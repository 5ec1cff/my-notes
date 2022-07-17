# 处理 dumpsys 的 protobuf dump

不知道从哪个版本的 Android 开始，dumpsys 支持 protobuf 输出了，这比起解析 dumpsys 的一大串字符串自然方便不少，也更加具有兼容性、稳定性，便于制作各种系统分析工具。

不过实际使用起来……着实费了一番功夫。

## 准备 proto 文件  


获取这些 proto 自然是从 AOSP 源码仓库，大部分的 proto 可以在下面三个目录找到：

```
platform/frameworks/proto_logging
platform/frameworks/base/core/proto
platform/frameworks/base/proto
```

SurfaceFlinger ：

```
platform/frameworks/native/services/surfaceflinger/layerproto
```

由于这些 proto 并没有单独的 git 模块，因此 clone 下来很不方便，所以用了朋友写的爬虫来下载。

```
platform/frameworks/proto_logging/+/refs/heads/master
platform/frameworks/base/+/refs/heads/master/core/proto
platform/frameworks/base/+/refs/heads/master/proto
```

实际上上面三个目录也并非包含了所有的依赖，而且并不是所有的 proto 都用得上，因此我用脚本处理了需要的依赖。每个支持 proto dump 的服务一般会有一个主 proto 文件，只要找到这个文件所需的依赖即可（参见最后的脚本）。

## 获取并处理 proto dump  

一般来说只要在 dumpsys 参数加上 `--proto` 即可得到 protobuf 格式的 dump 。

服务对应的 proto 一般在 `frameworks/base/core/proto/android/server/xxxServiceDumpProto.proto` （可以看一看 `base/core/proto/README.md`）

```sh
# wm
dumpsys window --proto > s.buf
# 没有数据类型
cat s.buf | protoc --decode_raw
# 有数据类型
cat s.buf|protoc --decode=com.android.server.wm.WindowManagerServiceDumpProto --proto_path=protos protos/frameworks/base/core/proto/android/server/windowmanagerservice.proto > dump.txt
# SF
dumpsys SurfaceFlinger --proto
cat d.proto.txt | protoc --decode=android.surfaceflinger.LayersProto --proto_path=proto proto/android/surfaceflinger/layers.proto > decoded.txt
```

## 向 Android Studio 项目引入 protobuf  

