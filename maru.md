# maru

越来越感觉到 Zygisk 目前的原理不适合隐藏，反而是 Riru 的隐藏更为简单，于是突发奇想，能不能用 Riru 的加载方式加载我们的代码，然后在其中加载 Zygisk 模块呢？

起初想通过一个 magisk 模块实现（~~maru~~），但实际尝试后，考虑到下面几个原因：

1. zygisk 被禁用后 zygisk 模块都不会被 magisk 加载，导致不仅要自己加载模块的 so ，还要自己处理 mount 和 props 。  
2. magisk 引入了大量外部依赖，直接把 magisk 代码复制到另一个项目，维护起来有一定难度。  
3. magisk 内部的基础设施更加完善，操作起来比自己造的模块会更方便。  

因此觉得直接修改 magisk 源码或许会更好（于是成为了 magisk 分支 maru）。

## Native Bridge 加载器  

### 加载原理

系统会在 `/system/lib(64)` 中查找名为 `ro.dalvik.vm.native.bridge` 的值的 lib 并作为 native bridge 加载。

```cpp
// art/runtime/runtime.cc
bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
  // ...
  // Look for a native bridge.
  //
  // The intended flow here is, in the case of a running system:
  //
  // Runtime::Init() (zygote):
  //   LoadNativeBridge -> dlopen from cmd line parameter.
  //  |
  //  V
  // Runtime::Start() (zygote):
  //   No-op wrt native bridge.
  //  |
  //  | start app
  //  V
  // DidForkFromZygote(action)
  //   action = kUnload -> dlclose native bridge.
  //   action = kInitialize -> initialize library
  //
  //
  // The intended flow here is, in the case of a simple dalvikvm call:
  //
  // Runtime::Init():
  //   LoadNativeBridge -> dlopen from cmd line parameter.
  //  |
  //  V
  // Runtime::Start():
  //   DidForkFromZygote(kInitialize) -> try to initialize any native bridge given.
  //   No-op wrt native bridge.
  {
    std::string native_bridge_file_name = runtime_options.ReleaseOrDefault(Opt::NativeBridge);
    is_native_bridge_loaded_ = LoadNativeBridge(native_bridge_file_name);
  }
  // ...
}

// art/libnativebridge/native_bridge.cc
bool LoadNativeBridge(const char* nb_library_filename,
                      const NativeBridgeRuntimeCallbacks* runtime_cbs) {
  // We expect only one place that calls LoadNativeBridge: Runtime::Init. At that point we are not
  // multi-threaded, so we do not need locking here.

  if (state != NativeBridgeState::kNotSetup) {
    // Setup has been called before. Ignore this call.
    if (nb_library_filename != nullptr) {  // Avoids some log-spam for dalvikvm.
      ALOGW("Called LoadNativeBridge for an already set up native bridge. State is %s.",
            GetNativeBridgeStateString(state));
    }
    // Note: counts as an error, even though the bridge may be functional.
    had_error = true;
    return false;
  }

  if (nb_library_filename == nullptr || *nb_library_filename == 0) {
    CloseNativeBridge(false);
    return false;
  } else {
    if (!NativeBridgeNameAcceptable(nb_library_filename)) {
      CloseNativeBridge(true);
    } else {
      // Try to open the library. We assume this library is provided by the
      // platform rather than the ART APEX itself, so use the system namespace
      // to avoid requiring a static linker config link to it from the
      // com_android_art namespace.
      void* handle = OpenSystemLibrary(nb_library_filename, RTLD_LAZY);

      if (handle != nullptr) {
        callbacks = reinterpret_cast<NativeBridgeCallbacks*>(dlsym(handle,
                                                                   kNativeBridgeInterfaceSymbol));
        if (callbacks != nullptr) {
          if (isCompatibleWith(NAMESPACE_VERSION)) {
            // Store the handle for later.
            native_bridge_handle = handle;
          } else {
            ALOGW("Unsupported native bridge API in %s (is version %d not compatible with %d)",
                  nb_library_filename, callbacks->version, NAMESPACE_VERSION);
            callbacks = nullptr;
            dlclose(handle);
          }
        } else {
          dlclose(handle);
          ALOGW("Unsupported native bridge API in %s: %s not found",
                nb_library_filename, kNativeBridgeInterfaceSymbol);
        }
      } else {
        ALOGW("Failed to load native bridge implementation: %s", dlerror());
      }

      // Two failure conditions: could not find library (dlopen failed), or could not find native
      // bridge interface (dlsym failed). Both are an error and close the native bridge.
      if (callbacks == nullptr) {
        CloseNativeBridge(true);
      } else {
        runtime_callbacks = runtime_cbs;
        state = NativeBridgeState::kOpened;
      }
    }
    return state == NativeBridgeState::kOpened;
  }
}
```

