# Windows 上构建 Magisk 踩坑  

你以为 magisk 的构建只要简单的 `python build.py all` 吗？

## ONDK

一上来就报错：NDK 找不到。我以为 Magisk 用的 Android NDK ，自己的 NDK 已经下了十甚至九个版本怎么会找不到，仔细一看，发现用的是自己造的轮子 `ONDK`

[topjohnwu/ondk: Oxidized NDK - NDK repackaged with Rust toolchain](https://github.com/topjohnwu/ondk)

考虑到用 `build.py ndk` 下载可能会很慢，因此自己找对版本下载了，然后解压到 `$ANDROID_SDK_ROOT/ndk/magisk` 即可。

## jlink

构建 stub 的时候报错 jlink 找不到，原来自己系统用的 LibericaJRE-16 竟然没这个玩意。

干脆用 AS 的 JDK:

```bat
:: magiskenv.bat
@echo off
set _PATH=%PATH%
set PATH=C:\Program Files\Android\Android Studio\jre\bin;%_PATH%
set ANDROID_SDK_ROOT=C:\Users\mspri\AppData\Local\Android\Sdk
set JAVA_HOME=C:\Program Files\Android\Android Studio\jre

cmd /c%*

set PATH=%_PATH%
```

## cxx、 git symbolic link 与 Windows

stub 构建顺利通过，接下来是 native ，然而很快又出了个问题：

```
Caused by:
  process didn't exit successfully: `D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\target\release\build\cxxbridge-cmd-983ad08e8f052e4a\build-script-build` (exit code: 1)
  --- stderr

  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  When building `cxx` from a git clone, git's symlink support needs
  to be enabled on platforms that have it off by default (Windows).
  Either use:

     $ git config --global core.symlinks true

  prior to cloning, or else use:

     $ git clone -c core.symlinks=true https://github.com/dtolnay/cxx

  for the clone.

  Symlinks are only required when compiling locally from a clone of
  the git repository---they are NOT required when building `cxx` as
  a Cargo-managed (possibly transitive) build dependency downloaded
  through crates.io.
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
error: failed to compile `cxxbridge-cmd v1.0.72 (D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\gen\cmd)`, intermediate artifacts can be found at `D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\target`

cxxbridge-cmd installation failed!
```

看上去是仓库里有 symbol link ，而 Windows git 由于某种原因没有正确处理，导致报这个错。进一步发现这是个子模块 `cxx` 。

[dtolnay/cxx: Safe interop between Rust and C++](https://github.com/dtolnay/cxx)

按照提示，似乎只要执行 `git config --global core.symlinks true` 就能打开 symbolic link 的支持了。

于是输入，然后重新 `git submodule update --init --recursive` ……仍然报这个错。

`git config core.symlinks` 发现仓库的 config 是 `false` ，改了，删掉 `native/src/external/cxx` 再重复，还是一样。

原仓库 `gen/build/src` 下就有符号链接，检查了一下本地的，原本该是符号链接的文件在本地只是一个包含了路径的普通文件。

一度怀疑是用的长期不更新版本 git 导致的问题，然而这个版本 `2.2.28` ，看了一下 [release notes](https://github.com/git-for-windows/build-extra/blob/main/ReleaseNotes.md) ，早就支持了。并且我单独 git clone 这个仓库是有符号链接的。

[Symbolic Links · git-for-windows/git Wiki](https://github.com/git-for-windows/git/wiki/Symbolic-Links)

最后发现 `.git\modules\$module\config` 下有对每个模块的配置文件，其中 `core.symlinks` 还是 `false` ……

手动改了一下，再次 update ，总算过了这一关。

## Duplicated symbols

过了上面那一关，还有更难的：十万甚至九万个 duplicated symbols

```
...

ld: error: duplicate symbol: unshare
>>> defined in ./obj/local/x86_64/libcompat.a(objs/compat/compat/compat.o)
>>> defined in C:\Users\mspri\AppData\Local\Android\Sdk\ndk\magisk\build\..\toolchains\llvm\prebuilt\windows-x86_64\bin/../sysroot/usr/lib/x86_64-linux-android\libc.a(syscalls-x86_64.o)

ld: error: duplicate symbol: inotify_init1
>>> defined in ./obj/local/x86_64/libcompat.a(objs/compat/compat/compat.o)
>>> defined in C:\Users\mspri\AppData\Local\Android\Sdk\ndk\magisk\build\..\toolchains\llvm\prebuilt\windows-x86_64\bin/../sysroot/usr/lib/x86_64-linux-android\libc.a(syscalls-x86_64.o)

ld: error: duplicate symbol: getline
>>> defined in ./obj/local/x86_64/libcompat.a(objs/compat/compat/compat.o)
>>> defined in C:\Users\mspri\AppData\Local\Android\Sdk\ndk\magisk\build\..\toolchains\llvm\prebuilt\windows-x86_64\bin/../sysroot/usr/lib/x86_64-linux-android\libc.a(stdio.o)
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
make: *** [C:/Users/mspri/AppData/Local/Android/Sdk/ndk/magisk/build/../build/core/build-binary.mk:670: obj/local/x86_64/magiskboot] Error 1

Build binary failed!
```

仔细一看，`build.py` 的 `setup_ndk` 下载解压后还有个 patch 的步骤，而我们没做。

我自然不想再让它下载了，手动修一下把下载的步骤跳过，然后 `build.py ndk` 执行修补

```py
def setup_ndk(args):
    os_name = platform.system().lower()
    ndk_ver = config['ondkVersion']
    url = f'https://github.com/topjohnwu/ondk/releases/download/{ndk_ver}/ondk-{ndk_ver}-{os_name}.tar.gz'
    ndk_archive = url.split('/')[-1]

    header(f'* Downloading and extracting {ndk_archive}')
    '''
    with urllib.request.urlopen(url) as response:
        with tarfile.open(mode='r|gz', fileobj=response) as tar:
            tar.extractall(ndk_root)

    rm_rf(ndk_path)
    mv(op.join(ndk_root, f'ondk-{ndk_ver}'), ndk_path)'''

    header('* Patching static libs')
    for target in ['aarch64-linux-android', 'arm-linux-androideabi',
                   'i686-linux-android', 'x86_64-linux-android']:
        arch = target.split('-')[0]
        lib_dir = op.join(
            ndk_path, 'toolchains', 'llvm', 'prebuilt', f'{os_name}-x86_64',
            'sysroot', 'usr', 'lib', f'{target}', '21')
        if not op.exists(lib_dir):
            continue
        src_dir = op.join('tools', 'ndk-bins', '21', arch)
        rm(op.join(src_dir, '.DS_Store'))
        shutil.copytree(src_dir, lib_dir, copy_function=cp, dirs_exist_ok=True)
```

改好后：

> 此处开了详细信息： `build.py -v all`

![](res/images/20220917_04.png)

## YATTAZE！

正确配置的 build 执行得不算太慢，一会就出结果了：

![](res/images/20220917_05.png)

编译好的 Magisk apk 和 stub 输出在 out 目录下。

![](res/images/20220917_06.png)

在 emulator 跑一下：

![](res/images/20220917_07.png)

# 2023.04.03

已经很久没更新过 avd 上的 magisk 了，现在打算用 lsp 的 metagisk ，因为有西大师亲自编写的 native bridge

实际上我自从用上 kernelsu 已经好久没有用过 magisk 了，wsl 嵌套 cuttlefish 虽然可以 kernelsu 但是很难用，又懒得研究 avd 怎么加载内核，而用现成的 apk 进行 avd magisk 又不好使了，只能自己编译。

## 吐血的 rust

```
C:\Users\mspri\AppData\Local\Android\Sdk\ndk\magisk\toolchains\rust\bin\cargo.exe external\cxx-rs\gen\cmd
  Installing cxxbridge-cmd v1.0.72 (D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\gen\cmd)
error: failed to compile `cxxbridge-cmd v1.0.72 (D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\gen\cmd)`, intermediate artifacts can be found at `D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\target`

Caused by:
  failed to get `clap` as a dependency of package `cxxbridge-cmd v1.0.72 (D:\codes\magisk_source\master\Magisk\native\src\external\cxx-rs\gen\cmd)`

Caused by:
  failed to load source for dependency `clap`

Caused by:
  Unable to update registry `crates-io`

Caused by:
  usage of sparse registries requires `-Z sparse-registry`

cxxbridge-cmd installation failed!

mv .cargo\config.toml.bak -> .cargo\config.toml
```

ONDK 的 cargo 还在 1.65 ，而我系统里面的 cargo 是 1.68 ，之前配置了 config ， registry 用了 sparse index

https://blog.rust-lang.org/inside-rust/2023/01/30/cargo-sparse-protocol.html

临时解决办法是把 `%UserProfile%\.cargo\config` 改名

## 我 jdk 去哪了？

```
ERROR: JAVA_HOME is set to an invalid directory: C:\Program Files\Android\Android Studio\jre

Please set the JAVA_HOME variable in your environment to match the
location of your Java installation.

Build app failed!
```

更新了 AS 之后，自带 jre 竟然消失了……

搜索了一下 jlink 的位置，发现没消失，只是改名叫 jbr 了：`C:\Program Files\Android\Android Studio\jbr`

## L.S.偏执狂

```
FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':buildSrc:compileKotlin'.
> Could not resolve all files for configuration ':buildSrc:compileClasspath'.
   > Could not resolve org.lsposed.lsparanoid:gradle-plugin:0.5.0.
     Required by:
         project :buildSrc
      > No matching variant of org.lsposed.lsparanoid:gradle-plugin:0.5.0 was found. The consumer was configured to find a library for use during compile-time, compatible with Java 11, preferably not packaged as a jar, preferably optimized for standard JVMs, and its dependencies declared externally, as well as attribute 'org.gradle.plugin.api-version' with value '8.0.1', attribute 'org.jetbrains.kotlin.platform.type' with value 'jvm' but:
          - Variant 'apiElements' capability org.lsposed.lsparanoid:gradle-plugin:0.5.0 declares a library for use during compile-time, packaged as a jar, preferably optimized for standard JVMs, and its dependencies declared externally, as well as attribute 'org.jetbrains.kotlin.platform.type' with value 'jvm':
              - Incompatible because this component declares a component, compatible with Java 17 and the consumer needed a component, compatible with Java 11
              - Other compatible attribute:
```

https://github.com/LSPosed/LSParanoid

> Note that you should use at least Java 17 to launch the gradle daemon for this plugin (this is also required by AGP 8+). The project that uses this plugin on the other hand does not necessarily to target Java 17.

看来 AS JDK 也不行了，只好用 MS 的 JDK 17

## 历史遗留产物

世界上最讨厌的事情就是执行 gradle task 报错，一看 stack trace 根本没有自己的类——我怎么知道哪里错了！

```
* What went wrong:
Execution failed for task ':stub:processDebugManifestForPackage'.
> A failure occurred while executing com.android.build.gradle.tasks.ProcessPackagedManifestTask$WorkItem
   > Attribute "appComponentFactory" bound to namespace "http://schemas.android.com/apk/res/android" was already specified for element "application".

```

看起来是合并 manifest 出现的问题，appComponentFactory 似乎被重复了。

在 stub 的 build 目录下查找：

```
stub\build\intermediates\manifest_merge_blame_file\debug\manifest-merger-blame-debug-report.txt

29    <application
29-->D:\codes\magisk_source\master\Magisk\stub\src\main\AndroidManifest.xml:10:5-11:19
30        android:name="o.x"
30-->D:\codes\magisk_source\master\Magisk\stub\src\debug\AndroidManifest.xml:12:9-27
31        android:allowBackup="false"
31-->[:app:shared] D:\codes\magisk_source\master\Magisk\app\shared\build\intermediates\merged_manifest\debug\AndroidManifest.xml:28:9-36
32        android:appComponentFactory="j.I"
32-->D:\codes\magisk_source\master\Magisk\stub\src\debug\AndroidManifest.xml:11:9-42
33        android:debuggable="true"
34        android:extractNativeLibs="false"
35        android:label="Magisk"
```

里面只有一个 appComponentFactory

但是看到最终合并的结果：

```
stub\build\intermediates\merged_manifest\debug\AndroidManifest.xml

    <application
        android:appComponentFactory="s.p"
        android:name="uGK.G1"
        android:name="o.x"
        android:allowBackup="false"
        android:appComponentFactory="j.I"
        android:debuggable="true"
        android:extractNativeLibs="false"
        android:label="Magisk"
        android:requestLegacyExternalStorage="true"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.Translucent.NoTitleBar"
        android:usesCleartextTraffic="true" >
```

不仅 appComponentFactory 重复了，甚至 name 也重复了。

发现 stub/src 下有 debug 和 release 目录包含了 AndroidManifest ，这两个目录被 gitignore 了，看起来是生成的目录，从创建时间上看也确实，因此是以前编译的遗留产物。

```
Magisk\stub\src\debug\AndroidManifest.xml

    <application
        android:appComponentFactory="j.I"
        android:name="o.x"
        tools:ignore="GoogleAppIndexingWarning,MissingApplicationIcon,UnusedAttribute">
```

就是这里提供了错误的 appComponentFactory 和 name ，在 `stub\build\generated\source\obfuscate` 还能看到这两个类。

上面的应该是旧的混淆方法，现在新的混淆类放在了 `stub\build\generated\source\app|factory` 里面。

删掉 src 下面的两个生成的目录，总算可以过编译了。然而为何生成的东西放在 src 下面，而且 clean 的时候竟然也没被清掉？magisk 的暗坑还是太多了。


