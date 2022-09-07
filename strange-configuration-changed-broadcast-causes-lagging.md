# 配置变更广播导致卡顿？

```js
WPC=use('com.android.server.wm.WindowProcessController');
t=trace(WPC.dispatchConfigurationChange);
```

正常分发 ConfigurationChanged （以屏幕旋转为例）

```js
[12] method com.android.server.wm.WindowProcessController#dispatchConfigurationChange called @ Thread[android.display,5,main]
this: type=undefined, realType=class:com.android.server.wm.WindowProcessController, value=ProcessRecord{6952c25 24498:com.android.inputmethod.latin/u0a107}
arg0: type=android.content.res.Configuration, realType=class:android.content.res.Configuration, value={1.0 310mcc260mnc [zh_CN_#Hans,en_US] ldltr sw392dp w781dp h362dp 440dpi nrml long land finger qwerty/v/v dpad/v winConfig={ mBounds=Rect(0, 0 - 2280, 1080) mAppBounds=Rect(0, 0 - 2148, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_90} s.45}
  com.android.server.wm.WindowProcessController.dispatchConfigurationChange(Native Method)
  com.android.server.wm.WindowProcessController.updateConfiguration(WindowProcessController.java:1201)
  com.android.server.wm.WindowProcessController.onConfigurationChanged(WindowProcessController.java:1167)
  com.android.server.wm.ConfigurationContainer.onRequestedOverrideConfigurationChanged(ConfigurationContainer.java:174)
  com.android.server.wm.ConfigurationContainer.onConfigurationChanged(ConfigurationContainer.java:128)
  com.android.server.wm.WindowContainer.onConfigurationChanged(WindowContainer.java:320)
  com.android.server.wm.DisplayContent.onConfigurationChanged(DisplayContent.java:2277)
  com.android.server.wm.ConfigurationContainer.onRequestedOverrideConfigurationChanged(ConfigurationContainer.java:174)
  com.android.server.wm.WindowContainer.onRequestedOverrideConfigurationChanged(WindowContainer.java:751)
  com.android.server.wm.DisplayContent.onRequestedOverrideConfigurationChanged(DisplayContent.java:5330)
  com.android.server.wm.DisplayContent.performDisplayOverrideConfigUpdate(DisplayContent.java:5301)
  com.android.server.wm.ActivityTaskManagerService.updateGlobalConfigurationLocked(ActivityTaskManagerService.java:5309)
  com.android.server.wm.DisplayContent.updateDisplayOverrideConfigurationLocked(DisplayContent.java:5275)
  com.android.server.wm.DisplayContent.updateDisplayOverrideConfigurationLocked(DisplayContent.java:5252)
  com.android.server.wm.DisplayContent.sendNewConfiguration(DisplayContent.java:1294)
  com.android.server.wm.DisplayRotation.continueRotation(DisplayRotation.java:538)
  com.android.server.wm.DisplayRotation.access$200(DisplayRotation.java:77)
  com.android.server.wm.DisplayRotation$2.lambda$continueRotateDisplay$0(DisplayRotation.java:218)
  com.android.server.wm.-$$Lambda$DisplayRotation$2$37vRmD77aVmzN2ixs0KjlN8wUX4.accept(Unknown Source:10)
  com.android.internal.util.function.pooled.PooledLambdaImpl.doInvoke(PooledLambdaImpl.java:292)
  com.android.internal.util.function.pooled.PooledLambdaImpl.invoke(PooledLambdaImpl.java:201)
  com.android.internal.util.function.pooled.OmniFunction.run(OmniFunction.java:97)
  android.os.Handler.handleCallback(Handler.java:938)
  android.os.Handler.dispatchMessage(Handler.java:99)
  android.os.Looper.loop(Looper.java:223)
  android.os.HandlerThread.run(HandlerThread.java:67)
  com.android.server.ServiceThread.run(ServiceThread.java:44)
```

一般来说，这样只会分发到显示窗口的进程。

finishReceiver 导致的：

