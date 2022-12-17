# View.FLAG_WINDOW_IS_OBSCURED

不知道什么时候开始 play 会检查窗口遮挡了，由于我日常开着 FV 悬浮球之类的工具，在 play 里面总是被弹提示，而且不能执行安装等操作，非常烦人。

反编译 + 搜索了一下，发现这个窗口遮挡的信息是从 MotionEvent 里面来的。

[Android detect or block floating/overlaying apps - Stack Overflow](https://stackoverflow.com/questions/36923301/android-detect-or-block-floating-overlaying-apps)

如果 View 设置了 `setFilterTouchesWhenObscured` ，并且触摸事件有 `FLAG_WINDOW_IS_OBSCURED` 这个 flag ， `View#onFilterTouchEventForSecurity` 会返回 false 。自定义 View 可以重载这个方法，调用 super 看是不是 false 来判断是否被遮挡。

```java
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }
```

在 XML 中就是这样：

```
android:filterTouchesWhenObscured="true"
```

如果把这个过滤属性放在父 view ，可以阻止被遮挡的窗口下的点击事件向下传播

对应用来说，隐藏的最简单的方法就是 hook `View#onFilterTouchEventForSecurity` ，让它返回 true 。

如果从系统层面下手比较困难，因为 input 模块主要在 native 代码里面。

处理输入的代码被编译在动态库 `libinput.so` 中，input 事件通过 socket 传输(似乎是 binder + socket pair fd)。

```cpp
// frameworks/native/include/input/InputTransport.h
status_t InputPublisher::publishMotionEvent ->
status_t sendMessage(const InputMessage* msg);
```

这个函数每个版本的签名几乎都不一样

```cpp
    // 13
    status_t publishMotionEvent(uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source,
                                int32_t displayId, std::array<uint8_t, 32> hmac, int32_t action,
                                int32_t actionButton, int32_t flags, int32_t edgeFlags,
                                int32_t metaState, int32_t buttonState,
                                MotionClassification classification, const ui::Transform& transform,
                                float xPrecision, float yPrecision, float xCursorPosition,
                                float yCursorPosition, const ui::Transform& rawTransform,
                                nsecs_t downTime, nsecs_t eventTime, uint32_t pointerCount,
                                const PointerProperties* pointerProperties,
                                const PointerCoords* pointerCoords);

    // 12.1
    status_t publishMotionEvent(uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source,
                                int32_t displayId, std::array<uint8_t, 32> hmac, int32_t action,
                                int32_t actionButton, int32_t flags, int32_t edgeFlags,
                                int32_t metaState, int32_t buttonState,
                                MotionClassification classification, const ui::Transform& transform,
                                float xPrecision, float yPrecision, float xCursorPosition,
                                float yCursorPosition, uint32_t displayOrientation,
                                int32_t displayWidth, int32_t displayHeight, nsecs_t downTime,
                                nsecs_t eventTime, uint32_t pointerCount,
                                const PointerProperties* pointerProperties,
                                const PointerCoords* pointerCoords);

    // 12
    status_t publishMotionEvent(uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source,
                                int32_t displayId, std::array<uint8_t, 32> hmac, int32_t action,
                                int32_t actionButton, int32_t flags, int32_t edgeFlags,
                                int32_t metaState, int32_t buttonState,
                                MotionClassification classification, const ui::Transform& transform,
                                float xPrecision, float yPrecision, float xCursorPosition,
                                float yCursorPosition, int32_t displayWidth, int32_t displayHeight,
                                nsecs_t downTime, nsecs_t eventTime, uint32_t pointerCount,
                                const PointerProperties* pointerProperties,
                                const PointerCoords* pointerCoords);

    // 11
    status_t publishMotionEvent(uint32_t seq, int32_t eventId, int32_t deviceId, int32_t source,
                                int32_t displayId, std::array<uint8_t, 32> hmac, int32_t action,
                                int32_t actionButton, int32_t flags, int32_t edgeFlags,
                                int32_t metaState, int32_t buttonState,
                                MotionClassification classification, float xScale, float yScale,
                                float xOffset, float yOffset, float xPrecision, float yPrecision,
                                float xCursorPosition, float yCursorPosition, nsecs_t downTime,
                                nsecs_t eventTime, uint32_t pointerCount,
                                const PointerProperties* pointerProperties,
                                const PointerCoords* pointerCoords);

    // 10.0
    status_t publishMotionEvent(
            uint32_t seq,
            int32_t deviceId,
            int32_t source,
            int32_t displayId,
            int32_t action,
            int32_t actionButton,
            int32_t flags,
            int32_t edgeFlags,
            int32_t metaState,
            int32_t buttonState,
            MotionClassification classification,
            float xOffset,
            float yOffset,
            float xPrecision,
            float yPrecision,
            nsecs_t downTime,
            nsecs_t eventTime,
            uint32_t pointerCount,
            const PointerProperties* pointerProperties,
            const PointerCoords* pointerCoords);

    // 9.0, 8.1
    status_t publishMotionEvent(
            uint32_t seq,
            int32_t deviceId,
            int32_t source,
            int32_t displayId,
            int32_t action,
            int32_t actionButton,
            int32_t flags,
            int32_t edgeFlags,
            int32_t metaState,
            int32_t buttonState,
            float xOffset,
            float yOffset,
            float xPrecision,
            float yPrecision,
            nsecs_t downTime,
            nsecs_t eventTime,
            uint32_t pointerCount,
            const PointerProperties* pointerProperties,
            const PointerCoords* pointerCoords);

    // 8.0
    status_t publishMotionEvent(
            uint32_t seq,
            int32_t deviceId,
            int32_t source,
            int32_t action,
            int32_t actionButton,
            int32_t flags,
            int32_t edgeFlags,
            int32_t metaState,
            int32_t buttonState,
            float xOffset,
            float yOffset,
            float xPrecision,
            float yPrecision,
            nsecs_t downTime,
            nsecs_t eventTime,
            uint32_t pointerCount,
            const PointerProperties* pointerProperties,
            const PointerCoords* pointerCoords);
```

上面的 publishMotionEvent 符号在 `libinput.so` 被导出，且被 `libinputflinger.so` 引用，可以考虑用 plt hook 。

```
_ZN7android14InputPublisher18publishMotionEventEjiiiiNSt3__15arrayIhLm32EEEiiiiiiNS_20MotionClassificationEfffffffflljPKNS_17PointerPropertiesEPKNS_13PointerCoordsE
```

我们看看这个「遮挡」的标准是什么：

[master](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp;l=2655;drc=91609c37dec6a06dc90cffd0ee69ce873bae44e1)

```cpp
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
static bool canBeObscuredBy(const sp<WindowInfoHandle>& windowHandle,
                            const sp<WindowInfoHandle>& otherHandle) {
    // Compare by token so cloned layers aren't counted
    if (haveSameToken(windowHandle, otherHandle)) {
        return false;
    }
    auto info = windowHandle->getInfo();
    auto otherInfo = otherHandle->getInfo();
    if (otherInfo->inputConfig.test(WindowInfo::InputConfig::NOT_VISIBLE)) {
        return false;
    } else if (otherInfo->alpha == 0 &&
               otherInfo->inputConfig.test(WindowInfo::InputConfig::NOT_TOUCHABLE)) {
        // Those act as if they were invisible, so we don't need to flag them.
        // We do want to potentially flag touchable windows even if they have 0
        // opacity, since they can consume touches and alter the effects of the
        // user interaction (eg. apps that rely on
        // FLAG_WINDOW_IS_PARTIALLY_OBSCURED should still be told about those
        // windows), hence we also check for FLAG_NOT_TOUCHABLE.
        return false;
    } else if (info->ownerUid == otherInfo->ownerUid) {
        // If ownerUid is the same we don't generate occlusion events as there
        // is no security boundary within an uid.
        return false;
    } else if (otherInfo->inputConfig.test(gui::WindowInfo::InputConfig::TRUSTED_OVERLAY)) {
        return false;
    } else if (otherInfo->displayId != info->displayId) {
        return false;
    }
    return true;
}
```

[android 11](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r1:frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp;l=2081;drc=be737674afd67f67e7053d3c0c87ed5dc61f052d)

```cpp
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
static bool canBeObscuredBy(const sp<InputWindowHandle>& windowHandle,
                            const sp<InputWindowHandle>& otherHandle) {
    // Compare by token so cloned layers aren't counted
    if (haveSameToken(windowHandle, otherHandle)) {
        return false;
    }
    auto info = windowHandle->getInfo();
    auto otherInfo = otherHandle->getInfo();
    if (!otherInfo->visible) {
        return false;
    } else if (info->ownerPid == otherInfo->ownerPid) {
        // If ownerPid is the same we don't generate occlusion events as there
        // is no in-process security boundary.
        return false;
    } else if (otherInfo->isTrustedOverlay()) {
        return false;
    } else if (otherInfo->displayId != info->displayId) {
        return false;
    }
    return true;
}
```

[trusted overlay](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r1:frameworks/native/libs/input/InputWindow.cpp;l=46;drc=466cdea8d2de302f6be4d9f59816dd2c6078decf)

```cpp
// frameworks/native/libs/input/InputWindow.cpp
bool InputWindowInfo::isTrustedOverlay() const {
    return layoutParamsType == TYPE_INPUT_METHOD || layoutParamsType == TYPE_INPUT_METHOD_DIALOG ||
            layoutParamsType == TYPE_MAGNIFICATION_OVERLAY || layoutParamsType == TYPE_STATUS_BAR ||
            layoutParamsType == TYPE_NOTIFICATION_SHADE ||
            layoutParamsType == TYPE_NAVIGATION_BAR ||
            layoutParamsType == TYPE_NAVIGATION_BAR_PANEL ||
            layoutParamsType == TYPE_SECURE_SYSTEM_OVERLAY ||
            layoutParamsType == TYPE_DOCK_DIVIDER ||
            layoutParamsType == TYPE_ACCESSIBILITY_OVERLAY ||
            layoutParamsType == TYPE_INPUT_CONSUMER ||
            layoutParamsType == TYPE_TRUSTED_APPLICATION_OVERLAY;
}
```

Android 11 中，`TYPE_ACCESSIBILITY_OVERLAY` 是可信的，因此不会被看作是遮挡。

不过我日常用的 FV 悬浮球有一些 application overlay 的窗口(type 2038, 0x7f6)，其中最后一个窗口覆盖了整个屏幕，而且是 NOT_TOUCHABLE 的，因此被检测到了。

```
# dumpsys input
      4: name='Window{e79218b mode=1 rootTaskId=1 u0 com.fooview.android.fooview}', displayId=0, portalToDisplayId=-1, paused=false, hasFocus=false, hasWallpaper=false, visible=true, canReceiveKeys=false, flags=0x00010308, type=0x000007f6, frame=[0,908][33,1128], globalScale=1.000000, windowScale=(1.000000,1.000000), touchableRegion=[0,908][33,1128], inputFeatures=0x00000000, ownerPid=5138, ownerUid=10211, dispatchingTimeout=8000ms
      5: name='Window{240c017 mode=1 rootTaskId=1 u0 com.fooview.android.fooview}', displayId=0, portalToDisplayId=-1, paused=false, hasFocus=false, hasWallpaper=false, visible=true, canReceiveKeys=false, flags=0x00020018, type=0x000007f6, frame=[971,95][972,2356], globalScale=1.000000, windowScale=(1.000000,1.000000), touchableRegion=[971,95][972,2356], inputFeatures=0x00000000, ownerPid=5138, ownerUid=10211, dispatchingTimeout=8000ms
      12: name='Window{a5aef30 mode=1 rootTaskId=1 u0 com.fooview.android.fooview}', displayId=0, portalToDisplayId=-1, paused=false, hasFocus=false, hasWallpaper=false, visible=true, canReceiveKeys=false, flags=0x00010318, type=0x000007f6, frame=[0,2355][1080,2356], globalScale=1.000000, windowScale=(1.000000,1.000000), touchableRegion=[0,2355][1080,2356], inputFeatures=0x00000000, ownerPid=5138, ownerUid=10211, dispatchingTimeout=8000ms

# dumpsys window
  Window #22 Window{a5aef30 mode=1 rootTaskId=1 u0 com.fooview.android.fooview}:
    mDisplayId=0 rootTaskId=1 mSession=Session{d602148 5138:u0a10211} mClient=android.os.BinderProxy@8bd48e2
    mOwnerUid=10211 showForAllUsers=false package=com.fooview.android.fooview appop=SYSTEM_ALERT_WINDOW
    mAttrs={(0,0)(fillx1) gr=BOTTOM LEFT CENTER sim={adjust=pan} blurRatio=1.0 blurMode=0 ty=APPLICATION_OVERLAY fmt=TRANSPARENT
      fl=NOT_FOCUSABLE NOT_TOUCHABLE LAYOUT_IN_SCREEN LAYOUT_NO_LIMITS LAYOUT_INSET_DECOR
      pfl=
      fitTypes=NAVIGATION_BARS CAPTION_BAR}
    Requested w=1080 h=1 mLayoutSeq=32042
    mHasSurface=true isReadyForDisplay()=true mWindowRemovalAllowed=false
    WindowStateAnimator{290a64 }:
      Surface: shown=true layer=0 alpha=1.0 rect=(0.0,0.0) 1080 x 1 transform=(1.0, 0.0, 1.0, 0.0)
```