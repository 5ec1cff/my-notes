# LSP Native Hook 踩坑

准备用 LSP 的 native hook 实现之前提到的 [`ImNotObscured`](im-not-obscured.md) ，不过遇到了不少坑，记录一下。

[Native Hook · LSPosed/LSPosed Wiki](https://github.com/LSPosed/LSPosed/wiki/Native-Hook)

## sepolicy

看上去 LSP 没有对在系统服务里面加载 so 做特殊处理，因此系统没法加载模块的 so

```
avc: denied { execute } for path="/data/app/~~VLQwcy9J5DQF_BVy2ONN7w==/fivecc.tools.im_not_obscured-dRL-Ki7aYUu5JDh-h2Poeg==/base.apk" dev="dm-5" ino=41029 scontext=u:r:system_server:s0 tcontext=u:object_r:apk_data_file:s0 tclass=file permissive=0
```

解决方法：临时加一个规则上去

```
echo "allow system_server apk_data_file file execute" > add_pol
magiskpolicy --apply add_pol --live
```

## dlopen

看上去模块的 namespace 也没法 dlopen 某些系统库，于是去 LSP 抄了一份 elf_utils 。

```
12-11 12:41:28.398 31376 31376 D linker  : dlopen(name="libinput.so", flags=0x0, extinfo=(null), caller="/data/app/~~VLQwcy9J5DQF_BVy2ONN7w==/fivecc.tools.im_not_obscured-dRL-Ki7aYUu5JDh-h2Poeg==/base.apk!/lib/x86_64/libim_not_obscured.so", caller_ns=classloader-namespace@0x7f2269859b10, targetSdkVersion=10000) ...
12-11 12:41:28.398 31376 31376 D linker  : find_libraries(ns=classloader-namespace): task=libinput.so, is_dt_needed=0
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so, flags=0x0, search_linked_namespaces=1): calling open_library with realpath=
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so, flags=0x0, realpath=/system/lib64/libinput.so, search_linked_namespaces=1)
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libbase.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: liblog.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libcutils.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libutils.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libbinder.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libui.so
12-11 12:41:28.398 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libinput.so): Adding DT_NEEDED task: libc++.so
--
12-11 12:41:28.407 31376 31376 D linker  : find_libraries(ns=classloader-namespace): task=libc.so, is_dt_needed=1
12-11 12:41:28.407 31376 31376 D linker  : find_library_internal(ns=classloader-namespace, task=libc.so): Already loaded (by soname): /apex/com.android.runtime/lib64/bionic/libc.so
12-11 12:41:28.408 31376 31376 D linker  : find_libraries(ns=classloader-namespace): task=libm.so, is_dt_needed=1
12-11 12:41:28.408 31376 31376 D linker  : find_library_internal(ns=classloader-namespace, task=libm.so): Already loaded (by soname): /apex/com.android.runtime/lib64/bionic/libm.so
12-11 12:41:28.408 31376 31376 D linker  : find_libraries(ns=classloader-namespace): task=libdl.so, is_dt_needed=1
12-11 12:41:28.408 31376 31376 D linker  : find_library_internal(ns=classloader-namespace, task=libdl.so): Already loaded (by soname): /apex/com.android.runtime/lib64/bionic/libdl.so
12-11 12:41:28.408 31376 31376 D linker  : find_libraries(ns=classloader-namespace): task=libdl_android.so, is_dt_needed=1
12-11 12:41:28.408 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libdl_android.so, flags=0x0, search_linked_namespaces=1): calling open_library with realpath=
12-11 12:41:28.408 31376 31376 D linker  : load_library(ns=classloader-namespace, task=libdl_android.so, flags=0x0, realpath=/apex/com.android.runtime/lib64/bionic/libdl_android.so, search_linked_namespaces=1)
12-11 12:41:28.408 31376 31376 D linker  : library "libdl_android.so" needed or dlopened by "/system/lib64/libvndksupport.so" is not accessible for the namespace "classloader-namespace"
12-11 12:41:28.408 31376 31376 E linker  : library "libdl_android.so" ("/apex/com.android.runtime/lib64/bionic/libdl_android.so") needed or dlopened by "/system/lib64/libvndksupport.so" is not accessible for the namespace: [name="classloader-namespace", ld_library_paths="", default_library_paths="/data/app/~~VLQwcy9J5DQF_BVy2ONN7w==/fivecc.tools.im_not_obscured-dRL-Ki7aYUu5JDh-h2Poeg==/base.apk!/lib/x86_64:/system/lib64:/system_ext/lib64", permitted_paths="/data:/mnt/expand"]
12-11 12:41:28.409 31376 31376 D linker  : find_library_internal(ns=classloader-namespace, task=libdl_android.so): Trying 4 linked namespaces
12-11 12:41:28.409 31376 31376 D linker  : find_library_in_linked_namespace(ns=(default), task=libdl_android.so): Not accessible (soname=libdl_android.so)
12-11 12:41:28.409 31376 31376 D linker  : find_library_in_linked_namespace(ns=com_android_art, task=libdl_android.so): Not accessible (soname=libdl_android.so)
12-11 12:41:28.409 31376 31376 D linker  : find_library_in_linked_namespace(ns=com_android_neuralnetworks, task=libdl_android.so): Not accessible (soname=libdl_android.so)
12-11 12:41:28.409 31376 31376 D linker  : find_library_in_linked_namespace(ns=com_android_os_statsd, task=libdl_android.so): Not accessible (soname=libdl_android.so)
12-11 12:41:28.409 31376 31376 D linker  : ... dlclose(realpath="/system/lib64/libinput.so"@0x7f2269931910) ... load group root is "/system/lib64/libinput.so"@0x7f2269931910
12-11 12:41:28.409 31376 31376 D linker  : ... dlclose: calling destructors for "/system/lib64/libinput.so"@0x7f2269931910 ...
12-11 12:41:28.409 31376 31376 D linker  : ... dlclose: calling destructors for "/system/lib64/libinput.so"@0x7f2269931910 ... done
12-11 12:41:28.409 31376 31376 D linker  : ... dlclose: calling destructors for "/system/lib64/libbinder.so"@0x7f2269932250 ...
12-11 12:41:28.409 31376 31376 D linker  : ... dlclose: calling destructors for "/system/lib64/libbinder.so"@0x7f2269932250 ... done
```

```cpp
extern "C" [[gnu::visibility("default")]] [[gnu::used]]
NativeOnModuleLoaded native_init(const NativeAPIEntries *entries) {
    hook_func = entries->hook_func;
    SandHook::ElfImg elf("libinput.so");
    auto target = elf.getSymbAddress("_ZN7android14InputPublisher18publishMotionEventEjiiiiNSt3__15arrayIhLm32EEEiiiiiiNS_20MotionClassificationEfffffffflljPKNS_17PointerPropertiesEPKNS_13PointerCoordsE");
    LOGD("function=%p", target);
    auto r = hook_func(target, (void *) PublishMotionEventFunc_API30_new,
                           (void **) &PublishMotionEventFunc_API30_old);
    LOGD("hook result=%d", r);
    return on_library_loaded;
}
```
