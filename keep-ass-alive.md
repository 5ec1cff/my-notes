# 无障碍服务保活探究  

## 启动无障碍  

无障碍服务的启动 API 并不在 IAccessibilityManagerService ，而是在安全设置(`content://settings/secure`)里面！

以下安全设置的值即为启用的无障碍服务列表，是 `:` 分隔的平坦化组件名 (`packageName/componentName`)。

```
enabled_accessibility_services
```

读写安全设置需要 `WRITE_SECURE_SETTINGS` 权限，这个权限是 `development` 的，所以默认第三方 app 即使声明也不获得权限，系统设置也不会有启用的开关，但可以通过 `pm grant` 授予。

> 类似的还有 `READ_LOGS` ，也被<ruby>利用<rt>滥用</rt></ruby>于在 Android 10 以上版本监听剪贴板的黑魔法。  

无障碍服务是否启用完全由该值决定，调用 disableSelf 禁用自身在内部也是更新这个值。

> 「设置」中更新无障碍服务的相关源码在此：`frameworks/base/packages/SettingsLib/src/com/android/settingslib/accessibility/AccessibilityUtils.java` ，竟然不在「Settings」包而是在「SettingsLib」，找起来着实费了一番功夫。

## 自动拉起

理论上来说，原生的无障碍服务在启用后是可以在进程异常结束后被自动拉起的，看起来类似于 TileService 、通知使用权或桌面。只要服务列表中存在，系统总会将其自动拉起，只有 force-stop 才会将其关闭（此时无障碍服务列表中该服务会被自动移除）。  
无障碍服务由 SystemServer 通过 bindService 启动，并通过服务的重启机制自动重启(`BIND_AUTO_CREATE`)，所以开发者应该不用处理崩溃后恢复的情况，系统会自动完成。

不过一些（大部分？）定制系统的默认行为阻止了应用的自启，可重启的服务显然属于「自启」的范围，因此需要手动设置允许自启，此外省电策略也是影响服务自启的关键问题（因为被省电策略停止的应用相当于被 force-stop ，服务会被自动移除）。

> 尽管无障碍服务自动拉起阻碍重重，但通知使用权和 TileService 的拉起就往往不受限制，看起来「无障碍」服务的「障碍」更大一些……不过自启机制更多被<ruby>利用<rt>滥用</rt></ruby>于满足某些应用的非正常保活需求，所以限制也并非是坏事，感觉 Android 原生更应该加入相关的权限控制。  
> 其实比起限制自启，更为恐怖的还是某些应用在系统具有的「免杀白名单」……此处就不点名了。

此前研究的时候，一直以为进程崩溃后服务不从列表中移除是 bug （实际上一直以为进程死亡就导致无障碍服务被移除），还尝试了反复将自己的服务从无障碍服务列表移除再添加来「解决问题」，最后却导致了更严重的问题，比如发现服务被反复 bind ，或者虽然 bind 了(也就是 onServiceConnected 了)却无法添加无障碍窗口（导致崩溃）。现在看来其实都是自启被限制了的锅。

> 不过能出现这种 bug ，就算我有 99% 的问题，难道你 AccessibilityManagerService 就没有 1% 的责任？←_←

## 附录：Service 的 Restart 机制

Service 的运行有两种：启动(startService) 和绑定(bindService)，它们是被分别对待的。也就是说，服务可以同时处于被启动和被绑定两种状态，且互不影响（？）。而服务的 Restart （重启）机制自然也是分开处理的。

被绑定的服务自启与 bindService 时的 flag `BIND_AUTO_CREATE` 有关。这样的服务就会被系统自动拉起，直到显式调用 unbind 为止。在 `dumpsys activity services` 中，如果服务的某个绑定者具有这个 flag ，会显示一个 `CREATE` 。这个重启发生在服务所在进程异常退出的时候（也就是非系统主动杀死，具体来说就是 `IApplicationThread` binderDead 的时候）。当然，系统主动重启服务会等待一定时间，次数也是有限制的，且等待时间会随着次数的增加而增加。

> 实际上在 Manifest 声明的 persistence （持久）服务可以无限制地被系统地主动重启，当然这个就是属于系统 App 的特权了。

```
  * ServiceRecord{64da221 u0 com.fooview.android.fooview/.fvprocess.FooAccessibilityService}
    intent={cmp=com.fooview.android.fooview/.fvprocess.FooAccessibilityService}
    packageName=com.fooview.android.fooview
    processName=com.fooview.android.fooview:fv
    permission=android.permission.BIND_ACCESSIBILITY_SERVICE
    baseDir=/data/app/com.fooview.android.fooview-UobdMJWrMIOstRT8Xauzqw==/base.apk
    dataDir=/data/user/0/com.fooview.android.fooview
    app=ProcessRecord{c1586f 5534:com.fooview.android.fooview:fv/u0a211}
    hasBindingWhitelistingBgActivityStarts=true
    allowWhileInUsePermissionInFgs=true
    startForegroundCount=0
    recentCallingPackage=android
    createTime=-14h26m19s327ms startingBgTimeout=--
    lastActivity=-3h32m4s228ms restartTime=-3h32m4s228ms createdFromFg=true
    callerPackage=android
    Bindings:
    * IntentBindRecord{cd2556f CREATE}:
      intent={cmp=com.fooview.android.fooview/.fvprocess.FooAccessibilityService}
      binder=android.os.BinderProxy@e70627c
      requested=true received=true hasBound=true doRebind=false
      * Client AppBindRecord{bc3e705 ProcessRecord{f8c2255 1489:system/1000}}
        Per-process Connections:
          ConnectionRecord{f246f4e u0 CR FGSA CAPS com.fooview.android.fooview/.fvprocess.FooAccessibilityService:@b622a49}
    All Connections:
      ConnectionRecord{f246f4e u0 CR FGSA CAPS com.fooview.android.fooview/.fvprocess.FooAccessibilityService:@b622a49}
```

无障碍服务绑定：

```
frameworks/base/services/accessibility/java/com/android/server/accessibility/AccessibilityServiceConnection.java
bindLocked
```

服务重启：

```
frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
killServicesLocked
-> scheduleServiceRestartLocked
```

重启日志：

```
06-28 15:06:17.132  1489  2429 W ActivityManager: Scheduling restart of crashed service five.ec1cff.assistant/.AssistantService in 1000ms for connection
```

有关 启动服务 的重启可参考 → [Android Service重启恢复（Service进程重启）原理解析 - 简书](https://www.jianshu.com/p/7a659ca38510)

> 简单来说就是 onStartCommand 的返回值决定了是否重启，返回 `START_STICKY` 就可以自动重启。