native bridge 需要一个特殊的接口 `NativeBridgeItf` ，如果 dlopen 失败或者 dlsym 这个接口失败，都会卸载我们的 native bridge ，实际上这利于我们的隐藏，因为不像 LD_PRELOAD 难以主动卸载自身。

### 实例分析：Riru

Riru 注入 zygote 是通过 native bridge 实现的，但它并不依赖 native bridge 提供的接口，仅仅是将其作为加载的工具人。下面我们来分析一下：

Riru 生成了一个 libriruloader.so ，作为 magisk 模块挂载到 `/system/lib(64)` 用于进一步加载。

Riru 中这样声明这个符号：

```cpp
// riru/src/main/cpp/loader/loader.cpp
extern "C" [[gnu::visibility("default")]] uint8_t NativeBridgeItf[
        sizeof(NativeBridgeCallbacks<__ANDROID_API_R__>) * 2]{0};
```

这个接口实际上是一个结构体 `NativeBridgeCallbacks` ，声明于 `art/libnativebridge/include/nativebridge/native_bridge.h` ，随着 Android 版本更新会逐渐升级接口版本，增加更多内容，它的第一个字段是版本号：

```cpp
// Native bridge interfaces to runtime.
struct NativeBridgeCallbacks {
  // Version number of the interface.
  uint32_t version;
```

而 riru 默认情况下直接全部填 0 ，实际上就阻止了系统调用接口中的函数，loader 会被直接卸载，不过由于 loader 是被 dlopen 的，因此 constructor 仍然会被调用，在 constructor 中可以执行任意代码，自然也可以 dlopen 其他库，并且不会被关闭。

riru 通过 socket 的方式与 rirud 守护进程通信，在 loader 中会读取原始的 native bridge ，如果存在，会尝试 dlopen 这个 native bridge ，并直接复制它的 NativeBridgeItf 给自己，这样系统就能加载原始的 native bridge 了。

### 如何读取原始的 NB ?

从 riru 的实现来看，这个 original native bridge 被 daemon 持有，而 daemon 在 post-fs-data 启动，orig nb 就是从参数传进去的。

```sh
# post-fs-data.sh
unshare -m sh -c "/system/bin/app_process -Djava.class.path=rirud.apk /system/bin --nice-name=rirud riru.Daemon $(magisk -V) $(magisk --path) $(getprop ro.dalvik.vm.native.bridge)&"
```

看看 Magisk 的加载：

post-fs-data 阶段处理模块的加载，首先会执行 `post-fs-data.sh` 然后再 magic mount ，挂载的同时处理 props 。因此我们在 `post-fs-data.sh` 可以读取原来的属性。

```cpp
// native/src/core/bootstages.cpp
void post_fs_data(int client) {
    // ...

    if (getprop("persist.sys.safemode", true) == "1" || check_key_combo()) {
        safe_mode = true;
        // Disable all modules and denylist so next boot will be clean
        disable_modules();
        disable_deny();
    } else {
        exec_common_scripts("post-fs-data");
        db_settings dbs;
        get_db_settings(dbs, ZYGISK_CONFIG);
        zygisk_enabled = dbs[ZYGISK_CONFIG];
        initialize_denylist();
        handle_modules(); // 首先执行 post-fs-data.sh
    }

early_abort:
    // We still do magic mount because root itself might need it
    magic_mount(); // magic mount 的同时加载 props
    DAEMON_STATE = STATE_POST_FS_DATA_DONE;

unblock_init:
    close(xopen(UNBLOCKFILE, O_RDONLY | O_CREAT, 0));
}

// native/src/core/module.cpp
void handle_modules() {
    prepare_modules();
    collect_modules(false);
    exec_module_scripts("post-fs-data");

    // Recollect modules (module scripts could remove itself)
    module_list->clear();
    collect_modules(true);
}

void magic_mount() {
    // ...
    LOGI("* Loading modules\n");
    for (const auto &m : *module_list) {
        const char *module = m.name.data();
        char *b = buf + sprintf(buf, "%s/" MODULEMNT "/%s/", MAGISKTMP.data(), module);

        // Read props
        strcpy(b, "system.prop");
        if (access(buf, F_OK) == 0) {
            LOGI("%s: loading [system.prop]\n", module);
            load_prop_file(buf, false);
        }
        // ...
    }
    // ...
}

// native/src/resetprop/resetprop.cpp
void load_prop_file(const char *filename, bool prop_svc) {
    auto impl = get_impl();
    LOGD("resetprop: Parse prop file [%s]\n", filename);
    parse_prop_file(filename, [=](auto key, auto val) -> bool {
        impl->setprop(key.data(), val.data(), prop_svc);
        return true;
    });
}
```

