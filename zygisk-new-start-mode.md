# 修改 Magisk 为 Zygisk 提供新启动方式  

一直对 zygisk 产生的那个[栈底的 `/dev/fd`](new-idea-detect-zygisk.md) 很在意，因为一般的 zygote 不可能通过 execveat 启动的，这自然是一个明显的特征，而 shamiko 之类的估计不好修复。因此，还需要修改 zygisk 的行为，让它真正 execve /system/bin/app_process 。

最近总算[配置好了](build-magisk-on-windows.md)在 windows 上开发 magisk 的环境，下面让我们试试吧！

下文以 zygisked-app_process 称呼被 magisk 修改后的 app_process （本质上就是 magisk 主程序）

## 尝试一

一开始的思路是：

```
exec zygisked-app_process -> umount zygisked-app_process -> exec orig app_process
```

失败。linker 提示 appwidget 无法链接到 app_process

```
09-17 11:52:01.206 24482 24482 E linker  : library "/system/bin/app_process" ("/system/bin/app_process64") needed or dlopened by "/system/bin/appwidget" is not accessible for the namespace: [name="(default)", ld_library_paths="", default_library_paths="/system/lib64:/system_ext/lib64", permitted_paths="/system/lib64/drm:/system/lib64/extractors:/system/lib64/hw:/system_ext/lib64:/system/framework:/system/app:/system/priv-app:/system_ext/framework:/system_ext/app:/system_ext/priv-app:/vendor/framework:/vendor/app:/vendor/priv-app:/system/vendor/framework:/system/vendor/app:/system/vendor/priv-app:/odm/framework:/odm/app:/odm/priv-app:/oem/app:/product/framework:/product/app:/product/priv-app:/data:/mnt/expand:/apex/com.android.runtime/lib64/bionic:/system/lib64/bootstrap"]
```

起初以为是 LD_PRELOAD 失败了，app_process 无法加载位于 /system/bin 的 so 作为 preload 。看到 /data 是允许的路径，还尝试把 HIJACK_BIN 改成 /data/adb ，结果发现 zygote 没权限访问（以前一直以为可以访问的）。

不过再仔细一看，其实是 appwidget 链接到 app_process 出现了问题。

研究了一下源码，发现现在的加载机制改了：一阶段是一个独立的 zygisk-loader.so ，编译好后包含在 magisk 程序的一个数组里面。在 zygisked-app_process 启动的时候发送给 magiskd ，然后接收并 mount 到 HIJACK_BIN 目录（64 位是 /system/bin/appwidget ，32 位是 /system/bin/bu ）。LD_PRELOAD 设置为 HIJACK_BIN ，随后 fexecve 执行原 app_process 。一阶段 loader 中会 dlopen `/system/bin/app_process` ，此处是强制重新打开(android_dlopen_ext flags=`ANDROID_DLEXT_FORCE_LOAD`)，因为这个 app_process 实际上是 zygisked-app_process ，二阶段的入口就包含在里面。

而现在的问题就是 dlopen 出现了问题，至于为何打开原 app_process 会出现 namespace 的问题暂不追究，至少如果这个 app_process 是「zygisked」的，就能打开。

因此，假如我们希望直接 execve 原 app_process，我们 umount 后还要再 mount 回去。

另外，考虑到目前的操作都在全局挂载命名空间似乎不妥，而 zygote 启动后会 unshare ，且 remount root 为 slave ，于是还尝试了在 zygisk 启动阶段就主动 unshare + slave mount ，然而这样直接导致 zygote 崩溃：

```
Process uptime: 0s
Cmdline: zygote64
pid: 8465, tid: 8465, name: main  >>> zygote64 <<<
uid: 0
signal 6 (SIGABRT), code -1 (SI_QUEUE), fault addr --------
Abort message: 'JNI FatalError called: (system_server) Not allowlisted (5): /trace_marker'
```

看上去是某个奇怪的 fd 泄露了，但是 zygisk 的 fd 应该都好好关闭或者 CLOEXEC 了吧？搜索源码也没有这个 `trace_marker` 的痕迹。

检查 tombstone ，发现打开的 fd 在 /proc/self/fd 的路径有部分变得很奇怪：

```
open files:
    fd 0: /dev/null (unowned)
    fd 1: /dev/null (unowned)
    fd 2: /dev/null (unowned)
    fd 3: socket:[126612] (unowned)
    fd 4: /dev/pmsg0 (unowned)
    fd 5: /trace_marker (unowned)
    fd 6: /javalib/core-oj.jar (unowned)
    fd 7: /javalib/core-libart.jar (unowned)
    fd 8: /javalib/okhttp.jar (unowned)
    fd 9: /javalib/bouncycastle.jar (unowned)
```

