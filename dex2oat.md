# dex2oat

PMS -> installd -> dex2oat

```
frameworks/base/services/core/java/com/android/server/pm/PackageDexOptimizer.java performDexOptLI
frameworks/base/services/core/java/com/android/server/pm/Installer.java
art/dex2oat/dex2oat.cc
```

## compile filter

[art/libartbase/base/compiler_filter.h](https://cs.android.com/android/platform/superproject/main/+/main:art/libartbase/base/compiler_filter.h;l=32;drc=bcb2b436bde55ee40050400783a9c083e77ce2fe)

```cpp
  enum Filter {
    kAssumeVerified,      // Skip verification but mark all classes as verified anyway.
    kVerify,              // Only verify classes.
    kSpaceProfile,        // Maximize space savings based on profile.
    kSpace,               // Maximize space savings.
    kSpeedProfile,        // Maximize runtime performance based on profile.
    kSpeed,               // Maximize runtime performance.
    kEverythingProfile,   // Compile everything capable of being compiled based on profile.
    kEverything,          // Compile everything capable of being compiled.
  };
```

## log

```
# dex2oat TAG
dex2oat[64]
# pm TAG
PackageDexOptimizer (Running dexopt ...) (PackageDexOptimizer#dexOptPath)
```

## pm compile

```
pm compile -f -m speed ${package}
```

如果不加 `-f` 可能不会重新编译，判断准则是和已有的 odex 比较

[art/runtime/native/dalvik_system_DexFile.cc GetDexOptNeeded](https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/native/dalvik_system_DexFile.cc;l=565;drc=bcb2b436bde55ee40050400783a9c083e77ce2fe)

此外，debug 包编译不产生 AOT 代码，compile filter 直接被换成 verify

[performDexOptLI -> getRealCompilerFilter](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/pm/PackageDexOptimizer.java;l=324;drc=bcb2b436bde55ee40050400783a9c083e77ce2fe)

[art/libartbase/base/compiler_filter.cc](https://cs.android.com/android/platform/superproject/main/+/main:art/libartbase/base/compiler_filter.cc;l=116;drc=bcb2b436bde55ee40050400783a9c083e77ce2fe)

...

[一次Art Hook失败问题的跟进 | 醉梦轩](https://www.drunkdream.com/2019/03/01/art-oat-build/)

## 解读 AOT 代码

### 静态方法调用

```
oatdump --oat-file=$(echo $(apkpath io.github.a13e300.demo)/oat/*/base.odex) --class-filter=NeverCall --method-filter=methodToDecompile
```

```java
    public static void methodToDecompileA() {
        Log.d("Test", "test");
    }

    public static void methodToDecompileC() {
        Log.w("test", "test");
    }

    public static void methodToDecompileD() {
        App.method();
    }

    public static void methodToDecompileB() {

    }
```

```
5: LNeverCall; (offset=0x00008bcc) (type_idx=11) (NotReady) (AllCompiled)
  3: void NeverCall.methodToDecompileA() (dex_method_idx=13)
    DEX CODE:
      0x0000: 1a00 e86e                	| const-string v0, "Test" // string@28392
      0x0002: 1a01 68f3                	| const-string v1, "test" // string@62312
      0x0004: 7120 3d13 1000           	| invoke-static {v0, v1}, int android.util.Log.d(java.lang.String, java.lang.String) // method@4925
      0x0007: 0e00                     	| return-void
    OatMethodOffsets (offset=0x00008be0)
      code_offset: 0x001a52a0 
    OatQuickMethodHeader (offset=0x001a529c)
      vmap_table: (offset=0x0014e794)
        CodeInfo CodeSize:131 FrameSize:48 CoreSpillMask:10028 FpSpillMask:0 NumberOfDexRegisters:2
          StackMap BitSize=44 Rows=4 Bits={Kind=0 PackedNativePc=7 DexPc=3 RegisterMaskIndex=1 StackMaskIndex=0 InlineInfoIndex=0 DexRegisterMaskIndex=0 DexRegisterMapIndex=0}
          RegisterMask BitSize=6 Rows=1 Bits={Value=3 Shift=3}
    QuickMethodFrameInfo
      frame_size_in_bytes: 48
      core_spill_mask: 0x00010028 (r3, r5, r16)
      fp_spill_mask: 0x00000000 
      vr_stack_locations:
      	locals: v0[sp + #12] v1[sp + #16]
      	method*: v2[sp + #0]
      	outs: v0[sp + #8] v1[sp + #12]
    CODE: (code_offset=0x001a52a0 size=131)...
      0x001a52a0:       4885842400E0FFFF    	       testq rax, [rsp + -8192]
        StackMap[0] (native_pc=0x1a52a8, dex_pc=0x0, register_mask=0x0, stack_mask=0b)
      0x001a52a8:                     55    	       push rbp
      0x001a52a9:                     53    	       push rbx
      0x001a52aa:               4883EC18    	       subq rsp, 24
      0x001a52ae:               48893C24    	       movq [rsp], rdi
      0x001a52b2: 65F704250000000007000000    	       test gs:[0], 7  ; state_and_flags
      0x001a52be:           0F8539000000    	       jnz/ne +57 (0x001a52fd)
      0x001a52c4:           8B35E6E2C400    	       mov esi, [RIP + 0xc4e2e6]
      0x001a52ca:     65833C251806000000    	       cmp gs:[1560], 0  ; pReadBarrierMarkReg06
      0x001a52d3:           0F852E000000    	       jnz/ne +46 (0x001a5307)
      0x001a52d9:                   85F6    	       test esi, esi
      0x001a52db:           0F8430000000    	       jz/eq +48 (0x001a5311)
      0x001a52e1:           8B15491EC200    	       mov edx, [RIP + 0xc21e49]
      0x001a52e7:                 4889F3    	       movq rbx, rsi
      0x001a52ea:                 4889D5    	       movq rbp, rdx
      0x001a52ed:           8B3D613AC200    	       mov edi, [RIP + 0xc23a61]
      0x001a52f3:                 FF5718    	       call [rdi + 24]
        StackMap[1] (native_pc=0x1a52f6, dex_pc=0x4, register_mask=0x28, stack_mask=0b)
      0x001a52f6:               4883C418    	       addq rsp, 24
      0x001a52fa:                     5B    	       pop rbx
      0x001a52fb:                     5D    	       pop rbp
      0x001a52fc:                     C3    	       ret 
      0x001a52fd:       65FF1425F8040000    	       call gs:[1272]  ; pTestSuspend
        StackMap[2] (native_pc=0x1a5305, dex_pc=0x0, register_mask=0x0, stack_mask=0b)
      0x001a5305:                   EBBD    	       jmp -67 (0x001a52c4)
      0x001a5307:       65FF142518060000    	       call gs:[1560]  ; pReadBarrierMarkReg06
      0x001a530f:                   EBC8    	       jmp -56 (0x001a52d9)
      0x001a5311:             B8E86E0000    	       mov eax, 28392
      0x001a5316:       65FF142558020000    	       call gs:[600]  ; pResolveString
        StackMap[3] (native_pc=0x1a531e, dex_pc=0x0, register_mask=0x0, stack_mask=0b)
      0x001a531e:                 4889C6    	       movq rsi, rax
      0x001a5321:                   EBBE    	       jmp -66 (0x001a52e1)
```

此处 offset 都是从 oat 文件开始的地方的偏移，即相对 oatdata 的偏移，可以检查一下上面反汇编的代码和实际位置的代码是否一致：

```
$ readelf -s base.odex -W

Symbol table '.dynsym' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000001000 0x1a5000 OBJECT  GLOBAL DEFAULT    1 oatdata
     2: 00000000001a6000     0 OBJECT  GLOBAL DEFAULT    2 oatexec

$ dd if=base.odex skip=$((0x1a52a0+0x1000)) count=16 bs=1c | xxd
00000000: 4885 8424 00e0 ffff 5553 4883 ec18 4889  H..$....USH...H.
```

这里应该就是调用 Log.d 的代码了，rdi 最终应该是一个 ArtMethod 地址，而 rdi + 24 就是 callee 的 aot 代码地址

```
      0x001a52ed:           8B3D613AC200    	       mov edi, [RIP + 0xc23a61]
      0x001a52f3:                 FF5718    	       call [rdi + 24]
```

`0x001a52f3 + 0xc23a61 + 0x1000 = 0xdc9d54` ，属于 `.data.bimg.rel.ro`

```
  [ 3] .data.bimg.rel.ro PROGBITS         0000000000dc6000  00dc6000
       0000000000003e68  0000000000000000   A       0     0     4096
     4: 0000000000dc6000 15976 OBJECT  GLOBAL DEFAULT    3 oatdatabimgrelro
     5: 0000000000dc9e64     4 OBJECT  GLOBAL DEFAULT    3 oatdatabimgrelrolastword
```

https://cs.android.com/android/platform/superproject/main/+/main:art/dex2oat/linker/oat_writer.h;l=434;drc=eddde90070822d22d47fa15cd4ee3f0fbae1324e

```cpp
  // Map for allocating .data.bimg.rel.ro entries. Indexed by the boot image offset of the
  // relocation. The value is the assigned offset within the .data.bimg.rel.ro section.
  SafeMap<uint32_t, size_t> data_bimg_rel_ro_entries_;
```

https://cs.android.com/android/platform/superproject/main/+/main:art/dex2oat/linker/oat_writer.cc;l=3077;drc=eddde90070822d22d47fa15cd4ee3f0fbae1324e

这是一个 uint32 数组

运行中进程 Log.d ArtMethod 地址 `1899166776 = 0x7132fc38` ，base.odex 基地址如下

```
7d316762d000-7d31677d3000 r--p 00000000 fe:21 49173                      /data/app/~~nxu8GrCHv1E0tvaK3CEl3Q==/io.github.a13e300.demo-T3m8kVyeLViPwaPqbcWelw==/oat/x86_64/base.odex
```

```sh
# 0x001a52f3 + 0xc23a61 + 0x1000 + 0x7d316762d000
emu64xa:/data/local/tmp # dd if=/proc/$(pidof io.github.a13e300.demo)/mem skip=137651155856724 bs=1c count=16|xxd
16+0 records in
16+0 records out
16 bytes (16 B) copied, 0.000119 s, 131 K/s
00000000: 38fc 3271 58fc 3271 78fc 3271 98fc 3271  8.2qX.2qx.2q..2q
```

可见该偏移的位置就是 ArtMethod 地址，不过直接从文件读取似乎并不正确：

```
# 0x001a52f3 + 0xc23a61 + 0x1000
$ dd if=base.odex skip=$((0xdc9d54)) count=16 bs=1c | xxd
00000000: 38cc 2001 58cc 2001 78cc 2001 98cc 2001  8. .X. .x. ... .
```

Log.d ArtMethod 地址属于下面这个内存区域：

```
70911000-71520000 rw-p 00000000 00:00 0                                  [anon:dalvik-/system/framework/boot-framework.art]
```

这个地址不像一般的内存区域，只有 32 位，而 oatdatabimgrelro 也是一个 32 位数组，可能是特意设计成这样的

文件中原本的地址是 `0x0120cc38` ，注意到 `0x7132fc38 - 0x70123000 = 0x0120cc38` ，因此我们就知道，这里存的是相对 boot.art 的地址，在加载的时候被修正，加上了 boot.art 的基地址。

上面是系统类方法调用，再来看个自己的：

```
      0x0000: 7100 ebc9 0000           	| invoke-static {}, void io.github.a13e300.demo.App.method() // method@51691
```

```
      0x001a53a2:         488B3D2FE0C300    	       movq rdi, [RIP + 0xc3e02f]
      0x001a53a9:                 FF5718    	       call [rdi + 24]
```

`0xc3e02f + 0x001a53a9 + 0x1000 = de43d8` ，位于 bss 中，这是一个无数据的节

```
  dca000 - df73b0
  [ 4] .bss              NOBITS          0000000000dca000 000000 02d3b0 00   A  0   0 4096
```

被调用方法的地址 `137652628902216`

```
emu64xa:/data/local/tmp # addrmap 137652628902216 $(pidof io.github.a13e300.demo)
7d31c00a5000-7d31c00e5000 rw-p 00000000 00:00 0                          [anon:dalvik-LinearAlloc]
```

内存中该位置地址 `0xc3e02f + 0x001a53a9 + 0x1000 + 0x7d316762d000 = 137651155964888` ，实际的值却并不是上面的 ArtMethod 地址，不过周围的地址看起来像是。

```
emu64xa:/data/local/tmp # dd if=/proc/$(pidof io.github.a13e300.demo)/mem skip=137651155964888 bs=1c count=16|xxd
16+0 records in
16+0 records out
16 bytes (16 B) copied, 0.000658 s, 24 K/s
00000000: 009b 3970 0000 0000 6851 0cc0 317d 0000  ..9p....hQ..1}..
```

这个地址 `0x70399b00` 同样指向 boot.art

```
emu64xa:/data/local/tmp # addrmap 0x70399b00 $(pidof io.github.a13e300.demo)
70123000-703b1000 rw-p 00000000 00:00 0                                  [anon:dalvik-/system/framework/boot.art]
```

可以发现，AOT 代码中调用方法需要引用 bss 节的地址，里面存储的应该是一个 ArtMethod 地址。下面分析一下 oat 中的 bss ：

bss 段实际上是全 0 的，因此在文件中没有实际分配。在加载 odex 的时候，内存区域是一个匿名映射 `[anon:bss]`

bss 节有两个符号 oatbssmethods 和 oatbssroots 。

`OatFileBase::ComputeFields` 获取了这两个符号，记录到 OatFile 结构中

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/oat_file.cc;l=374;drc=ac8d1d4155472f632356ae8e72ebe9bc68b68ff2

`OatFile::InitializeRelocations()` 初始化了 bss methods 数组，将所有 ArtMethod 指向 ResoutionMethod

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/oat_file.cc;l=2549;drc=ac8d1d4155472f632356ae8e72ebe9bc68b68ff2

```cpp
  // Initialize the .bss section.
  // TODO: Pre-initialize from boot/app image?
  ArtMethod* resolution_method = Runtime::Current()->GetResolutionMethod();
  for (ArtMethod*& entry : GetBssMethods()) {
    entry = resolution_method;
  }
```

在 `OatFileBase::Setup` 中，还要读取一系列 bss mappings ，这些数据位于 oatdata 中，与 Oat 中的 DexFile 有关，每个 OatDexFile 都有各自的 bss mappings ，最终都会被记录到 OatDexFile 结构体中。

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/oat_file.cc;l=1017;drc=e423e35dbd0915e19b8938bc6440955cda70caf9

```cpp
    const IndexBssMapping* method_bss_mapping;
    const IndexBssMapping* type_bss_mapping;
    const IndexBssMapping* public_type_bss_mapping;
    const IndexBssMapping* package_type_bss_mapping;
    const IndexBssMapping* string_bss_mapping;
    auto read_index_bss_mapping = [&](const char* tag, /*out*/const IndexBssMapping** mapping) {
      return ReadIndexBssMapping(this, &oat, i, dex_file_location, tag, mapping, error_msg);
    };
    if (!read_index_bss_mapping("method", &method_bss_mapping) ||
        !read_index_bss_mapping("type", &type_bss_mapping) ||
        !read_index_bss_mapping("public type", &public_type_bss_mapping) ||
        !read_index_bss_mapping("package type", &package_type_bss_mapping) ||
        !read_index_bss_mapping("string", &string_bss_mapping)) {
      return false;
    }
```

这个 mapping 实际上就是 `LengthPrefixedArray` ，即长度前缀的数组。其中的元素是 `IndexBssMappingEntry`

```cpp
// art/runtime/index_bss_mapping.h
using IndexBssMapping = LengthPrefixedArray<IndexBssMappingEntry>;

// IndexBssMappingEntry describes a mapping of one or more indexes to their offsets in the .bss.
// A sorted array of IndexBssMappingEntry is used to describe the mapping of method indexes,
// type indexes or string indexes to offsets of their assigned slots in the .bss.
//
// The highest index and a mask are stored in a single `uint32_t index_and_mask` and the split
// between the index and the mask is provided externally. The "mask" bits specify whether some
// of the previous indexes are mapped to immediately preceding slots. This is permissible only
// if the slots are consecutive and in the same order as indexes.
//
// The .bss offset of the slot associated with the highest index is stored in plain form as
// `bss_offset`. If the mask specifies any smaller indexes being mapped to immediately
// preceding slots, their offsets are calculated using an externally supplied size of the slot.
struct IndexBssMappingEntry {
  uint32_t index_and_mask;
  uint32_t bss_offset;
};
```

前面提到 bss methods 一开始指向的都是 resolution method ，首次调用会解析方法并调用 `MaybeUpdateBssMethodEntry` ，根据 OatDexFile 的 mapping 更新这些地址。

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/entrypoints/entrypoint_utils.cc;l=315;drc=ac8d1d4155472f632356ae8e72ebe9bc68b68ff2

resolution method 一般会调用到 `artQuickResolutionTrampoline` 

bss 未被解析的时候，AOT 代码调用的是 RuntimeMethod （即初始化填入的 resolution method），artQuickResolutionTrampoline 会根据 caller 的 dalvik 指令确定被调用方法的 index 和调用类型等信息，调用 `ClassLinker::ResolveMethod` 解析方法，并调用 `MaybeUpdateBssMethodEntry` 更新调用者所属 dexfile 的 bss

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/entrypoints/quick/quick_trampoline_entrypoints.cc;l=1283;drc=ac8d1d4155472f632356ae8e72ebe9bc68b68ff2

### 虚方法调用

```
      0x0000: 2200 0c00                	| new-instance v0, NeverCall$Y // type@TypeIndex[12]
      0x0002: 7010 0c00 0000           	| invoke-direct {v0}, void NeverCall$Y.<init>() // method@12
      0x0005: 6e10 0b00 0000           	| invoke-virtual {v0}, void NeverCall$X.y() // method@11

      0x001a5490:       65FF1425E8010000    	       call gs:[488]  ; pAllocObjectResolved
        StackMap[1] (native_pc=0x1a5498, dex_pc=0x0, register_mask=0x0, stack_mask=0b)
      0x001a5498:                 4889C6    	       movq rsi, rax
      0x001a549b:                 4889C3    	       movq rbx, rax
      0x001a549e:         488B3D633BC200    	       movq rdi, [RIP + 0xc23b63]
      0x001a54a5:                 FF5718    	       call [rdi + 24]
        StackMap[2] (native_pc=0x1a54a8, dex_pc=0x2, register_mask=0x8, stack_mask=0b)
      0x001a54a8:                 4889DE    	       movq rsi, rbx
      0x001a54ab:                   8B3E    	       mov edi, [rsi]
      0x001a54ad:         488BBFE0000000    	       movq rdi, [rdi + 224]
      0x001a54b4:                 FF5718    	       call [rdi + 24]
        StackMap[3] (native_pc=0x1a54b7, dex_pc=0x5, register_mask=0x8, stack_mask=0b)
```

可以发现，ArtMethod 不再从 bss 中取得，而是从对象取得。

此处 rsi 是刚创建的对象，`[rsi]` 指向对象头 4 字节，也就是 `art::mirror::Object` 的第一个成员 `HeapReference class`

HeapReference 实际上就是一个四字节的地址（ART 对象地址都是 4 字节的），因此现在 rdi 就是对象的类对象的地址。

之后 `movq rdi, [rdi + 224]` ，也就是访问类对象某个偏移处包含的地址。`art::mirror::Class` 的成员声明的末尾注释写道：

https://cs.android.com/android/platform/superproject/main/+/main:art/runtime/mirror/class.h;l=1587;drc=cc71b0afb121d2258094e3dc909e8dd6b3eb7a52

```cpp
  // The following data exist in real class objects.
  // Embedded Imtable, for class object that's not an interface, fixed size.
  // ImTableEntry embedded_imtable_[0];
  // Embedded Vtable, for class object that's not an interface, variable size.
  // VTableEntry embedded_vtable_[0];
  // Static fields, variable size.
  // uint32_t fields_[0];
```

可见，Class 对象除了有一些成员映射到 java 的 instance field ，还有一些数据没有被映射。这些数据包括 imtable, vtable 和静态 field 值。

因此我们猜测这个地址实际上应该是 embedded vtable 的地址

```cpp
  static constexpr MemberOffset EmbeddedVTableLengthOffset() {
    return MemberOffset(sizeof(Class));
  }

  static constexpr MemberOffset ImtPtrOffset(PointerSize pointer_size) {
    return MemberOffset(
        RoundUp(EmbeddedVTableLengthOffset().Uint32Value() + sizeof(uint32_t),
                static_cast<size_t>(pointer_size)));
  }

  static constexpr MemberOffset EmbeddedVTableOffset(PointerSize pointer_size) {
    return MemberOffset(
        ImtPtrOffset(pointer_size).Uint32Value() + static_cast<size_t>(pointer_size));
  }
```

从上面的代码可以看到，Class 对象结构体之后紧接着就是 嵌入虚表长 `EmbeddedVTableLength` (uint32) ，之后是 ImtPtr 指针，之后就是 vtable 的数据。

TODO……