## USAP

[Android Framework | 一种新型的应用启动机制:USAP - 掘金](https://juejin.cn/post/6922704248195153927#heading-6)

如何启用：

```
getprop persist.device_config.runtime_native.usap_pool_enabled
setprop persist.device_config.runtime_native.usap_pool_enabled true
```

在创建 usap 之前就已经加载了 native bridge 。USAP 仅仅是 zygote 的 fork ，也就是说其中不会重启 runtime ，也不会重新加载 native bridge 。

## hook JNI  

注入 Zygote 后一般要 hook JNI 的 RegisterNativeMethods ，以便我们得到 `com.android.internal.app.Zygote` 的 native 函数指针，进而 hook 它的关键方法。

### Zygisk 是怎么 hook jni 注册的  

Zygisk 的实现如下： 

```cpp
// native/src/zygisk/hook.cpp
#define ANDROID_RUNTIME ".*/libandroid_runtime.so$"
#define APP_PROCESS     "^/system/bin/app_process.*"

void hook_functions() {
#if MAGISK_DEBUG
    xhook_enable_debug(1);
    xhook_enable_sigsegv_protection(0);
#endif
    default_new(xhook_list);
    default_new(jni_hook_list);
    default_new(jni_method_map);

    XHOOK_REGISTER(ANDROID_RUNTIME, fork);
    XHOOK_REGISTER(ANDROID_RUNTIME, unshare);
    XHOOK_REGISTER(ANDROID_RUNTIME, jniRegisterNativeMethods);
    XHOOK_REGISTER(ANDROID_RUNTIME, selinux_android_setcontext);
    XHOOK_REGISTER_SYM(ANDROID_RUNTIME, "__android_log_close", android_log_close);
    hook_refresh();

    // Remove unhooked methods
    xhook_list->erase(
            std::remove_if(xhook_list->begin(), xhook_list->end(),
            [](auto &t) { return *std::get<2>(t) == nullptr;}),
            xhook_list->end());

    if (old_jniRegisterNativeMethods == nullptr) {
        ZLOGD("jniRegisterNativeMethods not hooked, using fallback\n");

        // android::AndroidRuntime::setArgv0(const char*, bool)
        XHOOK_REGISTER_SYM(APP_PROCESS, "_ZN7android14AndroidRuntime8setArgv0EPKcb", setArgv0);
        hook_refresh();

        // We still need old_jniRegisterNativeMethods as other code uses it
        // android::AndroidRuntime::registerNativeMethods(_JNIEnv*, const char*, const JNINativeMethod*, int)
        constexpr char sig[] = "_ZN7android14AndroidRuntime21registerNativeMethodsEP7_JNIEnvPKcPK15JNINativeMethodi";
        *(void **) &old_jniRegisterNativeMethods = dlsym(RTLD_DEFAULT, sig);
    }
}
```

首先尝试用 xhook hook `libandroid_runtime.so` 的 `jniRegisterNativeMethods` ，这个在高版本的 Android 一般是行不通的（？），于是转而使用 fallback 方案，这个方案就比较有意思了。

首先 hook `app_process` 的 `android::AndroidRuntime::setArgv0` ：

```cpp
// This method is a trampoline for swizzling android::AppRuntime vtable
bool swizzled = false;
DCL_HOOK_FUNC(void, setArgv0, void *self, const char *argv0, bool setProcName) {
    if (swizzled) {
        old_setArgv0(self, argv0, setProcName);
        return;
    }

    ZLOGD("AndroidRuntime::setArgv0\n");

    // We don't know which entry is onVmCreated, so overwrite every one
    // We also don't know the size of the vtable, but 8 is more than enough
    auto new_table = new void*[8];
    new_table[0] = reinterpret_cast<void*>(&vtable_entry<0>);
    new_table[1] = reinterpret_cast<void*>(&vtable_entry<1>);
    new_table[2] = reinterpret_cast<void*>(&vtable_entry<2>);
    new_table[3] = reinterpret_cast<void*>(&vtable_entry<3>);
    new_table[4] = reinterpret_cast<void*>(&vtable_entry<4>);
    new_table[5] = reinterpret_cast<void*>(&vtable_entry<5>);
    new_table[6] = reinterpret_cast<void*>(&vtable_entry<6>);
    new_table[7] = reinterpret_cast<void*>(&vtable_entry<7>);

    // Swizzle C++ vtable to hook virtual function
    gAppRuntimeVTable = *reinterpret_cast<void***>(self); // -> 对象 -> 虚表地址
    *reinterpret_cast<void***>(self) = new_table;
    swizzled = true;

    old_setArgv0(self, argv0, setProcName);
}
```

这里是一个虚表 hook 。虚表是 C++ 用于实现多态的一种手段，一个继承了虚函数的类，它的对象的内存布局中头部一般是指向虚表的指针，而虚表中包含了虚函数的实现的地址。

`android::AndroidRuntime` 是一个抽象类，其中虚函数 onVmCreated 就是我们的目标：

```cpp
// frameworks/base/core/jni/include/android_runtime/AndroidRuntime.h
namespace android {

class AndroidRuntime
{
public:
    AndroidRuntime(char* argBlockStart, size_t argBlockSize);
    virtual ~AndroidRuntime();

    enum StartMode {
        Zygote,
        SystemServer,
        Application,
        Tool,
    };

    void setArgv0(const char* argv0, bool setProcName = false);
    // ...

    /**
     * This gets called after the VM has been created, but before we
     * run any code. Override it to make any FindClass calls that need
     * to use CLASSPATH.
     */
    virtual void onVmCreated(JNIEnv* env);
```

那么为什么要 hook setArgv0 呢？有两个原因：

1. app_process 引入了 setArgv0 这个符号。  
2. setArgv0 后就调用了 `AndroidRuntime.start` ，因此首个被调用的虚函数就是 `onVmCreated` 。  

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
    // ...

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.isEmpty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

那么我们只要备份并替换对象指向的虚表指针，换成我们构造的假虚表，其中每一个表项都指向 hooker 函数，那么首次调用 onVmCreated 的时候就调用了我们的 hooker 函数，在这里我们可以用原先的备份调用原函数并处理 hook 逻辑。

```cpp
template<int N>
void vtable_entry(void *self, JNIEnv* env) {
    // The first invocation will be onVmCreated. It will also restore the vtable.
    onVmCreated(self, env);
    // Call original function
    reinterpret_cast<decltype(&onVmCreated)>(gAppRuntimeVTable[N])(self, env);
}

// This method is a trampoline for hooking JNIEnv->RegisterNatives
void onVmCreated(void *self, JNIEnv* env) {
    ZLOGD("AppRuntime::onVmCreated\n");

    // Restore virtual table
    auto new_table = *reinterpret_cast<void***>(self);
    *reinterpret_cast<void***>(self) = gAppRuntimeVTable;
    delete[] new_table;

    new_functions = new JNINativeInterface();
    memcpy(new_functions, env->functions, sizeof(*new_functions));
    new_functions->RegisterNatives = &env_RegisterNatives;

    // Replace the function table in JNIEnv to hook RegisterNatives
    old_functions = env->functions;
    env->functions = new_functions;
}
```

那么 hook onVmCreated 的目的又是什么呢？我们看上面的代码。

onVmCreated 的参数中包含了 JNIEnv 指针，系统类的 jni 函数注册都通过这个 env 注册，因此我们可以替换掉它的 `env->functions` 中的 `RegisterNatives` ，实现 hook jni 函数的注册。

### NB 能否像 zygisk 一样 hook ？

zygisk 绕了一个大弯路，总算是 hook 到 JNI 注册了，那么这个方法是否还适用于 native bridge 的注册呢？

回顾 native bridge 的加载，位于 `Runtime::Init` 。看一看如何从 AndroidRuntime 到 Runtime ：

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // ...
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);
    // ...
}

