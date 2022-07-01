> 下面研究两个同样会影响所有 ui 绘制的设置，以及如何用简单的 shell 命令修改。

# 深色模式

比较开启深色模式前后的 system settings ，发现变化的值（MIUI 12.5 上）：

```
# settings list system > d1 (d2)
# diff d1 d2
73c73
< dark_mode_enable=0
---
> dark_mode_enable=1
208c208
< smart_dark_enable=0
---
> smart_dark_enable=1
```

直接 `settings put` 并不能切换深色模式，看上去系统并不会监视值的变化，需要我们手动触发。

搜索设置相关源码（其实是反编译，直接搜上面的 key 找不到），发现是通过 UiModeManager 设置的，这是一个系统服务 (`uimode`)。

```
frameworks/base/core/java/android/app/UiModeManager.java
frameworks/base/core/java/android/app/IUiModeManager.aidl
```

它实现了 ShellCommand ，可以直接通过 shell 命令控制深色模式。

```
# service list | grep uimode
198     uimode: [android.app.IUiModeManager]
# cmd uimode
UiModeManager service (uimode) commands:
  help
    Print this help text.
  night [yes|no|auto|custom]
    Set or read night mode.
  time [start|end] <ISO time>
    Set custom start/end schedule time (night mode must be set to custom to apply).

# cmd uimode night yes
Night mode: yes
```

# 显示布局边界

这个设置存在了 sysprops 里面，同样 diff ：

```
# getprop > d1 (d2)
# diff d1 d2
62c62
< [debug.layout]: [false]
---
> [debug.layout]: [true]
```

直接 setprop ，也不会立即影响已有的 view ，只对新创建的 view 有效。

搜索 `debug_layout` ，在设置中：

```
packages/apps/Settings/src/com/android/settings/development/ShowLayoutBoundsPreferenceController.java
```

> 也可以找一找它的 Tile

```java
    @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        final boolean isEnabled = (Boolean) newValue;
        DisplayProperties.debug_layout(isEnabled);
        SystemPropPoker.getInstance().poke();
        return true;
    }
```

DisplayProperties 似乎是生成的类，反编译看到仅仅是调用 native 代码设置了 prop ，因此关键在下面的 `SystemPropPoker` 。

Poker 会创建一个 AsyncTask ，check 所有系统服务，并向它们发送一个 transact ，code 为 `IBinder.SYSPROPS_TRANSACTION`

```java
// frameworks/base/packages/SettingsLib/src/com/android/settingslib/development/SystemPropPoker.java
        @Override
        protected Void doInBackground(Void... params) {
            String[] services = listServices();
            if (services == null) {
                Log.e(TAG, "There are no services, how odd");
                return null;
            }
            for (String service : services) {
                IBinder obj = checkService(service);
                if (obj != null) {
                    Parcel data = Parcel.obtain();
                    try {
                        obj.transact(IBinder.SYSPROPS_TRANSACTION, data, null, 0);
                    } catch (RemoteException e) {
                        // Ignore
                    } catch (Exception e) {
                        Log.i(TAG, "Someone wrote a bad service '" + service
                                + "' that doesn't like to be poked", e);
                    }
                    data.recycle();
                }
            }
            return null;
        }
```

这个 code 值为 `1599295570` ，看起来是专门用于通知系统属性改变的。

```java
// frameworks/base/core/java/android/os/IBinder.java
    /** @hide */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    int SYSPROPS_TRANSACTION = ('_'<<24)|('S'<<16)|('P'<<8)|'R';
```

尝试修改 `debug.layout` 为 true 后给 `window`, `SurfaceFlinger`, `activity` 发这个 transact ，发现只有 `activity` 会提醒到 view 绘制边界。

于是利用 shell 命令开关「显示布局边界」如下：

```
# 开启
setprop debug.layout true; service call activity 1599295570
# 关闭
setprop debug.layout false; service call activity 1599295570
```

> 在 ViewRootImpl 中的 `loadSystemProperties` 负责处理这个属性的变化。在 WindowManagerGlobal 中通过 SystemProperties.addChangeCallback 设置了监听器。

> 参考：[“显示布局边界”的原理 | 姜康的技术博客](https://www.jiangkang.tech/2021/01/19/android/xian-shi-bu-ju-bian-jie-de-yuan-li/#toc-heading-2)

# 附录：props 的更新回调机制  

在 native 层 Binder 的 onTransact 中处理了 SYSPROPS_TRANSACTION ：

```cpp
// frameworks/native/libs/binder/Binder.cpp
status_t BBinder::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t /*flags*/)
{
    switch (code) {
        // ...
        case SYSPROPS_TRANSACTION: {
            report_sysprop_change();
            return NO_ERROR;
        }

        default:
            return UNKNOWN_TRANSACTION;
    }
}
```

而 AMS 中也有一个 onTransact ：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        if (code == SYSPROPS_TRANSACTION) {
            // We need to tell all apps about the system property change.
            ArrayList<IBinder> procs = new ArrayList<IBinder>();
            synchronized (mProcLock) {
                final ArrayMap<String, SparseArray<ProcessRecord>> pmap =
                        mProcessList.getProcessNamesLOSP().getMap();
                final int numOfNames = pmap.size();
                for (int ip = 0; ip < numOfNames; ip++) {
                    SparseArray<ProcessRecord> apps = pmap.valueAt(ip);
                    final int numOfApps = apps.size();
                    for (int ia = 0; ia < numOfApps; ia++) {
                        ProcessRecord app = apps.valueAt(ia);
                        final IApplicationThread thread = app.getThread();
                        if (thread != null) {
                            procs.add(thread.asBinder());
                        }
                    }
                }
            }

            int N = procs.size();
            for (int i=0; i<N; i++) {
                Parcel data2 = Parcel.obtain();
                try {
                    procs.get(i).transact(IBinder.SYSPROPS_TRANSACTION, data2, null,
                            Binder.FLAG_ONEWAY);
                } catch (RemoteException e) {
                }
                data2.recycle();
            }
        }
        try {
            return super.onTransact(code, data, reply, flags);
        } catch (RuntimeException e) {
            // The activity manager only throws certain exceptions intentionally, so let's
            // log all others.
            if (!(e instanceof SecurityException
                    || e instanceof IllegalArgumentException
                    || e instanceof IllegalStateException)) {
                Slog.wtf(TAG, "Activity Manager Crash."
                        + " UID:" + Binder.getCallingUid()
                        + " PID:" + Binder.getCallingPid()
                        + " TRANS:" + code, e);
            }
            throw e;
        }
    }
```

AMS 收到 SYSPROPS_TRANSACTION 后，向所有 app 的 IApplicationThread 也做这个 transact ，因此每个 app 进程都收到了属性变化的回调。