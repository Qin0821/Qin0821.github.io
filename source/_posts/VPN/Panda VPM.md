### Panda VPN 
[官网FAQ](https://www.pantavv.xyz/faq)

#### 使用技巧

##### 1. 怎么让Mac终端走VPN代理？


方法一：
1). 在终端执行brew install proxychains-ng，安装proxychains-ng
2). 编辑/usr/local/etc/proxychains.conf，在最后一行改为: socks5 127.0.0.1 1090
注：1090 是本地socks5监听端口, 根据PandaVPN客户端设置里面端口数进行设置即可，socks5默认端口1090


方法二：
在终端执行export http_proxy=127.0.0.1:41091 https_proxy=127.0.0.1:41091
注：41091 是本地HTTP/HTTPS监听端口, 根据PandaVPN客户端设置里面端口数进行设置即可，HTTP/HTTPS默认端口41091

>实际操作下来还是ping不通google

##### 2.PC端怎么添加自定义规则？

添加自定义规则：
未选择全局模式，针对特定的网址（如www.abc.com）使用代理，您可以在pc端添加自定义规则，请打开客户端设置-高级设置，英文状态下||abc.com即可；如果非全局模式下希望某些网站不使用代理（如www.def.com），在前面加@@即可，如@@||def.com