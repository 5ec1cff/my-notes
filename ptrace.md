# ptrace

[ptrace(2) - Linux manual page](https://man7.org/linux/man-pages/man2/ptrace.2.html)

[wait(2) - Linux manual page](https://man7.org/linux/man-pages/man2/waitpid.2.html)

本文相关代码都在下面的仓库：

https://github.com/5ec1cff/ptrace-examples

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

下面是一个 x86-64 上的汇编 hello world 实例。

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

### Step 2. 远程系统调用 mmap

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

这里使用 peekdata 和 pokedata 读写内存

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

### Step 3. 写入代码并执行

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

## ptrace POKEDATA 和 process_vm_writev

ptrace POKEDATA/POKETEXT 可以直接写入没有写权限的页，process_vm_writev 则不行。因此 POKEDATA 成为了在只有 ptrace ，且进程中没有 rwx 页面的情况下，写入任意代码的唯一方法。

PEEKDATA/POKEDATA 一次只能传输一个处理器字长 (long) 的数据，只能做一些替换少量代码的工作。相比之下，process_vm_readv/writev 使用 iovec ，可以实现从多个缓冲区读出/写入多个缓冲区，一次系统调用能传输指定长度的数据，比 ptrace 更适合传输大量数据。

## 系统调用劫持

考虑到 POKEDATA 存在一些问题，我们最好不要往上面写 syscall 的代码，那么怎么执行系统调用呢？可以想到的方法是借用已有的 syscall 。

一般来说，我们的 tracee 都会执行系统调用，下面我们将劫持第一个系统调用，借助它完成 mmap 并写入和执行代码，然后恢复原来的执行流程。

### Step 1. 等待第一个系统调用

使用 `PTRACE_SYSCALL` 可以在系统调用进入 (syscall-enter) 前停止。我们在这个阶段观察 rip, rax, orig_rax ，并备份寄存器上下文。

设置了 `PTRACE_O_TRACESYSGOOD` 之后，syscall stop 的 STOPSIG 为 `SIGTRAP|0x80` ，便于我们区分。

```cpp
    ON_ERROR_KILL("set options", ptrace(PTRACE_SETOPTIONS, pid, nullptr, PTRACE_O_TRACESYSGOOD));

    ON_ERROR_KILL("next syscall", ptrace(PTRACE_SYSCALL, pid, nullptr, nullptr));
    ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
    if (!WIFSTOPPED(status) || WSTOPSIG(status) != (SIGTRAP | 0x80)) {
        cout << "stopped by other non-syscall stop signal SIG" << sigabbrev_np(WSTOPSIG(status)) << endl;
        kill(pid, SIGKILL);
        return;
    }
    ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs));
    memcpy(&regs_backup, &regs, sizeof(struct user_regs_struct));
    cout << "rip=" << hex << regs.rip << " rax=" << hex << regs.rax << " orig_rax=" << dec << regs.orig_rax << endl;
    cin.get();
```

我们执行的 ls 在第一个系统调用停下了，可以检查 maps 发现 rip 所指的地址位于 ld-linux ，orig_rax 保存了系统调用号，此处为 12 ，也就是 brk 。rax 寄存器的值看起来像是某个 errno ，对不同的系统调用似乎还不一样，不过不需要管它，我们只要关注 orig_rax 即可。

另外，rip 指向的地址已经是 `syscall` 指令的下一个指令了，因此后来恢复的时候需要将 rip -2 。

```
forked pid 2880
child forked
stopped by SIGTRAP(5)


continue
trapped
stopped at execve
rip=7f27fa87caab rax=ffffffffffffffda orig_rax=12
```

### Step 2. 系统调用号，借用一下

接下来我们可以把 orig_rax 换掉，也就是替换系统调用号为 mmap ；同时也要把系统调用参数的寄存器设置成 mmap 的参数。

> 在 x86 上是替换 orig_rax/eax ，在 arm 上没有 orig rax 了，但是可以使用 PTRACE_SET_SYSCALL 替代。

```cpp
    cout << "exec mmap" << endl;
    regs.orig_rax = SYS_mmap;
    regs.rdi = 0;
    regs.rsi = sizeof(code);
    regs.rdx = PROT_READ | PROT_WRITE | PROT_EXEC;
    regs.r10 = MAP_ANONYMOUS | MAP_PRIVATE;
    regs.r8 = 0xffffffff; // -1
    regs.r9 = 0;
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs));
    ON_ERROR_KILL("next syscall", ptrace(PTRACE_SYSCALL, pid, nullptr, nullptr));
    ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
    if (!WIFSTOPPED(status) || WSTOPSIG(status) != (SIGTRAP | 0x80)) {
        cout << "stopped by other non-syscall stop signal SIG" << sigabbrev_np(WSTOPSIG(status)) << endl;
        kill(pid, SIGKILL);
        return;
    }
    ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs));
    cout << "mmap return:" << hex << regs.rax << endl;
    cin.get();
```

经过下一个 `PTRACE_SYSCALL` ，我们来到了 syscall-exit stop ，此时可以拿到 mmap 的返回值：

```
exec mmap
mmap return:7f27fa88e000
```

### Step 3. 写入代码和运行

步骤同「远程系统调用」一节的 Step 3 ，成功执行后：

```
write code and run
Hello, world
```

### Step 4. 恢复原来的系统调用

我们借用了第一个 syscall ，但是应该怎么还回去呢？

前面提到，在 syscall-enter stop 的时候，rip 已经走到了 syscall 的下一个指令，rax 的值也被装入 orig_rax 。

因此我们需要把 rip -2 ，同时 rax 的值也还原回去。

```cpp
    cout << "replay syscall" << endl;
    regs_backup.rip -= 2;
    regs_backup.rax = regs_backup.orig_rax;
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs_backup));
```

接下来可以直接 PTRACE_CONT 或者 PTRACE_SINGLESTEP 了，不过我们也可以看一看原先的系统调用有没有正确执行：

```cpp
    int i = 2;
    while (i--) {
        ON_ERROR_KILL("next syscall", ptrace(PTRACE_SYSCALL, pid, nullptr, nullptr));
        ON_ERROR_KILL("waitpid", waitpid(pid, &status, __WALL));
        if (!WIFSTOPPED(status) || WSTOPSIG(status) != (SIGTRAP | 0x80)) {
            cout << "stopped by other non-syscall stop signal SIG" << sigabbrev_np(WSTOPSIG(status)) << endl;
            kill(pid, SIGKILL);
            return;
        }
        ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs));
        cout << "rip=" << hex << regs.rip << " rax=" << hex << regs.rax << " orig_rax=" << dec << regs.orig_rax << endl;
    }

    ON_ERROR_KILL("cont", ptrace(PTRACE_CONT, pid, nullptr, nullptr));
```

执行结果：

```
replay syscall
rip=7f27fa87caab rax=ffffffffffffffda orig_rax=12
rip=7f27fa87caab rax=55d9159a9000 orig_rax=12
CMakeCache.txt  CMakeFiles  Makefile  Testing  cmake_install.cmake  exec  fork-exec  inject  memory-write  ptrace_learn  ptrace_learn.cbp  test  trace-exec  trace-fork-exec  trace-syscall
exited with 0
```

看起来 brk 被正确执行了，PTRACE_CONT 后，下面的 ls 程序也正常运行直到退出。

> 与 glibc 不同，真正的 syscall brk 实际上返回的是新的 brk 地址，而非 errno ，在 man brk 的 notes 有提到。

## 远程函数调用

我们的目标是附加任意进程，调用 puts 函数输出 hello world 。

首先明确，这个操作是针对动态链接的程序的。在这种情况下，处理 execve + traceme 就比较麻烦了，因为 linker 究竟何时完成初始化比较难以判断，所以这里使用 attach 。

> 当然我们可以用给真正的程序入口下断点等方式，确保 linker 完成加载。

### Step 1. 确定函数地址  

puts 函数位于 libc 中，由于是动态链接库的函数，因此获取地址分为两步：

1. 在目标进程找到 libc 的基地址。  
2. 在 libc 中找到 puts 函数。  

第一步寻找基地址，可以通过 /proc/pid/maps 确定，已经有现成的 [proc maps parser](https://github.com/ouadev/proc_maps_parser/blob/master/pmparser.c) 可以直接使用。我们将在 tracer 中完成这个工作。

> 上面的 proc-maps-parser 调用 fgets 的时候有些问题，可以参考 Rikka 的 [prefab](https://github.com/RikkaW/proc-maps-parser-prefab/blob/master/proc-maps-parser/src/main/cpp/pmparser.c) 版本，另外原版是 C 专用的，在 C++ 项目中使用需要在 header 加上 `extern "C"` 。

```cpp
    void* libc_base = nullptr;
    
    procmaps_iterator* maps = pmparser_parse(pid);
    if (maps == nullptr){
        cerr << "cannot parse the memory map of " << pid << endl;
        return;
    }
    procmaps_struct* maps_tmp;
    while ((maps_tmp = pmparser_next(maps)) != nullptr) {
        if (string_view(maps_tmp->pathname).find("libc.so.6") != string_view::npos && maps_tmp->offset == 0) {
            libc_base = maps_tmp->addr_start;
            cout << "found libc in maps, base=" << hex << libc_base << endl;
            break;
        }
    }
    pmparser_free(maps);
    if (!libc_base) {
        cout << "libc not found" << endl;
        return;
    }
```

接下来是寻找 puts 的相对地址，可以在 tracer 中解析 libc 这个 ELF 文件，找到 puts 的地址，但这里我偷懒了，直接 readelf 把地址读出来硬编码进 tracer 就完事了。

> 甚至可以在 tracer 中 dlsym 找到函数地址，然后结合 tracer 中 libc 基址（通过 dl_iterator_phdr 确定），得到 puts 的相对地址。这样的做法不适合不同指令集的进程的情况。（硬编码自然更不适合。）

```cpp
// readelf -s -W /lib/x86_64-linux-gnu/libc.so.6 | grep puts
//  1429: 0000000000080ed0   409 FUNC    WEAK   DEFAULT   15 puts@@GLIBC_2.2.5
    auto puts_off = 0x80ed0;
```

得到了进程中 libc 基址和 puts 的偏移地址后，加起来就是进程中的 puts 地址。

### Step 2. 构造调用代码

上面得到的是一个绝对地址，因此我们使用 `call *%rax` 进行调用，这样只要使用 ptrace 写 puts 的地址到 rax 寄存器即可。

我用 python 写了一个简陋的 shellcode 生成器 shell.py:

```py
import sys
import os
from pathlib import Path

TMP_PATH = Path('/tmp')
TEMPLATE = '''        .global _start

        .text
_start:
'''

temp_asm = TMP_PATH / "shellcode.s"
temp_exe = TMP_PATH / "shellcode"
temp_bin = TMP_PATH / "shellcode.bin"

with open(temp_asm, 'w') as f:
    f.write(TEMPLATE + sys.stdin.read())

cmd = f"gcc -static -nostdlib -fPIE {temp_asm} -o {temp_exe}"
print(cmd)
if os.system(cmd) != 0:
    print('failed to exec gcc')
    exit(1)

cmd = f"objcopy -O binary -j .text {temp_exe} {temp_bin}"
print(cmd)
if os.system(cmd) != 0:
    print('failed to exec objcopy')
    exit(1)

print('result:')

with open(temp_bin, 'rb') as f:
    i = 0
    for b in f.read():
        print(hex(b), end=', ')
        i += 1
        if i % 8 == 0:
            print()
```

用上面的生成器生成我们的代码，其实就是一个 call ，加上一个断点。

```
$ python3 ~/code/shell.py <<-EOF
> call *%rax
> int3
> EOF
gcc -static -nostdlib -fPIE /tmp/shellcode.s -o /tmp/shellcode
objcopy -O binary -j .text /tmp/shellcode /tmp/shellcode.bin
result:
0xff, 0xd0, 0xcc,
```

shellcode 不过 3 字节，方便 POKEDATA 写入。

```cpp
    struct user_regs_struct regs{}, regs_backup{};
    long ins_back;
    int status;

    ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs))
    memcpy(&regs_backup, &regs, sizeof(struct user_regs_struct));
    ins_back = ptrace(PTRACE_PEEKTEXT, pid, regs.rip, nullptr);
    cout << "rsp=" << hex << regs.rsp << ",rip=" << hex << regs.rip << ",backup instructions:" << hex << ins_back << endl;
    cin.get();
    ON_ERROR_KILL("poke", ptrace(PTRACE_POKETEXT, pid, regs.rip, 0xccd0ff));
```

确定了函数地址和调用的代码，还需要提供参数。这里我们选择在栈上写入 `Hello world` 。

```cpp
const char MSG[] = "Hello world!";

    auto arg = (void*) (regs.rsp - sizeof(MSG));
    struct iovec local{
            .iov_base = (void*) MSG,
            .iov_len = sizeof(MSG)
    }, remote{
            .iov_base = arg,
            .iov_len = sizeof(MSG)
    };
    ON_ERROR_KILL("write memory", process_vm_writev(pid, &local, 1, &remote, 1, 0))
    cin.get();
    regs.rax = (long long int) libc_base + puts_off; // call $rax
    regs.rdi = (long long int) arg; // arg1
    regs.rsp -= sizeof(MSG);
    ON_ERROR_KILL("set regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs));
```

> https://stackoverflow.com/a/2538212  
> https://zhuanlan.zhihu.com/p/27339191  

x86-64 中的函数<ruby>调用约定<rt>Calling Convention</rt></ruby>大概是：
前 6 个参数依次通过下面的寄存器传参： `%rdi, %rsi, %rdx, %rcx, %r8, %r9`；
多于 6 个参数则放到栈上。  

我们的 [puts](https://man7.org/linux/man-pages/man3/puts.3.html) 函数很简单，只接收一个参数，就是字符串的地址。

上面的代码将堆栈指针 rsp 向下移动 MSG 的长度，并把 MSG 复制到 rsp 所指的地址后面，此时 rsp 的值就是要传给 puts 的字符串的地址。

最后别忘了函数地址是赋给 rax 而不是 rip 。

完成了寄存器的布置后，继续运行程序，如果成功，应该可以观察到 tracee 输出 `Hello world!` ，随后 tracee 在我们布置的断点处停下。

### Step 3. 还原

这里就没什么好说的了，把之前修改的寄存器和指令用备份还原回去即可。

```cpp
    ON_ERROR_KILL("get regs", ptrace(PTRACE_GETREGS, pid, nullptr, &regs))
    ON_ERROR_KILL("restore text", ptrace(PTRACE_POKETEXT, pid, regs_backup.rip, ins_back))
    ON_ERROR_KILL("restore regs", ptrace(PTRACE_SETREGS, pid, nullptr, &regs_backup))
```

上面的例子演示了使用 ptrace 实现简单的函数调用。假如需要调用的函数参数很复杂，用上面的方法就很繁琐，需要自己定位函数地址，并且写入内存、设置寄存器以传参，这样编写的代码难以具有通用性。此外，如果需要实现更复杂的调用，那么用上面的方法依次进行远程调用，效率会非常低下。

我们更希望使用熟悉的高级编程语言编写逻辑，然后注入到进程中执行。可以想到的方法是使用 ptrace mmap 代码，跳转到对应位置执行。更进一步地想，在动态链接的程序中，ptrace 通过 dlopen 打开我们事先编写的动态共享库，加载代码执行，这样就比上面的方法方便多了。

## 信号处理和 PTRACE_SEIZE

siginfo:

https://man7.org/linux/man-pages/man2/sigaction.2.html

我们知道 ptrace 虽然强大，但是同一时间每个进程只允许一个进程作为它的 tracer 。当我们自己编写的 tracer 遇到了无法处理的问题，难以再挂接调试器到 tracee 上，除非我们主动 detach （当然，你也可以做调试 tracer ，进而间接调试 tracee 这样的套娃操作），那么这时候就需要我们 detach 的同时把进程留在 stop 状态。此外，我们也要知道如何附加一个一开始就是 stop 状态的进程。

之前我们附加进程都是使用 `PTRACE_ATTACH` 或者 `PTRACE_TRACEME` 。

……

### attach 与 SIGSTOP

下面考虑一个进程，一开始就通过 raise SIGSTOP 处于 stop 状态，然后使用 PTRACE_ATTACH 附加。

一开始会得到一个 SIGSTOP 。尝试 GETSIGINFO 会得到 EINVAL 。

```
waiting ...
process stopped by signal SIGSTOP (19), status=137f, event=(none)
get siginfo: Invalid argument
```

接下来我们 CONT ，注入信号 0 ，得到另一个 SIGSTOP ，这个 STOP 来源于内核，看起来似乎是由 attach 产生的：

```
inject signal SIG0
waiting ...
process stopped by signal SIGSTOP (19), status=137f, event=(none)
signo: 19, si_code:SI_KERNEL(128)
```

如果此时选择注入信号 0 ，那么原先 raise 的 stop 状态就被解除，进程随后执行完毕退出。

如果选择注入信号 SIGSTOP ，接下来 waitpid 又会收到一个 SIGSTOP ，和第一个一样，无法得到 siginfo 。

接下来如果注入信号 0 ，进程同样解除 stop ，执行完毕退出；如果选择不注入信号（也就是不调用 CONT），那么 waitpid 会继续等待，此时进程的状态是 tracing stop ，此时从外部发送 SIGCONT 信号，tracer 和 tracee 都没有任何反应。

另外一个例子：运行一个无限循环的程序，tracer attach 上去，然后从外部 kill SIGSTOP ，得到一个 signal delivery stop：

```
inject signal SIG0
inject 0 instead
waiting ...
process stopped by signal SIGSTOP (19), status=137f, event=(none)
signo: 19, si_code:SI_USER(0)
```

此时注入 SIGSTOP 信号后，waitpid 也会得到一个无法获取 siginfo 的 SIGSTOP ：

```
inject signal SIGSTOP
waiting ...
process stopped by signal SIGSTOP (19), status=137f, event=(none)
get siginfo: Invalid argument
```

我们期望当进程自己收到 SIGSTOP ，处于 stop 状态的时候，ptrace 能够等待一个 SIGCONT 让进程重启，而通过上面的例子来看，我们似乎没法做到这一点，因为不管是 attach 前 stop ，还是 attach 之后发生了 stop ，通过信号注入 SIGSTOP ，我们都会收到一个未知来源的 SIGSTOP 。此时注入任意信号，都会使进程继续运行；如果忽略它，则进程一直停留在 tracing stop 状态，且无法拦截任何信号。

其实上面的问题正是 PTRACE_ATTACH 的缺陷，在 [man 2 ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) 的 `Group-stops` 一节已经介绍了。而为了解决这个问题，在 Linux 3.4 推出了 PTRACE_SEIZE 。

### PTRACE_SEIZE 下的 Group-stop

ptrace 的 manual page 提到了几种不同的停止状态，包括 `signal-delivery-stop`, `group-stop`, `syscall-stop`, `PTRACE_EVENT stops` 四种。

Group-stop 就是 SIGSTOP 等停止信号导致的进程停止，正常情况下，停止导致进程状态变为 T (stopped) ，而在 ptrace 下，它们导致进程状态变成 t (tracing-stop) 。在 PTRACE_ATTACH 模式中，Group-stop 汇报为 SIGSTOP ，此时调用 PTRACE_GETSIGINFO 应当返回 EINVAL ；而 PTRACE_SEIZE 中，这种停止状态得到的 status 满足 `(status >> 8) == (SIGSTOP | PTRACE_EVENT_STOP << 8)`。同时，PTRACE_GETSIGINFO 是有效的，其 si_code 也设为前面的 `status >> 8` 的值。

……

当进程进入 Group-stop 的时候，可以使用 `PTRACE_LISTEN` 「恢复」进程。此时进程保持停止，但可以接收信号。

```
... The state of the tracee after PTRACE_LISTEN is somewhat of a gray
area: it is not in any ptrace-stop (ptrace commands won't work on
it, and it will deliver waitpid(2) notifications), but it also
may be considered "stopped" because it is not executing
instructions (is not scheduled), and if it was in group-stop
before PTRACE_LISTEN, it will not respond to signals until
SIGCONT is received.
```

#### SIGSTOP

如果 tracee 收到 SIGSTOP ，会进入 tracing-stop ，tracer 首先得到 signal delivery stop ，然后如果 tracer 注入了 SIGSTOP ，就会得到一个带有 PTRACE_EVENT_STOP 事件的 SIGTRAP。

> SIGSTOP 可以被 tracer 忽略。

```
process stopped by signal SIGSTOP (19), status=137f, event=(none)
signo: 19, si_code:SI_USER(0)

inject signal SIGSTOP
waiting ...
process stopped by signal SIGSTOP (19), status=80137f, event=PTRACE_EVENT_STOP
signo: 19, si_code:unknown(32787)
```

#### SIGCONT

如果 tracee 收到 SIGCONT ，无论 tracee 是否处于 group-stop ， tracer 都会首先得到一个 SIGTRAP ，带有 PTRACE_EVENT_STOP 事件。

```
process stopped by signal SIGTRAP (5), status=80057f, event=PTRACE_EVENT_STOP
signo: 5, si_code:PTRACE_EVENT_STOP(32773)
```

然后调用 PTRACE_CONT 注入信号 0，如果 SIGCONT 没有被屏蔽，才会进入 signal delivery stop ，注入 SIGCONT 可以让 tracee 继续运行。如果 SIGCONT 被 tracee 屏蔽了，进程会直接运行。

```
process stopped by signal SIGCONT (18), status=127f, event=(none)
signo: 18, si_code:SI_USER(0)
```

此时注入信号 0 也会使得 tracee 继续运行（就像注入了 SIGCONT 一样），而注入信号 SIGSTOP 会使 tracee 保持停止。

那么注入信号 0 和注入 SIGCONT 有什么区别呢？区别体现在是否调用信号处理器。

正常进程接收 SIGCONT 的流程是这样的：

1. 如果进程停止，则首先唤醒（此时忽视 SIGCONT 的屏蔽字）  
2. 如果进程设置了 handler ，且 SIGCONT 没有被屏蔽，则调用 handler  

在 ptrace 下，我们可以注入信号 0 ，以忽略第二步，也就是 SIGCONT 的 handler 调用，如果注入信号 SIGCONT ，就会调用 handler 。

我写了一个[交互式程序](https://github.com/5ec1cff/ptrace-examples/blob/master/trace-repl.cpp)，并观察 interrupt 、listen、cont 以及信号导致停止的行为，总结如下：

1. 向 tracee 发送 SIGSTOP ，首先产生 signal-delivery-stop ，可以注入或忽略 SIGSTOP ，注入后产生 SIGSTOP 的 PTRACE_EVENT_STOP  
2. PTRACE_INTERRUPT 产生一个 SIGTRAP 的 PTRACE_EVENT_STOP 。  
3. 所有的 PTRACE_EVENT_STOP 事件导致的停止都可以调用 PTRACE_LISTEN ，除此之外都不可以。  
4. PTRACE_EVENT_STOP 导致的停止，可以通过 PTRACE_CONT (0) 使其继续运行。  
5. 进入 listen 状态后，不能使用 PTRACE_CONT 继续 tracee 的运行，但是可以使用 PTRACE_INTERRUPT 中断。  
6. 进入 listen 状态后，tracee 停止运行，但可以接收信号产生 signal-delivery-stop ；如果进入 listen 状态之前发送了信号（也就是有 pending 信号），进入之后也会产生 signal-delivery-stop 。  
7. 向 tracee 发送 SIGCONT ，首先产生一个 SIGTRAP 的 PTRACE_EVENT_STOP ，行为和 PTRACE_INTERRUPT 相同。  

至于怎么 detach 并保持停止，其实也很简单，首先 kill/tgkill SIGSTOP ，然后 PTRACE_DETACH sig=SIGSTOP 即可。这样相当于注入一个 SIGSTOP 信号。

另外观察到一个现象：
如果 seize 的时候 tracee 是停止的，那么即使用过 cont 0 使 tracee 在跟踪期间继续运行，在 tracer detach 或者直接退出的时候进程会重新变成停止的，除非跟踪期间进程收到过 SIGCONT 并被放行，或者存在 pending 的 SIGCONT；
如果 seize 的时候 tracee 是运行的，则 tracer detach 或直接退出的时候 tracee 会保持运行，除非最后发送一个 SIGSTOP 。

上面的现象可以解释为进程有一个 run / stop 的状态，可以通过 SIGSTOP 和 SIGCONT 修改，tracer 可以通过 ptrace 的暂时修改这个状态，让本来处于 stop 状态的进程运行，但 tracer 离开之后，进程又恢复到本来的状态。

> 上文都是基于 man pages 和观察得出的结论，至于实际情况如何，还是要看内核源码的实现。

## 其他

chromium 项目的 errno 和 signal

[ChromiumOS Docs - Linux Error Number Table (errno)](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/errnos.md)

[ChromiumOS Docs - Linux Signal Table](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/constants/signals.md)

