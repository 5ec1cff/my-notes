# LSPlant

https://github.com/LSPosed/LSPlant

## 基本原理

修改 ArtMethod 的 entry_point_from_quick_compiled_code_ ，这里原先是 dex2oat 优化后的本机代码，现在指向我们控制的代码。同时通过各种手段使调用方法时走本机代码 。

为此，hook 一个 method 需要另外两个 method ，一个作为 hook 的 stub ；另一个 method 用于备份被 hook 的方法。LSPlant 使用 DexBuilder 生成包含这两个方法的辅助类。

## ArtMethod 调用约定

我们可以从反射执行看 oat 后 ArtMethod 的本机代码是如何调用的。

[ArtMethod::Invoke](https://cs.android.com/android/platform/superproject/+/master:art/runtime/art_method.cc;l=365;drc=473c5a01699e82723c936bfd47ceac9abee70e09)

根据方法是否为 static ，执行一段汇编编写的代码 art_quick_invoke_stub 或 art_quick_invoke_static_stub ，其代码位于：

`art/runtime/arch/{arch}/quick_entrypoints_{arch}.S`

[art/runtime/arch/x86_64/quick_entrypoints_x86_64.S](https://cs.android.com/android/platform/superproject/+/master:art/runtime/arch/x86_64/quick_entrypoints_x86_64.S;l=415;drc=630c507467f637a5c9221db1558cd679f494fb6a)

## Trampoline

根据调用约定，进入 art method 第一个参数应该是 ArtMethod 指针，而 hook 方法替换了 entry_point_from_quick_compiled_code_ ，但 caller 调用时仍然传入了原 ArtMethod 的指针，我们需要把它替换成自己的 ArtMethod 指针。因此 entry_point_from_quick_compiled_code_ 指向一段跳板代码完成这个工作，即把第一个参数换成自己的 ArtMethod ，并从

```cpp
auto [trampoline, entry_point_offset, art_method_offset] = GetTrampoline();
```

trampoline 表示跳板代码内容

entry_point_offset 表示 ArtMethod 的 entry_point_from_quick_compiled_code_ 偏移在指令中的位置 （按位），当 entry_point_from_quick_compiled_code_ 的偏移计算好后，会根据 entry_point_offset 修正 trampoline 指令。

art_method_offset 表示 ArtMethod 地址在指令中的位置（按字节）。LSPlant 为每个方法生成一个跳板代码，生成时在该偏移写入 hook 的 ArtMethod 指针。

下面具体分析一些 arch 的 code ：

### x86-64

```cpp
    if constexpr (kArch == Arch::kX86_64) {
        return std::make_tuple("\x48\xbf\x78\x56\x34\x12\x78\x56\x34\x12\xff\x77\x00\xc3"_uarr,
                               // NOLINTNEXTLINE
                               uint8_t{96u}, uintptr_t{2u});
    }
```

```
0x0000000000000000:  48 BF 78 56 34 12 78 56 34 12    movabs rdi, 0x1234567812345678 # ArtMethod 地址置于 rdi 中
0x000000000000000a:  FF 77 xx                         push   qword ptr [rdi + xx] # 取 hook ArtMethod 的 entry_point_from_quick_compiled_code_ 放到栈上
0x000000000000000d:  C3                               ret    # 跳转到 hook 的 entry_point_from_quick_compiled_code_
```

### arm64

```cpp
    if constexpr (kArch == Arch::kArm64) {
        return std::make_tuple(
            "\x60\x00\x00\x58\x10\x00\x40\xf8\x00\x02\x1f\xd6\x78\x56\x34\x12\x78\x56\x34\x12"_uarr,
            // NOLINTNEXTLINE
            uint8_t{44u}, uintptr_t{12u});
    }
```

```
0x0000000000000000:  60 00 00 58    ldr  x0, #0xc # 读相对第一条指令 0xc 偏移的位置的内存，即 hook 的 ArtMethod 地址到第一个参数 (x0)
0x0000000000000004:  10 x0 4x F8    ldur x16, [x0] # 取 entry_point_from_quick_compiled_code_
0x0000000000000008:  00 02 1F D6    br   x16 # 跳转到 hook
0x000000000000000c:  78 56 34 12    and  w24, w19, #0xfffff003 # ArtMethod 地址
0x0000000000000010:  78 56 34 12    and  w24, w19, #0xfffff003
```

### arm

```cpp
    if constexpr (kArch == Arch::kArm) {
        return std::make_tuple("\x00\x00\x9f\xe5\x00\xf0\x90\xe5\x78\x56\x34\x12"_uarr,
                               // NOLINTNEXTLINE
                               uint8_t{32u}, uintptr_t{8u});
    }
```

指令是 arm 模式而非 thumb （因此地址也是偶数）

```
       0: e59f0000      ldr     r0, [pc] # 加载 pc+8 到第一个参数，即 hook ArtMethod 地址
       4: e590f0xx      ldr     pc, [r0, #xx] # hook entry_point_from_quick_compiled_code_ 送 pc 直接跳转
       8: 12345678      # hook ArtMethod 地址
```

[PC+8](https://stackoverflow.com/questions/24091566/why-does-the-arm-pc-register-point-to-the-instruction-after-the-next-one-to-be-e)

### x86

```cpp
    if constexpr (kArch == Arch::kX86) {
        return std::make_tuple("\xb8\x78\x56\x34\x12\xff\x70\x00\xc3"_uarr,
                               // NOLINTNEXTLINE
                               uint8_t{56u}, uintptr_t{1u});
    }
```

```
       0: b8 78 56 34 12                movl    $0x12345678, %eax       # imm = 0x12345678
       5: ff 70 xx                      pushl   (%eax + xx)
       8: c3                            retl
```

## 疑难杂症

### 设置备份为 private

https://github.com/LSPosed/LSPlant/blob/ab5830a0207a76cc2abc82e6d4f15f2053f51523/lsplant/src/main/jni/lsplant.cc#L530

```cpp
if (!backup->IsStatic()) backup->SetPrivate();
```

和方法的解析有关，一般非 static 非 private 则认为是虚函数，会在虚表上解析，导致无法正确解析到备份方法的地址

可以参考 [ART深度探索开篇：从Method Hook谈起 | Weishu's Notes](https://weishu.me/2017/03/20/dive-into-art-hello-world/)

> 在调用的时候，如果不是static的方法，会去查找这个方法的真正实现；我们直接把原方法做了备份之后，去调用备份的那个方法，如果此方法是public的，则会查找到原来的那个函数，于是就无限循环了；我们只需要阻止这个过程，查看 FindVirtualMethodForVirtualOrInterface 这个方法的实现就知道，只要方法是 invoke-direct 进行调用的，就会直接返回原方法，这些方法包括：构造函数，private的方法( 见 https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html) 因此，我们手动把这个备份的方法属性修改为private即可解决这个问题。

## 参考

[YAHFA--ART环境下的Hook框架 - 记事本](http://rk700.github.io/2017/03/30/YAHFA-introduction/)

[在Android N上对Java方法做hook遇到的坑 - 记事本](http://rk700.github.io/2017/06/30/hook-on-android-n/)
