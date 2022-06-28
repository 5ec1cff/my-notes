# 动手做一个 Android FPS 监视器  

## 前言

最近在研究一个开源 App 的源码，看到一条 issue 提到在某个界面静止的情况下，Surface 持续刷新，FPS 很高，导致耗电更快。我就想解决这个问题，但首先应该自己验证一下，这个界面是不是真的静态时 FPS 高，但我对如何获取 FPS 一无所知，手上也没有合适的工具，于是决定自己钻研一番。

## 认识 FPS

FPS （每秒传输帧数, Frames Per Second），也叫帧率。在 Android 中，

刷新率

[VSYNC  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/devices/graphics/implement-vsync?hl=zh-cn)  
[“终于懂了” 系列：Android屏幕刷新机制—VSync、Choreographer 全面理解！_胡飞洋的博客-CSDN博客](https://blog.csdn.net/hfy8971613/article/details/108041504)  
[通俗易懂的Android屏幕刷新机制 - 掘金](https://juejin.cn/post/6976787089703010341)  
[Android-屏幕刷新机制 - 简书](https://www.jianshu.com/p/6958d3b11b6a)  



……先坑着

[vtools/FpsUtils.kt at master · helloklf/vtools](https://github.com/helloklf/vtools/blob/master/app/src/main/java/com/omarea/library/shell/FpsUtils.kt)

```
# 内核级 fps 数据
/sys/class/drm/sde-crtc-0/measured_fps
/sys/class/graphics/fb0/measured_fps
# 在 /sys 下寻找 fps
find /sys -name measured_fps 2>/dev/null
find /sys -name fps 2>/dev/null
# 利用 SurfaceFlinger 后门代码获取 PageFlipCount
service call SurfaceFlinger 1013
# 也可以通过 dumpsys SurfaceFlinger 获取上面的数据（不能 proto ）
# Android 9: grep '+ DisplayDevice' 找 flips=
# Android 11: grep 'Composition RenderSurface State' 找 flips
```