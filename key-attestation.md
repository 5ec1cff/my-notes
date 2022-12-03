# <ruby>密钥认证<rt>Key Attestation</rt></ruby>

[vvb2060/KeyAttestation](https://github.com/vvb2060/KeyAttestation)

[Verifying hardware-backed key pairs with Key Attestation  |  Android Developers](https://developer.android.com/training/articles/security-key-attestation)

[Android Keystore system  |  Android Developers](https://developer.android.com/training/articles/keystore)

```kt
// app/src/main/java/io/github/vvb2060/keyattestation/home/HomeViewModel.kt
    @Throws(GeneralSecurityException::class)
    private fun generateKey(alias: String, useStrongBox: Boolean, includeProps: Boolean) {
        val keyPairGenerator = KeyPairGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore")
        val now = Date()
        val originationEnd = Date(now.time + 1000000)
        val consumptionEnd = Date(now.time + 2000000)
        val builder = KeyGenParameterSpec.Builder(alias, KeyProperties.PURPOSE_SIGN)
                .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
                .setKeyValidityStart(now)
                .setKeyValidityForOriginationEnd(originationEnd)
                .setKeyValidityForConsumptionEnd(consumptionEnd)
                .setAttestationChallenge(now.toString().toByteArray())
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && includeProps) {
            builder.setDevicePropertiesAttestationIncluded(true)
        }
        if (Build.VERSION.SDK_INT >= 28 && useStrongBox) {
            builder.setIsStrongBoxBacked(true)
        }
        builder.setDigests(KeyProperties.DIGEST_NONE, KeyProperties.DIGEST_SHA256)
        keyPairGenerator.initialize(builder.build())
        keyPairGenerator.generateKeyPair()
    }

    @Throws(AttestationException::class)
    private fun doAttestation(alias: String, useStrongBox: Boolean, includeProps: Boolean
    ): AttestationResult {
        val certs: Array<X509Certificate?>?
        val attestation: Attestation
        val isGoogleRootCertificate: Boolean
        try {
            val keyStore = KeyStore.getInstance("AndroidKeyStore")
            keyStore.load(null)
            generateKey(alias, useStrongBox, includeProps)
            val certificates = keyStore.getCertificateChain(alias)
            certs = arrayOfNulls(certificates.size)
            for (i in certs.indices) certs[i] = certificates[i] as X509Certificate
        } catch (e: ProviderException) {
            if (Build.VERSION.SDK_INT >= 28 && e is StrongBoxUnavailableException) {
                throw AttestationException(AttestationException.CODE_STRONGBOX_UNAVAILABLE, e)
            } else if (e.cause?.message?.contains("device ids") == true) {
                // The device does not support device ids attestation
                throw AttestationException(AttestationException.CODE_DEVICEIDS_UNAVAILABLE, e)
            } else {
                // The device does not support key attestation
                throw AttestationException(AttestationException.CODE_NOT_SUPPORT, e)
            }
        } catch (e: Exception) {
            // Unable to get certificate
            throw AttestationException(AttestationException.CODE_NOT_SUPPORT, e)
        }
        try {
            isGoogleRootCertificate = VerifyCertificateChain.verifyCertificateChain(certs)
        } catch (e: GeneralSecurityException) {
            // Certificate is not trusted
            throw AttestationException(AttestationException.CODE_CERT_NOT_TRUSTED, e)
        }
        try {
            attestation = Attestation(certs[0])
        } catch (e: CertificateParsingException) {
            // Unable to parse attestation record
            throw AttestationException(AttestationException.CODE_CANT_PARSE_ATTESTATION_RECORD, e)
        }

        return AttestationResult(useStrongBox, attestation, isGoogleRootCertificate)
    }
```

关键在于 `KeyStore.getCertificateChain` ，返回的证书链的第一项包含一些信息与设备解锁状态、boot hash 有关。（API 24 开始）

> https://developer.android.com/training/articles/security-key-attestation#root_certificate

google 给出了根证书的公钥，如果用公钥验证匹配，说明根证书是 google 信任的。

```
-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAr7bHgiuxpwHsK7Qui8xU
FmOr75gvMsd/dTEDDJdSSxtf6An7xyqpRR90PL2abxM1dEqlXnf2tqw1Ne4Xwl5j
lRfdnJLmN0pTy/4lj4/7tv0Sk3iiKkypnEUtR6WfMgH0QZfKHM1+di+y9TFRtv6y
//0rb+T+W8a9nsNL/ggjnar86461qO0rOs2cXjp3kOG1FEJ5MVmFmBGtnrKpa73X
pXyTqRxB/M0n1n/W9nGqC4FSYa04T6N5RIZGBN2z2MT5IKGbFlbC8UrW0DxW7AYI
mQQcHtGl/m00QLVWutHQoVJYnFPlXTcHYvASLu+RhhsbDmxMgJJ0mcDpvsC4PjvB
+TxywElgS70vE0XmLD+OJtvsBslHZvPBKCOdT0MS+tgSOIfga+z1Z1g7+DVagf7q
uvmag8jfPioyKvxnK/EgsTUVi2ghzq8wm27ud/mIM7AY2qEORR8Go3TVB4HzWQgp
Zrt3i5MIlCaY504LzSRiigHCzAPlHws+W0rB5N+er5/2pJKnfBSDiCiFAVtCLOZ7
gLiMm0jhO2B6tUXHI/+MRPjy02i59lINMRRev56GKtcd9qO/0kUJWdZTdA2XoS82
ixPvZtXQpUpuL12ab+9EaDK8Z4RHJYYfCT3Q5vNAXaiWQ+8PTWm2QgBR/bkwSWc+
NpUFgNPN9PvQi8WEg5UmAGMCAwEAAQ==
-----END PUBLIC KEY-----
```

> https://github.com/vvb2060/KeyAttestation/blob/c5e18d915ae6335314fd940e0559958203e069da/app/src/main/java/io/github/vvb2060/keyattestation/attestation/VerifyCertificateChain.java

java.security.KeyStore:

```
libcore/ojluni/src/main/java/java/security/KeyStore.java
libcore/ojluni/src/main/java/java/security/Provider.java
libcore/ojluni/src/main/java/java/security/KeyStoreSpi.java
```

Android 的实现(现在是 Keystore2)：

```
frameworks/base/keystore/java/android/security/KeyStore2.java
frameworks/base/keystore/java/android/security/keystore2/AndroidKeyStoreSpi.java
frameworks/base/keystore/java/android/security/keystore2/AndroidKeyStoreProvider.java
```

binder 服务 IKeystoreService 由 /system/bin/keystore 实现。

```sh
# service list | grep keystore
18      android.security.keystore: [android.security.keystore.IKeystoreService]
# ps -ef|grep keyst
keystore  1537     1  0 05:01 ?        00:00:01 /system/bin/keystore /data/misc/keystore
# service call android.security.keystore 1599097156
Result: Parcel(00000601    '....') # 1537
```

服务接口和实现（目前是 keystore2）：

```
system/hardware/interfaces/keystore2/aidl/android/system/keystore2/IKeystoreService.aidl
system/security/keystore2/src/service.rs

system/security/keystore/
```

## 检测应用是否访问 keystore

如果应用访问了 keystore 服务，则应用可能进行了 key attestation 。

我们可以用 binder debugfs 来观察应用是否访问 keystore 服务：

```sh
cat /d/binder/proc/$(pidof keystore)
```

可以看到第一个 binder node 引用的进程较多

```
binder proc state:
proc 1537
context binder
  thread 1537: l 12 need_return 0 tr 0
  thread 22091: l 00 need_return 0 tr 0
  node 1803: ub4000079f9615560 cb4000079f960d5c8 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 7 iw 7 tr 1 proc 28455 29271 28303 28869 5251 1838 569
  node 191752788: ub4000079f96158c0 cb4000079f96066c0 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 29271
  node 191495115: ub4000079f9615ae0 cb4000079f9606720 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 28869
  node 191790298: ub4000079f9615da0 cb4000079f7c4e120 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 29271
  ref 1801: desc 0 node 4 s 1 w 1 d 0000000000000000
  ref 191790310: desc 1 node 191790309 s 1 w 1 d 0000000000000000
  ref 191790293: desc 3 node 191790292 s 1 w 0 d 0000000000000000
  ref 416917: desc 4 node 15351 s 1 w 1 d 0000000000000000
binder proc state:
proc 1537
context hwbinder
  thread 1537: l 00 need_return 0 tr 0
  thread 22091: l 00 need_return 0 tr 0
  ref 1733: desc 0 node 1 s 1 w 1 d 0000000000000000
  ref 1781: desc 2 node 122 s 1 w 1 d 0000000000000000
  buffer 1789: 0000000000000000 size 8:0:0 delivered
  buffer 191790316: 0000000000000000 size 168:24:176 delivered
```

我们启动 key attestation ，再 print ：

```
# cat /d/binder/proc/$(pidof keystore)
binder proc state:
proc 1537
context binder
  thread 1537: l 12 need_return 0 tr 0
  thread 22491: l 00 need_return 0 tr 0
  node 1803: ub4000079f9615560 cb4000079f960d5c8 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 8 iw 8 tr 1 proc 22457 28455 29271 28303 28869 5251 1838 569
  node 191752788: ub4000079f96158c0 cb4000079f96066c0 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 29271
  node 191495115: ub4000079f9615ae0 cb4000079f9606720 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 28869
  node 191790298: ub4000079f9615da0 cb4000079f7c4e120 pri 0:139 hs 1 hw 1 ls 0 lw 0 is 1 iw 1 tr 1 proc 29271
  ref 1801: desc 0 node 4 s 1 w 1 d 0000000000000000
  ref 191801757: desc 2 node 191801756 s 1 w 1 d 0000000000000000
  ref 416917: desc 4 node 15351 s 1 w 1 d 0000000000000000
```

引用的进程 pid 出现了 22457 ，正是 key attestation 的。启动 momo 也类似。这些应用都进行了 key attestation 来检测解锁状态。

```
# ps -ef|grep key
keystore  1537     1  0 Nov27 ?        00:00:33 /system/bin/keystore /data/misc/keystore
u0_a266  22457   762 12 12:42 ?        00:00:00 io.github.vvb2060.keyattestation
root     22518 22220  1 12:42 pts/0    00:00:00 grep --color key
```

我们也可以用代理 binder 的方法：

> 这是 Rikka 曾经提出的通过 addService 来代理 binder 的做法

```kt
class ProxyBinder(private val orig: IBinder) : Binder() {
    override fun onTransact(code: Int, data: Parcel, reply: Parcel?, flags: Int): Boolean {
        return try {
            println("transact $code from u${getCallingUid()}/p${getCallingPid()} (oneway=${flags and FLAG_ONEWAY != 0})")
            orig.transact(code, data, reply, flags)
        } catch (t: Throwable) {
            println("error occurred while proxy binder transaction")
            println(t)
            false;
        }
    }
}

fun runBinderProxy(arg: ArgParser) {
    val name = arg.getOption("n")
    val oldBinder = ServiceManager.getService(name)!!
    println("get binder for service $name : $oldBinder")
    try {
        val newBinder = ProxyBinder(oldBinder)
        ServiceManager.addService(name, newBinder)
        println("replaced service $name\nPress any key to exit")
        System.`in`.read()
    } finally {
        println("recovery service $name")
        ServiceManager.addService(name, oldBinder)
    }
}
```

上面的代码可以代理在 servicemanager 注册的 binder 服务，并观察 transaction ，打印 calling pid 。

这在 AVD Android 12 可用，在朋友的 MIUI (Android 12) 手机上也可用，但唯独我自己的 MIUI (Android 11) 始终没法代理到服务，而 addService 的过程并没有报错，目前暂时不清楚原因，也许这个版本魔改了 servicemanager 。
