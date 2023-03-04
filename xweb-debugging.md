# XWeb 调试

前段时间升级了微信，现在想要提取青年大学习的 cookie ，发现以前 8.0.7 开启调试的方法行不通了。

以前是强制开启 x5 内核，然后在 `debugx5.qq.com` 开启调试。据说现在换成了 XWeb 内核，方法又不一样了。

WebviewPP 的 hookXWeb 方法似乎行不通，不过稍微逆向了一下，发现其实不用 Xposed 也可以。

在任意聊天栏输入地址 `http://debugxweb.qq.com/?inspector=true` ，打开后如果跳转到微信首页说明成功。

另外，如果开启了 WebviewPP 反而会失效，原因不明。
