# Android 上的 ptrace 实践

[前篇](ptrace.md)

## dlopen

获取 dlopen 有两种方法：

1. 从 linker 获取 `__loader_dlopen` （或者 `__dl___loader_dlopen`），需要提供 caller_addr  
2. 从 libdl.so 获取 `dlopen`  

详细代码：

https://github.com/5ec1cff/ptrace-examples/tree/android

实现了注入到任意进程，调用 __loader_dlopen 打开指定路径的 lib ，或者调用 __loader_dlclose 关闭指定 handle 的 lib 。目前只支持 x86-64 和 arm64 。

### 对齐问题

一开始总是得到 SIGSEGV ，并且 fault addr 为 0 。出现问题的时候注入信号并 detach ，观察 crash dump ：

```
05-02 13:13:59.534 18907 18907 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
05-02 13:13:59.534 18907 18907 F DEBUG   : Build fingerprint: 'Android/sdk_phone64_x86_64/emulator64_x86_64:12/SE1A.220621.001/8752307:userdebug/test-keys'
05-02 13:13:59.534 18907 18907 F DEBUG   : Revision: '0'
05-02 13:13:59.534 18907 18907 F DEBUG   : ABI: 'x86_64'
05-02 13:13:59.534 18907 18907 F DEBUG   : Timestamp: 2023-05-02 13:13:59.436192500+0000
05-02 13:13:59.535 18907 18907 F DEBUG   : Process uptime: 0s
05-02 13:13:59.535 18907 18907 F DEBUG   : Cmdline: zygote64
05-02 13:13:59.535 18907 18907 F DEBUG   : pid: 17413, tid: 17413, name: main  >>> zygote64 <<<
05-02 13:13:59.535 18907 18907 F DEBUG   : uid: 0
05-02 13:13:59.535 18907 18907 F DEBUG   : signal 11 (SIGSEGV), code 128 (SI_KERNEL), fault addr 0x0
05-02 13:13:59.535 18907 18907 F DEBUG   :     rax 0000000000000000  rbx 00007ffe94362c77  rcx 0000000000000000  rdx 0000000000000000
05-02 13:13:59.536 18907 18907 F DEBUG   :     r8  0000000000000000  r9  0000000000000028  r10 0000000000000000  r11 0000000000000246
05-02 13:13:59.536 18907 18907 F DEBUG   :     r12 0000000000000002  r13 00007ffe94362dd0  r14 00007b1f79c6f8b8  r15 0000000000000000
05-02 13:13:59.536 18907 18907 F DEBUG   :     rdi 00007ffe94362c77  rsi 0000000000000002
05-02 13:13:59.536 18907 18907 F DEBUG   :     rbp 0000000000000000  rsp 00007ffe94361ad7  rip 00007b1f79b75d1d
05-02 13:13:59.536 18907 18907 F DEBUG   : backtrace:
05-02 13:13:59.536 18907 18907 F DEBUG   :       #00 pc 0000000000048d1d  /apex/com.android.runtime/bin/linker64 (__dl__Z9do_dlopenPKciPK17android_dlextinfoPKv+29) (BuildId: 82dabb4f8aa58c9aa33c9c6fcabc92a7)
05-02 13:13:59.536 18907 18907 F DEBUG   :       #01 pc 00000000000444f9  /apex/com.android.runtime/bin/linker64 (__loader_dlopen+57) (BuildId: 82dabb4f8aa58c9aa33c9c6fcabc92a7)
05-02 13:13:59.537 18907 18907 F DEBUG   :       #02 pc 00000000000b3aab  /apex/com.android.runtime/lib64/bionic/libc.so (__ppoll+11) (BuildId: 5db8d317d3741b337ef046540bbdd0f7)
05-02 13:13:59.537 18907 18907 F DEBUG   :       #03 pc 6f6c2f617461642f  <unknown>
```

gdb 反编译看一看：

```
disas __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
Dump of assembler code for function __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv:
   ...
   0x0000000000048d1d <+29>:    movaps %xmm0,0x40(%rsp)
```

movaps 是什么指令？问一问万能的 ChatGPT：

```
movaps是x86-64汇编语言中的一条指令，用于将数据块从一个位置移动到另一个位置。该指令可以在操作数和目的地操作数都是128位XMM寄存器时使用，它将操作数寄存器中128位的值复制到目标操作数寄存器中，覆盖其128位的值。"movaps"表示“移动对齐的128位数据”，如果任意一个操作数不是128位对齐的话，该指令将引发异常情况。
```

也就是说，这条指令要求 rsp 是 16 字节对齐的，而我们往栈上写入了字符串后， rsp 并不是对齐的。

