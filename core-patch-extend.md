# 核心破解

核心破解（[CorePatch](https://github.com/LSPosed/CorePatch)）是一个针对 PMS 进行修改的 Xposed 模块，它具有以下功能：

1. 允许缺少签名、签名错误的包安装到系统中；  
2. 允许降级安装；
3. 允许与系统中存在的包不同签名的新 apk 直接覆盖安装。（「禁用 APK 签名验证」）  

但是它并不能解决所有的问题，至今核心破解还存在两大难题：

## 两大难题

### 自定义权限

如果一个包声明了自定义权限，即使用了核心破解的禁用签名验证，仍然无法安装，并且有以下报错：

> 这个权限的保护等级不需要是「signature」的

```
Failure [INSTALL_FAILED_DUPLICATE_PERMISSION: Package fivecc.tools.signdemo attempting to redeclare permission fivecc.tools.MY_PERMISSION already owned by fivecc.tools.signdemo]
```

据说 3.8 是可用的，而 4.2 反而不可用。

是由于这个提交：

[Don't hook checkCapability() for permission usecase · LSPosed/CorePatch@65a3ee5](https://github.com/LSPosed/CorePatch/commit/65a3ee5245d42b0ac2348a32b074f298f359e295)

```java
            hookAllMethods("android.content.pm.PackageParser", loadPackageParam.classLoader, "checkCapability", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) {
                    // Don't handle PERMISSION (grant SIGNATURE permissions to pkgs with this cert)
                    // Or applications will have all privileged permissions
                    // https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/content/pm/PackageParser.java;l=5947?q=CertCapabilities
                    if ((Integer) param.args[1] != 4) {
                        param.setResult(true);
                    }
                }
            });
```

根据注释，是为了防止安装任意 apk 都被给予系统权限；但是这也导致了具有自定义权限的 app 在签名不同时无法通过核心破解覆盖安装。

现在包含了 androidx core 的 app 都会带一个 DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION 权限，因此核心破解对很多新 app 可能都失效了。

这个权限应该是为了向下移植 Android 13 的 registerReceiver 的 `RECEIVER_(NOT_)EXPORTED` flags 。

[功能和 API 概览  |  Android 开发者  |  Android Developers](https://developer.android.google.cn/about/versions/13/features?hl=zh-cn#runtime-receivers)

实际上，安装一个不同签名的 app ，重新定义其中自己的权限，对用户来说不应该是不安全的（既然用户都决定用核心破解了），因此这个还是有必要修复的。

### SharedUser  

```
Failure [INSTALL_FAILED_SHARED_USER_INCOMPATIBLE: Package fivecc.tools.signdemo has a signing lineage that diverges from the lineage of the sharedUserId]
```

虽然 shared user 已经[废弃](https://developer.android.com/guide/topics/manifest/manifest-element?hl=zh-cn)，但是还是有一些旧的 app 需要这个 feature （知名的如 Termux）。这个问题导致 termux 难以在不同签名的版本之间迁移（毕竟要把整个 termux 的用户目录搬走，移动到新的版本还是很麻烦的，有可能遇到各种问题）。

## 如何解决？

未完待续

<!--可以在安装之前临时修改系统已经记录的签名，然后再安装。

这样可能会有一些问题。CorePatch 目前的实现都是无状态的，由于 hook 点缺少签名与包的关联信息，因此难以实现我们的需求，而我们的解决方案涉及到改变 PMS 的状态，这样会有一些不确定因素。假如自动在安装之前处理签名的替换，安装失败后自动恢复，也可能会出现一些不可预料的结果。

因此我决定让用户自行处理，比如，用户需要在安装前，手动根据要安装的包去修改签名，并产生一个备份（只备份修改前的内容）。 -->
