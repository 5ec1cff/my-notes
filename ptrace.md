# ptrace

[ptrace(2) - Linux manual page](https://man7.org/linux/man-pages/man2/ptrace.2.html)

[wait(2) - Linux manual page](https://man7.org/linux/man-pages/man2/waitpid.2.html)

## tracee execve 时的行为

当被跟踪者(tracee)进程成功调用 execve 后，会立即产生一个 SIGTRAP ，并进入 signal-delivery-stop 状态，允许我们在进程的所有新代码执行之前进行处理。

这个状态不好辨识，在 man ptrace 中已经不建议使用。我们可以用 `PTRACE_O_TRACEEXEC` 选项，这样当成功 execve 的时候，原先的 SIGTRAP 不会产生，取而代之的是另一个 stop ，status 满足 `status >> 8 == (SIGTRAP | (PTRACE_EVENT_EXEC << 8))` ，并且我们可以通过 `PTRACE_GETEVENTMSG` 获得 execve 之前的 pid （在多线程中有用，因为在非主线程进行 execve ，pid 会发生改变）。

除了多线程中 pid 的问题之外，这两个 stop 看起来没什么区别，停止的时候，新的映像都已经装载，且没有执行。

```cpp
// trace-exec.cpp
#include <iostream>

#include <sys/ptrace.h>
#include <sys/wait.h>
#include <unistd.h>

using namespace std;

int main() {
    auto pid = fork();
    int status;
    if (pid < 0) {
        perror("fork");
        return 1;
    } else if (pid == 0) {
        ptrace(PTRACE_TRACEME);
        cout << "child forked" << endl;
        raise(SIGTRAP);
        cout << "trapped" << endl;
        execlp("ls", "ls", nullptr);
        perror("execve");
        return 1;
    }
    cout << "forked pid " << pid << endl;
    bool first_stop = true;
    for (;;) {
        if (waitpid(pid, &status, __WALL) == -1) {
            if (errno != EINTR) {
                perror("waitpid");
                return 1;
            }
        }
        if (WIFSTOPPED(status)) {
            if (status >> 8 == (SIGTRAP | (PTRACE_EVENT_EXEC << 8))) {
                cout << "stopped at execve" << endl;
            } else {
                cout << "stopped by " << WSTOPSIG(status) << endl;
                if (first_stop) {
                    ptrace(PTRACE_SETOPTIONS, pid, nullptr, PTRACE_O_TRACEEXEC);
                    first_stop = false;
                }
            }
            cin.get();
            cout << "continue" << endl;
            ptrace(PTRACE_CONT, pid, nullptr, nullptr);
        } else if (WIFEXITED(status)) {
            cout << "exited with " << WEXITSTATUS(status) << endl;
            break;
        } else {
            cout << "unknown status " << status << endl;
        }
    }
    return 0;
}
```

运行上面的程序，应该观察得到两次 ptrace stop ：

```
forked pid 1980
child forked
stopped by 5


continue
trapped
stopped at execve


continue
CMakeCache.txt  CMakeFiles  Makefile  Testing  cmake_install.cmake  exec  fork-exec  ptrace_learn  ptrace_learn.cbp  trace-exec  trace-fork-exec
exited with 0
```

在第二次停下的时候查看子进程 1980 的状态，已经 execve 到 ls ：

```
$ cat /proc/1980/status
Name:   ls
Umask:  0022
State:  t (tracing stop)
Tgid:   1980
Ngid:   0
Pid:    1980
PPid:   1979
TracerPid:      1979

$ ls /proc/1980/exe -l
lrwxrwxrwx 1 five_ec1cff five_ec1cff 0 Apr 24 21:11 /proc/2011/exe -> /mnt/f/works/ptrace-learn/cmake-build-debug-wsl/trace-exec
$ ls /proc/1980/exe -l
lrwxrwxrwx 1 five_ec1cff five_ec1cff 0 Apr 24 21:11 /proc/2011/exe -> /usr/bin/ls
```

## 跟踪 tracee fork

有这样一个程序，它会 fork ，并在子进程 execve ，主进程等待子进程结束。我们需要跟踪主进程，并跟踪子进程：

```cpp
// fork-exec
#include <iostream>

#include <sys/wait.h>
#include <unistd.h>

using namespace std;

int main() {
    cout << "my pid=" << getpid() << ",press enter to fork and exec" << endl;
    cin.get();
    auto pid = fork();
    int status;
    if (pid < 0) {
        perror("fork");
        return 1;
    } else if (pid == 0) {
        cout << "child forked" << endl;
        execlp("ls", "ls", nullptr);
        perror("execve");
        return 1;
    }
    cout << "forked pid " << pid << endl;
    if (waitpid(pid, &status, __WALL) == -1) {
        if (errno != EINTR) {
            perror("waitpid");
            return 1;
        }
    }
    if (WIFEXITED(status)) {
        cout << "exited with " << WEXITSTATUS(status) << endl;
    } else {
        cout << "unknown status " << status << endl;
    }
    return 0;
}
```

下面的程序使用 PTRACE_ATTACH 附加上面的进程，对主进程，设置 `PTRACE_O_TRACEFORK` 选项跟踪 fork 。

`PTRACE_O_TRACEFORK` 会在 tracee 原本的进程发生成功 fork 返回的时候停止，status 满足 `status >> 8 == (SIGTRAP | (PTRACE_EVENT_FORK << 8)` ，可以通过 PTRACE_GETEVENTMSG 得到 fork 的新进程的 pid 。同时，新的子进程也会处于 SIGSTOP 停止状态。

使用 `PTRACE_O_*` 自动附加的进程会继承原先的选项，在这里我们在第一次附加后直接覆盖掉了。

```cpp
// trace-fork-exec
#include <iostream>

#include <sys/ptrace.h>
#include <sys/wait.h>
#include <unistd.h>
#include <cstring>
#include <csignal>
#include <set>

using namespace std;

int main(int argc, char **argv) {
    if (argc <= 1) {
        cerr << "usage: " << argv[0] << " <pid>" << endl;
        return 1;
    }
    auto ipid = (int) strtol(argv[1], nullptr, 0);
    cout << "tracing pid " << ipid << endl;
    int status;
    ptrace(PTRACE_ATTACH, ipid, nullptr, nullptr);
    bool first_stop = true;
    set<int> pids{};
    pids.insert(ipid);
    for (;;) {
        auto pid = waitpid(-1, &status, __WALL);
        if (pid == -1) {
            if (errno != EINTR) {
                perror("waitpid");
                return 1;
            }
            continue;
        }
        if (WIFSTOPPED(status)) {
            int orig_sig = 0;
            if (status >> 8 == (SIGTRAP | (PTRACE_EVENT_EXEC << 8))) {
                cout << pid << " stopped at execve" << endl;
            } else if (status >> 8 == (SIGTRAP | (PTRACE_EVENT_FORK << 8))) {
                long msg;
                ptrace(PTRACE_GETEVENTMSG, pid, nullptr, &msg);
                cout << pid << " stopped at fork, child pid=" << msg << endl;
            } else {
                orig_sig = WSTOPSIG(status);
                cout << pid << " stopped by SIG" << sigabbrev_np(orig_sig) << "(" << orig_sig << ")" << endl;
                if (first_stop) {
                    ptrace(PTRACE_SETOPTIONS, pid, nullptr, PTRACE_O_TRACEFORK);
                    first_stop = false;
                }
            }
            if (pids.find(pid) == pids.end()) {
                pids.insert(pid);
                cout << "new process " << pid << " added" << endl;
                ptrace(PTRACE_SETOPTIONS, pid, nullptr, PTRACE_O_TRACEEXEC);
            }
            cin.get();
            cout << pid << " continue" << endl;
            ptrace(PTRACE_CONT, pid, nullptr, (void*) orig_sig);
        } else if (WIFEXITED(status)) {
            cout << pid << " exited with " << WEXITSTATUS(status) << endl;
            pids.erase(pid);
        } else {
            cout << pid << " unknown status " << status << endl;
        }
        if (pids.empty()) {
            cout << "all processes exited" << endl;
            break;
        }
    }
    return 0;
}
```

fork-exec:

```
my pid=2332,press enter to fork and exec


forked pid 2334
child forked
CMakeCache.txt  CMakeFiles  Makefile  Testing  cmake_install.cmake  exec  fork-exec  ptrace_learn  ptrace_learn.cbp  trace-exec  trace-fork-exec
exited with 0
```

trace-fork-exec:

```
$ ./trace-fork-exec 2332
tracing pid 2332
2332 stopped by SIGSTOP(19)

2332 continue
2332 stopped by SIGSTOP(19)

2332 continue
2332 stopped at fork, child pid=2334

2332 continue
2334 stopped by SIGSTOP(19)
new process 2334 added

2334 continue
2334 stopped by SIGSTOP(19)

2334 continue
2332 stopped by SIGCHLD(17)

2332 continue
2334 stopped at execve

2334 continue
2334 exited with 0
2332 stopped by SIGCHLD(17)

2332 continue
2332 exited with 0
all processes exited
```

反复执行，观察结果，发现父进程和子进程的 fork 停止究竟谁更先被收到，似乎是不确定的。

## 远程系统调用和执行任意代码

我们将使用 mmap 映射一块 rwx 内存，然后写入代码并执行。

### Step 1. 生成 shellcode

为此首先需要准备要执行的代码，看上去就像 CTF 中的 shellcode ，用 pwntools 等工具很容易可以生成，不过考虑到我们要在未来的项目中使用，还是要学会自己生成 shellcode 。我们先从汇编开始。

下面是~~不知道从哪个大学偷来的~~ x86-64 上的汇编 hello world 实例。

```
# hello.s
# https://cs.lmu.edu/~ray/notes/gasexamples/
# ----------------------------------------------------------------------------------------
# Writes "Hello, World" to the console using only system calls. Runs on 64-bit Linux only.
# To assemble and run:
#
#     gcc -c hello.s && ld hello.o && ./a.out
#
# or
#
#     gcc -nostdlib hello.s && ./a.out
# ----------------------------------------------------------------------------------------

        .global _start

        .text
_start:
        # write(1, message, 13)
        mov     $1, %rax                # system call 1 is write
        mov     $1, %rdi                # file handle 1 is stdout
        #mov     $message, %rsi          # address of string to output
        lea     message(%rip), %rsi
        mov     $13, %rdx               # number of bytes
        syscall                         # invoke operating system to do the write

        # exit(0)
        mov     $60, %rax               # system call 60 is exit
        xor     %rdi, %rdi              # we want return code 0
        syscall                         # invoke operating system to exit
message:
        .ascii  "Hello, world\n"
```

我们需要将代码编译为位置无关的，需要加上 `-fPIC` 或者 `-fPIE`

> pic 和 pie 的区别：[Gcc中编译和链接选项 -fpic -fPIC -fpie -fPIE -pie的含义](https://e-mailky.github.io/2018-03-06-gcc-pic) ~~其实我也没看懂~~

```
gcc -static -nostdlib -fPIE hello.s -o hello
```

需要注意代码中不能直接用 `mov $label` ，这样无法链接成 PIC 的程序，因此原来的代码中的 mov message 改成了 lea ，才能通过编译。

[c - relocation R_X86_64_32 against `.data' can not be used when making a shared object; - Stack Overflow](https://stackoverflow.com/questions/49434489/relocation-r-x86-64-32-against-data-can-not-be-used-when-making-a-shared-obje)

因为我们的代码和数据都放在 .text 段，且入口就在开头，所以这样我们只要得到 .text 段的内容，把它写入到 mmap 的区域即可。

下面的命令可以复制 text section 到单独的文件中。

```
objcopy -O binary -j .text hello hello_
```

可以用下面的命令 disassemble 刚才产生的文件：

```
objdump -b binary -m i386:x86-64 -D hello_
```

[简述获取shellcode的几种方式 - FreeBuf](https://www.freebuf.com/articles/system/237300.html)

接下来就是把 binary 嵌入到程序中，我们可以生成一个数组。

但是也有别的方法：

[graphitemaster/incbin: Include binary files in C/C++](https://github.com/graphitemaster/incbin)

## Step2. 远程系统调用 mmap

其实我们可以直接写代码到 rip ，然后直接执行，不过为了同时演示远程系统调用，还是用一下 mmap 。

```cpp
#define ON_ERROR_KILL(d, x) if ((x) == -1) { perror(d); kill(pid, SIGKILL); return; }
void inject(int pid) {
    struct user_regs_struct regs{}, regs_backup{};
    long ins_back;
    int status;
    ON_ERROR_KILL("single step", ptrace(PTRACE_SINGLESTEP, pid, nullptr, nullptr));
    ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
    if (!WIFSTOPPED(status) || WSTOPSIG(status) != SIGTRAP) {
        cout << "stopped by other signal SIG" << sigabbrev_np(WSTOPSIG(status))  << endl;
        kill(pid, SIGKILL);
        return;
    }
    ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs));
    memcpy(&regs_backup, &regs, sizeof(struct user_regs_struct));
    ins_back = ptrace(PTRACE_PEEKTEXT, pid, regs.rip, nullptr);
    ON_ERROR_KILL("poke", ptrace(PTRACE_POKETEXT, pid, regs.rip, 0x050f));
    cout << "rip=" << hex << regs.rip << ",backup instructions:" << hex << ins_back << endl;
    regs.rax = SYS_mmap;
    regs.rdi = 0;
    regs.rsi = 10;
    regs.rdx = PROT_READ | PROT_WRITE | PROT_EXEC;
    regs.r10 = MAP_ANONYMOUS | MAP_PRIVATE;
    regs.r8 = 0xffffffff; // -1
    regs.r9 = 0;
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs));
    ON_ERROR_KILL("single step", ptrace(PTRACE_SINGLESTEP, pid, nullptr, nullptr));
    cout << "waiting" << endl;
    ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
    if (!WIFSTOPPED(status) || WSTOPSIG(status) != SIGTRAP) {
        cout << "stopped by other signal SIG" << sigabbrev_np(WSTOPSIG(status))  << endl;
        kill(pid, SIGKILL);
        return;
    }
    cout << "done" << endl;
    ON_ERROR_KILL("get result", ptrace(PTRACE_GETREGS, pid, nullptr, &regs));
    cout << "rip=" << hex << regs.rip << ",mmap returned: " << hex << regs.rax << endl;
    cout << "check mappings" << endl;
    cin.get();
    ON_ERROR_KILL("restore text", ptrace(PTRACE_POKETEXT, pid, regs_backup.rip, ins_back));
    ON_ERROR_KILL("restore regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs_backup));
    ON_ERROR_KILL("cont", ptrace(PTRACE_CONT, pid, nullptr, nullptr));
}
```

这里使用 peekdata 和 pokedata 读写内存，也可以换成 process_vm_readv/process_vm_writev ，在下面的写代码过程就使用了后者。

此处我们的 tracee 是 fork + exec 产生的，当产生了 execve stop 之后，似乎需要先 single step 一下才能正常进入下面的流程。这一点不知道在 man page 的哪里提到了。

调用 syscall 非常简单，我们往 rip 所指的位置写入 syscall 指令 (`0x050f`)  ，并向寄存器写入参数即可。接下来使用 single step 进行调用，然后还原修改的指令和寄存器。

[ChromiumOS Docs - Linux System Call Table](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/syscalls.md#x86_64-64_bit)

```
forked pid 2159
child forked
stopped by SIGTRAP(5)


continue
trapped
stopped at execve
rip=7f076faa72b0,backup instructions:d98e8e78948
waiting
done
rip=7f076faa72b2,mmap returned: 7f076fabe000
check mappings
```

maps:

```
5568aff72000-5568aff76000 r--p 00000000 08:20 48289                      /usr/bin/ls
5568aff76000-5568aff8a000 r-xp 00004000 08:20 48289                      /usr/bin/ls
5568aff8a000-5568aff92000 r--p 00018000 08:20 48289                      /usr/bin/ls
5568aff93000-5568aff95000 rw-p 00020000 08:20 48289                      /usr/bin/ls
5568aff95000-5568aff96000 rw-p 00000000 00:00 0
7f076fa87000-7f076fa89000 r--p 00000000 08:20 2055                       /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f076fa89000-7f076fab3000 r-xp 00002000 08:20 2055                       /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f076fab3000-7f076fabe000 r--p 0002c000 08:20 2055                       /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f076fabe000-7f076fabf000 rwxp 00000000 00:00 0
7f076fabf000-7f076fac3000 rw-p 00037000 08:20 2055                       /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7ffec7de4000-7ffec7e05000 rw-p 00000000 00:00 0                          [stack]
7ffec7e70000-7ffec7e74000 r--p 00000000 00:00 0                          [vvar]
7ffec7e74000-7ffec7e76000 r-xp 00000000 00:00 0                          [vdso]
```

可以发现，mmap 返回值 7f076fabe000 就是我们需要的 rwx page 。

此外，首次停止时的 rip 为 7f076faa72b0 ，对应内存区域属于 `/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2` ，即动态链接器，使用 readelf 查看 entry ，计算 entry 发现和 rip 刚好相等，因此 execve 确实停在了程序的入口（对于动态链接的程序，入口应该是动态链接器的入口）；进行系统调用的 single step 之后，rip 为 7f076faa72b2 ，即原来的 rip+2 ，刚好执行完 `syscall` 指令。

```
$ readelf -h /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
  Entry point address:               0x202b0

0x7f076fa87000 + 0x202b0 = 0x7f076faa72b0
```

可以使用 dd 查看修改了指令后的内存：

```
dd if=/proc/2159/mem skip=139669914940080 bs=1c count=16 | xxd
dd: /proc/2159/mem: cannot skip to specified offset
16+0 records in
16+0 records out
00000000: 0f05 0000 0000 0000 4989 c48b 05f7 9701  ........I.......
16 bytes copied, 0.00011526 s, 139 kB/s
```

### Step3. 写入代码并执行

接下来我们把之前产生的代码写入并执行：

```cpp
const unsigned char code[] = {
        0x48, 0xc7, 0xc0, 0x1, 0x0, 0x0, 0x0, 0x48,
        0xc7, 0xc7, 0x1, 0x0, 0x0, 0x0, 0x48, 0x8d,
        0x35, 0x15, 0x0, 0x0, 0x0, 0x48, 0xc7, 0xc2,
        0xd, 0x0, 0x0, 0x0, 0xf, 0x5, 0x48, 0xc7,
        0xc0, 0x3c, 0x0, 0x0, 0x0, 0x48, 0x31, 0xff,
        0xf, 0x5, 0x48, 0x65, 0x6c, 0x6c, 0x6f, 0x2c,
        0x20, 0x77, 0x6f, 0x72, 0x6c, 0x64, 0xa
};

    struct iovec local{
        .iov_base = (void*) code,
        .iov_len = sizeof(code)
    }, remote{
        .iov_base = (void*) regs.rax,
        .iov_len = sizeof(code)
    };
    ON_ERROR_KILL("write memory", process_vm_writev(pid, &local, 1, &remote, 1, 0));
    // ON_ERROR_KILL("restore text", ptrace(PTRACE_POKETEXT, pid, regs_backup.rip, ins_back));
    // ON_ERROR_KILL("restore regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs_backup));
    regs.rip = regs.rax;
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs));
    ON_ERROR_KILL("cont", ptrace(PTRACE_CONT, pid, nullptr, nullptr));