其中 `/trace_marker` 就是 `/sys/kernel/tracing/trace_marker` 。这个路径的 fd 在 zygote 是允许的，然而现在变成了这样的名字，就没法允许了。下面的 /javalib 似乎来自 apex 。总之可能是两次 unshare + mount slave 导致的奇怪问题，不过自己没法复现，也不知道具体是哪里的原因。

## 尝试二

既然 zygisk 自己都是在全局命名空间处理 HIJACK_BIN 的卸载的，那我们也不必保证它的隔离性，毕竟不会同时有两个 zygote 启动（指同 64/32）

这次尝试不 umount ，而是直接把 orig app_process 又挂到原目录上，相当于挂了三层，进入之后再 umount 。

这样 linker 的问题也没了，但是 hook 却无法生效。检查发现，本应 hook app_process 的 setArgv0 却没 hook 到，导致后面的工作都无法进行；进一步发现，maps 中名为 app_process 的内存是 zygisked-app_process 的，而真正的 app_process 名字却是 `/` （只能通过 inode 号查找了），由于 xhook 搜索 so 是通过 maps 进行的，这就导致无法找到正确的 app_process 。

> 对比了一下，正常执行的 zygisk ，在内存里面应该会有两个 app_process 文件的映射 **（这样真的不会对 xhook 有问题吗？）**

看上去是我们 umount 导致的路径变成 `/` ，自己写了个程序验证了一下，确实如此，也许和内核查找路径的方式有关。

## 终极尝试

看上去叠加 mount 并不好，最好还是 umount 掉 zygisked-app_process ，execve orig，然后在 loader 中请求 magiskd mount zygisked-app_process （因为 zygote 自己无法 mount）。

不过 loader 是独立于 magisk 其他部分的，写起来、调试起来并不方便，不过最后还是写出了一个可行的版本。

loader 中连接 magiskd socket 请求 mount ，这里也采用了 patch 方法，发送 loader 的同时 patch socket name （使用了和 MAIN_SOCKET 不同的魔数）。

> 突然想到，其实根本不用我们亲自 patch ，直接用和 MAIN_SOCKET 相同的魔数就好。因为 loader 自身也包含在 magisk 主程序中，patch 主程序的时候肯定会 patch 这部分的。

## 第一次终极尝试  

考虑到在 loader 中和 magiskd 通信较为麻烦，而且这样不可避免地要增加 Request code ，而这部分位于 C++ ，code 写在 enum 里面，不太好迁移到 C ，于是干脆使用更加原始的通信机制——信号。

具体思路是：zygisked-app_process 将自己的某个信号阻塞，然后请求 magiskd umount zygisked-app_process ，发送 loader 后不 close socket，直接 exec 。对面 magiskd read 等待回应，如果发现连接断开，认为 zygote 已经 exec （假如 exec 失败，zygisked-app_process 会回应，以分辨是否成功），mount zygisked-app_process 并发送信号给 zygote ，此时 loader 使用 sigsuspend 等待信号到来，之后直接执行 loader 逻辑。

观察 zygote 的 SIGBLK ，发现 SIGUSR1 一般是 blocked 的，因此适合作为我们需要的信号。

按照这个想法写了一下，确实成功了。不过这个做法似乎有些 ub ，因为我们不知道 exec 导致的 fd close 并通知到 magiskd mount 和内核加载对应的路径的程序到底谁先执行。

于是为了确保正确性，又增加了 stat ，首先 stat 原始 app_process 的 inode ，然后 execve 连接断开的时候在 3s 内 stat /proc/zygote/exe 的 inode ，如果是 app_process 才 mount 并发信号。

实现为 stat 一次等待 100ms ，共 30 次。不过从实际情况来看，基本都是第一次就成功了。

最后又给 loader 加了个 sigalrm ，确保 3s 内不成功能自杀（下一次就走正常 app_process 了）。刚好 Zygote 不 block SIGALRM ，因此不需要我们主动还原 mask 。

> 至于 Zygote 是不是真的有固定的信号 mask ，其实我也不太清楚……反正看了几个设备的 SigBlk 都是 `80001204`

## 感想

Android.mk 的项目写起来真的不方便，因为 AS 不提供语言支持（~~会不会是有能用的 IDE 我不知道呢~~）。所有的语法问题都只能等编译才能发现。并且这个构建系统似乎 native 部分不是增量构建，每次都要重新编译，所以开发周期被大大地延长了（毕竟我脱离了强大 IDE 的支持写代码就寸步难行了）。总之，开发 magisk 十分考验功底，很难不佩服创造者 wu 和其他 magisk 的开发者。
