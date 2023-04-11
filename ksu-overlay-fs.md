# Overlayfs

## 子挂载点问题

overlayfs 不会合并挂载点下子挂载点的内容。

例如，如果 lowerdir 指向的目录下存在 bind mount 或者 tmpfs ，则 overlayfs 并不会合并它们。

下面是一个例子：

```sh
root@fivewsl:/mnt/overlay-test# mkdir lower1
root@fivewsl:/mnt/overlay-test# mkdir lower2
root@fivewsl:/mnt/overlay-test# echo lower1 > lower1/a
root@fivewsl:/mnt/overlay-test# echo lower2 > lower2/a
root@fivewsl:/mnt/overlay-test# mkdir merged
root@fivewsl:/mnt/overlay-test# touch lower1/lower1
root@fivewsl:/mnt/overlay-test# touch lower2/lower2
root@fivewsl:/mnt/overlay-test# umount merged
root@fivewsl:/mnt/overlay-test# ls merged
root@fivewsl:/mnt/overlay-test# touch merged/merged
root@fivewsl:/mnt/overlay-test# echo merged > merged/merged
root@fivewsl:/mnt/overlay-test# touch bound
root@fivewsl:/mnt/overlay-test# echo bind-mounted > bound
# 在 merged 目录下绑定挂载
root@fivewsl:/mnt/overlay-test# mount --bind ./bound ./merged/merged
root@fivewsl:/mnt/overlay-test# cat merged/merged
# bind mount 有作用
bind-mounted
# merged 作为 lowerdir ，也作为新 overlayfs 的位置
root@fivewsl:/mnt/overlay-test# mount -t overlay -o lowerdir=$PWD/lower1:$PWD/lower2:$PWD/merged overlay merged
root@fivewsl:/mnt/overlay-test# ls merged
a  lower1  lower2  merged
root@fivewsl:/mnt/overlay-test# cat merged/merged
# 仍然是 merged 目录的内容而非 bind mount 的内容
merged

# mountinfo
root@fivewsl:/mnt/overlay-test# grep overlay-test /proc/self/mountinfo
118 74 8:32 /mnt/overlay-test/bound /mnt/overlay-test/merged/merged rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
122 74 0:61 / /mnt/overlay-test/merged rw,relatime - overlay overlay ro,lowerdir=/mnt/overlay-test/lower1:/mnt/overlay-test/lower2:/mnt/overlay-test/merged
```

当然不一定要 overlay 覆盖原来的目录，比如 bind mount 到 lower1 也是一样的效果——bind mount 没有被合并到新的 overlayfs 。实际上， lowerdir 下的任何 sub mount 都无法合并。

在 KSU 曾经就出现过这样的问题：