int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote, bool primary_zygote)
{
    // ...
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

    return 0;
}

// art/runtime/jni/java_vm_ext.cc
extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  // ...
  if (!Runtime::Create(options, ignore_unrecognized)) {
    return JNI_ERR;
  }
  // ...
  android::InitializeNativeLoader();

  Runtime* runtime = Runtime::Current();
  bool started = runtime->Start();
  if (!started) {
    delete Thread::Current()->GetJniEnv();
    delete runtime->GetJavaVM();
    LOG(WARNING) << "CreateJavaVM failed";
    return JNI_ERR;
  }

  *p_env = Thread::Current()->GetJniEnv();
  *p_vm = runtime->GetJavaVM();
  return JNI_OK;
}

bool Runtime::Create(const RuntimeOptions& raw_options, bool ignore_unrecognized) {
  RuntimeArgumentMap runtime_options;
  return ParseOptions(raw_options, ignore_unrecognized, &runtime_options) &&
      Create(std::move(runtime_options));
}

bool Runtime::Create(RuntimeArgumentMap&& runtime_options) {
  // TODO: acquire a static mutex on Runtime to avoid racing.
  if (Runtime::instance_ != nullptr) {
    return false;
  }
  instance_ = new Runtime;
  Locks::SetClientCallback(IsSafeToCallAbort);
  if (!instance_->Init(std::move(runtime_options))) {
    // TODO: Currently deleting the instance will abort the runtime on destruction. Now This will
    // leak memory, instead. Fix the destructor. b/19100793.
    // delete instance_;
    instance_ = nullptr;
    return false;
  }
  return true;
}
```

由此可见，顺序是 `AndroidRuntime::setArgv0` -> `AndroidRuntime::start` -> 加载 native bridge ，因此这个阶段注入必然不能用 zygisk 的方法 hook 到。

> 实际上，native bridge 注入后可以用 `getprogname` 得到自己的名字 (`zygote`) ，riru 中就有这样的代码，因此更说明了 setArgv0 在加载 native bridge 之前调用。

### Riru 的实现

既然如此，我们看看 Riru 如何 hook 的。

Riru 本体的入口：

```cpp
// riru/src/main/cpp/entry.cpp
extern "C" [[gnu::visibility("default")]] [[maybe_unused]] void
// NOLINTNEXTLINE
init(void *handle, const char* magisk_path, const RirudSocket& rirud) {
    self_handle = handle;

    magisk::SetPath(magisk_path);
    hide::PrepareMapsHideLibrary();
    jni::InstallHooks();
    modules::Load(rirud);
}
```

关键在于 `jni::InstallHooks`

```cpp
// riru/src/main/cpp/jni_hooks.cpp
void jni::InstallHooks() {
    XHOOK_REGISTER(".*\\libandroid_runtime.so$", jniRegisterNativeMethods)

    if (xhook_refresh(0) == 0) {
        xhook_clear();
        LOGI("hook installed");
    } else {
        LOGE("failed to refresh hook");
    }

    useTableOverride = old_jniRegisterNativeMethods == nullptr;

    if (useTableOverride) {
        LOGI("no jniRegisterNativeMethods");

        SandHook::ElfImg art("libart.so");

        auto *GetJniNativeInterface = art.getSymbAddress<GetJniNativeInterface_t *>(
                "_ZN3art21GetJniNativeInterfaceEv");
        setTableOverride = art.getSymbAddress<SetTableOverride_t *>(
                "_ZN3art9JNIEnvExt16SetTableOverrideEPK18JNINativeInterface");

        if (setTableOverride != nullptr && GetJniNativeInterface != nullptr) {
            auto functions = GetJniNativeInterface();
            auto new_JNINativeInterface = new JNINativeInterface();
            memcpy(new_JNINativeInterface, functions, sizeof(JNINativeInterface));
            old_RegisterNatives = functions->RegisterNatives;
            new_JNINativeInterface->RegisterNatives = new_RegisterNative;

            setTableOverride(new_JNINativeInterface);
            LOGI("override table installed");
        } else {
            if (GetJniNativeInterface == nullptr) LOGE("cannot find GetJniNativeInterface");
            if (setTableOverride == nullptr) LOGE("cannot find setTableOverride");
        }

        auto *handle = dlopen("libnativehelper.so", 0);
        if (handle) {
            old_jniRegisterNativeMethods = reinterpret_cast<jniRegisterNativeMethods_t *>(dlsym(
                    handle,
                    "jniRegisterNativeMethods"));
        }
    }
}
```

一开始也是在 `libandroid_runtime.so` 寻找 `jniRegisterNativeMethods` 。

如果失败，则会寻找 `libart.so` 的镜像，并查找下面这两个符号：

```cpp
art::GetJniNativeInterface()
art::JNIEnvExt::SetTableOverride(JNINativeInterface const*)
```

此处用了 SandHook 的 ElfImg ，它会解析 maps ，并直接打开 `libart.so` 文件，解析符号。

```
$ readelf -s /apex/com.android.art/lib64/libart.so -W | grep _ZN3art21GetJniNativeInterfaceEv
  2360: 0000000000393bd4    40 FUNC    GLOBAL PROTECTED   14 _ZN3art21GetJniNativeInterfaceEv
 24323: 0000000000393bd4    40 FUNC    GLOBAL PROTECTED   14 _ZN3art21GetJniNativeInterfaceEv