```js
[16] method com.android.server.wm.WindowProcessController#dispatchConfigurationChange called @ Thread[Binder:23227_C,5,main]
this: type=undefined, realType=class:com.android.server.wm.WindowProcessController, value=ProcessRecord{7265af 24908:com.topjohnwu.magisk/u0a131}
arg0: type=android.content.res.Configuration, realType=class:android.content.res.Configuration, value={1.0 310mcc260mnc [zh_CN_#Hans,en_US] ldltr sw392dp w781dp h362dp 440dpi nrml long land finger qwerty/v/v dpad/v winConfig={ mBounds=Rect(0, 0 - 2280, 1080) mAppBounds=Rect(0, 0 - 2148, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_90} s.45}
  com.android.server.wm.WindowProcessController.dispatchConfigurationChange(Native Method)
  com.android.server.wm.WindowProcessController.onProcCachedStateChanged(WindowProcessController.java:1333)
  com.android.server.am.ProcessRecord.setCached(ProcessRecord.java:799)
  com.android.server.am.OomAdjuster.computeOomAdjLocked(OomAdjuster.java:1125)
  com.android.server.am.OomAdjuster.updateOomAdjLockedInner(OomAdjuster.java:561)
  com.android.server.am.OomAdjuster.updateOomAdjLocked(OomAdjuster.java:363)
  com.android.server.am.ActivityManagerService.updateOomAdjLocked(ActivityManagerService.java:18158)
  com.android.server.am.ActivityManagerService.trimApplicationsLocked(ActivityManagerService.java:18412)
  com.android.server.am.ActivityManagerService.finishReceiver(ActivityManagerService.java:16857)
  android.app.IActivityManager$Stub.onTransact(IActivityManager.java:2310)
  com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2883)
  android.os.Binder.execTransactInternal(Binder.java:1159)
  android.os.Binder.execTransact(Binder.java:1123)
```

跟踪这个 finishReceiver 的来源：

首先尝试 hook 然后 `Binder.getCallingPid` ，然而这个方法是 oneway 的，pid 总是返回 0 ，试图通过丢异常来找发送者也不好使，因为 oneway call 也不会往调用进程丢异常。

finishReceiver 有一个 IIntentReceiver 的 binder ，尝试在 broadcastQueue （有三个不同的）中获取它的 BroadcastReceiver 对象，然而总是返回 null ：

```js
[Remote::PID::23227]-> AMS.finishReceiver.implementation = function(...args) { console.log(this.mFgBroadcastQueue.value
.getMatchingOrderedReceiver(args[0])); return this.finishReceiver(...args); }
function
[Remote::PID::23227]-> null
android.os.BinderProxy@8069149 null
[Remote::PID::23227]-> AMS.finishReceiver.implementation = function(...args) { console.log(java.lang.Object.toString.call(args[0]), this.mOffloadBroadcastQueue.value.getMatchingOrderedReceiver(args[0])); return this.finishReceiver(...args
); }
function
[Remote::PID::23227]-> android.os.BinderProxy@8069149 null
[Remote::PID::23227]-> AMS.finishReceiver.implementation = function(...args) { console.log(java.lang.Object.toString.call(args[0]), this.mBgBroadcastQueue.value.getMatchingOrderedReceiver(args[0])); return this.finishReceiver(...args); }
function
[Remote::PID::23227]-> android.os.BinderProxy@8069149 null
[Remote::PID::23227]->
[Remote::PID::23227]-> AMS.finishReceiver.implementation = null
null
```

