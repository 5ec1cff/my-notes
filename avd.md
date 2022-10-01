# AVD 记录

## llkd

2022.08.16

最近在用 avd (Android 11 x86-64)上安装 lsposed 调试 xposed 模块，但是总是遇到 lspd 被杀的情况：

```log
2022-08-16 22:07:58.753 2501-2646/system_process E/LSPosed Bridge: service is dead
```

进一步查明原因，在 logcat 中过滤 lspd 的 pid ：

```log
08-16 13:57:36.990  2501  3238 I ActivityManager: Force stopping com.android.shell appid=2000 user=0: from pid 2366
08-16 13:57:36.990  2501  3238 I ActivityManager: Killing 3831:com.android.shell/2000 (adj 935): stop com.android.shell due to from pid 2366
08-16 14:07:58.728   515   515 W livelock: Z 600.000s 2366->2497->2497 [idmap2] [kill]
08-16 14:07:58.728   515   515 I livelock: Killing 'lspd' (2366) to check forward scheduling progress in Z state for '[idmap2]' (2497)
```

发现被一个叫做 livelock 的东西给杀了。

[Android Live-LocK 守护程序 (llkd)](https://source.android.com/devices/architecture/kernel/llkd#coverage)

根据文档所说，有一个设置免杀（黑名单）的属性 `ro.llk.blacklist.process` ，于是尝试 `resetprop ro.llk.blacklist.process lspd`

观察到 livelock 的日志开头有输出一些 ro 属性，怀疑是启动时加载一次，因此重启一下（首先确认服务名为 `llkd-1`）：

```sh
generic_x86_64:/data/local/tmp # getprop | grep llkd
[init.svc.llkd-1]: [running]
[init.svc_debug_pid.llkd-1]: [515]
generic_x86_64:/data/local/tmp # stop llkd-1
generic_x86_64:/data/local/tmp # start llkd-1
generic_x86_64:/data/local/tmp # logcat -s livelock:*
--------- beginning of kernel
--------- beginning of system
--------- beginning of main
--------- beginning of crash
08-16 14:07:58.728   515   515 W livelock: Z 600.000s 2366->2497->2497 [idmap2] [kill]
08-16 14:07:58.728   515   515 I livelock: Killing 'lspd' (2366) to check forward scheduling progress in Z state for '[idmap2]' (2497)
08-16 14:17:50.948  6087  6087 I livelock: started
08-16 14:17:50.949  6087  6087 W livelock: [khungtaskd] not configurable
08-16 14:17:50.950  6087  6087 I livelock: ro.config.low_ram=false
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.sysrq_t=true
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.enable=true
08-16 14:17:50.950  6087  6087 I livelock: ro.khungtask.enable=true
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.mlockall=true
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.killtest=true
08-16 14:17:50.950  6087  6087 I livelock: ro.khungtask.timeout=720s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.timeout_ms=600.000s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.D.timeout_ms=600.000s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.Z.timeout_ms=600.000s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.stack.timeout_ms=600.000s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.check_ms=120.000s
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.stack=wait_on_page_bit_killable,bit_wait_io,__get_user_pages,cma_alloc
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.ignorelist.process.stack=llkd,lmkd.llkd,apexd,ueventd,keystore,init
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.ignorelist.process=[watchdog/3],[watchdog/2],[watchdog/1],[watchdogd/0],2,6087,[watchdogd],llkd,lmkd,[khungtaskd],[kthreadd],init,watchdogd,1,0
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.ignorelist.parent=[kthreadd],2,0,adbd&[setsid]
08-16 14:17:50.950  6087  6087 I livelock: ro.llk.ignorelist.uid=
```

发现属性并没有发生任何变化，而且属性的名字 `ro.llk.ignorelist.process` 似乎也与文档描述不符，最后，在 props 根本找不到文档所说的默认值：

```sh
130|generic_x86_64:/data/local/tmp # getprop ro.llk.ignorelist.process

```

既然没法配置黑名单，干脆直接 `stop llkd-1` 好了。

## AVD 安装 Magisk

在 Android 11 的 AVD 上安装 Magisk 一直以来令人头疼，因为这个版本具有奇怪的「双层」 ramdisk 镜像，而 Magisk 的 `avd_patch.sh` 似乎并没有处理这一特性。

此前一直使用 [MagiskOnEmulator](https://github.com/shakalaca/MagiskOnEmulator) 和官方发布的 magisk 来修补 ramdisk ，它确实可以正确处理 Android 11 的 ramdisk 。但随着版本更新，到了 Magisk 24 就会出现一些诡异问题，比如在 x86-64 的 avd 上它只会复制 x64 的 so ，导致 32 位 zygisk 无法启动，尝试修复未果。

既然这样，不如试试另一种方式 —— [`avd_magisk.sh`](https://github.com/topjohnwu/Magisk/blob/c2f96975cef433a67dc036fdd6ea3736f69d6763/scripts/avd_magisk.sh)，也就是利用 avd 本身具有的 root 权限模拟 magisk 运行。考虑到我的需求只是用 zygisk 、lsposed 调试模块，其实完全没必要修补 ramdisk 。

正确的用法应该是用 magisk 的构建脚本从头构建一遍，然后用 `build.py emulator` 的，不过我不想那么麻烦，直接用官方编译好的 apk 凑合用吧。

首先在一个目录准备好 `scripts/avd_magisk.sh` 和 magisk 的 apk (这里用 debug 版 `app-debug.apk`)，然后写一个脚本（此处在 windows 下是 `avd.bat`）

```bat
adb push busybox app-debug.apk avd_magisk.sh /data/local/tmp
adb shell sh /data/local/tmp/avd_magisk.sh
```

打开 avd ，执行 `avd.bat` 。完成后 avd 会软重启，并且直接启动 magisk 的 post-fs-data 和 service 。此时就可以愉快地用 magisk 了。如果要开启 zygisk ，应该在 app 打开开关后，再次执行 `avd.bat` 。安装模块后的重启也应该执行这个脚本。如果只是需要 zygote 重启的话，也可以在 adb shell 中 `stop;start` 。

用这样的方法调试了几次模块，总的来说还是非常方便的，也没有像 patch 方法要考虑 magisk 升级的麻烦。

## 冷启动

调试的时候经常搞坏系统。比如：

1. 之前调试 sui 的时候把 adbd 杀了，却没有被 init 自动拉起  
2. magisk 模块导致 zygote bootloop ，**stop 之后在这个阶段安装模块**会导致诡异的情况——执行任何程序都会提示 not found （包括 sh），至今没找明白原因。  

这种情况下就需要彻底关闭系统，然而 AVD 的 UI 甚至没有强制重启的选项，并且由于 AVD 有快照功能，直接关闭的话，下次启动又恢复上次坏掉的系统。

虽然确实有冷启动的方法，不过我总是记不住，现在记在下面：

启动的时候添加这个参数：

```
    -no-snapshot                                                        perform a full boot and do not auto-save, but qemu vmload and vmsave operate on snapstorage
```
