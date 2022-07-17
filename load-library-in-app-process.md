# 在 app process 中使用 jni  

## 加载 so

相比 zygote 启动的应用，app_process 启动虽然可以直接加载 apk ，但是并不会设置 nativeLibraryPath ，无法直接从 apk 加载 so ，因此我们 hack 一下：

```kotlin
class JNI {
    companion object {
        init {
            JNI::class.java.classLoader!!.getField("pathList")!!.let { pathList ->
                val firstElement = (pathList.getField("dexElements") as Array<*>)[0]!!
                val path = firstElement.getField("path") as File
                val abi = if (Process.is64Bit()) Build.SUPPORTED_64_BIT_ABIS[0] else Build.SUPPORTED_32_BIT_ABIS[0]
                pathList.invokeMethod("addNativePath",
                    arrayOf(Collection::class.java),
                    arrayOf<Any>(listOf("${path.absolutePath}!/lib/$abi"))
                )
            }
            System.loadLibrary("ash")
        }
    }

    external fun hello(): String
}
```

这里假定 ClassLoader 是 PathClassLoader 。

## 在 signal handler 与 jvm 交互

研究这个 jni 其实主要目的是在我的 shell 程序中处理 Ctrl+C ，在 linux 下自然是要处理 SIGINT 信号。

```cpp
static JavaVM* gVM = nullptr;
static jobject gInstance = nullptr;
static void handler(int sig) {
    JNIEnv *env;
    LOGD("received signal %d at thread %d", sig, gettid());
    int e = gVM->GetEnv((void**) &env, JNI_VERSION_1_6);
    if (e == JNI_OK) {
        jmethodID callback = env->GetMethodID(env->GetObjectClass(gInstance), "onInterrupted", "()V");
        env->CallVoidMethod(gInstance, callback);
    } else {
        LOGE("failed to getEnv %d", e);
    }
}

extern "C"
JNIEXPORT void JNICALL
Java_five_ec1cff_ash_JNI_nativeInitSignalHandler(JNIEnv *env, jobject thiz) {
    env->GetJavaVM(&gVM);
    gInstance = env->NewGlobalRef(thiz);
    signal(SIGINT, handler);
}
```

```kotlin
    external fun nativeInitSignalHandler()

    fun onInterrupted() {
        // Thread.currentThread().interrupt()
        println("test")
    }
```

代码看起来没有任何问题，然而实际运行起来按 Ctrl + C 后总是报错 StackOverflowError （如下）：

```java
^Cjava.lang.StackOverflowError: stack size 8192KB
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:442)
        at java.lang.Thread.sleep(Thread.java:358)
        at five.ec1cff.ash.Main$run$15.onCommand(Main.kt:158)
        at five.ec1cff.ash.ArgParser.runHandler(ArgParser.java:80)
        at five.ec1cff.ash.Main.run(Main.kt:181)
        at five.ec1cff.ash.Main$Companion.main$lambda-1(Main.kt:190)
        at five.ec1cff.ash.Main$Companion.$r8$lambda$_3oe5AW9liX-gULKkqL_PYYbzDA(Unknown Source:0)
        at five.ec1cff.ash.Main$Companion$$ExternalSyntheticLambda0.run(Unknown Source:2)
        at five.ec1cff.ash.SystemHelperKt.runOnActivityThread(SystemHelper.kt:46)
        at five.ec1cff.ash.SystemHelperKt.runOnActivityThread$default(SystemHelper.kt:19)
        at five.ec1cff.ash.Main$Companion.main(Main.kt:190)
        at five.ec1cff.ash.Main.main(Unknown Source:2)
        at com.android.internal.os.RuntimeInit.nativeFinishInit(Native Method)
        at com.android.internal.os.RuntimeInit.main(RuntimeInit.java:463)
```

搜索 `jni signal` 查到这么一条 issue ：

[ART prevents any Java calls from JNI during native signal handling &#x5b;37035211&#x5d; - Visible to Public - Issue Tracker](https://issuetracker.google.com/issues/37035211)

看上去 ART 虚拟机不支持在 signal handler 中和 jvm 交互，原因似乎是 ART 使用了 sigaltstack （其实我没懂）。官方也表示不会修复。看起来在 app process 中处理信号相当麻烦了。（难怪没有实现 `sun.misc.SignalHandler` ）