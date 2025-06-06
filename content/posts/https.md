---
title: Https
date: 2019-11-15 14:50:20
lastmod: 2019-11-15 14:50:20
cover: http://img.sysummery.top/https_cover.jpeg
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - 后端
tags:
  - https
---

https是在应用层(http)与传输层之间多了一个SSL(TSL)层。这个层负责先将http要发送的数据使用一个密室加密之后再交给传输层。相应的，接收方的传输层先将加密的数据交给SSL(TLS)层解密后，再发送给应用层(http)。
<!--more-->
## 对称加密与非对称加密
对称加密: 通信双方的密匙是一样的，这个密匙既用于加密有用于解密。

非对称加密：有一对公匙和密匙。公匙加密的内容公匙自己不能解密，必须得私匙解密。同样的，私匙加密的内容私匙自己不能解密，得用公匙解密。

摘要：对一段内用使用hash函数生成一个固定长度的字符串。

数字签名：对摘要使用一个密匙(公匙)加密。

## Https通信过程

![https的简易流程](https://img.sysummery.top/https.jpg)

上图中还缺少一些步骤，比如通信双方还会对双方密码能力的信息，比如支持的SSL(TSL)版本，加密组件等进行协商约定等。但是本文只想介绍https的原理和关键步骤，所以没有画出。

总的来说，https最开始使用非对称加密在客户端与服务器之间产生一个密匙。之后的数据通信就使用这个密匙进行对称加密。

其中第三步是重点，服务器会给客户端发送一个证书，然后客户端根据验证这个证书的结果来判断是否执行下一步。

### CA颁发的证书
CA是一个在互联网上权威的第三方机构，我们可以简单的认为所有的网站和用户都信任他。

如果一个网站要支持https协议，那么他就得向CA申请一个证书。假如证书最开始是一张白纸，申请证书的流程如下

1. 网站向CA提供自身的一些信息，如果域名、公匙等，CA验证这些信息没问题后就把这些信息以明文的方式写到证书上。
2. 生成数字签名。对第一步的明文信息使用hash函数取摘要，然后用CA的密匙(私匙)对这个摘要加密就行程了数字签名。把这个数字签名也放到证书上。
3. 在证书上附上颁发证书机构的信息，以及证书的有效期等信息。

到此证书的制作就完成了。以后任何一个客户端想和网站发起https协议的时候网站都会先把这个证书给发起端检验。

### 发起端验证证书
为什么我用发起端这个词而不用客户端呢？因为服务器或网站也可以对客户端进行https校验，过程和原理也是一样的。那么当发起端收到证书后会有以下步骤的校验。

1. 颁发证书的CA机构是否合法，是否仍然有资质。
2. 验证证书是否在有效期内。
3. 验证数字签名。我们可以简单的认为，浏览器具有所有CA机构的公钥。浏览器用颁发该证书的CA的公匙，去解密数字签名。如果解密成功说明这个证书是该CA发的，否则证书有问题。解密之后就获得了这个证书明文部分的摘要。
4. 客户端用同样的hash函数对证书中的明文部分取摘要，如果与第三步中的摘要相同，那么说明证书的内容是准确的，没有被篡改过 。
5. 对明文部分的信息进行一些校验，比如域名。

此时发起端已经拿到了对方的公匙。

### 发起端与对方交换"对称加密"的密匙
发起端随机生成一个加密字符串，然后使用对方的公匙非对称加密，发送给对方。对方收到后用私匙解密获取这个字符串。然后对方会告诉发起端，我已收到这个加密字符串，以后咱俩再通信的时候咱么都用这个字符串加密通信的内容。

之后双方都是用这个加密字符串对称加密发送的内容。
