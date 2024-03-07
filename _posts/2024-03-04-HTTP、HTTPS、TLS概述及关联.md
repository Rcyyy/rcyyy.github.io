---
title: HTTP、HTTPS、TLS概述及关联
date: 2024-03-04 09:00:00 +0800
categories: [Network Protocols]
tags: [HTTP,TLS]
math: false
mermaid: true
image:
  path: /assets/img/posts/2024-03-04-title.webp
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: versions
---
这篇博客目的是对`HTTP、HTTPS、TLS/SSL`这几个网络协议的概念和版本做一个梳理，包括各个协议之间的关联，比如HTTPS是协议吗？有版本之分吗？HTTP2一定建立在TLS之上吗？HTTP2对TLS版本有要求吗？HTTP3可以不使用TLS吗？HTTP3对TLS版本有要求吗？

## HTTP/HTTPS协议
### HTTP/0.9
HTTP协议起源于1990年的万维网项目(World Wide Web)，起初是希望在互联网基础上建立一个超文本系统，这个项目的实现主要包含了四项内容：
- 定义了超文本文档的格式（HTML）
- 定义了一个简单的协议用于在互联网上交换HTML文档（HyperText Transfer Protocol）
- 一个客户端来请求并展示HTML文档（第一个浏览器WorldWideWeb）
- 一个服务端来提供HTML文档（httpd的早期版本）
  
最初的HTTP没有版本，后来随着演变，为了区分将最初的HTTP协议版本定义为HTTP/0.9，最初版本协议内容极为简洁：
- 请求只有GET方法。
- 请求内容只有URL：`GET /mypage.html`。
- 返回内容只有HTML文档。

最初版本的HTTP没有Header、没有返回码，也就是最初的HTTP协议只能用来获取某个主机上的某个HTML文档。
### HTTP/1.0 
随着HTTP协议的普及，由于最初版本的协议过于简单，功能也很受限，因此功能也很快得到了扩展，并在1996年提出了一份`Informational`的HTTP/1.0的文档[RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945)。HTTP主要拓展如下：
- 协议版本信息需要添加到请求行中。
- 定义请求状态码。
- 定义HTTP Header。
- 可以传输多种数据依据` Content-Type`。
  