实践表明，ptrace 停止后当前的 rsp 也未必就是 128 位对齐的，因此修改代码，在我们调用前将 rsp 调整为比它小的最大的 128 位对齐的地址。

> 但是为什么 linker64 的函数需要堆栈对齐，而我们在 libc 停下的地方却没有对齐呢？

```cpp
bool make_call(int pid, void* addr, struct user_regs_struct &regs, void** result) {
    // ...
    regs.rsp = regs.rsp & (~0xf); // here
    regs.rax = (unsigned long long) addr;
    if (!ptrace_set_regs(pid, regs)) {
```

libdl:

```
(gdb) disas dlopen
Dump of assembler code for function dlopen:
   0x0000000000001b50 <+0>:     mov    (%rsp),%rdx
   0x0000000000001b54 <+4>:     jmp    0x1ce0 <__loader_dlopen@plt>
End of assembler dump.
(gdb) disas 0x1ce0
Dump of assembler code for function __loader_dlopen@plt:
   0x0000000000001ce0 <+0>:     jmp    *0x11ca(%rip)        # 0x2eb0 <__loader_dlopen@got.plt>
   0x0000000000001ce6 <+6>:     push   $0x1
   0x0000000000001ceb <+11>:    jmp    0x1cc0
End of assembler dump.
```

### 权限问题

解决了上面的问题，dlopen 总算不会出现 SIGSEGV 了，但是注入 zygote 返回的 handle 是 0 。

```
05-02 13:45:45.136 26460 26460 D linker  : ... dlopen failed: library "/data/local/tmp/libinject-lib.so" not found
05-02 13:45:45.132 26460 26460 W main    : type=1400 audit(0.0:59): avc: denied { search } for name="tmp" dev="dm-5" ino=65538 scontext=u:r:zygote:s0 tcontext=u:object_r:shell_data_file:s0 tclass=dir permissive=0
```

看起来我们放在 `/data/local/tmp` 会导致 zygote 无法访问。

> linker 日志： `setprop debug.ld.all dlopen`  并重启 zygote 。

那么先临时关闭 SELinux ：

```
setenforce 0
injector open `pidof zygote64` /data/local/tmp/libinject-lib.so
dlopen /data/local/tmp/libinject-lib.so on 27974

found linker in maps, base=0x74a3bf78a000,path=/apex/com.android.runtime/bin/linker64
dlopen_addr: 0x74a3bf7ce4c0
rsp=0x7ffdf5619708
string pushed to 0x7ffdf5619680 (size=33)
rip=0x74a3a9d08aaa instruction: 972fffff0013d48
handle: 0x91b6696f12f223bf
```

这样我们的代码可以正常执行：

```
05-02 13:59:21.475 27974 27974 D linker  : ... dlopen calling constructors: realpath="/data/local/tmp/libinject-lib.so", soname="libinject-lib.so", handle=0x91b6696f12f223bf
05-02 13:59:21.475 27974 27974 D pt-injector: injected
05-02 13:59:21.475 27974 27974 D linker  : ... dlopen successful: realpath="/data/local/tmp/libinject-lib.so", soname="libinject-lib.so", handle=0x91b6696f12f223bf
```

那么有什么方法可以在开启 selinux 的情况下执行呢？可以考虑修改 selinux 规则，或者换个位置存放，比如放在 `/dev` ，context 改为 `u:object_r:system_file:s0`

```
05-03 01:29:45.954  3237  3237 D linker  : ... dlopen calling constructors: realpath="/dev/libinject-lib.so", soname="libinject-lib.so", handle=0x17a87dbf82e7009f
05-03 01:29:45.954  3237  3237 D pt-injector: injected
05-03 01:29:45.954  3237  3237 D linker  : ... dlopen successful: realpath="/dev/libinject-lib.so", soname="libinject-lib.so", handle=0x17a87dbf82e7009f
^C
130|emulator64_x86_64:/data/local/tmp # getenforce
Enforcing
```

这样注入到 init 也是没问题的。

### linker 日志

LinkerLogger 会在 dlopen 和 dlsym 的时候刷新，设置了系统属性 `debug.ld.*` 之后，app 进程都可以得到动态更新，注入 dlopen 会产生 linker 日志，但是注入到 zygote 进行 dlopen 却不会发生更新，进而无法产生日志，不得不重启。

实际上只有进程是 dumpable 的时候才会发生更新：

https://android.googlesource.com/platform/bionic/+/ebd654640abdc3f048f0a676741b79dd3d23b766/linker/linker_logger.cpp#89

模拟器是 debug 构建的系统，因此 app 进程都是 dumpable 的，不过 zygote 自身仍然不是 dumpable 的。

我们可以用 ptrace 注入系统调用查看或修改 dumpable ，这样 debug.ld 属性就可以被 zygote 读取了：

