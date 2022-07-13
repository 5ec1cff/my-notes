# 分析手机的电池使用情况  

感觉手上这台 K30 5G 的耗电越来越快了，记得一次晚上睡觉前充满了电，醒来（6 小时后）发现竟然掉了 7% 。使用系统自带的电量分析，发现主要是 QQ 这个大毒瘤害的。

没记错的话，MIUI 专门为 QQ 和微信开了后门，看似「智能管控」实则「无限制」。不过最令我气愤的还是这个 QQ 明明耗电如此过分，消息还常常收不到。于是尝试了 [TSBattery](https://github.com/fankes/TSBattery) 这个专门针对 QQ 耗电优化的 xposed 模块，效果显著——一晚上只耗 2% 的电了，虽然和某为的电量管控尚有差距，起码是个很大的进步。

不过这也激起了我对这个模块的好奇心——究竟如何分析特定 App 的耗电原因？

其实我也很好奇那些我「允许自启、省电策略无限制」还总是被系统警告「耗电过多」的那些 App 到底耗了多少电，然而 MIUI 的电量统计看起来不想让我们知道这些——只要 app 设置成「无管控」，就不显示电量消耗信息——我寻思不管控就不耗电了？

因此决定自己探索一下。

## 基础知识

首先了解一下电量统计的基础知识：一个 App 的耗电，主要取决于它使用硬件的多少，包括 CPU、Wifi、蓝牙、传感器、相机、闪光灯等。系统统计耗电的时候，以 Uid 为单位。

> 可以参考这篇文章：[Android耗电统计算法 - Gityuan博客 | 袁辉辉的技术博客](http://gityuan.com/2016/01/10/power_rank/)

导致 App 耗电的因素，从 TSBattery 的简介来看，对于 QQ 主要是获取 WakeLock 和通过循环占用 CPU 。

```
Q.这个模块是做什么的？
A.此模块的诞生来源于国内厂商毒瘤 APP 强行霸占后台耗电，QQ 在 8.6.0 版本以后也只是接入了 HMS 推送，但是可笑的是开发组却并没有删除之前疯狂耗电的接收消息方法，于是这个模块就因此而诞生了。
Q.原理是什么？
A.模块有两套工作方式，一种是针对 QQ、TIM Hook 掉系统自身的电源锁“WakeLock”使其不能影响系统休眠，这样子在锁屏的时候 QQ、TIM 就可以进入睡眠状态。第二种就是针对 QQ、TIM 删除其自身的无用耗电疯狂循环检测后台强行保活服务。
Q.如何使用？
A.目前模块支持 LSPosed、EdXposed 以及太极(无极)框架，在太极和 LSPosed 的作用域中，只需勾选 QQ、TIM、微信即可，模块可以做到即插即用，激活后无需重启手机，重启 QQ、TIM 或微信就可以了。
Q.激活后一定可以非常省电吗？
A.并不，模块只能减少 QQ、TIM、微信的耗电，但是请务必记住这一点，省电只是一个理论上的东西，实际水平由你使用的系统和硬件决定，如果你在前台疯狂使用 QQ、TIM，那么照样会耗电，模块只能保证后台运行和锁屏时毒瘤不会消耗过多的无用的电量，仅此而已。
```

## 获取电量统计数据

检查服务列表，发现一些我们感兴趣的服务：

```sh
# service list | grep battery
28      battery: []
29      batteryproperties: [android.os.IBatteryPropertiesRegistrar]
30      batterystats: [com.android.internal.app.IBatteryStats]
```

显然，`batterystats` 就是我们需要的「电池使用统计」。直接 `dumpsys batterystats` 打印所有的统计，加上包名可以打印部分的。

……

[使用 Batterystats 和 Battery Historian 分析电池用量  |  Android 开发者  |  Android Developers](https://developer.android.com/topic/performance/power/setup-battery-historian?hl=zh-cn)