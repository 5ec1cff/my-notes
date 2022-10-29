# 检测 zygisk 新思路

在 Zygisk 时代，尽管有 Shamiko 的存在，却还是难以抵挡检测，相比 riru 时代而言似乎是一种倒退。就在最近的研究中，LSP 开发者发现了 zygisk 仍然存在 [环境变量泄露](https://t.me/c/1188689189/993251) 而导致被检测的问题，下面我们来研究一下。

> 以下测试基于 Magisk 25.2 Release

## 无效的 sanitize  

Zygisk 自身虽然难以隐藏，但起码下了一些功夫去清理对 env 的修改，源码在此：[sanitize_environ](https://github.com/topjohnwu/Magisk/blob/b496923cbb0de7312de837d69d0a5faf342982e2/native/src/zygisk/entry.cpp#L22)

```cpp
// Make sure /proc/self/environ is sanitized
// Filter env and reset MM_ENV_END
static void sanitize_environ() {
    char *cur = environ[0];

    for (int i = 0; environ[i]; ++i) {
        // Copy all env onto the original stack
        size_t len = strlen(environ[i]);
        memmove(cur, environ[i], len + 1);
        environ[i] = cur;
        cur += len + 1;
    }

    prctl(PR_SET_MM, PR_SET_MM_ENV_END, cur, 0, 0);
}
```

分析一下：这个函数把 env 内存中可能的空洞都填补了，并且用 prctl 设置环境变量的结尾地址

如此一来，搜索 environ ，调用 getenv ，或者查找 `/proc/self/environ` 似乎都找不到被 Magisk 修改的蛛丝马迹了：

```sh
# cat /proc/$(pidof zygote64)/environ | tr '\000' '\n'
PATH=/product/bin:/apex/com.android.runtime/bin:/apex/com.android.art/bin:/system_ext/bin:/system/bin:/system/xbin:/odm/bin:/vendor/bin:/vendor/xbin
ANDROID_BOOTLOGO=1
ANDROID_ROOT=/system
ANDROID_ASSETS=/system/app
ANDROID_DATA=/data
ANDROID_STORAGE=/storage
ANDROID_ART_ROOT=/apex/com.android.art
ANDROID_I18N_ROOT=/apex/com.android.i18n
ANDROID_TZDATA_ROOT=/apex/com.android.tzdata
EXTERNAL_STORAGE=/sdcard
ASEC_MOUNTPOINT=/mnt/asec
BOOTCLASSPATH=/apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:/apex/com.android.art/javalib/core-icu4j.jar:/apex/com.android.art/javalib/okhttp.jar:/apex/com.android.art/javalib/bouncycastle.jar:/apex/com.android.art/javalib/apache-xml.jar:/system/framework/framework.jar:/system/framework/miuisdk@boot.jar:/system/framework/miuisystemsdk@boot.jar:/system/framework/ext.jar:/system/framework/telephony-common.jar:/system/framework/voip-common.jar:/system/framework/ims-common.jar:/system/framework/framework-atb-backward-compatibility.jar:/system/framework/tcmiface.jar:/system/framework/telephony-ext.jar:/system/framework/qcom.fmradio.jar:/system/framework/com.nxp.nfc.nq.jar:/system/framework/QPerformance.jar:/system/framework/UxPerformance.jar:/system/framework/WfdCommon.jar:/apex/com.android.conscrypt/javalib/conscrypt.jar:/apex/com.android.media/javalib/updatable-media.jar:/apex/com.android.mediaprovider/javalib/framework-mediaprovider.jar:/apex/com.android.os.statsd/javalib/framework-statsd.jar:/apex/com.android.permission/javalib/framework-permission.jar:/apex/com.android.sdkext/javalib/framework-sdkextensions.jar:/apex/com.android.wifi/javalib/framework-wifi.jar:/apex/com.android.tethering/javalib/framework-tethering.jar
DEX2OATBOOTCLASSPATH=/apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:/apex/com.android.art/javalib/core-icu4j.jar:/apex/com.android.art/javalib/okhttp.jar:/apex/com.android.art/javalib/bouncycastle.jar:/apex/com.android.art/javalib/apache-xml.jar:/system/framework/framework.jar:/system/framework/miuisdk@boot.jar:/system/framework/miuisystemsdk@boot.jar:/system/framework/ext.jar:/system/framework/telephony-common.jar:/system/framework/voip-common.jar:/system/framework/ims-common.jar:/system/framework/framework-atb-backward-compatibility.jar:/system/framework/tcmiface.jar:/system/framework/telephony-ext.jar:/system/framework/qcom.fmradio.jar:/system/framework/com.nxp.nfc.nq.jar:/system/framework/QPerformance.jar:/system/framework/UxPerformance.jar:/system/framework/WfdCommon.jar
SYSTEMSERVERCLASSPATH=/system/framework/com.android.location.provider.jar:/system/framework/services.jar:/system/framework/ethernet-service.jar:/apex/com.android.permission/javalib/service-permission.jar:/apex/com.android.wifi/javalib/service-wifi.jar:/apex/com.android.ipsec/javalib/android.net.ipsec.ike.jar
DOWNLOAD_CACHE=/data/cache
ANDROID_SOCKET_zygote=18
ANDROID_SOCKET_usap_pool_primary=23
```

LD_PRELOAD, MAGISK_INJ_1 这些都不存在，看起来没问题，真的是这样吗？

我们知道 execve 传入内核的 env 的字符串一般都放在新进程的 stack 上，那么我们 dump 一下（感谢 Mufanc 提供的脚本）：

```sh
# cat /sdcard/Documents/codes/stack.sh
LENGTH=$((0x2000))
ADDRESS=$((0x$(cat /proc/$1/maps | awk '{if ($6=="[stack]") print $1}' | awk -F- '{print $2}') - $LENGTH))
dd if=/proc/$1/mem skip=$ADDRESS bs=1c count=$LENGTH 2>/dev/null 2>/dev/null | hexdump -C
# sh /sdcard/Documents/codes/stack.sh $(pidof zygote64)
00001f70  52 4f 49 44 5f 53 4f 43  4b 45 54 5f 7a 79 67 6f  |ROID_SOCKET_zygo|
00001f80  74 65 3d 31 38 00 41 4e  44 52 4f 49 44 5f 53 4f  |te=18.ANDROID_SO|
00001f90  43 4b 45 54 5f 75 73 61  70 5f 70 6f 6f 6c 5f 70  |CKET_usap_pool_p|
00001fa0  72 69 6d 61 72 79 3d 32  33 00 4c 44 5f 50 52 45  |rimary=23.LD_PRE|
00001fb0  4c 4f 41 44 3d 2f 73 79  73 74 65 6d 2f 62 69 6e  |LOAD=/system/bin|
00001fc0  2f 61 70 70 77 69 64 67  65 74 00 4d 41 47 49 53  |/appwidget.MAGIS|
00001fd0  4b 5f 49 4e 4a 5f 31 3d  31 00 4d 41 47 49 53 4b  |K_INJ_1=1.MAGISK|
00001fe0  54 4d 50 3d 2f 64 65 76  2f 48 4c 37 69 00 2f 64  |TMP=/dev/HL7i./d|
00001ff0  65 76 2f 66 64 2f 34 00  00 00 00 00 00 00 00 00  |ev/fd/4.........|
00002000
```

怎么会是呢？本来该被 unset 还 sanitize 的环境变量老老实实地呆在那儿，最后甚至还有个 `/dev/fd/4` ，似乎暗示这个 zygote 是被 `fexecve` 出来的。

我们来分析一下 [bionic 中 unsetenv 的代码](https://android.googlesource.com/platform/bionic/+/1031e0da388ad42b8c33dedb8e6243b9fdab9b15/libc/upstream-openbsd/lib/libc/stdlib/setenv.c)：

```cpp
// bionic/libc/upstream-openbsd/lib/libc/stdlib/setenv.c
int
unsetenv(const char *name)
{
	char **P;
	const char *np;
	int offset = 0;

	if (!name || !*name) {
		errno = EINVAL;
		return (-1);
	}
	for (np = name; *np && *np != '='; ++np)
		;
	if (*np) {
		errno = EINVAL;
		return (-1);			/* has `=' in name */
	}

	/* could be set multiple times */
	while (__findenv(name, (int)(np - name), &offset)) {
		for (P = &environ[offset];; ++P)
			if (!(*P = *(P + 1)))
				break;
	}
	return (0);
}
DEF_WEAK(unsetenv);
```

看起来删除一个 env 仅仅只是删除了数组中的地址，并未把地址指向的内容一并填 0 ，而 magisk 修改的 env 显然在原有的 env 之后，因此也不会被 sanitize 的移动所填充。

解决方法是主动将其填 0 ，在随后发布的 shamiko 就解决了这一点，此时栈内存已经不存在 magisk 修改的 env 了。

不过这样似乎还没有完全「隐藏」，考虑 env 在 stack 中的分布，也许我们还要把 env 整体向下移动到合适的位置，才像是「无修改」。

## 内核 execve 设置程序 stack  

[common/fs/exec.c](https://android.googlesource.com/kernel/common/+/7d5fc93e8cc467ca63af9d75bdd5973958f5d5c8/fs/exec.c)

```
common/fs/exec.c
-> do_execveat_common
-> bprm_stack_limits
```

```c
static int do_execveat_common(int fd, struct filename *filename,
			      struct user_arg_ptr argv,
			      struct user_arg_ptr envp,
			      int flags)
{
    // ...
	retval = bprm_stack_limits(bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_string_kernel(bprm->filename, bprm);
	if (retval < 0)
		goto out_free;
	bprm->exec = bprm->p;

	retval = copy_strings(bprm->envc, envp, bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_strings(bprm->argc, argv, bprm);
	if (retval < 0)
		goto out_free;

	/*
	 * When argv is empty, add an empty string ("") as argv[0] to
	 * ensure confused userspace programs that start processing
	 * from argv[1] won't end up walking envp. See also
	 * bprm_stack_limits().
	 */
	if (bprm->argc == 0) {
		retval = copy_string_kernel("", bprm);
		if (retval < 0)
			goto out_free;
		bprm->argc = 1;
	}
    // ...
}
```

从低地址到高地址依次：

```
argv
envp
filename
(8 bytes zero)
```

filename 是 execve 的可执行文件路径参数，并非 `argv[0]`，对于 execveat ，这个路径是 `/dev/fd/xxx` 。尽管看上去我们传入了 `/system/bin/app_process` 的 argv ，但实际上 fexecve 的痕迹还是存在的，检测该字符串即可发现异常。因此，为了使隐藏更保险，zygisk 或许该在启动 zygote 时 umount 之前挂载的 app_process ，再通过这个路径 execve 原程序？

## `/proc/self/attr/prev`

[proc(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html)

这个文件用于指示 exec 前的 selinux label ，正常来说 zygote 是从 `u:r:init:s0` exec 出来的，然而在 magisk 加了一层包装后就变成了 `u:r:zygote:s0` 。由于 zygote 到 app 进程并未经过 exec ，因此这个 prev 自然被保留了下来，因此可以根据这点说明 zygote 遭到了修改。

不过最强检测器似乎没有利用这一特性，自己写了个简单的 wrapper 验证了一下，并没有检测到 zygote 被注入（看起来真正起作用的都是些我不懂的内存扫描魔法了）

```cpp
#include <unistd.h>
#include <sys/mount.h>
#include <android/log.h>
#include <cstring>
#include <cerrno>
#include <cstdlib>

#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, "exec_wrapper", __VA_ARGS__)
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, "exec_wrapper", __VA_ARGS__)

int main(int argc, char **argv) {
    LOGD("exec wrapper start");
    char buf[256];
    memset(buf, 0, sizeof(buf));
    if (readlink("/proc/self/exe", buf, sizeof(buf)) < 0) {
        LOGE("stat:%s", strerror(errno));
        exit(-1);
    }
    LOGD("exe:%s", buf);
    if (umount2(buf, MNT_DETACH)) {
        LOGE("umount:%s", strerror(errno));
        exit(-1);
    }
    if (execve(argv[0], argv, environ)) {
        LOGE("execve:%s", strerror(errno));
    }
    return -1;
}
```

实际上正常 android 的 zygote 在 selinux 规则中是不带 umount 权限的，因此需要修改 selinux 规则，此处直接借用了 magisk 规则， `magiskpolicy --magisk` 。

另外，如果直接 bind mount /data 下的 wrapper 到 app_process ，init 是无法执行的，看日志似乎报了个 `nosuid_transition` 和一个 `execute_no_trans` 。解决方法是 mount 一个 tmpfs ，在这里面有 suid ，把 wrapper 放在这里再 bind mount ，执行成功。一开始不能理解，因为看 magisk 的内置规则也没有给 execute_no_trans 权限，不过考虑前一个 denied 原因，可以猜测其中的原因：是 data 的挂载参数中的 nosuid 惹的祸，无法 suid 就无法改变 selinux label ，只好 fallback (?)到 execute_no_trans ，而这也是 selinux 规则不允许的。这下总算理解为什么 magisk 要把包装放在 tmpfs 里面了。

## 基于 smaps 检测代码注入

之前研究 livin 的时候，strace 发现了它读取 /proc/self/smaps 。

虽然没有仔细逆向过具体的逻辑，不过总感觉 smaps 里面大有文章，于是自己推测了一下 smaps 在检测方面的利用。

### maps

我们知道，读取 maps 可以得到进程自身的内存映射情况，进而实施检测，如检查路径、设备号特征等，更进一步地，还可以扫描特定页面的内存，寻找特征。对于检查路径这种简单的手段，我们可以使用匿名 mmap 替代原有的文件 map 来简单 bypass ，riru 的 riruhide 就是这个做法。而如果是已经被 linker 完全卸载的 so ，简单检查 maps 也不可能找到。

### smaps

smaps 是 maps 的扩充，它除了提供 maps 提供的内容，即内存映射的地址、文件路径、权限之外，还提供了一些统计信息。

[proc(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html)

下面就是一个典型的 smaps ：

```
00400000-0048a000 r-xp 00000000 fd:03 960637       /bin/bash
Size:                552 kB
Rss:                 460 kB
Pss:                 100 kB
Shared_Clean:        452 kB
Shared_Dirty:          0 kB
Private_Clean:         8 kB
Private_Dirty:         0 kB
Referenced:          460 kB
Anonymous:             0 kB
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
ProtectionKey:         0
VmFlags: rd ex mr mw me dw
```

那么，这些统计数字又能告诉我们什么呢？

### 数字中暗藏玄机

我们关注 `{Shared,Private}_{Clean,Dirty}` 字段。Clean 即未修改的页面，Dirty 即修改过的页面；Shared 表示有多个进程引用，Private 表示只有一个进程（自然是当前进程）引用。

我们知道，xhook 的 hook 原理是修改重定位地址，而 unhook 就是把重定位的地址改回去。

重定位地址一般位于 `.data.rel.ro` 节，也就是位于 elf 文件内，或者说对应的 map 必然是文件 map 。因此我们修改这个段，就会产生 dirty page 。

我们知道 zygisk hook 了 libandroid_runtime.so 的很多函数，因此必然产生了 dirty page ，这样特征就体现在 smaps 中了。

也许你会想，linker 自己重定位的时候不也要写吗，这样怎么区分 xhook 修改的和 linker 修改的呢？

我们知道，zygisk 的 unhook 发生在 fork 后的新进程，而 unhook 也是要修改 page 的，并且修改后的 page 一般不会被其他进程共享（除非 fork 了）。

因此，只要发现 libandroid_runtime.so 的重定位地址对应的页有 Private_Dirty ，就说明遭到修改。

这样就导致我们总可以检测到 zygisk 的 hook 。假如我们不 unhook ，则我们的 so 也没法 unload ，修改的地址也无法还原，同样可以被检测。

那么关键是如何找到重定位地址的页。实际上我们可以模仿 linker 的行为，从 dynamic 段 -> DT_ANDROID_RELA 遍历地址，找到对应的 page ；或者解析 elf sections ，找到 `.data.rel.ro` 。

我们也可以直接找 `r--p` 的页，因为重定位节在系统重定位结束后就会重新映射为只读了。

经过测试，一般情况下系统的 so 都不存在 权限为 `r--p` 且有 private dirty page 的页，不过也有例外：EMUI 上有一些奇怪的 so ，从 r-xp 页看上去像是 zygote 加载的，然而 r--p 页有 private dirty。

因此看上去只能拉白名单了，主动检测 /system /apex 下的 libandroid_runtime.so, libart.so, app_process 之类的路径的 map 。

```
10-16 13:08:55.247 17647 17677 D d2.c    : found dirty ro page 7b173f2000-7b173f3000 r--p 0000f000 fd:00 1524802                        /system/lib64/libhwetrace_jni.so (Private_Dirty:         4 kB)
10-16 13:08:55.404 17647 17677 D d2.c    : found dirty ro page 7ba82bd000-7ba82be000 r--p 0002f000 fd:00 1524842                        /system/lib64/libhwgl.so (Private_Dirty:         4 kB)
10-16 13:08:55.445 17647 17677 D d2.c    : found dirty ro page 7ba8c9e000-7ba8ca0000 r--p 0000e000 fd:00 1524295                        /system/lib64/libgpuassistant_client.so (Private_Dirty:         4 kB)
10-16 13:08:55.553 17647 17677 D d2.c    : found dirty ro page 7baa2ee000-7baa2ef000 r--p 0000f000 fd:00 1525078                        /system/lib64/libiAwareSdkAdapter.so (Private_Dirty:         4 kB)
10-16 13:08:55.599 17647 17677 D d2.c    : found dirty ro page 7bab3ed000-7bab3f0000 r--p 0001d000 fd:00 1513074                        /system/lib64/android.hardware.configstore@1.0.so (Private_Dirty:         4 kB)

7b173d0000-7b173d2000 r-xp 00000000 fd:00 1524802                        /system/lib64/libhwetrace_jni.so
Size:                  8 kB
Rss:                   8 kB
Pss:                   0 kB
Shared_Clean:          8 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            8 kB
Anonymous:             0 kB
AnonHugePages:         0 kB
--
7b173ef000-7b173f0000 r--p 0000f000 fd:00 1524802                        /system/lib64/libhwetrace_jni.so
Size:                  4 kB
Rss:                   4 kB
Pss:                   4 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         4 kB
Referenced:            4 kB
Anonymous:             4 kB
AnonHugePages:         0 kB
--
7b173f0000-7b173f1000 rw-p 00010000 fd:00 1524802                        /system/lib64/libhwetrace_jni.so
Size:                  4 kB
Rss:                   4 kB
Pss:                   4 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         4 kB
Referenced:            4 kB
Anonymous:             4 kB
AnonHugePages:         0 kB
```

### 重定位节的权限

```
five_ec1cff@LAPTOP-H42AMUM5:/mnt/d/Documents/tmp$ readelf -l libandroid_runtime.so

Elf file type is DYN (Shared object file)
Entry point 0x9c000
There are 10 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x0000000000000230 0x0000000000000230  R      0x8
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x000000000009ba24 0x000000000009ba24  R      0x1000
  LOAD           0x000000000009c000 0x000000000009c000 0x000000000009c000
                 0x00000000000f1b20 0x00000000000f1b20  R E    0x1000
  LOAD           0x000000000018e000 0x000000000018e000 0x000000000018e000
                 0x0000000000018720 0x0000000000018720  RW     0x1000
  LOAD           0x00000000001a6720 0x00000000001a7720 0x00000000001a7720
                 0x0000000000000f48 0x0000000000003340  RW     0x1000
  DYNAMIC        0x00000000001a1208 0x00000000001a1208 0x00000000001a1208
                 0x00000000000005f0 0x00000000000005f0  RW     0x8
  GNU_RELRO      0x000000000018e000 0x000000000018e000 0x000000000018e000
                 0x0000000000018720 0x0000000000019000  R      0x1
  GNU_EH_FRAME   0x000000000007339c 0x000000000007339c 0x000000000007339c
                 0x00000000000079f4 0x00000000000079f4  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x0
  NOTE           0x0000000000000270 0x0000000000000270 0x0000000000000270
                 0x0000000000000038 0x0000000000000038  R      0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .note.android.ident .note.gnu.build-id .dynsym .gnu.version .gnu.version_r .gnu.hash .dynstr .rela.dyn .relr.dyn .rela.plt .rodata .eh_frame_hdr .eh_frame
   02     .text .plt
   03     .data.rel.ro .fini_array .init_array .dynamic .got .got.plt
   04     .data .bss
   05     .dynamic
   06     .data.rel.ro .fini_array .init_array .dynamic .got .got.plt
   07     .eh_frame_hdr
   08
   09     .note.android.ident .note.gnu.build-id
```

可以看到， `.data.rel.ro` 位于一个 RW 的 LOAD 段，但是又在 GNU_RELRO 段，因此是只读的。

### 参考

soinfo::link_image bionic/linker/linker.cpp

https://android.googlesource.com/platform/bionic/+/4c524711a19a5f02000122b2118017e128c537eb/linker/linker.cpp#3231

soinfo::relocate bionic/linker/linker_relocate.cpp

https://android.googlesource.com/platform/bionic/+/81f0bdb6f303d5281b7082dc7badb248667a5f0c/linker/linker_relocate.cpp#588

phdr_table_protect_gnu_relro bionic/linker/linker_phdr.cpp

https://android.googlesource.com/platform/bionic/+/4c524711a19a5f02000122b2118017e128c537eb/linker/linker_phdr.cpp#933

[&#x5b;原创&#x5d;Android Linker详解(二)-bbs.pediy.com](https://bbs.pediy.com/thread-269891.htm#so%E9%87%8D%E5%AE%9A%E4%BD%8D)

GNU_RELRO:

[linux - Why ELF program headers have two LOAD entries, while the program layout three sections - Stack Overflow](https://stackoverflow.com/questions/28114323/why-elf-program-headers-have-two-load-entries-while-the-program-layout-three-se)

[Shared Libraries on Android](https://chromium.googlesource.com/chromium/src/+/d8bc2922c32620414673d34b6334b2fd8a688c7b/docs/android_native_libraries.md#RELRO-Sharing)

### 如何规避

可以尝试在 hook 前把要修改的 page mremap 到一个地方备份，我们自己 map 并复制原来的 page 作为 hook 后的重定位表；所有函数都 unhook 的时候再把备份的 map 给 remap 回来。（就是不知道此时 smaps 会如何显示这部分 maps 了）