> 广播相关可参考：[Broadcast 源码分析 | 小强的开发笔记](https://solarqiang.github.io/posts/3752956090/)

注意到这个 binder 总是同一个对象，干脆利用 Android 11 的 transact 后门，暴力获取 PID ：

```js
[Remote::PID::23227]-> b=null
AMS.finishReceiver.implementation = function(...args) {
    b = Java.retain(args[0]);
    return this.finishReceiver(...args);
}
function
// 此处触发屏幕旋转，得到 binder
[Remote::PID::23227]-> b==null
false
[Remote::PID::23227]-> Parcel = Java.use('android.os.Parcel')
result = Parcel.obtain(), data = Parcel.obtain();
// 利用 _PID transact 获取对方 pid
b.transact(1599097156, data, result, 0);
data.recycle();
[Remote::PID::23227]-> result
"<instance: android.os.Parcel>"
[Remote::PID::23227]-> result.readInt()
4487
```

这个 pid 竟然是 lspd ！

```sh
130|generic_x86_64:/ # ps -p 4487
USER            PID   PPID     VSZ    RSS WCHAN            ADDR S NAME
system         4487      1 12394572 113436 do_epoll_+         0 S lspd
```

在 lsp 源码中搜索 `finishReceiver` 找到了这样的代码：

```java
// daemon/src/main/java/org/lsposed/lspd/service/LSPosedService.java
    private void registerConfigurationReceiver() {
        try {
            IntentFilter intentFilter = new IntentFilter();
            intentFilter.addAction(Intent.ACTION_CONFIGURATION_CHANGED);

            ActivityManagerService.registerReceiver("android", null, new IIntentReceiver.Stub() {
                @Override
                public void performReceive(Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                    getExecutorService().submit(() -> dispatchConfigurationChanged(intent));
                    try {
                        ActivityManagerService.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                    } catch (Throwable e) {
                        Log.e(TAG, "finish receiver", e);
                    }
                }
            }, intentFilter, null, 0, 0);
        } catch (Throwable e) {
            Log.e(TAG, "register configuration receiver", e);
        }
        Log.d(TAG, "registered configuration receiver");
    }
```

通过上面的文章，我们知道 `finishReceiver` 似乎是广播的重要环节，一个应用的广播处理完成后要主动通知 ams 继续发送广播，让下一个应用处理……

而在 finishReceiver 中一定有 trimApplication ，在这里进而导致了 ConfigurationChanged 被分发到所有进程。

难道注册了 `CONFIGURATION_CHANGED` 广播一定会导致这样吗？

我们来做一个实验：启动一个 app ，分别启动 activity 并执行以下两种操作：

1. 保持 activity 在前台  
2. 注册 CONFIGURATION_CHANGED 广播，并 finish activity  

同时，应用的 Application.onConfigurationChanged 、 Activity.onConfigurationChanged 和 BroadcastReceiver 都会 log 以表示自己接收到了配置变更。

在关闭 lsp 的情况下，比较发现，对于旋转屏幕导致的 Configuration Change ，都不会出现 finishReceiver 而导致分发到所有进程，并且，Activity 不可见时， Application 的 onConfigurationChanged 始终不被调用。

而开启 lsp 的情况下， Application 总是能接收到 onConfigurationChanged 调用。

## 修改 LSP 源码

研究发现，`finishReceiver` 一般用于有序广播，所以并不是每次都要调用的，而 LSPosed 在 lspd 中注册的 IntentReceiver 都会在最后调用一次，于是我们把它改成仅 `ordered` 时调用，编译到设备上运行。

在 AVD 上测试，发现分发到的进程数量减少了，不再出现 trimApplication 导致的分发。

然而在真机(MIUI)测试，发现它在即使是屏幕旋转导致的分发，也仍然分发到了近百个进程……旋转屏幕还是卡顿的。

---

> 还有另一个

```js
[24] method com.android.server.wm.WindowProcessController#dispatchConfigurationChange called @ Thread[Binder:30246_8,5,main]
this: type=undefined, realType=class:com.android.server.wm.WindowProcessController, value=ProcessRecord{b91adae 32041:com.android.settings/1000}
arg0: type=android.content.res.Configuration, realType=class:android.content.res.Configuration, value={1.0 310mcc260mnc [zh_CN_#Hans,en_US] ldltr sw392dp w781dp h362dp 440dpi nrml long land finger qwerty/v/v dpad/v winConfig={ mBounds=Rect(0, 0 - 2280, 1080) mAppBounds=Rect(0, 0 - 2148, 1080) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_90} s.41}
  com.android.server.wm.WindowProcessController.dispatchConfigurationChange(Native Method)
  com.android.server.wm.WindowProcessController.onProcCachedStateChanged(WindowProcessController.java:1333)
  com.android.server.am.ProcessRecord.setCached(ProcessRecord.java:799)
  com.android.server.am.OomAdjuster.computeOomAdjLocked(OomAdjuster.java:1125)
  com.android.server.am.OomAdjuster.updateOomAdjLockedInner(OomAdjuster.java:561)
  com.android.server.am.OomAdjuster.updateOomAdjLocked(OomAdjuster.java:363)
  com.android.server.am.ActivityManagerService.updateOomAdjLocked(ActivityManagerService.java:18158)
  com.android.server.am.BroadcastQueue.processNextBroadcastLocked(BroadcastQueue.java:1044)
  com.android.server.am.ActivityManagerService.finishReceiver(ActivityManagerService.java:16854)
  android.app.IActivityManager$Stub.onTransact(IActivityManager.java:2310)
  com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2883)
  android.os.Binder.execTransactInternal(Binder.java:1159)
  android.os.Binder.execTransact(Binder.java:1123)

```

# 最终结论

看上去 ConfigurationChanged 会传播到非 cached 状态的进程（具体可参考 WindowProcessController 和 ProcessRecord）。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java
// onConfigurationChanged ->
    private void updateConfiguration() {
        final Configuration config = getConfiguration();
        if (mLastReportedConfiguration.diff(config) == 0) {
            // Nothing changed.
            if (Build.IS_DEBUGGABLE && mHasImeService) {
                // TODO (b/135719017): Temporary log for debugging IME service.
                Slog.w(TAG_CONFIGURATION, "Current config: " + config
                        + " unchanged for IME proc " + mName);
            }
            return;
        }

        if (mListener.isCached()) {
            // This process is in a cached state. We will delay delivering the config change to the
            // process until the process is no longer cached.
            if (mPendingConfiguration == null) {
                mPendingConfiguration = new Configuration(config);
            } else {
                mPendingConfiguration.setTo(config);
            }
            return;
        }

        dispatchConfigurationChange(config);
    }

    private void dispatchConfigurationChange(Configuration config) {
        // ...

        try {
            config.seq = mAtm.increaseConfigurationSeqLocked();
            mAtm.getLifecycleManager().scheduleTransaction(mThread,
                    ConfigurationChangeItem.obtain(config));
            setLastReportedConfiguration(config);
        } catch (Exception e) {
            Slog.e(TAG_CONFIGURATION, "Failed to schedule configuration change", e);
        }
    }
```

其中 `mListener` 是 `WindowProcessListener` ， `ProcessRecord` 实现了该接口。

`isCached` 取决于 `ProcessRecord.mCached` 。

可以通过 `dumpsys` 简单确定哪些进程属于 cached 状态。

```sh
# dumpsys activity lru
ACTIVITY MANAGER LRU PROCESSES (dumpsys activity lru)
  Activities:
  #106: fg     TOP  LCM 4162:com.miui.home/u0a88 act:activities
  #105: cch    CRE  --- 14129:com.absinthe.libchecker/u0a246 act:recents
  #104: prcp   LAST --- 12114:tv.danmaku.bili/u0a209 act:activities|recents
  #103: prcp   LAST --- 12384:tv.danmaku.bili:download/u0a209 act:client
  #102: prcp   LAST --- 12460:tv.danmaku.bili:ijkservice/u0a209 act:client
```

```
AMS#dumpLruLocked
   #dumpLruEntryLocked
```

第一列字母： `ProcessList.makeOomAdjString` ，其中带有 `cch` 的属于缓存的进程。

第二列字母： `ProcessList.makeProcStateString` ，其中带有 `CAC` 或者 `CACC` 的属于缓存的进程。

值得注意的是，前者取决于 `OomAdj(setAdj)` ，后者取决于 `ProcState(mCurProcState)` ，与 `ProcessRecord` 的 `mCached` 都是各自独立的字段，因此仅凭这两个值也不能保证一定是 cached （不过它既然都写 cache 在上面了，那多半就是）。

而实际运行的系统中，处于 cached 状态的进程少之又少……以我的 miui 为例，平时运行 120 个进程，只有 30 个处于 cached 状态，即不到 25% 的概率。

那么卡顿是否由 onConfigurationChanged 分发到大量进程造成呢？我暂时无法下结论，但对比了 bilibili 和其他视频播放器的旋转屏幕速度，让我感觉，这个卡顿是 bilibili 自身的拉跨所造成的可能性更大一些。

总之，看起来上面又是我做的一些没用的研究——总之，想解决性能问题，试图通过任何 hack 手段「加速」别人写的屎山软件，只能是异想天开；随着软件版本的更迭，屎山只会更大，无用的功能只会不断堆叠，资源消耗只会更多，指望软件厂商的优化更是天方夜谭了，也许只有我们适应「时代的发展」，不断更新更快更强更贵的手机，才能解决问题罢。