$ readelf -s /apex/com.android.art/lib64/libart.so -W | grep _ZN3art9JNIEnvExt16SetTableOverrideEPK18JNINativeInterface
  2601: 000000000038e900   232 FUNC    GLOBAL PROTECTED   14 _ZN3art9JNIEnvExt16SetTableOverrideEPK18JNINativeInterface
 24338: 000000000038e900   232 FUNC    GLOBAL PROTECTED   14 _ZN3art9JNIEnvExt16SetTableOverrideEPK18JNINativeInterface
```

两个函数的声明如下：

```cpp
// art/runtime/jni/jni_internal.h
const JNINativeInterface* GetJniNativeInterface();

// art/runtime/jni/jni_env_ext.h
  // Set the function table override. This will install the override (or original table, if null)
  // to all threads.
  // Note: JNI function table overrides are sensitive to the order of operations wrt/ CheckJNI.
  //       After overriding the JNI function table, CheckJNI toggling is ignored.
  static void SetTableOverride(const JNINativeInterface* table_override)
      REQUIRES(!Locks::thread_list_lock_, !Locks::jni_function_table_lock_);
```

```cpp
// art/runtime/jni/jni_internal.cc
const JNINativeInterface* GetJniNativeInterface() {
  // The template argument is passed down through the Encode/DecodeArtMethod/Field calls so if
  // JniIdType is kPointer the calls will be a simple cast with no branches. This ensures that
  // the normal case is still fast.
  return Runtime::Current()->GetJniIdType() == JniIdType::kPointer
             ? &JniNativeInterfaceFunctions<false>::gJniNativeInterface
             : &JniNativeInterfaceFunctions<true>::gJniNativeInterface;
}