[vendor overlay breaks stock vendor mount · Issue #233 · tiann/KernelSU](https://github.com/tiann/KernelSU/issues/233)

这种情况下，`/vendor` 下有一些挂载点，不过不是 bind mount ，因此 KernelSU 的修复直接使用 mount 用同样的参数重新挂载到 overlayfs 上，解决了问题。

[KernelSU/mount.rs at main · tiann/KernelSU](https://github.com/tiann/KernelSU/blob/main/userspace/ksud/src/mount.rs#L304)

但是最近发现在 GSI 系统的 `/system` 下有一些 bind mount 挂载，这种情况下不能通过简单的相同参数 mount 实现重新挂载。

[GSI 系统上模块挂载失败 · Issue #365 · tiann/KernelSU](https://github.com/tiann/KernelSU/issues/365)

magisk_overlayfs 或许能解决这个问题，因为它的处理方式是把原来的 bind mount 到 /system ，现在被 overlayfs 盖住的 mount 重新 bind mount 到 overlayfs 之上。可惜该模块在 ksu 上还无法正常工作。

[magisk_overlayfs/main.cpp at main · HuskyDG/magisk_overlayfs](https://github.com/HuskyDG/magisk_overlayfs/blob/main/native/jni/main.cpp#L532)

因此，解决 overlayfs 不支持 child mount 合并的方法可以是把 child mount 「移动」或者「复制」到新的 overlayfs 下。

既然「复制」的方法已经有了，我也提出一个「移动」的想法，也就是使用 move mount 。

1. 使用 mountinfo 建立 mount tree 。  
2. 在 mount tree 中搜索距离 /system 最近的子挂载点，如某个挂载点 dest 为 `/system/xxx/yyy` （也就是以 `/system/` 开头），其 parent 的 dest 为 `/system` 或者 `/` （也就是不以 `/system/` 开头），则它就是距离 `/system` 最近的子挂载点。  
3. 对 2 中的挂载点，通过 move mount 可以移动其挂载树，因此 open 打开其 path。  
4. 挂载 overlayfs  
5. 将之前打开的最近子挂载点 move mount 到 overlay 的 `/system` 上。  


## 系统 overlayfs 

Magisk 存在多年的 overlayfs 不兼容问题，前段时间被西大师的 PR 解决：

[Refactor magic mount to support overlayfs by yujincheng08 · Pull Request #6588 · topjohnwu/Magisk](https://github.com/topjohnwu/Magisk/pull/6588/commits/5817cda0a4b30d18d48a04613fe1ed47f3e0a01d)

[Magisk Modules Incompatible with overlayfs · Issue #2359 · topjohnwu/Magisk](https://github.com/topjohnwu/Magisk/issues/2359)

看起来系统 overlayfs 一般都是在 magic mount 前（也就是 post-fs-data）

## ksu 的挂载方式

https://github.com/tiann/KernelSU/blob/f2d8f1ee60474699656c24293344c4d56186d393/userspace/ksud/src/event.rs#L188

ksu 在挂载自己的 overlayfs 的时候，会先 umount 所有其他的 overlayfs ，自己的 mount 上去后再把其他 overlayfs mount 回来

这也反映到了 zksu 的 revert umount：

https://github.com/Dr-TSNG/ZygiskOnKernelSU/blame/master/loader/src/injector/unmount.cpp#L87

按理来说 overlayfs 可以层叠，直接把 ksu 自己的 overlay 挂在系统上面也没事，除非系统的 overlay 不是刚好和 ksu 重叠。

> 也就是系统挂到了 `/vendor/xxx` 的位置，而 ksu 需要 overlay 在 `/vendor` 上

这种情况不能像 bind mount 那样通过简单的 move 解决。

## umount 不如 mount

改变系统已有的 mount 层次结构太麻烦，容易出现各种问题。因此我们想到采取另一种方法：有多少个 child mount 就创建多少个 overlay 。

这样的做法和西大师的那个提交差不多，不过我们没有递归复制根挂载点作为 mirror ，而是使用 dir fd 。

打开每个需要的子挂载点的目录 fd ，然后 lowerdir 传 `/proc/self/fd/xxx` ，并上底层的 overlayfs 即可。

[linux - Using bind mounts with Overlayfs - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/624019/using-bind-mounts-with-overlayfs)

[linux - How to mount overlayfs where lower has child mounts? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/547232/how-to-mount-overlayfs-where-lower-has-child-mounts)

但是这样又有问题了：

[linux - How to use multiple lower layers in overlayfs - Stack Overflow](https://stackoverflow.com/questions/31044982/how-to-use-multiple-lower-layers-in-overlayfs)

overlayfs 最大堆叠层数不能超过两层！

[fs.h « linux « include - kernel/git/apw/overlayfs.git - overlayfs playground](https://git.kernel.org/pub/scm/linux/kernel/git/apw/overlayfs.git/tree/include/linux/fs.h?h=overlayfs.v12apw1#n491)

这个限制是硬编码的，用户不能更改

这样一来上面的方法就受到了限制，如果系统本来只有一个 mount 还好说，要是超过两层就没法用了。

不过想一想，或许还有别的出路。

既然不让叠加，那就主动打开模块的相应目录，如果存在就放到 lowerdir 。

比如 `/system/xxx` 是一个挂载点，模块 `/data/adb/modules/module_a/system/xxx` 就作为 lowerdir 即可。

但这样就有一个问题，如果模块没有对应目录怎么办？overlayfs 是不允许 lowerdir 只写一个目录的（哪怕只读了）。

https://cs.android.com/android/kernel/superproject/+/refs/heads/common-android-mainline:common/fs/overlayfs/super.c;l=1753;drc=d20537027c4e11417a1934cc00069eed39bfde81

```c
static struct ovl_entry *ovl_get_lowerstack(struct super_block *sb,
				const char *lower, unsigned int numlower,
				struct ovl_fs *ofs, struct ovl_layer *layers)
{

	if (!ofs->config.upperdir && numlower == 1) {
		pr_err("at least 2 lowerdir are needed while upperdir nonexistent\n");
		return ERR_PTR(-EINVAL);
	}
```

方法无非两个：提供 upperdir ，或提供另一个空的目录作为 lowerdir 。

考虑到节省系统资源，我们 mount 一个大小 0k 的只读 tmpfs ，作为所有 lowerdir 单一的挂载点的填充，注意这种情况下，它必须作为 lowerdir 的最下层，否则文件会被加上奇怪的 t 属性位。此外，似乎不能打开 fd ，umount 后传给 overlay ，只能在没有 umount 的情况下先挂载好 overlayfs ，再把 tmpfs umount 了。

另外，在给子目录挂载的时候也需要考虑所在位置是不是目录，如果被模块换成了非目录的文件，可以直接跳过。

这样 ro 挂载的问题算是有一个好的解决方案了，不过这种情况下想要扩展成 rw 就更加困难了。

挂载树如下， /mnt/move-test/a 上 bind mount 了 b ，b 上又 bind mount c ，然后由上面的方法挂载 overlayfs 。

```
 118 74 8:32 /mnt/move-test/b /mnt/move-test/a/b rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
  119 118 8:32 /mnt/move-test/c /mnt/move-test/a/b/c rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
 122 74 0:61 / /mnt/move-test/a ro,relatime - overlay KSU ro,lowerdir=/mnt/move-test/a:/mnt/move-test/g
  126 122 0:65 / /mnt/move-test/a/b ro,relatime - overlay KSU ro,lowerdir=/proc/self/fd/4:/dev/KSU_DUMMY
   129 126 0:69 / /mnt/move-test/a/b/c ro,relatime - overlay KSU ro,lowerdir=/proc/self/fd/3:/dev/KSU_DUMMY
```
