# 分析 Zygisk

半年多前在 gist 写了一篇分析 zygisk 源码的文章，时过境迁，Zygisk 的源码也不断更新，因此旧的文章可能不再适用了。刚好最近研究使用 native bridge 重写 zygisk 加载，也了解了一下新的变化，于是重新写一篇分析。

## pre fork

Zygisk 执行了一个「预 fork」的操作，具体来说就是调用 forkAndSpecialize 的时候主动 fork ，并记录 fork pid ，等到真正的 fork 要执行的时候再把这个 pid 传过去。这个操作有利于和 USAP 机制保持统一性，以及最重要的，确保模块能够不进入 zygote 主进程而干预 Specialize pre 的逻辑。

由于 USAP 的存在，fork 的时候不一定就会 specialize ，因此只有当 HookContext 存在且记录的 pid 有效的时候才执行 pre fork ，否则执行原逻辑。

```cpp
// Skip actual fork and return cached result if applicable
DCL_HOOK_FUNC(int, fork) {
    return (g_ctx && g_ctx->pid >= 0) ? g_ctx->pid : old_fork();
}
```

## fd sanitize

Zygote 在某个版本引入了 fd 检查机制，确保设计上不存在把 zygote 的 fd 泄露给 app 进程的情况。

这个检查发生在任何 fork 之前。

```cpp
// frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
// Utility routine to fork a process from the zygote.
NO_STACK_PROTECTOR
pid_t zygote::ForkCommon(JNIEnv* env, bool is_system_server,
                         const std::vector<int>& fds_to_close,
                         const std::vector<int>& fds_to_ignore,
                         bool is_priority_fork,
                         bool purge) {
  SetSignalHandlers();

  // Curry a failure function.
  auto fail_fn = std::bind(zygote::ZygoteFailure, env,
                           is_system_server ? "system_server" : "zygote",
                           nullptr, _1);

  // Temporarily block SIGCHLD during forks. The SIGCHLD handler might
  // log, which would result in the logging FDs we close being reopened.
  // This would cause failures because the FDs are not allowlisted.
  //
  // Note that the zygote process is single threaded at this point.
  BlockSignal(SIGCHLD, fail_fn);

  // Close any logging related FDs before we start evaluating the list of
  // file descriptors.
  __android_log_close();
  AStatsSocket_close();

  // If this is the first fork for this zygote, create the open FD table,
  // verifying that files are of supported type and allowlisted.  Otherwise (not
  // the first fork), check that the open files have not changed.  Newly open
  // files are not expected, and will be disallowed in the future.  Currently
  // they are allowed if they pass the same checks as in the
  // FileDescriptorTable::Create() above.
  if (gOpenFdTable == nullptr) {
    gOpenFdTable = FileDescriptorTable::Create(fds_to_ignore, fail_fn);
  } else {
    gOpenFdTable->Restat(fds_to_ignore, fail_fn);
  }

  android_fdsan_error_level fdsan_error_level = android_fdsan_get_error_level();

```

## Zygisk hook 和 zygote

除了 jni hook ，zygisk 还设置了一系列 libc 的 hook ，用于更精细地控制模块的加载和卸载。

```cpp
    XHOOK_REGISTER(ANDROID_RUNTIME, fork);
    XHOOK_REGISTER(ANDROID_RUNTIME, unshare);
    // XHOOK_REGISTER(ANDROID_RUNTIME, jniRegisterNativeMethods);
    XHOOK_REGISTER(ANDROID_RUNTIME, selinux_android_setcontext);
    XHOOK_REGISTER_SYM(ANDROID_RUNTIME, "__android_log_close", android_log_close);
```

在 SpecializeCommon 中，执行顺序：

```
unshare(可选)
mount
setresgid
setresuid (此时 cap 会消失)
__android_log_close
selinux_android_setcontext
```

1. unshare

unshare 是分离挂载命名空间。在 Android 11 以前，这是个可选的操作，只有需要 sdcard 的 app 才会 unshare 。Android 11 及之后的版本都是强制执行。

Zygote 本身启动的时候就会 unshare ，并 remount `/` 为 `MS_REC|MS_SLAVE` ，确保在 specialize 中的 mount 操作不会影响全局挂载命名空间。

Zygisk 在这个阶段记录，确认 app 进程是否 unshare mnt ns ，如果 unshare 了则可以对处于 denylist 的 app umount magisk 模块（但是不会强制不可 unshare 的进行 umount）。

> TODO: 此处是我修改的代码，应该改成原来的

```cpp
// Unmount stuffs in the process's private mount namespace
DCL_HOOK_FUNC(int, unshare, int flags) {
    int res = old_unshare(flags);
    if (g_ctx && (flags & CLONE_NEWNS) != 0 && res == 0 &&
        // For some unknown reason, unmounting app_process in SysUI can break.
        // This is reproducible on the official AVD running API 26 and 27.
        // Simply avoid doing any unmounts for SysUI to avoid potential issues.
        g_ctx->process && g_ctx->process != "com.android.systemui"sv) {
        if (g_ctx->flags[DO_REVERT_UNMOUNT]) {
            revert_unmount();
        }
        // Restore errno back to 0
        errno = 0;
    }
    return res;
}
```



