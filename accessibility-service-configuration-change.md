# 无障碍服务和配置变更

## 问题

我一直都有一个利用无障碍服务的 app ，其中内置了使用读写设置的方法在 app 主动拉起无障碍服务。

App 需要使用无障碍窗口，为了支持横屏和竖屏两种模式，自然要监听屏幕旋转。

一般来说， Activity 都是在旋转的时候自动 recreate ，当然也可以使用 onConfigurationChanged 判断屏幕旋转的状态（需要在 Manifest 声明）

实际上，onConfigurationChanged 是 [ComponentCallbacks](https://developer.android.com/reference/android/content/ComponentCallbacks#onConfigurationChanged(android.content.res.Configuration)) 的一部分，Application, Service 这些都继承了它(ComponentCallback2)，因此这些组件里面理论上也可以用 onConfigurationChanged 。

但是令我百思不得其解的是，我这个无障碍 app 竟然无法接收到 onConfigurationChanged 回调——无论是 Application, AccessibilityService ，甚至 Activity 也无法接到正确的配置变更，比如切换夜间模式，虽然 Activity 确实重建了，但主题没有发生变化。

经过实验，发现只要无障碍服务启动过，就无法得到回调，即使关闭，仍无法恢复，除非杀掉进程重启；正常来说，只要 App 进程启动，应该能收到回调的，即使所有 activity 退出，只留有 Application ，在一段时间内也可以收到。

进一步发现，这似乎和无障碍启动的方式有很大关系。

我的 App 里面有一个 Preference Activity ，其中有一个 SwitchPreference 用于打开无障碍服务，如果有写安全设置的权限，就会尝试启动。假如通过这种方式，就会出现上面的问题。

同样是改写设置，如果我们在不启动 activity 的情况下，直接 `settings put secure enabled_accessibility_services` ，竟然就一切正常了，配置变更可以被所有组件正常接收。

测试了 MIUI 和模拟器的 Android 11 ，都是如此。

## 分析

前置：[Configuration 变更时Activity的生命周期探究](https://juejin.cn/post/6844903869814669319)

给进程分发配置变更的主要逻辑在 `WindowProcessController`

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java
    @Override
    public void onConfigurationChanged(Configuration newGlobalConfig) {
        super.onConfigurationChanged(newGlobalConfig);
        updateConfiguration();
    }

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
        if (mThread == null) {
            if (Build.IS_DEBUGGABLE && mHasImeService) {
                // TODO (b/135719017): Temporary log for debugging IME service.
                Slog.w(TAG_CONFIGURATION, "Unable to send config for IME proc " + mName
                        + ": no app thread");
            }
            return;
        }
        if (DEBUG_CONFIGURATION) {
            Slog.v(TAG_CONFIGURATION, "Sending to proc " + mName + " new config " + config);
        }
        if (Build.IS_DEBUGGABLE && mHasImeService) {
            // TODO (b/135719017): Temporary log for debugging IME service.
            Slog.v(TAG_CONFIGURATION, "Sending to IME proc " + mName + " new config " + config);
        }

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

看起来和无障碍没什么关系，只是类中确实有和无障碍相关的：

```java
    /**
     * Called to notify {@link WindowProcessController} of a started service.
     *
     * @param serviceInfo information describing the started service.
     */
    public void onServiceStarted(ServiceInfo serviceInfo) {
        String permission = serviceInfo.permission;
        if (permission == null) {
            return;
        }

        // TODO: Audit remaining services for disabling activity override (Wallpaper, Dream, etc).
        switch (permission) {
            case Manifest.permission.BIND_INPUT_METHOD:
                mHasImeService = true;
                // Fall-through
            case Manifest.permission.BIND_ACCESSIBILITY_SERVICE:
            case Manifest.permission.BIND_VOICE_INTERACTION:
                // We want to avoid overriding the config of these services with that of the
                // activity as it could lead to incorrect display metrics. For ex, IME services
                // expect their config to match the config of the display with the IME window
                // showing.
                mIsActivityConfigOverrideAllowed = false;
                break;
            default:
                break;
        }
    }
```

如果进程启动过 `permission.BIND_ACCESSIBILITY_SERVICE` 权限的服务，就会设置 `mIsActivityConfigOverrideAllowed=false` 。默认来说是 true ，对于系统进程(uid 1000)则是 false 。

记得以前注入 systemui ，需要接收配置变更，然而 onConfigurationChanged 根本收不到，只好用广播接收器。

设置这个 field 的「持久性」与现象一致。难道就是这里导致的？

通过 frida 观察，启动无障碍前，我们的 `mIsActivityConfigOverrideAllowed` 是 true ，启动后确实变成了 false 。

尝试主动修改成，发现只要改成 true ，行为就恢复正常；但改成 false 并不是总能复现异常的。具体来说，要在启动 activity 前改成 true ，启动后改成 false ，才能复现上面的现象。

再看看这个 field 影响了什么：

```java
    /**
     * Check if activity configuration override for the activity process needs an update and perform
     * if needed. By default we try to override the process configuration to match the top activity
     * config to increase app compatibility with multi-window and multi-display. The process will
     * always track the configuration of the non-finishing activity last added to the process.
     */
    private void updateActivityConfigurationListener() {
        if (!mIsActivityConfigOverrideAllowed) {
            return;
        }

        for (int i = mActivities.size() - 1; i >= 0; i--) {
            final ActivityRecord activityRecord = mActivities.get(i);
            if (!activityRecord.finishing) {
                // Eligible activity is found, update listener.
                registerActivityConfigurationListener(activityRecord);
                return;
            }
        }

        // No eligible activities found, let's remove the configuration listener.
        unregisterActivityConfigurationListener();
    }

    void addActivityIfNeeded(ActivityRecord r) {
        // even if we already track this activity, note down that it has been launched
        setLastActivityLaunchTime(r.lastLaunchTime);
        if (mActivities.contains(r)) {
            return;
        }
        mActivities.add(r);
        updateActivityConfigurationListener();
    }

    void removeActivity(ActivityRecord r) {
        mActivities.remove(r);
        updateActivityConfigurationListener();
    }
```

看起来，每当进程的 activity 变化，需要寻找一个 activity ，通过 registerActivityConfigurationListener 注册自己为 ConfigurationListener ；假如 mIsActivityConfigOverrideAllowed 是 false ，就不会进行注册。

ConfigurationListener 是这样的接口：

```java
// frameworks/base/services/core/java/com/android/server/wm/ConfigurationContainerListener.java
public interface ConfigurationContainerListener {

    /** {@see ConfigurationContainer#onRequestedOverrideConfigurationChanged} */
    default void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {}

    /** Called when new merged override configuration is reported. */
    default void onMergedOverrideConfigurationChanged(Configuration mergedOverrideConfiguration) {}
}
```

ActivityRecord 和 WindowProcessController 都是 ConfigurationContainer ，在 `onConfigurationChanged` 中：

```java
// frameworks/base/services/core/java/com/android/server/wm/ConfigurationContainer.java
    public void onConfigurationChanged(Configuration newParentConfig) {
        mResolvedTmpConfig.setTo(mResolvedOverrideConfiguration);
        resolveOverrideConfiguration(newParentConfig);
        mFullConfiguration.setTo(newParentConfig);
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
        onMergedOverrideConfigurationChanged();
        if (!mResolvedTmpConfig.equals(mResolvedOverrideConfiguration)) {
            // This depends on the assumption that change-listeners don't do
            // their own override resolution. This way, dependent hierarchies
            // can stay properly synced-up with a primary hierarchy's constraints.
            // Since the hierarchies will be merged, this whole thing will go away
            // before the assumption will be broken.
            // Inform listeners of the change.
            for (int i = mChangeListeners.size() - 1; i >= 0; --i) {
                mChangeListeners.get(i).onRequestedOverrideConfigurationChanged(
                        mResolvedOverrideConfiguration);
            }
        }
        for (int i = mChangeListeners.size() - 1; i >= 0; --i) {
            mChangeListeners.get(i).onMergedOverrideConfigurationChanged(
                    mMergedOverrideConfiguration);
        }
        for (int i = getChildCount() - 1; i >= 0; --i) {
            final ConfigurationContainer child = getChildAt(i);
            child.onConfigurationChanged(mFullConfiguration);
        }
    }
```

现在我们理解什么是 `Activity Config Override` 了。也就是说，进程不使用全局的 Configuration ，而是 Activity 专属的。

我们注意到，当 Activity 启动的时候，此时无障碍服务未启动，`mIsActivityConfigOverrideAllowed=true` ，因此调用了 `registerActivityConfigurationListener` ；

现在启动了无障碍服务，导致 allowd 为 true ，因此 Activity 将要 finish 的时候， `unregisterActivityConfigurationListener` 并没有被调用。

那么具体怎么会导致无法收到配置变更呢？

hook 观察发现 WindowProcessController.dispatchConfigurationChanged 没有被调用，显然 updateConfiguration 一定会被调用，因此只可能是下面返回了：

```java
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
```

也就是说，getConfiguration 得到的的，和上次汇报的是一样的。

注册：

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java
    private void registerActivityConfigurationListener(ActivityRecord activityRecord) {
        if (activityRecord == null || activityRecord.containsListener(this)) {
            return;
        }
        // A process can only register to one activityRecord to listen to the override configuration
        // change. Unregister existing listener if it has one before register the new one.
        unregisterDisplayConfigurationListener();
        unregisterActivityConfigurationListener();
        mConfigActivityRecord = activityRecord;
        activityRecord.registerConfigurationChangeListener(this);
    }
```

activityRecord.registerConfigurationChangeListener 最终调用到 ConfigurationContainer 的：

```java
    void registerConfigurationChangeListener(ConfigurationContainerListener listener) {
        if (mChangeListeners.contains(listener)) {
            return;
        }
        mChangeListeners.add(listener);
        listener.onRequestedOverrideConfigurationChanged(mResolvedOverrideConfiguration);
        listener.onMergedOverrideConfigurationChanged(mMergedOverrideConfiguration);
    }
```

注册同时还要回调 WPC 的 onRequestedOverrideConfigurationChanged ，最终又来到 ConfigurationContainer ：

```java
// frameworks/base/services/core/java/com/android/server/wm/ConfigurationContainer.java
    /**
     * Update override configuration and recalculate full config.
     * @see #mRequestedOverrideConfiguration
     * @see #mFullConfiguration
     */
    public void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {
        // Pre-compute this here, so we don't need to go through the entire Configuration when
        // writing to proto (which has significant cost if we write a lot of empty configurations).
        mHasOverrideConfiguration = !Configuration.EMPTY.equals(overrideConfiguration);
        mRequestedOverrideConfiguration.setTo(overrideConfiguration);
        // Update full configuration of this container and all its children.
        final ConfigurationContainer parent = getParent();
        onConfigurationChanged(parent != null ? parent.getConfiguration() : Configuration.EMPTY);
    }
```

会将 override 值 `mRequestedOverrideConfiguration` 设为一个非 EMPTY 值。

override 值怎么用呢？在 `ConfigurationContainer#onConfigurationChanged` ：

```java
// frameworks/base/services/core/java/com/android/server/wm/ConfigurationContainer.java
        mResolvedTmpConfig.setTo(mResolvedOverrideConfiguration);
        resolveOverrideConfiguration(newParentConfig);
        mFullConfiguration.setTo(newParentConfig);
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);

    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration);
    }
```

resolveOverrideConfiguration 默认直接把 mResolvedOverrideConfiguration 设为 mRequestedOverrideConfiguration ，然后 mFullConfiguration 被 mResolvedOverrideConfiguration 覆盖，最终返回的应该是一个 override 和 onConfigurationChanged 传进来的混合值。

正常来说，Activity 配置变更的时候，ActivityRecord 会通知 WPC ，设置这个 override 值，但是现在 activity 退出了，仍然没有被 unregister ，因此这个值一直被保留，导致 getConfiguration 每次都返回一样的值，配置无法变更。

假如正确调用了 unregister ，会 `onMergedOverrideConfigurationChanged` 一个 `Configuration.EMPTY` 。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java
    private void unregisterActivityConfigurationListener() {
        if (mConfigActivityRecord == null) {
            return;
        }
        mConfigActivityRecord.unregisterConfigurationChangeListener(this);
        mConfigActivityRecord = null;
        onMergedOverrideConfigurationChanged(Configuration.EMPTY);
    }

// frameworks/base/services/core/java/com/android/server/wm/ConfigurationContainer.java
    public void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {
        // Pre-compute this here, so we don't need to go through the entire Configuration when
        // writing to proto (which has significant cost if we write a lot of empty configurations).
        mHasOverrideConfiguration = !Configuration.EMPTY.equals(overrideConfiguration);
        mRequestedOverrideConfiguration.setTo(overrideConfiguration);
        // Update full configuration of this container and all its children.
        final ConfigurationContainer parent = getParent();
        onConfigurationChanged(parent != null ? parent.getConfiguration() : Configuration.EMPTY);
    }
```

此时 mRequestedOverrideConfiguration 也会被重置，最终就可以正常更新配置。

正常来说启动无障碍服务都是在设置里面手动打开的，因此不会碰到这种问题。

MT 管理器里面有个自动开启无障碍的功能，因此同样的问题也能复现。

看了一下 blame ，这个 bug 在 2021 年 9 月被汇报，并在 2021 年 10 月提交了修复，看起来在 Android 12.1 才修复了这个问题。

[_/android/platform/frameworks/base - Android Code Search](https://cs.android.com/android/_/android/platform/frameworks/base/+/425a952eb883516d7f2583c1e08e9ebc03bbaedf)

[The voice interaction service may never receive new configuration if it was started by an activity &#x5b;201457378&#x5d; - Visible to Public - Issue Tracker](https://issuetracker.google.com/issues/201457378)

## 对策

显然，不在应用中内建「无障碍自启」是最好的办法（x

如果一定要有这个功能，且应用的其他功能需要 Configuration Changed ，应该确保启动无障碍服务的同一个进程没有任何 activity （可以使用多进程，无障碍服务独立一个进程来确保）。


## 总结

看来主动启动无障碍的坑还是太多了，Android 11 前有 crash 状态难以解除，后有这个 listener 泄露，为了分析这些问题又浪费了大量时间，也许这就是我这个 app 永远写不出来的原因罢。

**远离魔法，从你我做起！**
