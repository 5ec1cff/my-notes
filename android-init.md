# init

已经很久没有研究过 init 了，发现对一些细节不太了解，复习一下。

## rc 语法

https://android.googlesource.com/platform/system/core/+/main/init/README.md

https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/README.md;drc=5c4217cf6eb0686fcab73cd55d216859549b6250

要点

1. `on <trigger> ...` 下面是一系列 command ，顺序执行(?)  
2. `service ...` 定义服务，服务可以有 class ，`start` 和 `class_start` 启动一个或一类服务。  
3. triggers 排队执行，可以在 command 中触发某个 trigger ，即添加到队列  
4. `exec` command 阻塞等待被执行的程序退出  

## rc 解析

[LoadBootScripts](https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/init.cpp;l=337;drc=e274a5ef062925c35b042726f9dff8c0fd6cabd6)

actions 的顺序和 rc 文件顺序的关系（TODO）

## boot 执行

init 里面执行事件循环之前已经预先 queue 了一系列 trigger ，在 hw/init.rc 则会触发更多 trigger

[init.cpp](https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/init.cpp;l=1088;drc=55ef3d6104032dcaa773e486405cb9f8978fd5ce)

[hw/init.rc](https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.rc;l=542;drc=b5ce7aa444f7abfd62649c329f76c2b9e728cf92)

[nonencrypted](https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/builtins.cpp;l=581;drc=f06e218e826039bf5113de5a773803fccfcac561)

关键流程

```
(init)
trigger early-init
trigger init
trigger late-init
loop {
 on early-init
 on init
 on late-init
  trigger late-fs
  trigger post-fs-data
 on late-fs # 由厂商实现，例如 /vendor/etc/init/hw/init.target.rc
  mount_all --late 
   (do_mount_all -> queue_fs_event)
    trigger nonencrypt
 on post-fs-data
 on nonencrypt
  class_start main
}
```

post-fs-data 之后 /data 已经被挂载

start main 就会启动 zygote 和一系列服务

## 主流 root 框架对 init 的注入

post-fs-data.sh 在 post-fs-data trigger 执行，并且是阻塞的

service.sh 在 nonencrypted trigger 执行，按理来说此时 zygote 已经启动

https://topjohnwu.github.io/Magisk/guides.html#boot-scripts

### magisk

https://github.com/topjohnwu/Magisk/blob/cf43c562185cb359063ee36bfd879310b43dc949/native/src/init/rootdir.cpp#L16

选择的是 hw/init.rc 的尾部

```
on post-fs-data
    start logd
    exec %2$s 0 0 -- %1$s/magisk --post-fs-data

on property:vold.decrypt=trigger_restart_framework
    exec %2$s 0 0 -- %1$s/magisk --service

on nonencrypted
    exec %2$s 0 0 -- %1$s/magisk --service

on property:sys.boot_completed=1
    exec %2$s 0 0 -- %1$s/magisk --boot-complete

on property:init.svc.zygote=restarting
    exec %2$s 0 0 -- %1$s/magisk --zygote-restart

on property:init.svc.zygote=stopped
    exec %2$s 0 0 -- %1$s/magisk --zygote-restart
```

### ksu

注入到 /system/etc/init/atrace.rc 的头部，这么做的原因也许是 kprobes 注入到文件尾部太麻烦，于是选择了 a 开头的文件的头部，确保跟在 init.rc 后。

https://github.com/tiann/KernelSU/blob/ce892bc439272a0ec41498a5b46df143ca11f68b/kernel/ksud.c#L21

```
static const char KERNEL_SU_RC[] =
	"\n"

	"on post-fs-data\n"
	"    start logd\n"
	// We should wait for the post-fs-data finish
	"    exec u:r:su:s0 root -- " KSUD_PATH " post-fs-data\n"
	"\n"

	"on nonencrypted\n"
	"    exec u:r:su:s0 root -- " KSUD_PATH " services\n"
	"\n"

	"on property:vold.decrypt=trigger_restart_framework\n"
	"    exec u:r:su:s0 root -- " KSUD_PATH " services\n"
	"\n"

	"on property:sys.boot_completed=1\n"
	"    exec u:r:su:s0 root -- " KSUD_PATH " boot-completed\n"
	"\n"

	"\n";
```