```
./injector get-dumpable `pidof zygote64`
get dumpable for 365
rip=0x74d24725eaaa instruction: 972fffff0013d48
dumpable:0x0
./injector set-dumpable `pidof zygote64` 1
set dumpable to 1 for 365
rip=0x74d24725eaaa instruction: 972fffff0013d48
result:0x0
./injector get-dumpable `pidof zygote64`
get dumpable for 365
rip=0x74d24725eaaa instruction: 972fffff0013d48
dumpable:0x1
```

另外，init 的日志比较特殊，在我的 AVD 上，无法通过 `logcat --pid 1` 获取 init 的日志，而实际上 init 仍然会往 logd 写入日志，只是 pid 为 0 ，我们无法通过 `logcat --pid 0` 获取 init 的日志。更加奇怪的是，debug.ld 也无法影响 init 的 linker ，也就是无法产生 linker 日志，尽管 init 本来就是 dumpable 的，可能和 init 自身的 linker 有关。

考虑到 init 是一个非常特殊的进程，它的死亡会直接导致 kernel panic ，因此 ptrace 注入它还是要小心谨慎。

## 其他指令集

### arm64-v8a

#### 函数调用

函数调用的汇编代码：

```
blr x9
brk #0x1
```

编译成机器码就是 `0x20, 0x1, 0x3f, 0xd6, 0x20, 0x0, 0x20, 0xd4` 或者 `0xd4200020d63f0120` 。

其中 `brk #0x1` 是从 `__builtin_trap()` 得来的，cpu 到这条指令断下来之后似乎不会增加 pc 的值。

https://stackoverflow.com/questions/11345371/how-do-i-set-a-software-breakpoint-on-an-arm-processor

https://stackoverflow.com/questions/44949124/absolute-jump-with-a-pc-relative-data-source-aarch64

arm64 的调用约定：x0~x7 作为函数参数，x0 也作为返回值。由于 x9 是被调用方保存的寄存器，因此选择它来传递函数的绝对地址。

#### 系统调用

```
svc 0
```

机器码：`0x1, 0x0, 0x0, 0xd4` 或者 `0xd4000001` 。

参数：x0~x5 ，返回值是 x0 ，系统调用号 NR 是 x8 。

https://syscall.sh/

下面是直接注入当前 shell 的效果：

```
u0_a212@192.168.1.xxx ~
$ ./injector open $$ $PWD/libinject-lib.so; logcat --pid $$
dlopen /data/data/com.termux/files/home/libinject-lib.so on 26496

found linker in maps, base=0x701ef49000,path=/apex/com.android.runtime/bin/linker64
dlopen_addr: 0x701ef7a190
sp=0x7ffa8d84e0
string pushed to 0x7ffa8d84ae (size=50)
pc=0x701d9c1c94 instruction: b140041fd4000001
handle: 0xfcff7941be05802d
--------- beginning of main
05-03 20:23:54.309 26496 26496 D pt-injector: injected
^C
u0_a212@192.168.1.xxx ~
$ ./injector close $$  0xfcff7941be05802d; logcat --pid $$
dlclose 0xfcff7941be05802d on 26496

found linker in maps, base=0x701ef49000,path=/apex/com.android.runtime/bin/linker64
dlclose_addr: 0x701ef7a2d8
pc=0x701d9c1c94 instruction: b140041fd4000001
--------- beginning of main
05-03 20:23:54.309 26496 26496 D pt-injector: injected
05-03 20:24:08.152 26496 26496 D pt-injector: closed
```

### armeabi-v7a

arm 的适配应该是最艰难的。

https://developer.android.com/ndk/guides/abis?hl=zh-cn#v7a

#### 断点

`__builtin_trap`:

```asm
defe            udf     #254    ; 0xfe
```

直接执行并不产生 trap ，而是得到一个 SIGILL ：

```
u0_a212@192.168.1.144 ~/codes
$ strace ../bkpt_exe
execve("../bkpt_exe", ["../bkpt_exe"], 0xb400007268e0e000 /* 47 vars */ <unfinished ...>
[ Process PID=19460 runs in 32 bit mode. ]
<... execve resumed>)                   = 0
--- SIGILL {si_signo=SIGILL, si_code=ILL_ILLOPC, si_addr=0x11128} ---
+++ killed by SIGILL +++
Illegal instruction
```

https://developer.arm.com/documentation/dui0801/h/A32-and-T32-Instructions/UDF  
https://stackoverflow.com/questions/11345371/how-do-i-set-a-software-breakpoint-on-an-arm-processor  

`bkpt #0` ：

```
e1200070        bkpt    0x0000
```

得到 Trap ：

