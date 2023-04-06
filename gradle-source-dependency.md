# 令人困惑的 Gradle 源码依赖

最近想试用一下 LSPosed 的新 API ，clone  example 下来发现无法 sync ，提示 `Unknown build name 'api'` ，但是可以正常 build 。找同学测试了一下，也是一样的结果。

https://github.com/libxposed/example

发现这是由于 libxposed example 使用了 gradle 的 source dependencies 导致的，删除相关代码后可以 sync 成功，但是必须用其他的方法引入依赖。

https://blog.gradle.org/introducing-source-dependencies

类似的还有 LSPosed ，自从下面这个提交开始：

https://github.com/LSPosed/LSPosed/commit/03d2cea093db056d56dbbea2ee13831e3dc5253a

source dependencies 在 Android Studio 稳定版 (E) 会出错，想着升级肯定就没问题了，于是升级到 canary (G) 后，虽然能 sync 成功，但 IDE 的依赖解析几乎完全坏掉，每个模块依赖外部的类都无法显示。

当然可以自己主动删掉 source dependencies ，改用 maven local ，但是这样是否有些不方便了？看到 LSPosed 的开发者这两个星期还在不断提交，怎么看都像是能正常使用 IDE 的。

在 Discussion 看到类似问题，也去下面提问，却在群里收到西大师的一个问号。

https://github.com/orgs/LSPosed/discussions/2444

我以为是自己无知，问了一个如此愚蠢的问题，一时间不知道如何回应。但是在群里翻了一下，从那个提交到现在也没有讨论过相关问题，想来大家都能正常工作。也许其他开发者本地就是改用 maven local ；或者干脆就不使用 IDE 开发——这我是做不到的。

后来我反复测试，确认确实是 IDE 的问题。

在 jetbrains 的 issue tracker 看到这个问题在四年前就有人反馈，至今似乎也没解决。

https://youtrack.jetbrains.com/issue/IDEA-208388/New-Gradle-source-dependencies-integration

我想新 API 的 example 既然是要给开发者使用的，起码要能正常在 IDE 中工作。如果 source dependencies 真的还没有被 IDE 所集成，还是应该改掉的。