template<bool kEnableIndexIds>
struct JniNativeInterfaceFunctions {
  using JNIImpl = JNI<kEnableIndexIds>;
  static constexpr JNINativeInterface gJniNativeInterface = {
    // ...
    JNIImpl::RegisterNatives,
    // ...
  }
}

template <bool kEnableIndexIds>
class JNI {
  // ...
  static jint RegisterNatives(JNIEnv* env,
                              jclass java_class,
                              const JNINativeMethod* methods,
                              jint method_count) {
    // ...
  }
}
```

### 为什么那么麻烦呢？

看了 Zygisk 和 Riru 的 hook ，虽然有一定差别，但大体上思路是相同的：首先尝试 plt hook `libandroid_runtime.so` 对 `jniRegisterNativeMethods` 的调用，如果失败，就尝试替换 JNIEnv 的函数表。

那么什么样的情况下会失败呢？我记得之前编译 magisk 的时候开启了详细日志，在 Android 12 的 AVD 上，直接 plt hook 是失败的。

首先，我们的主要目标是 hook Zygote 的 native methods 注册（当然 zygisk 提供了 hook 其他 jni 函数的 api），它们都在 libandroid_runtime 中：

```cpp
// libandroid_runtime.so
// frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
int register_com_android_internal_os_Zygote(JNIEnv* env) {
  // ...
  RegisterMethodsOrDie(env, "com/android/internal/os/Zygote", gMethods, NELEM(gMethods));

  return JNI_OK;
}

// frameworks/base/core/jni/core_jni_helpers.h
static inline int RegisterMethodsOrDie(JNIEnv* env, const char* className,
                                       const JNINativeMethod* gMethods, int numMethods) {
    int res = AndroidRuntime::registerNativeMethods(env, className, gMethods, numMethods);
    LOG_ALWAYS_FATAL_IF(res < 0, "Unable to register native methods.");
    return res;
}

// frameworks/base/core/jni/AndroidRuntime.cpp
/*
 * Register native methods using JNI.
 */
