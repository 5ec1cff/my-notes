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

对应用来说，隐藏的最简单的方法就是 hook `View#onFilterTouchEventForSecurity` ，让它返回 true 。

如果从系统层面下手比较困难，因为 input 模块主要在 native 代码里面。

处理输入的代码被编译在动态库 `libinput.so` 中，input 事件通过 socket 传输(似乎是 binder + socket pair fd)。

```cpp
status_t InputPublisher::publishMotionEvent ->
status_t sendMessage(const InputMessage* msg);
```

上面的 publishMotionEvent 符号在 `libinput.so` 被导出，且被 `libinputflinger.so` 引用，可以考虑用 plt hook 。

```
_ZN7android14InputPublisher18publishMotionEventEjiiiiNSt3__15arrayIhLm32EEEiiiiiiNS_20MotionClassificationEfffffffflljPKNS_17PointerPropertiesEPKNS_13PointerCoordsE
```
