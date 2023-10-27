# sockets

## INET

正常的 linux 是允许任何用户访问网络的

https://t.me/real5ec1cff/15

https://t.me/real5ec1cff/16

曾经是 CONFIG_ANDROID_PARANOID_NETWORK 这个内核选项，如果有 AID_INET 则允许

这个源码找起来比较费劲，因为 mainline 内核是没有的

https://android.googlesource.com/kernel/msm/+/2038692b9acf7ae3054a35e879039ff062316bf0/net/ipv4/af_inet.c#265

## ICMP 

正常的 linux ping ：

```
getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep
```

```
socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = -1 EACCES (Permission denied)
socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = -1 EPERM (Operation not permitted)
socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6) = -1 EACCES (Permission denied)
socket(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6) = -1 EPERM (Operation not permitted)
```

root:

```
socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = -1 EACCES (Permission denied)
socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = 3
socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6) = -1 EACCES (Permission denied)
socket(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6) = 4
```

Android:

```
getcap /system/bin/ping
# 无
```

```
socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = 3
```

这里用的是 SOCK_DGRAM + IPPROTO_ICMP

这实际上是内核的免特权 icmp 特性

https://stackoverflow.com/a/20105379

https://lkml.org/lkml/2011/5/18/305

正常 linux 默认不开启:

```
$ sysctl net.ipv4.ping_group_range
net.ipv4.ping_group_range = 1   0
```

1 0 表示 `1<uid<0` ，这是个空集，也就是没有 gid 能使用

Android:

```
# sysctl net.ipv4.ping_group_range
net.ipv4.ping_group_range = 0   2147483647
```

这是在 init.rc 写入的

https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.rc;l=315;drc=b5ce7aa444f7abfd62649c329f76c2b9e728cf92

因此一般的 android app 也不能用 SOCK_RAW （网络权限只给了 AID_INET ，没有 AID_NET_RAW），ping 能用则是因为用了 SOCK_DGRAM + IPPROTO_ICMP 特性。