/*static*/ int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}
```

libandroid_runtime 调用了 `jniRegisterNativeMethods` ，这个在各个版本的 android 基本一致。进一步搜索可以发现 `jniRegisterNativeMethods` 来自 `libnativehelper.so` ，这个方法最终还是调用了 `JNIEnv` 函数表的 `RegisterNative` 进行注册。

看起来我们的 libandroid_runtime 是动态链接到 libnativehelper 的，那怎么会出现找不到的情况呢？

先来看一看 android runtime 链接的 so ：

```sh
$ readelf -d /system/lib64/libandroid_runtime.so|grep native
 0x0000000000000001 (NEEDED)             Shared library: [libnativehelper.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativebridge_lazy.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativeloader_lazy.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativedisplay.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativewindow.so]
$ readelf --dyn-syms /system/lib64/libandroid_runtime.so -W | grep jniR
    46: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND jniRegisterNativeMethods@LIBNATIVEHELPER_1 (5)
```

看上去没问题，顺带一提，这是 Android 11 。

再看看 AVD 的 Android 12 ：

```sh
$ readelf -d libandroid_runtime.so | grep native
 0x0000000000000001 (NEEDED)             Shared library: [libnativebridge_lazy.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativeloader_lazy.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativedisplay.so]
 0x0000000000000001 (NEEDED)             Shared library: [libnativewindow.so]
$ readelf --dyn-syms libandroid_runtime.so | grep jni
  2439: 00000000001de2b0    51 FUNC    GLOBAL PROTECTED   14 jniGetNioBufferBaseArrayO
  2561: 00000000001de270    51 FUNC    GLOBAL PROTECTED   14 jniGetNioBufferBaseArray
  2936: 00000000001de380    29 FUNC    GLOBAL PROTECTED   14 jniUninitializeConstants
  3486: 00000000001de2f0    76 FUNC    GLOBAL PROTECTED   14 jniGetNioBufferFields
  3724: 00000000001de340    51 FUNC    GLOBAL PROTECTED   14 jniGetNioBufferPointer
```

并没有我们要找的符号，它也没有链接到 libnativehelper 。

观察 android runtime 的构建文件 `frameworks/base/core/jni/Android.bp` ，在 Android 11 中搜索 `libnativehelper` ：

```js
    shared_libs: [
        "libbase",
        "libcutils",
        "libharfbuzz_ng",
        "libhwui",
        "liblog",
        "libminikin",
        "libnativehelper",
        "libz",
        "libziparchive",
    ],
```

Android 12 中：

```js
    static_libs: [
        "libnativehelper_lazy",
        "libziparchive_for_incfs",
    ],
```

可见，android 11 以前是动态链接，android 12 后是静态链接到一个 `_lazy` 后缀的 nativehelper 。

最早引入的提交(2021-02-16)：

[_/android/platform/frameworks/base - Android Code Search](https://cs.android.com/android/_/android/platform/frameworks/base/+/33cbf8b14c9805c86a3db7361110d7188f7ad4b6)

```
libandroid_runtime,libhwui: use libnativehelper_lazy

This change updates the dependencies of bootanimation to use a new
static library, libnativehelper_lazy, that loads libnativehelper on
demand. This means boot animation no longer depends on
libnativehelper.so. The affected frameworks libraries will load
libnativehelper lazily as-needed.

