# 读 Riru-ClipboardWhitelist 源码

## ClipboardWhiteList

### 原理

https://github.com/Kr328/Riru-ClipboardWhitelist/blob/main/module/src/main/java/com/github/kr328/clipboard/ClipboardProxy.java#L1

将白名单中的 app 进行的 IClipboard 的 binder 调用来源（包括包名和 uid）换成默认输入法的。

方法还是传统的 Binder Proxy ，记得以前是用 Java proxy 实现的，似乎近期改换了实现。

比较有意思的是 uid 的替换，因为 uid 不通过调用参数传递。在 `Binder.restoreCallingIdentity` 中做手脚，替换 uid ，使得系统通过 `Binder.getCallingUid` 读取到的就是我们想让它读到的 uid 。

ClipboardService 中在 `clipboardAccessAllowed` 方法处理的剪贴板访问权限

对于读剪贴板操作来说，`clipboardAccessAllowed` 可以通过。对于监听器来说，系统会记录调用者的包名，在剪贴板变更，分发回调的时候再次调用 `clipboardAccessAllowed` ，因此需要修改记录的包名。

疑问：为什么不直接用 `android` 或者 `com.android.systemui`

## ZygoteLoader

### customize.sh

ZygoteLoader 的 gradle plugin 收集插件自身和用户定义的 sh ，按照文件名顺序将不同阶段任务的 sh 整合到最终生成一个的 `customize.sh` 中依次调用，用户可以自行扩展。

### 数据目录

https://github.com/Kr328/ZygoteLoader/blob/16d550b499fc273191f63987ff0971efc797646c/runtime/src/main/assets/customize.d/40-initialize-data-directory.sh#L24

数据目录在 sh 里面初始化，随机生成名字写到 `/data/system/zloader-xxxxxx` ，并记录到 `/data/adb/zloader/data-root-directory` 里面（可能是用于共享）

还会写到模块的 module.prop 。ZygoteLoader 会读取 module.prop 上报到 java 层，便于修改配置。