```

这样的结果是 ls 没被执行，直接退出了，因为我们写的程序就是打印 hello world 然后退出。

```cpp
stopped at execve
rip=7fa1a1dd72b0,backup instructions:d98e8e78948
waiting
done
rip=7fa1a1dd72b2,mmap returned: 7fa1a1dee000
Hello, world
exited with 0
```

现在我们希望执行完成后继续执行 ls ，我们修改上面的汇编程序，把 exit(0) 改成 `int3` ，这样打印了 hello world 后会产生一个断点，我们捕获断点后恢复修改的代码和上下文。

此时直接编译并执行 hello ，应该是这样的：

```
./hello
Hello, world
Trace/breakpoint trap
```

相应地修改 inject 的代码

```cpp
const unsigned char code[] = {
        0x48, 0xc7, 0xc0, 0x1, 0x0, 0x0, 0x0, 0x48,
        0xc7, 0xc7, 0x1, 0x0, 0x0, 0x0, 0x48, 0x8d,
        0x35, 0xa, 0x0, 0x0, 0x0, 0x48, 0xc7, 0xc2,
        0xd, 0x0, 0x0, 0x0, 0xf, 0x5, 0xcc, 0x48,
        0x65, 0x6c, 0x6c, 0x6f, 0x2c, 0x20, 0x77, 0x6f,
        0x72, 0x6c, 0x64, 0xa
};
    ON_ERROR_KILL("write memory", process_vm_writev(pid, &local, 1, &remote, 1, 0));
    regs.rip = regs.rax;
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs));
    ON_ERROR_KILL("cont", ptrace(PTRACE_CONT, pid, nullptr, nullptr));
    ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
    if (!WIFSTOPPED(status) || WSTOPSIG(status) != SIGTRAP) {
        cout << "stopped by other signal SIG" << sigabbrev_np(WSTOPSIG(status))  << endl;
        kill(pid, SIGKILL);
        return;
    }
    ON_ERROR_KILL("restore text", ptrace(PTRACE_POKETEXT, pid, regs_backup.rip, ins_back));
    ON_ERROR_KILL("restore regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs_backup));
    ON_ERROR_KILL("cont", ptrace(PTRACE_CONT, pid, nullptr, nullptr));
```

现在就可以在注入的代码执行完后继续执行原来的程序了：

```
rip=7f0dc58f52b2,mmap returned: 7f0dc590c000
Hello, world
CMakeCache.txt  CMakeFiles  Makefile  Testing  cmake_install.cmake  exec  fork-exec  inject  ptrace_learn  ptrace_learn.cbp  trace-exec  trace-fork-exec
exited with 0
```

## 远程函数调用和 dlopen

## ……

## 其他

chromium 项目的 errno 和 signal

[ChromiumOS Docs - Linux Error Number Table (errno)](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/errnos.md)

[ChromiumOS Docs - Linux Signal Table](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/signals.md)

