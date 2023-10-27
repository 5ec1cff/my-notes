# Frida Java Bridge

https://github.com/frida/frida-java-bridge

## implementation

https://github.com/frida/frida-java-bridge/blob/58030ace413a9104b8bf67f7396b22bf5d889e43/lib/class-factory.js#L1676

https://github.com/frida/frida-java-bridge/blob/58030ace413a9104b8bf67f7396b22bf5d889e43/lib/android.js#L3347

`ArtMethodMangler.replace`

复制原 ArtMethod 得到 replacement 方法，是一个 native 方法，实现放在 jniCode 中。

```js
    patchArtMethod(replacementMethodId, {
      jniCode: impl,
      accessFlags: ((originalFlags & ~(kAccCriticalNative | kAccFastNative | kAccNterpEntryPointFastPathFlag)) | kAccNative | kAccCompileDontBother) >>> 0,
      quickCode: api.artClassLinker.quickGenericJniTrampoline,
      interpreterCode: api.artInterpreterToCompiledCodeBridge
    }, vm);
```

原方法强制走解释器

```js
    // Remove kAccFastInterpreterToInterpreterInvoke and kAccSkipAccessChecks to disable use_fast_path
    // in interpreter_common.h
    let hookedMethodRemovedFlags = kAccFastInterpreterToInterpreterInvoke | kAccSingleImplementation | kAccNterpEntryPointFastPathFlag;
    if ((originalFlags & kAccNative) === 0) {
      hookedMethodRemovedFlags |= kAccSkipAccessChecks;
    }

    patchArtMethod(hookedMethodId, {
      accessFlags: ((originalFlags & ~(hookedMethodRemovedFlags)) | kAccCompileDontBother) >>> 0
    }, vm);

    const quickCode = this.originalMethod.quickCode;

    // Replace Nterp quick entrypoints with art_quick_to_interpreter_bridge to force stepping out
    // of ART's next-generation interpreter and use the quick stub instead.
    if (artNterpEntryPoint !== undefined && quickCode.equals(artNterpEntryPoint)) {
      patchArtMethod(hookedMethodId, {
        quickCode: api.artQuickToInterpreterBridge
      }, vm);
    }
```

### kAccFastInterpreterToInterpreterInvoke

强制走 switch 实现而不是 mterp ，这样才能进入 DoCall

A11：

http://aospxref.com/android-11.0.0_r21/xref/art/runtime/art_method.h#319

```cpp
bool UseFastInterpreterToInterpreterInvoke() const {
     // The bit is applicable only if the method is not intrinsic.
     constexpr uint32_t mask = kAccFastInterpreterToInterpreterInvoke | kAccIntrinsic;
     return (GetAccessFlags() & mask) == kAccFastInterpreterToInterpreterInvoke;
}
```

http://aospxref.com/android-11.0.0_r21/xref/art/runtime/interpreter/interpreter_common.h?fi=DoInvoke#319

```cpp
 static ALWAYS_INLINE bool DoInvoke(Thread* self,
                                    ShadowFrame& shadow_frame,
                                    const Instruction* inst,
                                    uint16_t inst_data,
                                    JValue* result)
use_fast_path = called_method->UseFastInterpreterToInterpreterInvoke();

if (use_fast_path) {
    // mterp
}

return DoCall<is_range, do_access_check>(called_method, self, shadow_frame, inst, inst_data,
                                            result);
```

## art controller

https://github.com/frida/frida-java-bridge/blob/58030ace413a9104b8bf67f7396b22bf5d889e43/lib/android.js#L1841

hook `art::interpreter::DoCall`

```js
function instrumentArtMethodInvocationFromInterpreter () {
  const apiLevel = getAndroidApiLevel();

  let artInterpreterDoCallExportRegex;
  if (apiLevel <= 22) {
    artInterpreterDoCallExportRegex = /^_ZN3art11interpreter6DoCallILb[0-1]ELb[0-1]EEEbPNS_6mirror9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE$/;
  } else {
    artInterpreterDoCallExportRegex = /^_ZN3art11interpreter6DoCallILb[0-1]ELb[0-1]EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE$/;
  }

  for (const exp of Module.enumerateExports('libart.so').filter(exp => artInterpreterDoCallExportRegex.test(exp.name))) {
    Interceptor.attach(exp.address, artController.hooks.Interpreter.doCall);
  }
}
```

https://github.com/frida/frida-java-bridge/blob/58030ace413a9104b8bf67f7396b22bf5d889e43/lib/android.js#L1706C1-L1716C2

替换第一个参数为被替换的方法

```c
void
on_interpreter_do_call (GumInvocationContext * ic)
{
  gpointer method, replacement_method;

  method = gum_invocation_context_get_nth_argument (ic, 0);

  replacement_method = get_replacement_method (method);
  if (replacement_method != NULL)
    gum_invocation_context_replace_nth_argument (ic, 0, replacement_method);
}
```

...

```
    #22 pc 000000000013f7d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #23 pc 0000000000a2d58c  /data/app/~~sZ208PWnIVkknU3lQtjBAQ==/io.github.a13e300.demo-SSntgDgXpiAKakwGSEo49A==/oat/arm64/base.vdex (io.github.a13e300.demo.SyncCallActivity.onCreate)
    #24 pc 0000000000306dc0  /apex/com.android.art/lib64/libart.so (art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) (.llvm.11487796752256266877)+532) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #25 pc 000000000030eca8  /apex/com.android.art/lib64/libart.so (art::interpreter::ArtInterpreterToInterpreterBridge(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame*, art::JValue*)+200) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #26 pc 000000000030f6a0  /apex/com.android.art/lib64/libart.so (bool art::interpreter::DoCall<false, false>(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, art::JValue*)+968) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #27 pc 00000000001489d0  /apex/com.android.art/lib64/libart.so (void art::interpreter::ExecuteSwitchImplCpp<false, false>(art::interpreter::SwitchImplContext*)+28080) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #28 pc 000000000013f7d8  /apex/com.android.art/lib64/libart.so (ExecuteSwitchImplAsm+8) (BuildId: 734ef1b8ef156af69af806e34ced7fac)
    #29 pc 00000000001b012c  /system/framework/framework.jar (android.app.Activity.performCreate)
```


