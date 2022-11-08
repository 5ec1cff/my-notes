# 初探 oicq

[takayama-lily/oicq: Tencent QQ Bot Library for Node.js](https://github.com/takayama-lily/oicq)

## sendUni

[oicq/group.ts at main · takayama-lily/oicq](https://github.com/takayama-lily/oicq/blob/dbfb9d7a12929e2490adbb13670e70a17e7d054d/lib/group.ts#L59)

```ts
	/** 发送一条消息 */
	async sendMsg(content: Sendable): Promise<MessageRet> {
		const { rich, brief } = await this._preprocess(content)
		const body = pb.encode({
			1: { 4: { 1: this.gid } },
			2: PB_CONTENT,
			3: { 1: rich },
			4: randomBytes(2).readUInt16BE(),
			5: randomBytes(4).readUInt32BE(),
			8: 0,
		})
		const payload = await this.c.sendUni("MessageSvc.PbSendMsg", body)
		const rsp = pb.decode(payload)
		if (rsp[1] !== 0) {
			this.c.logger.error(`failed to send: [Discuss(${this.gid})] ${rsp[2]}(${rsp[1]})`)
			drop(rsp[1], rsp[2])
		}
		this.c.logger.info(`succeed to send: [Discuss(${this.gid})] ` + brief)
		return {
			message_id: "",
			seq: 0,
			rand: 0,
			time: 0,
		}
	}
```

sendUni 搜索发现来自这里：

[oicq/base-client.ts at 5201db4edf48267920b664fd605b7d35e97e3d74 · takayama-lily/oicq](https://github.com/takayama-lily/oicq/blob/5201db4edf48267920b664fd605b7d35e97e3d74/lib/core/base-client.ts#L519)

```ts
	/** 发送一个业务包并等待返回 */
	async sendUni(cmd: string, body: Uint8Array, timeout = 5) {
		if (!this[IS_ONLINE])
			throw new ApiRejection(-1, `client not online`)
		return this[FN_SEND](buildUniPkt.call(this, cmd, body), timeout)
	}
```

反编译 Android QQ 并搜索字符串 `MessageSvc` ，还是能找到不少东西的。

这些类似 `MessageSvc.xxx` 的命令被放入一个叫 `ToServiceMsg` 的类中，类全名：

`com/tencent/qphone/base/remote/ToServiceMsg`

同包名下还有 `IBaseService` 接口，似乎是个 Binder 接口，也许是为了方便 QQ 的多进程调用，其中一个方法 `sendToServiceMsg` 引起了我的注意。

`IBaseService$Stub` 的实现类的类名被混淆，在 8850 下是 `com/tencent/mobileqq/msf/service/p`

方法实现调用了 `MsfService;->(field)msfServiceReqHandler->(method)a(Context, ToServiceMsg, int)void`

```
com/tencent/mobileqq/msf/service/MsfService
msfServiceReqHandler: com/tencent/mobileqq/msf/service/t
```

## 小程序获取 url

```
接口：com/tencent/qqmini/sdk/launcher/core/proxy/ChannelProxy getAppInfoByLink
实现：com/tencent/qqmini/proxyimpl/ChannelProxyImpl

-> com/tencent/mobileqq/mini/reuse/MiniAppCmdUtil getAppInfoByLinkForSDK
-> mqq/app/AppRuntime startServlet (mqq/app/NewIntent)

com/tencent/mobileqq/mini/servlet/MiniAppGetAppInfoByIdForSDKServlet onSend
继承 mqq/app/MSFServlet service -> onSend -> sendToMSF (ToServiceMsg)

-> mqq/app/MainService sendMessageToMSF
-> msfSub com/tencent/mobileqq/msf/sdk/MsfServiceSdk sendMsg
```

看上去最终也是从 QQ 协议发出去的，因此可能可以放到 oicq 里面？
