* 如何配置
* 每个选项需要怎么设置，与啥用
* 如何过来pc的网络，只看手机的
* 如何排查问题（抓手机浏览器验证）
* 常见使用技巧

在 Android 的网络安全配置 (Network Security Configuration) 中，`<debug-overrides>` 部分的 `<trust-anchors>` 定义了调试时信任的证书来源。`<certificates src="user">` 和 `<certificates src="system">` 分别指的是不同的证书存储位置：

1. `<certificates src="user">`：

   - 指用户添加的证书。
   - 这些是用户手动安装到设备中的证书。
   - 通常位于设备的用户证书存储区域。
   - 用户可以通过设置应用程序添加这些证书。
   - 这允许开发者在调试时使用自签名证书或自定义证书颁发机构（CA）的证书。

2. `<certificates src="system">`：

   - 指系统预装的证书。
   - 这些证书是 Android 系统自带的，通常由知名的证书颁发机构（CA）提供。
   - 存储在系统的证书存储区域。
   - 这些证书通常被认为是可信的，包括了大多数公共网站使用的证书的根证书。

在调试配置中包含这两种来源意味着：

- 应用在调试模式下将信任both用户安装的证书和系统预装的证书。
- 这对于开发和测试很有用，特别是当你需要使用自签名证书或内部CA签发的证书时。
- 它允许你在开发环境中模拟各种SSL/TLS场景，而不影响生产版本的安全性。

重要注意事项：

- 这种配置只应该在调试版本中使用，不应该在发布版本中使用。
- 在生产环境中，通常只信任系统证书，以确保最高级别的安全性。
- 使用用户证书可能会带来安全风险，因为恶意用户可能会安装自己的证书来进行中间人攻击。

总的来说，这个配置为开发者提供了更大的灵活性，以便在开发和测试过程中处理各种SSL/TLS情况，同时不影响最终用户的安全性。

https://juejin.cn/post/6844903783672053773
https://www.jianshu.com/p/ad917b896635
https://blog.csdn.net/lanlangaogao/article/details/125161392
https://blog.csdn.net/weixin_41913122/article/details/104992412
https://blog.csdn.net/zhangkaiyazky/article/details/102854222