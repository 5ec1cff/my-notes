# 系统属性

[Android Property 实现解析与黑魔法 - 残页的小博客](https://blog.canyie.top/2022/04/09/property-implementation-and-isolation/)

## bionic

### 属性系统概述

[bionic/libc/bionic/system_property_api.cpp](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/bionic/system_property_api.cpp;drc=b481a2e743102efc6b18cc586aa979a70e575d64)

[bionic/libc/include/sys/_system_properties.h](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/include/sys/_system_properties.h;l=44;drc=a505b2d37a4ed106925c480b8c1c3ffc442d6ec5)

属性系统实际上就是一个整个系统共享的键值对存储器，通过共享文件内存映射实现。

存放属性的文件位于 `/dev/__properties__` ，这下面的多个文件，实际上是将属性按照前缀划分为不同的 SELinux 上下文，存放在对应的文件中。

划分的规则可以在 `/{system,vendor,...}/etc/selinux/{plat,...}_property_contexts` 等文件找到。

init 进程负责初始化属性存储区域，且是系统中唯一能够修改属性的进程（当然，只要某个进程能 rw 映射存放属性的文件，也是可以修改的），其他进程通过属性服务（一个名为 property_service 的 socket）修改属性。其他进程会以 ro 形式共享映射属性存储，用于读取。

### 初始化

一般进程：

[bionic/libc/bionic/libc_init_common.cpp __libc_init_common](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/bionic/libc_init_common.cpp;l=171;drc=2557f73c05f256db3ffa9ac9892b13e226b6ea4c)

`__system_properties_init`

只能映射 ro 的属性区域共享内存

init：

[system/core/init/property_service.cpp PropertyInit](https://cs.android.com/android/platform/superproject/+/master:system/core/init/property_service.cpp;l=1368;drc=da5323e2d6be16470b7ce2be118d41a497c7d9a6)

`__system_property_area_init`

创建属性区域，并映射 rw 共享内存

### 结构

bionic/libc/system_properties/include/system_properties/prop_area.h  
bionic/libc/system_properties/prop_area.cpp  

prop_area 结构体对应了整个存储区域的结构。大小 128K ，前 32*4 字节包含已分配字节、用于同步的原子 serial、魔数和版本

```cpp
struct prop_trie_node {
  uint32_t namelen;
  atomic_uint_least32_t prop; // 指向 prop_info 结构体
  atomic_uint_least32_t left; // 二叉树左子
  atomic_uint_least32_t right; // 二叉树右子
  atomic_uint_least32_t children; // 下一层级的二叉树
  char name[0];
};
```

prop_trie_node(prop_bt) 是二叉搜索树。属性名按 `.` 分割为数个层级，每个层级都由二叉搜索树用于按名字索引。prop_area 数据区的第一个对象（偏移 0）就是根结点，其 children 是第一层级的根树。可以看到，每个结点还要记录自己的名字（在最末尾）。

这一点在源码的注释已经说得比较清楚了，总的来说，这就是一个二叉搜索树 + 字典树的混合体。

```cpp
// Properties are stored in a hybrid trie/binary tree structure.
// Each property's name is delimited at '.' characters, and the tokens are put
// into a trie structure.  Siblings at each level of the trie are stored in a
// binary tree.  For instance, "ro.secure"="1" could be stored as follows:
//
// +-----+   children    +----+   children    +--------+
// |     |-------------->| ro |-------------->| secure |
// +-----+               +----+               +--------+
//                       /    \                /   |
//                 left /      \ right   left /    |  prop   +===========+
//                     v        v            v     +-------->| ro.secure |
//                  +-----+   +-----+     +-----+            +-----------+
//                  | net |   | sys |     | com |            |     1     |
//                  +-----+   +-----+     +-----+            +===========+

// Represents a node in the trie.
```

bionic/libc/system_properties/include/system_properties/prop_info.h

prop_info 就是属性对象，记录了属性的值。这个值可以是最长 92 字节的短值，也可以是更长的长值。长值会作为独立的对象进行分配。

```cpp
struct prop_info {
  atomic_uint_least32_t serial;
  union {
    char value[PROP_VALUE_MAX];
    struct {
      char error_message[kLongLegacyErrorBufferSize];
      uint32_t offset;
    } long_property;
  };
  char name[0];
};

// bionic/libc/include/sys/system_properties.h
#define PROP_VALUE_MAX  92
```

### 对象分配

用于分配 prop_trie_node 、 prop_info 和 long value 。所有对象都在内存区域上依次分配。

```cpp
// bionic/libc/system_properties/prop_area.cpp
void* prop_area::allocate_obj(const size_t size, uint_least32_t* const off) {
  const size_t aligned = __BIONIC_ALIGN(size, sizeof(uint_least32_t));
  if (bytes_used_ + aligned > pa_data_size_) {
    return nullptr;
  }

  *off = bytes_used_;
  bytes_used_ += aligned;
  return data_ + *off;
}
```

prop_area 没有实现值的释放，因为 prop_info 是一个固定大小的结构体，只要已经分配过 prop 的结点，今后改写都在这个结点进行。而 long value 是 ro 属性专属的特性（众所周知 ro 不需要修改，且 bionic 也没有实现修改 long value 属性）。

## Magisk resetprop

Magisk resetprop 并没有调用系统实现，而是根据 [bionic](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/system_properties/) 自行实现了 property 的操作。

https://github.com/topjohnwu/Magisk/blob/350d0d600cd8f24d82c83876d79e74ade5c9db64/native/src/core/resetprop/resetprop.cpp#L254

在这里， `__system_property_xxx` 不是系统 API ，而是自己实现的，原始的系统 API 通过 dlsym 获得。

就在今年五月的一个[提交](https://github.com/topjohnwu/Magisk/commit/f36b21bae55d13317b126ddf1489719870739801)后，该模块从 magisk 独立出来

https://github.com/topjohnwu/system_properties

实际上大部分源码都照搬 bionic ，属性的初始化也是从 `__system_properties_init` -> `SystemProperties::Init` ，即一般进程的调用，映射的是 ro 属性区域。

不同之处在于给 `prop_area::map_fd_ro` 加了一个 rw 映射的实现，并且这个 rw 是可选的，如果不能 rw 才映射 ro 。

https://github.com/topjohnwu/system_properties/blob/3b4b3f0c64bb83f955acb10fe1f2716a91c335dd/prop_area.cpp#L107

此外该库还实现了属性的 delete 

https://github.com/topjohnwu/system_properties/blob/3b4b3f0c64bb83f955acb10fe1f2716a91c335dd/prop_area.cpp#L433

```cpp
bool prop_area::remove(const char *name, bool prune) {
  prop_bt *node = traverse_trie(root_node(), name, false);
  if (!node) return false;

  uint_least32_t prop_offset = get_offset(&node->prop);
  if (prop_offset == 0) return false;

  prop_info *prop = to_prop_info(&node->prop);

  // Detach the property from trie ASAP
  set_offset(&node->prop, 0u);

  // Then wipe out the property from memory
  if (prop->is_long()) {
    char *value = const_cast<char*>(prop->long_value());
    memset(value, 0, strlen(value));
  }
  memset(prop->name, 0, strlen(prop->name));
  memset(prop, 0, sizeof(*prop));

  if (prune) {
    prune_trie(root_node());
  }

  return true;
}
```

方法是将 prop info 及 long value 填 0 ，并从 prop_bt 删除 prop ，还可能执行 prune ，会把没有 prop info 的树结点也全部填 0 。

为了实现修改 ro 属性，需要删除原先的 ro 属性，再用 add 重新分配结点（因为 update 不允许增加 long value）

## 权限检查

曾经尝试提权某 tv os ，由于其 selinux permissive ，利用 magica 拿到受限的 root ，尝试修改属性的 ro.debuggable 进一步提权，但是没有文件系统的 cap ，因此直接修改属性文件权限为 777 ，这样确实可以直接写文件了，但是 resetprop 报错初始化失败，启动其他新程序也无法正确初始化属性，将权限恢复后才正常，可能原因来自于 prop_area map_fd_ro 的一段检查：

https://cs.android.com/android/platform/superproject/+/master:bionic/libc/system_properties/prop_area.cpp;l=113;drc=c37aa7ad3c9767257dfcfd978a2527dc63fb57cd

```cpp
prop_area* prop_area::map_fd_ro(const int fd) {
  struct stat fd_stat;
  if (fstat(fd, &fd_stat) < 0) {
    return nullptr;
  }

  if ((fd_stat.st_uid != 0) || (fd_stat.st_gid != 0) ||
      ((fd_stat.st_mode & (S_IWGRP | S_IWOTH)) != 0) ||
      (fd_stat.st_size < static_cast<off_t>(sizeof(prop_area)))) {
    return nullptr;
  }
```
