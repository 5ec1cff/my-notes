# 基于 Overlayfs 的模块系统新思路

## 概述

KernelSU 在实现模块系统的时候采用了 overlayfs ，看起来利用这个文件系统实现模块比起 magisk 的基于 bind mount 的 magic mount 更加方便，但是实践表明，坑点仍然不少。

### 子挂载点问题

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

这种情况下，`/vendor` 下有一些挂载点，不过它们恰好可以直接复制参数重新 mount 回去，因此 KernelSU 的修复直接使用 mount 用同样的参数重新挂载到 overlayfs 上，解决了问题。

[KernelSU/mount.rs at main · tiann/KernelSU](https://github.com/tiann/KernelSU/blob/d065a7ca22e1d0ffd8c121f35aa3f3466db1521d/userspace/ksud/src/mount.rs#L304)

但是最近发现在 GSI 系统的 `/system` 下有一些 tmpfs 挂载，这种情况下不能通过简单的相同参数 mount 实现重新挂载。

[GSI 系统上模块挂载失败 · Issue #365 · tiann/KernelSU](https://github.com/tiann/KernelSU/issues/365)

此外还有个别系统厂商的 overlayfs (下称 stock overlay)，ksu 的处理方法是把自己盖在它们的最下层。

https://github.com/tiann/KernelSU/blob/f2d8f1ee60474699656c24293344c4d56186d393/userspace/ksud/src/event.rs#L188

ksu 在挂载自己的 overlayfs 的时候，会先 umount 所有其他的 overlayfs ，自己的 mount 上去后再把其他 overlayfs mount 回来

这也反映到了 zksu 的 revert umount：

https://github.com/Dr-TSNG/ZygiskOnKernelSU/blame/master/loader/src/injector/unmount.cpp#L87

按理来说 overlayfs 可以层叠，直接把 ksu 自己的 overlay 挂在系统上面也没事，除非系统的 overlay 不是刚好和 ksu 重叠。

总之，目标分区下复杂的挂载树结构，简单的一个 overlayfs 是无法处理的，而现有的方案也只是对症下药，难以解决更复杂的情况。

## 如何解决？

我们先参考现有的项目是如何解决这些问题的。

### magisk_overlayfs

HuskyDG 的 magisk_overlayfs ，或许处理了这个问题，因为它的处理方式是把原来的 bind mount 到 /system ，现在被 overlayfs 盖住的 mount 重新 bind mount 到 overlayfs 之上。可惜该模块在 ksu 上还无法正常工作。

[magisk_overlayfs/main.cpp at main · HuskyDG/magisk_overlayfs](https://github.com/HuskyDG/magisk_overlayfs/blob/main/native/jni/main.cpp#L532)

因此，解决 overlayfs 不支持 child mount 合并的方法可以是把 child mount 「移动」或者「复制」到新的 overlayfs 下。

这个模块甚至提供了 rw 挂载方案，但是 hijack KSU 的 mount 比较麻烦。

既然「复制」的方法已经有了，我也提出一个「移动」的想法，也就是使用 move mount 。

1. 使用 mountinfo 建立 mount tree 。  
2. 在 mount tree 中搜索距离 /system 最近的子挂载点，如某个挂载点 dest 为 `/system/xxx/yyy` （也就是以 `/system/` 开头），其 parent 的 dest 为 `/system` 或者 `/` （也就是不以 `/system/` 开头），则它就是距离 `/system` 最近的子挂载点。  
3. 对 2 中的挂载点，通过 move mount 可以移动其挂载树，因此 open 打开其 path。  
4. 挂载 overlayfs  
5. 将之前打开的最近子挂载点 move mount 到 overlay 的 `/system` 上。  

### Magisk

Magisk 存在多年的 overlayfs 不兼容问题，前段时间被西大师的 PR 解决（即将出现在 Magisk 26.1）：

[Refactor magic mount to support overlayfs by yujincheng08 · Pull Request #6588 · topjohnwu/Magisk](https://github.com/topjohnwu/Magisk/pull/6588/commits/5817cda0a4b30d18d48a04613fe1ed47f3e0a01d)

[Magisk Modules Incompatible with overlayfs · Issue #2359 · topjohnwu/Magisk](https://github.com/topjohnwu/Magisk/issues/2359)

> 看起来系统 overlayfs 一般都是在 magic mount 前（也就是 post-fs-data）

西大师的方法是：在一个 private ns 中，将原来的根目录整个 bind mount 到一个新目录，并且取消其 slave mount ，作为 mirror 。这样就解决了原来通过 mknod 创建 mirror 导致目标分区下层挂载点被忽略的问题。

## umount 不如 mount

在我看来，改变系统已有的 mount 层次结构太麻烦，容易出现各种问题，也不利于 hide 的 revert。

在 stackoverflow 搜索 overlayfs 相关问答的时候，看到下面两个问答，得到了启发。

[linux - Using bind mounts with Overlayfs - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/624019/using-bind-mounts-with-overlayfs)

[linux - How to mount overlayfs where lower has child mounts? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/547232/how-to-mount-overlayfs-where-lower-has-child-mounts)

我的另一种方法是：有多少个 child mount 就创建多少个 overlay 。

这些 overlayfs 的层叠方式和原本系统的挂载点层叠方式一样，例如，本来 `/system` 下的挂载树是这样的：

```
1 /system
| - 2 /system/xxx stock overlay
| - 3 /system/yyy bind mount
```

加上模块后：

```
1 /system
| - 2 /system/xxx stock overlay
| - 3 /system/yyy bind mount
| - 4 /system (ksu overlay)
    | - 5 /system/xxx (ksu overlay)
    | - 6 /system/yyy (ksu overlay)
```

这样的问题是，每一个 ksu overlay 都需要 lowerdir 提供原先目录的内容，然而在 `/system` 下挂载第一个 overlay 后，原先的内容被屏蔽了，怎么得到 lowerdir 呢？

这就遇到了和 magisk 一样的问题，magisk 用的是 mirror 的方式。我们可以学习西大师的那个做法，递归复制 private 的根挂载点作为 mirror 。

但是受到上面答案的启发，我们也可以不使用 mirror ，而是在挂载 overlay 前打开所需要目录的 fd ，然后将 `/proc/self/fd` 传给 overlayfs 。

考虑到 /proc/self/fd 是一个魔法的软链接，我们相信即使传给 overlay 也能让它正常工作（误）。

这样 lowerdir 的原始目录有了，剩下的就是模块文件。

一开始想法是，只要最底层的 overlay 的 lowerdir 加上模块目录，上层 overlay 的 lowerdir 就依赖下层的 overlay 就行。

但是这样又有问题了——overlayfs 最大堆叠层数不能超过两层！并且这个限制是硬编码的，用户不能更改。

[linux - How to use multiple lower layers in overlayfs - Stack Overflow](https://stackoverflow.com/questions/31044982/how-to-use-multiple-lower-layers-in-overlayfs)

[fs.h « linux « include - kernel/git/apw/overlayfs.git - overlayfs playground](https://git.kernel.org/pub/scm/linux/kernel/git/apw/overlayfs.git/tree/include/linux/fs.h?h=overlayfs.v12apw1#n491)

好吧，既然不让叠加，那就主动打开模块的相应目录，如果存在就放到 lowerdir 。

比如 `/system/xxx` 是一个挂载点，如果有模块提供了 `/data/adb/modules/{module}/system/xxx` ，就作为它的 lowerdir 。

但这样就有一个问题，如果所有模块都没这个目录怎么办？在没有 upperdir 的情况下， overlayfs 是不允许 lowerdir 只包含一个目录的。

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

方法无非两个：选择另一个空的目录作为 upperdir 或加入 lowerdir 。

考虑到节省系统资源，我们 mount 一个大小 0k 的只读 tmpfs ，作为所有 lowerdir 单一的挂载点的填充（dummy tmpfs）。

注意这种情况下，它必须作为 lowerdir 的最下层，否则文件会被加上奇怪的 t 属性位。

此外，似乎不能打开 fd ，umount 后传给 overlay ，只能在没有 umount 的情况下先挂载好 overlayfs ，再把 tmpfs umount 了。

另外，在给子目录挂载的时候也需要考虑所在位置是不是目录，如果被模块换成了非目录的文件，可以直接跳过。

这样 ro 挂载的问题算是有一个好的解决方案了，不过这种情况下想要扩展成 rw 就更加困难了。

### 其他问题

在群里讨论之后，发现上面的做法仍然存在问题，比如 vfat 无法作为 overlay 的 lowerdir (`/vendor/bt_firmware`)，这种情况下也许只能 fallback 为 bind mount 了。

> 也就是说，不能修改上面的文件，不过应该没人会改 firmware 吧。

如果系统会叠两层 overlayfs 也无法处理。总之，mount overlay 失败的时候，fallback 成 mount bind 原目录 /proc/self/fd 是比较好的方法，起码确保了系统原始文件完整，而大部分模块也能正常工作。
 
此外还有直接 bind mount 一个文件的奇葩情况（上文的 GSI 系统就属此例），此时原挂载点不能作为 overlayfs 的 lowerdir 。

重新考虑了一下，新方案可以处理 regular file bind mount 的情况：

1. 挂载点下没有模块修改，此时也不需要 overlay 了，直接 bind mount （dummy tmpfs 可以丢掉了）。  
2. 挂载点下有模块修改，原本是目录，此时 overlay ，合并模块和 stock 挂载点。  
3. 挂载点下有模块修改，原本是普通文件，此时不需要任何 mount （因为没有旧目录需要合并），下层的 overlay 处理了一切。  

判断 stock mount 是不是 regular file 只要 stat 我们打开的 fd 即可

## POC

https://github.com/5ec1cff/ksu-mount-POC

~~由于不会 Rust 只好用 C++ 写了~~

### 实现分析

mount-scan.h 中，通过解析 mountinfo 建立了挂载树。

mountinfo 提供了每个挂载的 id 和父挂载 id ，而我们需要的是每个挂载的子挂载信息。

MountNode 记录了每个挂载的子挂载，最终我们得到一个根挂载。

根挂载的 id 是 `parent_id == id` ，或者 `parent_id` 不存在的挂载（chroot 后的 mountinfo 就是这样的，某些挂载不可见）。

```cpp
    MountNodePtr root;
    for (const auto &[_, node]: node_map) {
        auto parent = node_map.find(node->parent_id);
        if (parent != node_map.end()) {
            parent->second->children.push_back(node);
        }
        if (root == nullptr && (
                node->id == node->parent_id
                || parent == node_map.end())) {
            root = node;
        }
    }
```

下面三个方法用于确定哪些挂载点是有效的。

findMountForPath: 找路径 path 下最上层的挂载点（也就是能被我们看到的）

例如路径 /system 所属的挂载点，在 SAR 系统上一般就是根挂载点 `/` 。

```cpp
    static MountNodePtr findMountForPath(const MountNodePtr &self, const std::string &path) {
        if (path.starts_with(self->mount_point)) {
            for (auto & node : reversed(self->children)) {
                auto new_result = findMountForPath(node, path);
                if (new_result != nullptr) {
                    return new_result;
                }
            }
            return self;
        } else {
            return nullptr;
        }
    }
```

findChildMountForPath： 找某个路径的直接子挂载点下的最上层挂载点。

这个说法有点绕口令，实际上是为了后面的方法做铺垫。

比如这样的挂载树：

```
1 - /system
  | 2 - /system/xxx
    | 6 - /system/xxx/www
    | 7 - /system/xxx
  | 3 - /system/yyy
      | 4 - /system/yyy
        | 5 - /system/yyy/zzz
```

则 findChildMountForPath 找到的是 4, 7 。

2, 3 是直接子挂载点，然后再找它们挂载点路径的最上层挂载点。

```cpp
    static void findChildMountForPath(std::vector<MountNodePtr> &children, const MountNodePtr &self, const std::string &path) {
        auto mount_for_path = findMountForPath(self, path);
        if (mount_for_path == nullptr) return;
        for (auto & node : reversed(mount_for_path->children)) {
            if (node->mount_point.starts_with(path)) {
                children.push_back(findMountForPath(node, node->mount_point));
            }
        }
    }
```

findTopMostMountsUnderPath：实际上就是找某个路径下所有有效的（最上层的）挂载点。

```cpp
    static void findTopMostMountsUnderPath(std::vector<MountNodePtr> &seq, const MountNodePtr &self, const std::string &path) {
        if (self != nullptr) {
            auto children = self->findChildMountForPath(self, path);
            for (auto &c: children) {
                findTopMostMountsUnderPath(seq, c, c->mount_point);
            }
            seq.push_back(self);
        }
    }
```

比如上面的树中，/system 下 1, 4, 5, 7 都是有效的（在最上层）。这些挂载点按照倒序排列，就是最终要被挂上 overlayfs 的地方。

### 实验

考虑 /mnt/move-test 下面的四个目录 a,b,c,g ，其中模块目录为 g ，使用上面的 POC 挂载 overlayfs 。

```
./of-poc $PWD/a $PWD/g
```

a, b, c 目录结构如下：

```
tree a b c
a
├── b # dir
└── c # dir
b
├── c # dir
└── x # regular
c
└── zz # regular
```

执行下面两个 bind mount ：

```
mount --bind b a/b
mount --bind c a/b/c
```

则 a 目录结构如下：

```
tree a
a
├── b
│   ├── c
│   │   └── zz # regular
│   └── x # regular
└── c # dir
```

如果模块 g 下没有给 `/mnt/move-test/b` 等准备文件，就是这个效果：

```
tree g
g
└── mnt
    └── move-test
        └── a
```

```
 118 74 8:32 /mnt/move-test/b /mnt/move-test/a/b rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
  119 118 8:32 /mnt/move-test/c /mnt/move-test/a/b/c rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
 122 74 0:61 / /mnt/move-test/a ro,relatime - overlay KSU ro,lowerdir=/mnt/move-test/a:/mnt/move-test/g
  126 122 0:65 / /mnt/move-test/a/b ro,relatime - overlay KSU ro,lowerdir=/proc/self/fd/4:/dev/KSU_DUMMY
   129 126 0:69 / /mnt/move-test/a/b/c ro,relatime - overlay KSU ro,lowerdir=/proc/self/fd/3:/dev/KSU_DUMMY
```

现在模块的 `a/b` 是目录，`a/b/c` 是文件。

```
tree g
g
└── mnt
    └── move-test
        └── a
            └── b # dir
                └── c # regular file
```

```
 118 74 8:32 /mnt/move-test/b /mnt/move-test/a/b rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
  119 118 8:32 /mnt/move-test/c /mnt/move-test/a/b/c rw,relatime - ext4 /dev/sdc rw,discard,errors=remount-ro,data=ordered
 122 74 0:61 / /mnt/move-test/a ro,relatime - overlay KSU ro,lowerdir=/mnt/move-test/g/mnt/move-test/a:/mnt/move-test/a
  125 122 0:64 / /mnt/move-test/a/b ro,relatime - overlay KSU ro,lowerdir=/mnt/move-test/g/mnt/move-test/a/b:/proc/self/fd/4
```

```
tree a
a
├── b
│   ├── c # regular
│   └── x # regular
└── c
```


