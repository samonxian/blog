# HTTPS 的基本原理

`HTTPS = HTTP + TLS/SSL`，简单理解 HTTPS 其实就是在 HTTP 上面加多了一层安全层。HTTP 可以是 Http2.0 也可以是 Http1.1，不过现在 Http2.0 是强制要求使用 Https 的。

## HTTPS 基本原理

首先需要一个第三方认证机构（[CA认证](https://baike.baidu.com/item/CA认证)），确保公钥的合法性（即证书，不合法的证书浏览器会警告），然后利用非对称加密（公钥私钥）方式加密并传输共享密钥到服务器，可以确保共享密钥无法被拦截被获取到（共享密钥被公钥加密了，只有对应的私钥才能解密，服务器有私钥），最终的客户端和服务端 HTTP 传输就是使用共享秘钥加密进行通信。

## HTTPS 流程图

首先，我们先看下 HTTPS 请求的整个流程。

```sequence
客户端->服务器: 1.发送 ClientHello 报文
Note right of 服务器: 服务器可进行 SSL 通信时
Note right of 服务器: 发送 ServerHello
Note left of 客户端: 如果没收到 SeverHello 回应
Note left of 客户端: 响应无效，说明不支持 HTTPS
服务器->客户端: 2.发送 ServerHello 报文
服务器->客户端: 3.发送 Certificate 报文（包含公开密钥证书）  
服务器->客户端: 4.发送 ServerHelloDone 报文
Note left of 客户端: SSL 第一次握手结束
Note left of 客户端: 客户端会验证公钥是否合法
Note left of 客户端: 取出公钥
Note left of 客户端: 公钥加密
Note left of 客户端: 得到 Pre-master secret
客户端->服务器: 5.发送 ClientKeyExchange 报文\n 包含公钥加密的共享密钥 \n （Pre-master secret）
客户端->服务器: 6.发送 ChangeCipherSpec 报文 \n 提示服务器使用 Pre-master secret 密钥加密
客户端->服务器: 7.发送 Finished 报文
服务器->客户端: 8.发送 ChangeCipherSpec 报文 \n 提示客户端使用 Pre-master secret 密钥加密
服务器->客户端: 9.发送 Finished 报文
Note right of 服务器: SSL 链接建立完成
Note right of 服务器: 通信会受到 SSL 的保护
Note right of 服务器: 后续的 HTTP 通信
Note right of 服务器: 使用 Pre-master secret 加密
Note right of 服务器: 后续的流程跟 HTTP 请求一致
客户端->服务器: 10.发送 HTTP 请求报文
服务器->客户端: 11.发送 HTTP 响应报文
客户端->服务器: 12.发送 close_notify 报文 \n 客户端断开连接
Note right of 服务器: 到此整个 Https 请求结束
```



## HTTPS 是如何确保安全的？

- 使用非对称密钥（即[公钥私钥]([https://zh.wikipedia.org/wiki/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86](https://zh.wikipedia.org/wiki/公开密钥加密))）和[对称密钥]([https://zh.m.wikipedia.org/zh-hans/%E5%B0%8D%E7%A8%B1%E5%AF%86%E9%91%B0%E5%8A%A0%E5%AF%86](https://zh.m.wikipedia.org/zh-hans/對稱密鑰加密))（即共享密钥）相结合

  通过公钥私钥的方式，避免了共享密钥发送途中被第三方拦截获取密钥的安全问题。

  通过公钥和私钥加密建立保护层（即 SSL 保护层），后续的 Http 请求就会使用共享密钥进行加密通信（共享的密钥已经被 SSL 保护起来了，外面无法拦截到），即所谓的安全层。

  所以建立了安全层后，即使 HTTP 报文被拦截到，也无法解密。

- CA 认证

  由于公钥这个环节是公开的，存在被替换的风险，所以就有了第三方证书认证公司（[CA认证](https://baike.baidu.com/item/CA认证)），浏览器通过判断证书是否有效，发现网站是否值得信任。

  一般系统或者浏览器都会内置信任的根证书（这些 CA 组织都是非常可信的），浏览器可以根据这个根证书判断网站的证书是否合法。

  证书如果不合法，那么浏览器就会警告，不给用户访问证书不合法的网站，除非用户跳过这个警告。















































