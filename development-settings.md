# 开发者选项的 shell 命令

主要方法：

1. settings diff  
修改设置前后使用 `settings list` 记录，然后对两个文本作 diff 找到修改的地方。
2. 查找源码
```
packages/apps/Settings/res/xml/development_settings.xml
packages/apps/Settings/res/values-zh-rCN/strings.xml
packages/apps/Settings/src/com/android/settings/development
```

## 显示刷新频率

这个似乎不会保存到设置存储中，即不能通过设置存储来控制。

`packages/apps/Settings/src/com/android/settings/development/ShowRefreshRatePreferenceController.java`

```
# 开启显示刷新频率
service call SurfaceFlinger 1034 i32 1
# 关闭显示刷新频率
service call SurfaceFlinger 1034 i32 0
```

> 这里使用了 SurfaceFlinger backdoor ，需要 root/system uid 才能调用，shell 一般是不行的。  
> [`frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp`](https://android.googlesource.com/platform/frameworks/native/+/213454462ca60ec8af20f69a797bbef19712b85b/services/surfaceflinger/SurfaceFlinger.cpp#5980)

## 时间悬浮窗(MIUI)

在 system settings 中，key 为 miui_time_floating_window

> 这种原生没有的情况下使用 settings diff 就比较好找。

```
settings put system miui_time_floating_window 0|1
```

## 显示触摸

`packages/apps/Settings/src/com/android/settings/development/ShowTapsPreferenceController.java`

key: show_touches

> AOSP 的 strings.xml 竟然没有翻译……

```
settings put system show_touches 0|1
```

## 指针位置

`packages/apps/Settings/src/com/android/settings/development/PointerLocationPreferenceController.java`

pointer_location

> 同上，也没有翻译

`settings put system pointer_location 0|1`

## 显示布局边界

[详情](dark-mode-and-debug-layout.md)

`packages/apps/Settings/src/com/android/settings/development/ShowLayoutBoundsPreferenceController.java`

```
# 开启
setprop debug.layout true; service call activity 1599295570
# 关闭
setprop debug.layout false; service call activity 1599295570
```