最终目的当然是让我们的程序能够处理这些 proto 数据。在这里我们使用 google 的 [protobuf gradle 插件](https://github.com/google/protobuf-gradle-plugin)

build.gradle.kts:

> 配置 protobuf 的代码参考了 [哔哩漫游](https://github.com/yujincheng08/BiliRoaming)

```kotlin
// 项目级
buildscript {
    dependencies {
        // ...
        classpath("com.google.protobuf:protobuf-gradle-plugin:0.8.19")
    }
}

// 模块级
plugins {
    // ...
    id("com.google.protobuf")
}

// https://github.com/google/protobuf-gradle-plugin/issues/540#issuecomment-1001053066
fun com.android.build.api.dsl.AndroidSourceSet.proto(action: SourceDirectorySet.() -> Unit) {
    (this as? ExtensionAware)
        ?.extensions
        ?.getByName("proto")
        ?.let { it as? SourceDirectorySet }
        ?.apply(action)
}

android {
    // ...

    sourceSets {
        named("main") {
            proto {
                srcDir("src/main/proto")
                include("**/*.proto")
            }
        }
    }
}

dependencies {
    // ...
    implementation("com.google.protobuf:protobuf-kotlin-lite:3.20.1")
    compileOnly("com.google.protobuf:protoc:3.20.1")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.19.4"
    }

    generatedFilesBaseDir = "$projectDir/src/generated"

    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                id("java") {
                    option("lite")
                }
                id("kotlin") {
                    option("lite")
                }
            }
        }
    }
}
```

配置好后，将我们的 proto 文件放在 `src/main/proto` 下，直接 Run generate source gradle tasks 即可在 `src/generated` 下生成 proto 的对应代码。（此处目录为相对模块的目录）

## 踩坑  

### 预构建的 protoc 不包含 include  

一些 proto 文件 import 了 `include/google/protobuf/descriptor.proto` ，然而通过 gradle 自动下载的 protoc 竟然没有这些 include 。好在之前下载了压缩包版的 protoc 包含了这些 include ，暂且放在 `src/main/proto` 下了。

### proto 类与 android 类重名  

处理好后能够生成代码了，但是和项目一起编译就报错，发现一些 proto 类的类名竟然和 android SDK 的类重名，如 `android.content.pm.PackageInfo` 。

观察这些类，发现基本是个空壳，都是对应 proto 文件名生成的类，本来应该把 proto 中的数据类放在这个类里的，但是由于存在 `option java_multiple_files = true;` 这个选项，这些类被分开放到单独的文件去了，但是剩下的这个空壳似乎没法清除掉。

按照官方的说法，这些 proto 不应该改名，但是这样在编译时就会和 SDK 冲突，不知道 AOSP 是如何编译的（似乎没有用标准的 protoc ，而是定制的编译器）。此外，运行时的 frameworks 也包含了一些 proto 类，就算过了编译这一关，也会在运行时链接发生冲突。总之，想在 android 上使用 proto ，必须修改类名。

经过一番考察，决定用脚本修补这些 proto 文件，用 `option java_package=` 指定新的包名，加上一个后缀 `.proto_` ，具体做法可以参见最后附的脚本。

## 优雅地 dumpsys

下面给一个简单的例子，不需要创建进程 `dumpsys` 即可获取 proto ，对应 `dumpsys activity --proto activities` ：

```kotlin
    val iam = ServiceManager.getService("activity")
    // 创建管道，write 端发送给系统服务，read 端读取
    val pipes = ParcelFileDescriptor.createPipe()
    val r = pipes[0]!!
    val w = pipes[1]
    // dump 似乎是阻塞的，考虑使用线程，不过正常情况一般会很快返回
    iam.dump(w.fileDescriptor, arrayOf("--proto", "activities"))
    w.close() // 必须关闭，否则由于 write 端未关闭，read 端仍然阻塞
    val ins = AssetFileDescriptor(r, 0L, AssetFileDescriptor.UNKNOWN_LENGTH)
        .createInputStream()
    val proto = ActivityManagerServiceDumpActivitiesProto.parseFrom(ins)
    // 这个结构比较复杂，就不进一步处理了，直接 toString 可以输出结构化的数据
    println(proto)
```

## 处理脚本

下面给出用于寻找依赖和修补 proto 的脚本：

```py
import os
import re

os.chdir("protos")

target = r"frameworks\base\core\proto\android\server\activitymanagerservice.proto".replace("\\", "/")
q = set()
deps = set()

def read_import(path):
    if not os.path.exists(path):
        print(f"warn: import {path} does not exists!")
        return
    print(f"reading import for {path}")
    with open(path, 'r', encoding='utf-8') as f:
        ls = f.readlines()
        for l in ls:
            r = re.match(r'import "(.*)";', l)
            if r is not None:
                q.add(r.group(1))

q.add(target)
while len(q) > 0:
    c = q.pop()
    read_import(c)
    deps.add(c)

for d in deps:
    newPath = os.path.join('..', 'ams', *os.path.split(d))
    os.makedirs(os.path.dirname(newPath), exist_ok=True)
    with open(d, 'r', encoding='utf-8') as f:
        ls = f.readlines()
        pn, pi = None, 0
        jp = False
        for i, l in enumerate(ls):
            r = re.match(r'option java_package\s*=\s*"(.*)";', l)
            if r is not None:
                jp = True
                pn, pi = r.group(1), i
                break
            r = re.match(r'package (.*);', l) 
            if r is not None:
                pn, pi = r.group(1), i
        if pn.startswith('com.android') or pn.startswith("android"):
            newLine = f'option java_package = "{pn}.protos_";\n'
            if jp:
                ls[i] = newLine
            else:
                ls.insert(pi + 1, newLine)
        with open(newPath, 'w', encoding='utf-8') as ff:
            ff.writelines(ls)
```

## 参考

[&#x5b;译&#x5d;Protobuf 语法指南](https://colobu.com/2015/01/07/Protobuf-language-guide/#%E9%80%89%E9%A1%B9%EF%BC%88Options%EF%BC%89)  
[CookBook/Protobuf基础教程.md at master · Byron4j/CookBook](https://github.com/Byron4j/CookBook/blob/master/Protobuf/ProtobufTutorial/Protobuf%E5%9F%BA%E7%A1%80%E6%95%99%E7%A8%8B.md)  
[Language Guide (proto3)  |  Protocol Buffers  |  Google Developers](https://developers.google.com/protocol-buffers/docs/proto3)  
[Java Generated Code  |  Protocol Buffers  |  Google Developers](https://developers.google.com/protocol-buffers/docs/reference/java-generated#package)  
[protocol buffers - Do I need to download all .proto files in Android framework source codes in order to parse proto message of adb dumpsys? - Stack Overflow](https://stackoverflow.com/questions/68608099/do-i-need-to-download-all-proto-files-in-android-framework-source-codes-in-orde)  

## 思考

1. 应该寻找一些使用了 protobuf 处理 dumpsys 的项目作为参考。  
2. 有没有可能利用 frameworks 中存在的 proto 类？（不过里面甚至没有反序列化的方法）  
