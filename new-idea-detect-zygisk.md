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