### HTTP/1.1
HTTP/1.0之后没多久就在1997年推出了HTTP/1.1版本的正式RFC文档[RFC 2068](https://datatracker.ietf.org/doc/html/rfc2068)，新增功能如下：
- 可以保持长链接，而不是每次请求都要重新链接。
- 增加缓存机制。
- 定义分块(Chunk)传输。
- 增加内容协商，包括语言、类型、编码。
- 必须携带Host Header（意味着一个server可以实现多个虚拟Host）。

### HTTP/2
之前版本的迭代多是对协议的功能进行拓展和完善，HTTP/1.1已经能满足大部分场景，HTTP协议也是在1.1之后经历了15年的稳定，并没有大版本的迭代。而在近10年提出的HTTP/2、HTTP/3新版本也是针对HTTP性能和安全方面进行的优化。

随着互联网的飞速发展，网上的数据呈现海量级，HTTP也仍然是当今互联网的主流协议，对HTTP性能优化对互联网具有重要意义。HTTP/2前身是Google发明的SPDY协议，对协议大小、传输方式等进行了优化，最终被正式定义为HTTP/2 [RFC 7540](https://datatracker.ietf.org/doc/html/rfc7540)。协议主要改进如下：
- 从文本格式改为二进制格式，减少协议所占字节。
- 新增steam的概念，解决HTTP/1.1在HTTP层面上的队头阻塞。
- HTTP头压缩。很多头都是重复的，通过协议减少了HTTP Header的重复传输。
- 通过协议server端可以主动推数据,减少了请求的消耗。

### HTTP/3
最后也是目前最新的HTTP版本在2022年被正式定义，[RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)。最大的改进不是HTTP协议本身，而是在`QUIC`([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000))上，QUIC可以看作四层模型的传输层协议，HTTP/3要求使用QUIC作为传输层协议，HTTP/3所得到的优化基本上也都源于QUIC：
- QUIC使用内核的UDP协议传输数据，而不是TCP。解决了HTTP/2版本下由于TCP重传造成的队头阻塞。
- 优化初始延迟。
- 集成TLS。

## TLS/SSL协议
![](/assets/img/posts/transport-layer-security-secure-socket-layer-ssl-and-ssl-architecture.png)

TLS(Transport Layer Security)、SSL(Secure Sockets Layer)都是一种网络协议，七层OSI模型下属于表示层，用于保证网络传输数据的安全性，安全性主要针对三个问题：
- 窃听风险。第三方攻击者可以随意窃听通信内容，比如获取支付账号密码
- 冒充风险。第三方攻击者可以冒充他人身份与你通信，比如冒充银行网站以窃取银行账号密码。
- 篡改风险。第三方攻击者可以随意修改通信内容，比如在响应上加入钓鱼网址，或者污染原数据。
  
**非对称加密（身份认证、密钥交换）、对称加密（明文加密）、哈希算法（身份认证、防篡改）**是安全协议的基石和保障，最初在1994年`Netscape`公司设计了SSL协议，SSL1.0由于存在缺陷未被公开，一年以后SSL2.0被公开，但2.0同样被发现有安全漏洞随即1996年颁布了SSL3.0，同时3.0协议不向后兼容，这也是目前网络上唯一有可能使用的SSL版本。

SSL3.0协议获得互联网广泛认可和支持，并将其重命名为传输层安全（TLS）协议。TLS协议的第一个版本[RFC 2246](https://datatracker.ietf.org/doc/html/rfc2246)于1999年1月发布，但是该版本实际上就是SSL 3.0协议的适度改进版。虽然TLS协议和SSL协议是同一个协议的迭代升级，但是其重命名后在名称上造成的混淆一直延续到今天，所以一直到今天仍然有人称之为SSL/TLS协议。

2006年4月，IETF发布TLS 1.1版本[RFC4346](https://datatracker.ietf.org/doc/html/rfc4346)，包含一些小的安全改进。2008年8月，TLS 1.2版本[RFC5246](https://datatracker.ietf.org/doc/html/rfc5246)发布，移除了老旧加密套件，增强了协议的安全性，TLS 1.2协议是目前主流的TLS协议版本。

2018年3月，TLS 1.3 [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)协议正式批准问世，成为了新一代TLS协议版本。TLS 1.3版本历经长达10年28次草案修改，是迄今为止改动最大的一次，减少了握手连接的RTT(握手只需1 RTT，某些场景下可为0)，废弃了不安全的算法，极大精简了密码套件。

在安全协议迭代更新的同时，相对安全性不那么强容易被攻破的旧协议也逐渐被废弃。截至2015年 [RFC 7568](https://datatracker.ietf.org/doc/html/rfc7568) SSL 2.0 3.0都正式被IETF废弃。

TLS 1.0 1.1也在2021年被IETF提出废弃 [RFC 8996](https://datatracker.ietf.org/doc/html/rfc8996)。

**因此除非受限于某些历史遗留，版本限制，不论是出于安全性还是速度都应该优先选择TLS 1.3，然后是TLS 1.2，其他的版本已被标准废弃。**


## HTTP与TLS之间的约束
![](/assets/img/posts/http-https.webp)

HTTPS是HTTP协议的拓展，是安全版本的HTTP，就是需要在TLS/SSL连接之上传输数据，保证了数据的安全性。在当代，加密已经越来越重要，在许多场景下必须使用HTTPS，比如浏览器会默认使用HTTPS，对HTTP协议提示不安全。

此外，HTTP/3版本协议天生需要加密，因此HTTP/3无法使用不加密的HTTP协议。虽然HTTP/2在标准上仍然支持HTTP与HTTPS，但当前主流浏览器已经事实上对HTTP/2使用来说，也必须加密。

至于使用什么版本，理论上HTTP与TLS之间是独立的，可以使用不同的版本组合，但正如之前TLS所说，新版本的TLS更安全也更快，因此新版本的HTTP协议如HTTP/2在标准中要求必须使用TLS 1.2以上版本，HTTP/3依赖的QUIC内置使用的为最新的TLS 1.3版本。