```
u0_a212@192.168.1.144 ~/codes
$ strace ../bkpt_exe
execve("../bkpt_exe", ["../bkpt_exe"], 0xb40000775740e000 /* 47 vars */ <unfinished ...>
[ Process PID=19616 runs in 32 bit mode. ]
<... execve resumed>)                   = 0
--- SIGTRAP {si_signo=SIGTRAP, si_code=TRAP_BRKPT, si_addr=0x110f8} ---
+++ killed by SIGTRAP +++
Trap
```

此外观察 si_addr 发现两者都不会越过断点。

https://developer.arm.com/documentation/dui0473/m/arm-and-thumb-instructions/bkpt

#### thumb 模式

arm 上有 arm 和 thumb 两种指令集，根据跳转地址的最后一位来区分并切换 cpu 工作状态。Android 上 armeabi-v7a 默认都是编译为 thumb 指令集，那么 pc 寄存器里面的实际值是什么呢？

使用 debuggerd 观察 32 位 arm 进程的堆栈：

```
"pal.androidterm" sysTid=24586
    #00 pc 0009c2fc  /apex/com.android.runtime/lib/bionic/libc.so!libc.so (offset 0x9b000) (__epoll_pwait+20) (BuildId: edc23b5a08cb25fcac190e6392a4d537)
    #01 pc 0006dbc9  /apex/com.android.runtime/lib/bionic/libc.so!libc.so (offset 0x6c000) (epoll_wait+16) (BuildId: edc23b5a08cb25fcac190e6392a4d537)
```

libc.so 是一个 thumb 指令集的 so ，看起来 pc 寄存器保存的还是实际的地址，也就是只有偶数，而堆栈回溯可以看到奇数地址实际上是因为这个地址不是来自 pc ，而是来自栈上。

……

实际上 Android 大部分进程都运行在 thumb 模式，对于 arm 模式和 thumb 模式，我们需要不同的 shellcode ，而上面生成的是 arm 模式的机器码。

为了确保一致性，我直接用 NDK 编译。

```cmake
enable_language(ASM)

if(ANDROID_ABI STREQUAL "armeabi-v7a")
    message("current abi is armeabi-v7a")
    add_executable(libshellcode.so shellcode/armeabi-v7a.s)
    target_link_libraries(libshellcode.so -nostdlib -static)
endif()
```

https://stackoverflow.com/questions/22396214/understanding-this-part-arm-assembly-code

下面的代码加上了 `.thumb_func` 确保生成 thumb 代码。

```asm
        .global _start

        .text
        .thumb_func
_start:
call_code:
    blx r4
break_code:
    bkpt #0
svc_code:
    svc 0
```

```
$ arm-linux-gnueabi-objdump -d /mnt/f/works/ptinjector/app/build/intermediates/merged_native_libs/debug/out/lib/armeabi-
v7a/libshellcode.so

/mnt/f/works/ptinjector/app/build/intermediates/merged_native_libs/debug/out/lib/armeabi-v7a/libshellcode.so:     file format elf32-littlearm


Disassembly of section .text:

000110f8 <_start>:
   110f8:       47a0            blx     r4

000110fa <break_code>:
   110fa:       be00            bkpt    0x0000

000110fc <svc_code>:
   110fc:       df00            svc     0

$ dd if=/mnt/f/works/ptinjector/app/build/intermediates/merged_native_libs/debug/out/lib/armeabi-v7a/libshellcode.so skip=$((0xf8)) count=16 bs=1c |xxd
00000000: a047 00be 00df 411d 0000 0061 6561 6269  .G....A....aeabi
```

最终 shellcode （注释的代码为 arm 模式的，暂不考虑支持）：

```cpp
// const unsigned long CODE_FOR_CALL[] = { 0x34ff2fe1lu, 0x700020e1lu }; // blx r4; bkpt #0
const unsigned long CODE_FOR_CALL[] = { 0xbe0047a0lu }; // blx r4; bkpt #0
// const unsigned long CODE_FOR_SYSCALL[] = { 0x000000eflu }; // svc 0
const unsigned long CODE_FOR_SYSCALL[] = { 0xdf00lu }; // svc 0
// const auto CALL_EXPECTED_OFFSET = 4;
```

至于进程运行在什么模式，可以通过 cpsr 寄存器的 T 标志（第五位）判断：

https://developer.arm.com/documentation/den0013/d/ARM-Processor-Modes-and-Registers/Registers/Program-Status-Registers

```cpp
#define ARM_IS_THUMB(regs) (((regs).ARM_cpsr & (1 << 5)) != 0)
```

实践表明，通过 ptrace 修改 cpsr 的 T 标志不会使进程切换模式，因此如果要同时支持 thumb 和 arm ，必须写两套代码。