This change enables the removal of the ART APEX from the bootstrap
APEXes. The ART APEX provides libnativehelper.so and this is no longer
required during boot for bootanimation. Removing ART from the
bootstrap APEXes allows for compressing the system installed ART APEX
after an ART APEX update is applied and saves tens of megabytes of
storage.
```

大意是引入了一个 `libnativehelper_lazy` 库，确保 apex 未挂载的时候能够正常使用（`libnativehelper` 属于 `com.android.art` 这个 apex 包，apex 在 Android 10 引入。）

```cpp
// libnativehelper/Android.bp
// Lazy loading version of libnativehelper that can be used by code
// that is running before the ART APEX is mounted and
// libnativehelper.so is available.
cc_library_static {
    name: "libnativehelper_lazy",
    defaults: ["libnativehelper_defaults"],
    bootstrap: true,
    host_supported: true,
    export_include_dirs: [
        "header_only_include",
        "include",
        "include_jni",
        "include_platform",
        "include_platform_header_only",
    ],
    apex_available: ["//apex_available:platform"],
    srcs: ["libnativehelper_lazy.c"],
    target: {
        linux: {
            version_script: "libnativehelper.map.txt",
        },
    },
}
```

相关源码在 `libnativehelper_lazy.c` ，大概就是封装了一下，打开真正的 `libnativehelper` 调用里面的函数。

Riru 对该变化引入的提交(2021/3/1)：[Prepare for libnativehelper_lazy · RikkaApps/Riru@5b6113d](https://github.com/RikkaApps/Riru/commit/5b6113d59ebf8744aebede5e9382c7a3ae7ddcd1)

> riru 早期的 hook 逻辑写在 `main.cpp`

因此这部分逻辑完全是为了适配 Android 12 以上而编写的。

### 总结

Zygisk 中替换 JNIEnv 的方案无法适用于 native bridge 加载，考虑改成 Riru 的方案，缺点是要打开 libart 。

## maru 实现方案讨论

### 0x01 作为 magisk 模块实现

作为 magisk 模块实现，首先有一个最重要的问题，就是 Magisk 的 Zygisk 开关应该是开还是关。

如果关，那么 zygisk 模块不会被加载，一切都需要自己处理，这自然是很麻烦的。

因此应该开，那么我们如何阻断原先的 zygisk 执行呢？

首先 post-fs-data 先于 mount 执行，而 mount 后似乎没有什么明确的接口指示「mount 完成」。

注入 magisk 是不现实的，不过仍然有可行的方案：

我们看看 post-fs-data :

```cpp
// native/src/core/bootstages.cpp
void post_fs_data(int client) {
    // ...

    if (getprop("persist.sys.safemode", true) == "1" || check_key_combo()) {
        safe_mode = true;
        // Disable all modules and denylist so next boot will be clean
        disable_modules();
        disable_deny();
    } else {
        exec_common_scripts("post-fs-data");
        db_settings dbs;
        get_db_settings(dbs, ZYGISK_CONFIG);
        zygisk_enabled = dbs[ZYGISK_CONFIG];
        initialize_denylist();
        handle_modules();
    }

early_abort:
    // We still do magic mount because root itself might need it
    magic_mount();
    DAEMON_STATE = STATE_POST_FS_DATA_DONE;

unblock_init:
    close(xopen(UNBLOCKFILE, O_RDONLY | O_CREAT, 0));
}

// native/src/include/magisk.hpp
#define UNBLOCKFILE     "/dev/.magisk_unblock"
```

先是执行脚本，然后 magic mount ，最后创建了一个文件 `/dev/.magisk_unblock` ，这个文件是干什么用的？

再来看看 magisk 注入的 rc ：

```cpp
// native/src/init/magiskrc.inc
"on post-fs-data\n"
"    start logd\n"
"    rm " UNBLOCKFILE "\n"
"    start %2$s\n"
"    wait " UNBLOCKFILE " " str(POST_FS_DATA_WAIT_TIME) "\n"
"    rm " UNBLOCKFILE "\n"
"\n"
```

看看 [init](https://android.googlesource.com/platform/system/core/+/b2d8315f10b242daf604c24a5f1e8007b13f86fe/init/README.md) 对 `wait` 的解释：

`wait <path> [ <timeout> ]`

> Poll for the existence of the given file and return when found, or the timeout has been reached. If timeout is not specified it currently defaults to five seconds. The timeout value can be fractional seconds, specified in floating point notation.

总之，Magisk 创建了这个文件，并且是在 post-fs-data 的 mount 之后，因此我们也可以用 inotify 之类的东西 wait 这个文件，等到创建后 umount zygisk ，这样就阻止了原 zygisk 的执行。

实测发现 shell 的 inotifyd 反应速度太慢，感觉不如循环检测 app_process 的 inode ，发现不对就 umount （虽然仍然有 race ）。

init 的实现：`system/core/init/util.cpp`

```cpp
int wait_for_file(const char* filename, std::chrono::nanoseconds timeout) {
    android::base::Timer t;
    while (t.duration() < timeout) {
        struct stat sb;
        if (stat(filename, &sb) != -1) {
            LOG(INFO) << "wait for '" << filename << "' took " << t;
            return 0;
        }
        std::this_thread::sleep_for(10ms);
    }
    LOG(WARNING) << "wait for '" << filename << "' timed out and took " << t;
    return -1;
}
```

实际上也是循环 stat 。

感觉用模块代替 zygisk 还是有一定难度的。

### 0x02 修改 magisk 源码实现

主要问题：如何存放 zygisk-ld

如果像 magisk 的其他 bin 一样放在 /data/adb ，需要大量修改（构建脚本、安装脚本……）。

如果像原来一样把 zygisk-ld 存在 magisk 里面，需要少量修改（构建脚本）。但是 magisk 内要包含所有架构的 zygisk-ld 。

此外直接存在 magisk 里面，也要考虑容易被内存扫描到的问题（如果要模仿 riru 行为，不卸载模块而隐藏的情况下）
