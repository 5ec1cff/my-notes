# MIUI 权限  

## 「自启动」权限  

自启动看起来包括了服务重启（导致无障碍服务 crash 后无法自动重启，看上去被系统认为是结束了），此外还有 wakepath 之类的东西。

包含自启动设置的 Activity :`com/miui/appmanager/ApplicationsDetailsActivity` ，属于 `com.miui.securitycenter` 包（也就是应用详情）。

`com/miui/permission/PermissionManager` 包含了一系列权限控制的方法，以及常量（见[附录 1](#constants)）

权限控制由另一个包 `com.lbe.security.miui` 的 ContentProvider 提供

```xml
<provider
    android:name="com.lbe.security.service.provider.PermissionManagerProvider"
    android:permission="miui.permission.READ_AND_WIRTE_PERMISSION_MANAGER"
    android:exported="true"
    android:authorities="com.lbe.security.miui.permmgr" />
```

有一个自定义权限，这个权限是 signatureOrSystem 的，而且 shell 没有权限，只能 root 访问了。

## 设置 App 权限

参照 PermissionManager 的 setApplicationPermissions 方法写了一个设置权限的方法：

```kotlin
private const val AUTHORITY = "com.lbe.security.miui.permmgr"

fun setAppPermission(packageName: String, permId: Long, action: Int, flags: Int=0) {
    // Os.seteuid(1000)
    useContentProvider(AUTHORITY) {
        println(it.callCompat(AUTHORITY, "6", null, Bundle().apply {
            putLong("extra_permission", permId)
            putInt("extra_action", action)
            putStringArray("extra_package", arrayOf(packageName))
            putInt("extra_flags", flags)
            println(this)
        }))
    }
}
```

> 其中，call 的方法是 `6` （并未在常量给出，具体作用可以反编译 `com.lbe.security.miui` 查看。  
> `extra_permission` 是一个类似于 flags 的东西（与内部存储方式有关，权限存储在 long 的每个位上），值为 `PERM_` 前缀的常量。`16384` 为自启动。  
> `extra_action` 为操作，`1` 禁止，`3` 允许。值为 `ACTION_` 前缀的常量。  
> `extra_package` 是包的列表（就是这个导致没法用 content 命令写😅）  
> `extra_flags` 默认为 0 。值为 `FLAG_` 前缀的常量。观察到有一个 flag 与杀死进程有关，而关闭自启时应用会被杀死（实在不懂这个意图），但 flags 仍为 0 。根据反编译发现是关闭的时候默认总是杀死应用（😅😅😅）  

## 查询 App 权限

查询权限的方法被混淆，参考了下面这个方法：

```
com/miui/permcenter/j a(Landroid/content/Context;JLjava/lang/String;)Lcom/miui/permcenter/d;
```

可以用 content query 查询：

```
content query --uri content://com.lbe.security.miui.permmgr/active --where 'pkgName="five.ec1cff.assistant"'
```

得到的结果如下：

```
Row: 0 _id=506
pkgName=five.ec1cff.assistant
installTime=1656347084014
uninstallTime=0
present=1
pruneAfterDelete=0
lastConfigured=0
lastUpdated=1656332731269
permMask=6309578099143000064
suggestAccept=1441168485102157824
suggestPrompt=283203338271
suggestReject=112607600083747328
forcedBits=0
permDesc=BLOB
userAccept=103079231488
userPrompt=0
userReject=0
suggestBlock=0
suggestForeground=4755801756259057664
userForeground=0
```

我们编写的设置权限方法会影响 `userAccept` 和 `userReject` （应该就是权限的 bitset）

## bug

如果应用详情的自启动显示为「允许」是在这个 ui 设置的，那么用上面的 call 方法设置为「禁止」之后并不会反映在 ui 中，也就是仍然显示为「允许」（甚至强杀重启都不会改变），但是 query 的结果明明和手动关闭一致。但如果是我们用 call 方法设置过的「允许」，就不会有上述问题。至于 call 了之后是不是真的起作用了，也有待进一步研究。

## 附录 1

<a id="constants"></a>

```java
    public static final int ACTION_ACCEPT = 3;
    public static final int ACTION_BLOCK = 4;
    public static final int ACTION_DEFAULT = 0;
    public static final int ACTION_FOREGROUND = 6;
    public static final int ACTION_NONBLOCK = 5;
    public static final int ACTION_PROMPT = 2;
    public static final int ACTION_REJECT = 1;
    public static final int ACTION_VIRTUAL = 7;
    public static final int FLAG_GRANT_ONTTIME = 4;
    public static final int FLAG_GRANT_THREESEC = 8;
    public static final int FLAG_KILL_PROCESS = 2;
    public static final int GET_APP_COUNT = 1;
    public static final int GROUP_CHARGES = 1;
    public static final int GROUP_MEDIA = 4;
    public static final int GROUP_PRIVACY = 2;
    public static final int GROUP_SENSITIVE_PRIVACY = 16;
    public static final int GROUP_SETTINGS = 8;
    public static final long PERM_ID_ACCESS_XIAOMI_ACCOUNT = 4294967296L;
    public static final long PERM_ID_ACTIVITY_RECOGNITION = 137438953472L;
    public static final long PERM_ID_ADD_VOICEMAIL = 281474976710656L;
    public static final long PERM_ID_AUDIO_RECORDER = 131072;
    public static final long PERM_ID_AUTOSTART = 16384;
    public static final long PERM_ID_BACKGROUND_LOCATION = 2305843009213693952L;
    public static final long PERM_ID_BACKGROUND_START_ACTIVITY = 72057594037927936L;
    public static final long PERM_ID_BLUR_LOCATION = 8589934592L;
    public static final long PERM_ID_BODY_SENSORS = 70368744177664L;
    public static final long PERM_ID_BOOT_COMPLETED = 0x08000000;
    public static final long PERM_ID_BT_CONNECTIVITY = 4194304;
    public static final long PERM_ID_CALENDAR = 0x01000000;
    public static final long PERM_ID_CALLLOG = 16;
    public static final long PERM_ID_CALLMONITOR = 2048;
    public static final long PERM_ID_CALLPHONE = 2;
    public static final long PERM_ID_CALLSTATE = 1024;
    public static final long PERM_ID_CLIPBOARD = 4611686018427387904L;
    public static final long PERM_ID_CONTACT = 8;
    public static final long PERM_ID_DEAMON_NOTIFICATION = 1152921504606846976L;
    public static final long PERM_ID_DISABLE_KEYGUARD = 8388608;
    public static final long PERM_ID_EXTERNAL_STORAGE = 35184372088832L;
    public static final long PERM_ID_GALLERY_RESTRICTION = 68719476736L;
    public static final long PERM_ID_GET_ACCOUNTS = 140737488355328L;
    public static final long PERM_ID_GET_INSTALLED_APPS = 144115188075855872L;
    public static final long PERM_ID_GET_TASKS = 18014398509481984L;
    public static final long PERM_ID_INSTALL_PACKAGE = 65536;
    public static final long PERM_ID_INSTALL_SHORTCUT = 4503599627370496L;
    public static final long PERM_ID_LOCATION = 32;
    public static final long PERM_ID_MEDIA_VOLUME = 549755813888L;
    public static final long PERM_ID_MMSDB = 262144;
    public static final long PERM_ID_MOBILE_CONNECTIVITY = 1048576;
    public static final long PERM_ID_NETDEFAULT = 128;
    public static final long PERM_ID_NETWIFI = 256;
    public static final long PERM_ID_NFC = 2251799813685248L;
    public static final long PERM_ID_NOTIFICATION = 32768;
    public static final long PERM_ID_PHONEINFO = 64;
    public static final long PERM_ID_PROCESS_OUTGOING_CALLS = 1125899906842624L;
    public static final long PERM_ID_READCALLLOG = 0x40000000;
    public static final long PERM_ID_READCONTACT = 2147483648L;
    public static final long PERM_ID_READMMS = 0x20000000;
    public static final long PERM_ID_READSMS = 0x10000000;
    public static final long PERM_ID_READ_CLIPBOARD = 274877906944L;
    public static final long PERM_ID_READ_NOTIFICATION_SMS = 9007199254740992L;
    public static final long PERM_ID_REAL_READ_CALENDAR = 4398046511104L;
    public static final long PERM_ID_REAL_READ_CALL_LOG = 8796093022208L;
    public static final long PERM_ID_REAL_READ_CONTACTS = 2199023255552L;
    public static final long PERM_ID_REAL_READ_PHONE_STATE = 17592186044416L;
    public static final long PERM_ID_REAL_READ_SMS = 1099511627776L;
    public static final long PERM_ID_ROOT = 512;
    public static final long PERM_ID_SENDMMS = 524288;
    public static final long PERM_ID_SENDSMS = 1;
    public static final long PERM_ID_SERVICE_FOREGROUND = 288230376151711744L;
    public static final long PERM_ID_SETTINGS = 8192;
    public static final long PERM_ID_SHOW_WHEN_LOCKED = 36028797018963968L;
    public static final long PERM_ID_SMSDB = 4;
    public static final long PERM_ID_SOCIALITY_RESTRICTION = 34359738368L;
    public static final long PERM_ID_SYSTEMALERT = 0x02000000;
    public static final long PERM_ID_UDEVICEID = 576460752303423488L;
    public static final long PERM_ID_USE_SIP = 562949953421312L;
    public static final long PERM_ID_VIDEO_RECORDER = 4096;
    public static final long PERM_ID_WAKELOCK = 0x04000000;
    public static final long PERM_ID_WIFI_CONNECTIVITY = 2097152;
```