## 起因

最近看到 Weishu 发了一个 KernelSU 移植到旧版非 GKI 内核的视频。

https://www.bilibili.com/video/BV1ge4y1G7Dy/

我大概两年前折腾过内核，当时是为了弄出一个 unix diag 模块，最后用 hack 的手段搞定了（其实就是把内核的 symver 给倒出来，然后 patch 到模块的 symver 上去）。

如今那个模块因为内核版本更新也变得不能用了，原先 hack symvers 的手段虽然能够加载，但是不能正常使用，可能是源码中的结构体变化了导致的。

虽然早有耳闻屑 MIUI 的内核基本上是假开源，代码编译出来完全不能开机，但看到 weishu 那么顺利（？）地在同样是 MIUI 系统的内核里面植入了 KernelSU ，又让我有信心去折腾一番。

## 预备

[&#x5b;内核向&#x5d; 论如何优雅的刷入内核 - AKR社区](https://www.akr-developers.com/d/125)

[关于Image.xx-dtb 和 Image.xx + dtb的区别 - AKR社区](https://www.akr-developers.com/d/482)

[kernel dtb 与 dtbo - AKR社区](https://www.akr-developers.com/d/353)

[Pixel 3 AOSP 自编译内核 adb无法识别设备 - AKR社区](https://www.akr-developers.com/d/529-pixel-3-aosp-adb)

[Pixel3 Aosp自编译内核如何正确的驱动设备正常运行 - AKR社区](https://www.akr-developers.com/d/526-pixel3-aosp)

```
内核有自带了一个模块与内核本体是否匹配的判断，内核模块必须和内核来自于同一次编译才能被正确加载

同时使用这三个提交可以破除这一限制

https://github.com/luk1337/android_kernel_oneplus_sm8250/commit/153318ee16f44c26a8a3d2da6fecdb9f8c266c18

https://github.com/luk1337/android_kernel_oneplus_sm8250/commit/eaed1807d37a3201e1fcf31b3d148f4d3f39e27d

https://github.com/luk1337/android_kernel_oneplus_sm8250/commit/2ac459f2b93908c5418c7afa7643f290107196d9
```

编译内核还要考虑系统原有的模块（`/vendor/lib/modules`）能否加载，看了一下自己编译的和已有模块的 module_layout 之类的基本都不一致，所以可能要合上面三个补丁

不过还是要先编译出一个能开机的内核

## 官方

当前内核(12.5.7)：

```
Linux version 4.19.113-perf-g481f117a1553 (builder@c4-xm-ota-bd016.bj) (clang version 10.0.6 for Android NDK) #1 SMP PREEMPT Mon Oct 25 00:44:13 CST 2021
```

[MiCode/Xiaomi_Kernel_OpenSource at picasso-r-oss](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/tree/picasso-r-oss)

[How to compile kernel standalone · MiCode/Xiaomi_Kernel_OpenSource Wiki](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/wiki/How-to-compile-kernel-standalone)

从 Qualcomm 官网下的 SDLLVM 8.0.6 ，内置的 dtc (export DTC_EXT 去掉) -> 无法开机，fastboot boot 后仍然有连接

boot 镜像是用 magiskboot 制作的，直接把 kernel, dtb, 原 img 的 ramdisk 放到目录， `magiskboot repack <原 img>`

把原版 img 解包然后重打包，md5 是一样的，说明 magiskboot 应该不会有问题。

而直接拿现有的内核或者 stock 内核都可以通过 fastboot boot 引导进入系统（并且 boot 执行完成后和电脑的 fastboot 连接会断开），说明 fastboot boot 也没有问题。

进入系统想从 /sys/fs/pstore 拿点信息，但是似乎只有原先的启动记录

官方编译用的 10.0.6 ，但是只能找到 8.0.6 的下载，新的版本似乎要用专用的 Qualcomm Package Manager 下载，还要注册一大堆东西，并且是 GUI 界面的，wsl 不太好搞。

于是在这里找到了 SDLLVM 10.0.9 版本的第三方发布：

[xiangfeidexiaohuo/Snapdragon-LLVM: 高通Clang](https://github.com/xiangfeidexiaohuo/Snapdragon-LLVM)

[kernel dtb 与 dtbo - AKR社区](https://www.akr-developers.com/d/353/5)

另外这里说 dtc 可能有问题，必须要用 Android 魔改的，于是去 AOSP 下了一份源码

[refs/heads/android11-release - platform/external/dtc - Git at Google](https://android.googlesource.com/platform/external/dtc/+/refs/heads/android11-release)

[dtc编译错误 · Issue #1115 · MiCode/Xiaomi_Kernel_OpenSource](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/issues/1115)

稍微改了一下可以 make 编译了，DTC_EXT 指定成编译后的 dtc 程序位置。

现在这样 10.0.9 + Android 11 dtc 还是没法开机

顺便贴上编译脚本：

```sh
export ARCH=arm64
export SUBARCH=arm64
export DTC_EXT=${PWD}/dtc/dtc
export CROSS_COMPILE=${PWD}/toolchain/bin/aarch64-linux-android-
REAL_CC='REAL_CC=${PWD}/toolchains/Snapdragon-LLVM-10.0.9/bin/clang'
CLANG_T='CLANG_TRIPLE=aarch64-linux-gnu-'

export LOCALVERSION=-g481f117a1553

make O=out $REAL_CC $CLANG_T picasso_user_defconfig
make -j$(nproc) O=out $REAL_CC $CLANG_T 2>&1 | tee kernel.log
```

## 第三方

[使用小米开源代码编译内核卡第一屏 - AKR社区](https://www.akr-developers.com/d/577)

看来官方内核源码靠不住，而自己又没那个本事弄出能开机的内核，还是找找第三方的碰碰运气吧

[Redmi K30 5G ROMs, Kernels, Recoveries, & Other D | XDA Forums](https://forum.xda-developers.com/f/redmi-k30-5g-roms-kernels-recoveries-other-d.9887/)

在 XDA 找了一下，发现这么些内核：

[Forenche/kernel_xiaomi_picasso: Moved to https://github.com/stormbreaker-project/kernel_xiaomi_picasso](https://github.com/Forenche/kernel_xiaomi_picasso)

[stormbreaker-project/kernel_xiaomi_picasso](https://github.com/stormbreaker-project/kernel_xiaomi_picasso)

前两个很久没维护了，并且 release 都写着这种警告：

> **NOTE: DO NOT FLASH ON MIUI**

下面这个最近才维护过，不过仓库也在近期 archive 了。

[EndCredits/android_kernel_xiaomi_sm7250: CAF rebased kernel based on MiCode/Xiaomi Kernel OpenSouce for Redmi K30 5G ( picasso ) | Thanks to @Hadenix @LynnrinChan](https://github.com/EndCredits/android_kernel_xiaomi_sm7250)

这个内核的小版本号甚至和官方的不一致

第三方内核的主要问题就是魔改太多，也不知道能不能用在官方的系统上。

……

## 转机

抱着试试的心态把自己编译的内核和原来的 dtb 放在一起打包，没想到可以开机

当然开机后 WIFI 不能正常工作，查看 `/proc/modules` 也是空的，已经预料到了。

于是手动把之前提到的三个提交合并进去，但启动后仍然无法加载模块，报错如下：

```
[    4.045536] init: starting service 'exec 2 (/vendor/bin/modprobe -a -d /vendor/lib/modules audio_q6_pdr audio_q6_notifier audio_snd_event audio_apr audio_adsp_loader audio_q6 audio_native audio_usf audio_pinctrl_lpi audio_swr audio_platform audio_hdmi audio_stub audio_wcd_core audio_tfa98xx audio_cs35l41 audio_wsa881x audio_wsa883x audio_bolero_cdc audio_wsa_macro audio_va_macro audio_rx_macro audio_tx_macro audio_wcd938x audio_wcd938x_slave audio_wcd937x audio_wcd937x_slave audio_machine_lito)'...
[    4.047025] init: SVC_EXEC service 'exec 2 (/vendor/bin/modprobe -a -d /vendor/lib/modules audio_q6_pdr audio_q6_notifier audio_snd_event audio_apr audio_adsp_loader audio_q6 audio_native audio_usf audio_pinctrl_lpi audio_swr audio_platform audio_hdmi audio_stub audio_wcd_core audio_tfa98xx audio_cs35l41 audio_wsa881x audio_wsa883x audio_bolero_cdc audio_wsa_macro audio_va_macro audio_rx_macro audio_tx_macro audio_wcd938x audio_wcd938x_slave audio_wcd937x audio_wcd937x_slave audio_machine_lito)' pid 553 (uid 0 gid 0+0 context u:r:vendor_modprobe:s0) started; waiting...
[    4.073236] q6_pdr_dlkm: version magic '4.19.113-perf-g481f117a1553 SMP preempt mod_unload modversions aarch64' should be '4.19.113-perf SMP preempt mod_unload modversions aarch64'
[    4.073452] q6_pdr_dlkm: version magic '4.19.113-perf-g481f117a1553 SMP preempt mod_unload modversions aarch64' should be '4.19.113-perf SMP preempt mod_unload modversions aarch64'
```

看上去还要 version magic 对应，我们的版本号后面没有那个看起来像是 commit hash 的东西。

参考 weishu 视频 `07:10` 的构建代码，似乎是加一个环境变量 `LOCALVERSION`

https://www.bilibili.com/video/BV1ge4y1G7Dy?t=430.9

```
LOCALVERSION=-g481f117a1553 ./build.sh
```

编译后可以看到已经有我们的版本号了。

```
strings out/vmlinux | grep 'perf-'
Linux version 4.19.113-perf-g481f117a1553 (five_ec1cff@LAPTOP-H42AMUM5) (clang version 10.0.9) #5 SMP PREEMPT Wed Jan 11 22:27:31 CST 2023
4.19.113-perf-g481f117a1553 SMP preempt mod_unload modversions aarch64
```

开机，速度似乎很慢，不过启动已经有模块能加载了。

```
rmnet_perf 36864 0 - Live 0x0000000000000000 (O)
rmnet_shs 114688 0 - Live 0x0000000000000000 (O)
wlan 6918144 0 - Live 0x0000000000000000 (O)
exfat 139264 0 - Live 0x0000000000000000 (O)
machine_dlkm 180224 0 - Live 0x0000000000000000 (O)
wcd937x_slave_dlkm 16384 0 - Live 0x0000000000000000 (O)
wcd937x_dlkm 98304 1 machine_dlkm, Live 0x0000000000000000 (O)
wcd938x_slave_dlkm 16384 0 - Live 0x0000000000000000 (O)
wcd938x_dlkm 110592 2 machine_dlkm, Live 0x0000000000000000 (O)
wcd9xxx_dlkm 57344 20 wcd937x_dlkm,wcd938x_dlkm, Live 0x0000000000000000 (O)
mbhc_dlkm 81920 2 wcd937x_dlkm,wcd938x_dlkm, Live 0x0000000000000000 (O)
tx_macro_dlkm 114688 1 mbhc_dlkm, Live 0x0000000000000000 (O)
rx_macro_dlkm 106496 0 - Live 0x0000000000000000 (O)
va_macro_dlkm 106496 0 - Live 0x0000000000000000 (O)
wsa_macro_dlkm 69632 1 machine_dlkm, Live 0x0000000000000000 (O)
swr_ctrl_dlkm 73728 4 tx_macro_dlkm,rx_macro_dlkm,va_macro_dlkm,wsa_macro_dlkm, Live 0x0000000000000000 (O)
bolero_cdc_dlkm 57344 6 machine_dlkm,tx_macro_dlkm,rx_macro_dlkm,va_macro_dlkm,wsa_macro_dlkm, Live 0x0000000000000000 (O)
wsa883x_dlkm 57344 1 machine_dlkm, Live 0x0000000000000000 (O)
wsa881x_dlkm 57344 1 machine_dlkm, Live 0x0000000000000000 (O)
cs35l41_dlkm 147456 0 - Live 0x0000000000000000 (O)
tfa98xx_dlkm 180224 1 - Live 0x0000000000000000 (O)
wcd_core_dlkm 32768 10 machine_dlkm,wcd937x_dlkm,wcd938x_dlkm,mbhc_dlkm,tx_macro_dlkm,rx_macro_dlkm,va_macro_dlkm,wsa_macro_dlkm,wsa883x_dlkm,wsa881x_dlkm, Live 0x0000000000000000 (O)
stub_dlkm 16384 1 - Live 0x0000000000000000 (O)
hdmi_dlkm 24576 1 - Live 0x0000000000000000 (O)
swr_dlkm 24576 7 wcd937x_slave_dlkm,wcd937x_dlkm,wcd938x_slave_dlkm,wcd938x_dlkm,swr_ctrl_dlkm,wsa883x_dlkm,wsa881x_dlkm, Live 0x0000000000000000 (O)
pinctrl_lpi_dlkm 24576 7 - Live 0x0000000000000000 (O)
usf_dlkm 57344 0 - Live 0x0000000000000000 (O)
native_dlkm 200704 0 - Live 0x0000000000000000 (O)
platform_dlkm 2965504 48 native_dlkm, Live 0x0000000000000000 (O)
q6_dlkm 1134592 12 machine_dlkm,wcd9xxx_dlkm,va_macro_dlkm,swr_ctrl_dlkm,bolero_cdc_dlkm,tfa98xx_dlkm,pinctrl_lpi_dlkm,usf_dlkm,native_dlkm,platform_dlkm, Live 0x0000000000000000 (O)
adsp_loader_dlkm 16384 0 - Live 0x0000000000000000 (O)
apr_dlkm 233472 4 usf_dlkm,platform_dlkm,q6_dlkm,adsp_loader_dlkm, Live 0x0000000000000000 (O)
snd_event_dlkm 16384 5 machine_dlkm,bolero_cdc_dlkm,pinctrl_lpi_dlkm,q6_dlkm,apr_dlkm, Live 0x0000000000000000 (O)
q6_notifier_dlkm 16384 2 pinctrl_lpi_dlkm,apr_dlkm, Live 0x0000000000000000 (O)
q6_pdr_dlkm 16384 1 q6_notifier_dlkm, Live 0x0000000000000000 (O)
```

因为之前没记录原来内核的 /proc/modules ，因此暂时无法比较。不过理论上来说应该都能加载。

看一看 proc version ，确实是自己编译的。

```
# cat /proc/version
Linux version 4.19.113-perf-g481f117a1553 (five_ec1cff@xxxxxx) (clang version 10.0.9) #5 SMP PREEMPT Wed Jan 11 22:27:31 CST 2023
```

WLAN，声音之类的都没问题，其他担心的项目如相机，指纹、存储、TEE 等也都正常，就是速度感觉慢了一些，也许是心理作用，先这么用一晚上看看效果。

新内核运行了一晚上，用起来和以前没什么两样，于是我决定加一些东西

defconfig 里面加上：

```
CONFIG_KPROBES=y
CONFIG_UNIX_DIAG=y
```

同时创建了 git 仓库，注意到这样会自动加上一个 hash 导致模块的 vermagic 又不对了，因此还要加一项：

```
CONFIG_LOCALVERSION_AUTO=n
```

编译好了，继续 fastboot boot 启动。然而这个内核并不像原先那么顺利，开机后无法进入系统。一开始是黑屏，怎么按电源键都没响应，后来才出现了 MIUI 图标，说明屏幕没有问题，但仍无法进入系统。

> 值得注意的是此时 scrcpy 能打开，但也是黑屏，这似乎只能证明 system server 和 SurfaceFlinger 正常工作

好在重启到原先的内核还是能正常使用的，重启前记录了 dmesg 和 logcat 。

先看日志，似乎看不出什么问题，于是看一看 tombstone

观察到那次启动后只有一个 tombstone ，是一个 `vendor.qti.hardware.display.composer-service` 进程，看起来与显示相关：

```
Build fingerprint: 'Redmi/picasso/picasso:11/RKQ1.200826.002/V12.5.7.0.RGICNXM:user/release-keys'
Revision: '0'
ABI: 'arm64'
Timestamp: 2023-01-12 20:08:45+0800
pid: 934, tid: 2478, name: LTM_THREAD  >>> /vendor/bin/hw/vendor.qti.hardware.display.composer-service <<<
uid: 1000
signal 6 (SIGABRT), code -1 (SI_QUEUE), fault addr --------
Abort message: 'Attempted to retrieve value from failed HIDL call: Status(EX_TRANSACTION_FAILED): 'DEAD_OBJECT: ''
    x0  0000000000000000  x1  00000000000009ae  x2  0000000000000006  x3  000000792c7574e0
    x4  0000000000000000  x5  0000000000000000  x6  0000000000000000  x7  000000000001601c
    x8  00000000000000f0  x9  14a87dbf0b025cb5  x10 0000000000000000  x11 ffffffc0ffffffdf
    x12 0000000000000001  x13 0000000000000062  x14 b40000793701b760  x15 0000000000080000
    x16 00000079bb29c948  x17 00000079bb27b390  x18 000000792bfa4000  x19 00000000000003a6
    x20 00000000000009ae  x21 00000000ffffffff  x22 000000792c758000  x23 0000000000000000
    x24 000000792c758000  x25 000000792d9ed970  x26 000000792c758000  x27 000000000000aa40
    x28 000000792d9ece28  x29 000000792c757560
    lr  00000079bb22eaa0  sp  000000792c7574c0  pc  00000079bb22eacc  pst 0000000000000000

backtrace:
      #00 pc 0000000000089acc  /apex/com.android.runtime/lib64/bionic/libc.so (abort+164) (BuildId: a790cdbd8e44ea8a90802da343cb82ce)
      #01 pc 00000000000062b0  /system/lib64/liblog.so (__android_log_default_aborter+12) (BuildId: 53286795d97fe56c1e70dca740895c84)
      #02 pc 0000000000012fa4  /apex/com.android.vndk.v30/lib64/libbase.so (android::base::LogMessage::~LogMessage()+320) (BuildId: 80f255c2fa17369bce8002e5e367ad65)
      #03 pc 0000000000045158  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::details::return_status::onValueRetrieval() const+236) (BuildId: 441744a3e8eac97619ec117a3898fc51)
      #04 pc 000000000008c1ac  /vendor/lib64/libdpps.so (AlsNative::AlsEnable()+368) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #05 pc 00000000000b4214  /vendor/lib64/libdpps.so (DppsAlsSubject::Register(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, DppsNotifyInterface<int>*)+344) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #06 pc 00000000000ec034  /vendor/lib64/libdpps.so (ltmfeature::LtmOff::PostSetParam(LtmSwParams, LtmPayload const&, std::__1::vector<std::__1::function<int ()>, std::__1::allocator<std::__1::function<int ()> > >*, bool*)+768) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #07 pc 00000000000eb708  /vendor/lib64/libdpps.so (ltmfeature::LtmOff::SetParameter(LtmSwParams, LtmPayload const&)+396) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #08 pc 00000000000e3780  /vendor/lib64/libdpps.so (ltmfeature::LtmFeatureImp::SetParameter(LtmSwParams, LtmPayload const&)+116) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #09 pc 00000000000d8ad8  /vendor/lib64/libdpps.so (DppsLtmClientImp::ProcessOnCommand(LTM::LtmCommandPayload<LTM::LtmCommand> const&)+184) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #10 pc 00000000000d792c  /vendor/lib64/libdpps.so (DppsLtmClientImp::HandleQueueMsg(DppsEventRespPayload const&)+240) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #11 pc 00000000000e00fc  /vendor/lib64/libdpps.so (LtmEventImp::HandleQueueMsg()+104) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #12 pc 00000000000e15dc  /vendor/lib64/libdpps.so (LtmEventImp::ProcessNextEvent()+440) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #13 pc 00000000000d6858  /vendor/lib64/libdpps.so (DppsLtmClientImp::Process()+468) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #14 pc 00000000000d6644  /vendor/lib64/libdpps.so (DppsLtmClientImp::thread_handler(DppsLtmClientImp*) (.cfi)+112) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #15 pc 00000000000db01c  /vendor/lib64/libdpps.so (void* std::__1::__thread_proxy<std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct> >, void (*)(DppsLtmClientImp*), DppsLtmClientImp*> >(void*) (.cfi)+60) (BuildId: 88e1d2eb7acbab67389d68e0866af277)
      #16 pc 00000000000eb868  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+64) (BuildId: a790cdbd8e44ea8a90802da343cb82ce)
      #17 pc 000000000008ba88  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64) (BuildId: a790cdbd8e44ea8a90802da343cb82ce)
```

于是把 `CONFIG_KPROBES=y` 去掉，只保留 `CONFIG_UNIX_DIAG=y`

unix diag 是起作用了（主要就是用来获取 unix socket 的 peer 信息的）。不过观察到开机后软重启了数次，但之后就稳定下来，看日志也没看出有什么问题，tombstone 也没有产生相关的信息。

看来 KPROBES 还是有一些问题

未完待续……
