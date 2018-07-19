---
layout: post
title:  "HTTP(s) proxy 的实现"
date:   2017-11-19 20:46:43
categories: 造轮子
---

## 代理服务器程序(proxy)
最近项目调整到了web网关业务，之前对网关的daemon 功能有个大致的了解。
网关的流量模型本质是proxy, 在转流量的同时做virus-scanning， url filtering, traffic control 以及一些零零散散的统计报表，blocking page 的显示。

总体上没什么难理解的， 无非是non-block + epoll 的实现， 可以参考 [tigerso](https://github.com/YingshuLu/tigerso)

但是正如之前文章(HTTPS Decryption)讲的， 现在的web整体趋势是往https转，
所以作为web安全网关应该尽可能去解密HTTPS traffic, 才能充分调用网关的功能。

[Https Decryption](https://yingshulu.github.io/https/decryption/2017/09/11/HTTPS-Decryption.html) 介绍了解密（MITM）的原理是重签证书与客户端信任。

真正实现的过程中有两种方法：

1. 拿到client connect 请求现行与 server 进行SSL握手, 然后重签证书， 与client 进行SSL 握手
2. 拿到client connect 请求， 接收client 的 ssl-client-hello, 此时不进行ssl accept 而是将 ssl-client-hello 的所有信息(ssl version, cipher, alpn) 复制到 与server的ssl-client-hello中， 与 server 进行SSL握手, 然后重签证书， 与client 继续进行SSL 握手

方案1 最简单直接， 缺点是proxy不能模拟client的ssl 行为，会造成特殊站点（可能是用户内部自建website）的握手失败。

网关程序一般需要精确的模拟client行为（方案2），将ssl的设置留给使用者，为不是写死在程序里。

但是方案2需要hook "SSL_accept", 改变api的行为，能够两次重入函数：
第一次 只拿client hello, 不做任何check， 不发ssl response
第二次 继续回到ssl 握手的流程

## proxy 实现 (tigerso)
我想每个有理想的 programmer 内心都是有些排除他人程序的配置，希望能够完全掌控软件的所有细节， 有些程序洁癖。
我也一样，看了项目文档， 顿感手痒，花来三个星期自己造的个 forward http(s) proxy: [tigerso](https://github.com/YingshuLu/tigerso)

为了简单起见，tigerso 只实现了方案1的https MITM，做了简单的用户控制，websites控制。
目前已经将tigerso部署到公司的linux跳机上，成功替换掉squid, 完成了从公司翻墙代理的任务。因为是自己翻墙使用，现在已经关掉了Https Decryption功能。

在一个月的试用里面， 实测方案1对于一些不知名的https 站点确实不能完美解密。 对google这样的大站没什么影响。 但是youtube还是bypass吧，毕竟想要好的体验，视频流还是不要倒手一遍了。 另外mobile app / 网页app 的https基本没办法解开， 毕竟让我写app也会内嵌信任证书的，这写流量可以通过requst的agent值bypass。

epoll + non-block Reactor模型确实能够轻松的撑住10000级别的并发访问量， 性能测试中我用 4台linux肉鸡多线程不停打老虎(tigerso), 这只老虎也没死，最后是一般是卡在带宽上了， tcp 握手慢， tigerso就踢掉这些连接了。

值得吐槽的是这种异步回调方式，真的很考验人的心智， callback会不断的传播，每一步的状态都要保留并在callback中恢复。 如果只是http server其实还好， 但是https proxy就需要triple这些状态，后续再接上公司认证服务的话，又会是一大堆状态和callback.

如果后面还有时间和兴趣，还想集成些开源的virus scanning， url classification 库，看看效果。

***二维码在线要饭***  
![打赏](https://github.com/YingshuLu/yingshulu.github.io/blob/master/images/reward.png?raw=true)
