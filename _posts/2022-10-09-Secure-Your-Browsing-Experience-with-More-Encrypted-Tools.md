---
title: TLS/ESNI/ECH/DoT/DoH/JA3 - 谈谈 HTTPS 网络下浏览体验安全性的最后缝隙
layout: post
thread: 281
date: 2022-10-09
author: Joe Jiang
categories: Document
tags: [TLS, ESNI, ECH, DoT, DoH, JA3, HTTPS, 网络安全, 加密, DNS]
excerpt: 有了 HTTPS 我们的网络访问就完美无缺了吗？TLS 在握手 Client Hello 阶段的数据包，包含客户端想访问的地址等信息，而这些信息是明文传输的，这使得第三方可以通过监听等手段知道你想访问哪些网址。本文会按照加密方式的加工流程倒序介绍，先介绍关于 TLS 握手阶段的 SNI 相关信息，再谈谈 DNS 安全层，最后会介绍下 TLS 握手指纹……
header:
  image: ../assets/in-post/2022-10-09-Secure-Your-Browsing-Experience-with-More-Encrypted-Tools-0.png
  caption: "©️hijiangtao"
---

![](/assets/in-post/2022-10-09-Secure-Your-Browsing-Experience-with-More-Encrypted-Tools-Teaser.png )



虽然 TLS 能够加密整个通信过程，但是在协商的过程中依旧有很多隐私敏感的参数不得不以明文方式传输，其中最为重要且棘手的就是将要访问的域名，即 SNI。

加密 SNI (Encrypted Server Name Indication, ESNI) 通过加密客户端问候的 SNI 部分来添加到 SNI 扩展，这可以防止任何在客户端和服务器之间窥探的人看到客户端正在请求哪个证书，从而进一步保护客户端。Cloudflare 和 Mozilla Firefox 于 2018 年推出了对 ESNI 的支持。

ESNI 可以确保正在侦听的第三方无法监视 TLS 握手流程并以此确定用户正在访问哪些网站。

### ESNI 是如何工作的

ESNI 通过公钥加密客户端问候消息的 SNI 部分（仅此部分），来保护 SNI 的私密性。加密仅在通信双方（在此情况下为客户端和服务器）都有用于加密和解密信息的密钥时才起作用，就像两个人只有在都有储物柜密钥时才能使用同一储物柜一样。

由于客户端问候消息是在客户端和服务器协商 TLS 加密密钥之前发送的，因此 ESNI 加密密钥必须以其他方式进行传输。

Web 服务器可以在其 DNS 记录中添加一个公钥，这样，当客户端查找正确的服务器地址时，同时能找到该服务器的公钥。这有点像将房门钥匙放在屋外的密码箱中，以便访客可以安全地进入房屋。然后，客户端即可使用公钥来加密 SNI 记录，以便只有特定的服务器才能解密它。

![](/assets/in-post/2022-10-09-Secure-Your-Browsing-Experience-with-More-Encrypted-Tools-1.png )

## Encrypted Client Hello (ECH)

### 使用 ESNI 仍然存在的问题

ESNI 存在问题。首先是密钥的分发，Cloudflare 在部署时每个小时都会轮换密钥，这样可以降低密钥泄露带来的损失，但是 DNS 有缓存的机制，客户端很可能获得的是过时的密钥，此时客户端就无法用 ESNI 继续进行连接。其次是对网站的 DNS 请求可能返回有几个 IP 地址，每个地址分别代表了不同的 CDN 服务器，然而 ESNI 的 TXT 记录只有一个，可能会将该密钥发送给了错误的服务器导致握手失败。

自从 ESNI 规范草案在 IETF 上发布以来，分析表明仅仅加密 SNI 扩展提供的保护是不完整的。为了解决 ESNI 的问题，最近发布的规范不再只加密 SNI 扩展，而是对整个 Client Hello 信息进行加密，因此标准名称从 ESNI 也变成了 ECH。

### 什么是 ECH

**ECH（Encrypted Client Hello，也称加密客户端问候）**出现的目标就是就是为了克服 ESNI 的缺陷，同时也正如其名，ECH 有着更大的野心，不单单加密 SNI，而是要加密整个 Client Hello。

### ECH 的工作原理

ECH 同样采用 DoH （后文会进行详细介绍，此处可暂时理解成 DNS 安全层的一种实现）进行密钥的分发，但是在分发过程上进行了改进。如果解密失败，ESNI 服务器会中止连接，而 ECH 服务器会提供给客户端一个公钥供客户端重试连接，以期可以完成握手。

**ECH 协议实际上涉及两个 ClientHello 消息：ClientHelloOuter，它像往常一样以明文形式发送；ClientHelloInner，它被加密并作为 ClientHelloOuter 的扩展发送。**服务器仅使用其中一个 ClientHello 完成握手：如果解密成功，则继续使用 ClientHelloInner；否则，它只使用 ClientHelloOuter。

ClientHelloInner 由客户端想要用于连接的握手参数组成。这包括它想要访问的源服务器的 SNI、ALPN 列表等。ClientHelloOuter 虽然也是一个成熟的 ClientHello 消息，但不用于预期的连接。相反，握手是由 ECH 服务提供者完成，在无法完成解密的情况下，它会向客户端发出信号，表明由于解密失败而无法到达其预期的目的地。此时，服务提供商还会发送正确的 ECH 公钥，客户端可以使用该公钥重试握手，从而“更正”客户端的配置。

![](/assets/in-post/2022-10-09-Secure-Your-Browsing-Experience-with-More-Encrypted-Tools-2.png )

## DNS 安全层

### 为什么 DNS 需要额外的安全层

DNS 是互联网的电话簿；DNS 解析器将人类可读的域名转换为机器可读的 IP 地址。默认情况下，DNS 查询和响应以明文形式（通过 UDP）发送，这意味着它们可以被网络、ISP 或任何能够监视传输的人读取。即使网站使用 HTTPS，也会显示导航到该网站所需的 DNS 查询。 

这种隐私上的欠缺对安全有着巨大影响，如果 DNS 查询不是私密的，则攻击者可以轻松跟踪用户的Web 浏览行为。

### 基于 TLS 的 DNS 标准（DoT）

**基于 TLS 的 DNS（DNS over TLS，简称 DoT）**是加密 DNS 查询以确保其安全和私密的一项标准。DoT 使用安全协议 TLS，这与 HTTPS 网站用来加密和认证通信的协议相同。DoT 在用于 DNS 查询的用户数据报协议（TCP）的基础上添加了 TLS 加密，这可以确保 DNS 请求和响应不会被[在途攻击](https://www.cloudflare.com/learning/security/threats/on-path-attack/)篡改或伪造。

与 DoT 采用在 TCP 协议上处理稍有不同的是，还存在另一个在 UDP 协议上加密的标准，**DNS over DTLS，**即基于 UDP 的 DNS TLS 加密协议标准。

### 基于 HTTPS 的 DNS 标准（DoH）

基于 HTTPS 的 DNS 或 DoH 是 DoT 的替代标准。使用 DoH 时，DNS 查询和响应会得到加密，但它们是通过 HTTP 或 HTTP/2 协议发送，而不是直接通过 UDP 发送。与 DoT 一样，DoH 也能确保攻击者无法伪造或篡改 DNS 流量。从网络管理员角度来看，DoH 流量表现为与其他 HTTPS 流量一样，如普通用户与网站和 Web 应用进行的交互。

### DoT 与 DoH 的区别

这两项标准都是单独开发的，并且各有各的 RFC 文档，但 DoT 和 DoH 之间最重要的区别是它们使用的端口。DoT 仅使用端口 853，DoH 则使用端口 443，后者也是所有其他 HTTPS 流量使用的端口。

由于 DoT 具有专用端口，因此即使请求和响应本身都已加密，具有网络可见性的任何人都发现来回的 DoT 流量。DoH 则相反，DNS 查询和响应伪装在其他 HTTPS 流量中，因为它们都是从同一端口进出的。

### 其他加密 DNS 请求的方式

除了上文提到的几种标准外，想要达到 DNS 请求加密的目的，我们仍然有多种其他实现方式：

1. VPN：通过 VPN，你不仅可以加密 DNS 请求，甚至可以将所有网络流量进行加密传输；
2. DNSCurve：DNSCurve 使用 Curve25519 椭圆曲线加密算法创建 Salsa20 使用的密钥，配以 MAC 函数 Poly1305，用来加密和验证解析器与身份验证服务器之间的 DNS 网络数据包。它的提出者希望用 DNSCurve 替代 DNSSEC；
3. DNSCrypt：DNSCrypt 是由 Frank Denis 及付业成（Yecheng Fu）主导设计的网络协议，用于用户计算机与递归域名服务器之间的域名系统（DNS）通信的身份验证。DNSCrypt 将未修改的 DNS 查询与响应以密码学结构打包来检测是否被伪造；
4. IPsec：IPsec（Internet Protocol Security）是为 IP 网络提供安全性的协议和服务的集合，它是VPN（Virtual Private Network，虚拟专用网）中常用的一种技术。 由于 IP 报文本身没有集成任何安全特性，IP 数据包在公用网络如 Internet 中传输可能会面临被伪造、窃取或篡改的风险。通信双方通过 IPsec 建立一条 IPsec 隧道，IP数据包通过 IPsec 隧道进行加密传输，有效保证了数据在不安全的网络环境如 Internet 中传输的安全性；

## 概念释义与其他

1. **服务器名称**：服务器名称就是计算机的名称。对于Web服务器，此名称通常对最终用户不可见，除非该服务器仅托管一个域并且该服务器名称与该域名等同。
2. **主机名**：主机名是连接到网络的设备的名称。在互联网背景中，域名或网站名称是一种主机名。两者都与与域名关联的 IP 地址分开。
3. **虚拟主机名**：虚拟主机名是没有自己的IP地址的主机名，它与其他主机名一起托管在服务器上。它是"虚拟的" ，因为它没有专用的物理服务器，就像虚拟现实仅以数字形式存在，而不是在物理世界中一样。
4. **AdGuard 进展**：AdGuard 是一款跨平台内容过滤的广告拦截软件。其对 DoH、ESNI 等的支持仍在筹划中 [https://github.com/AdguardTeam/CoreLibs/issues/722](https://github.com/AdguardTeam/CoreLibs/issues/722)
5. **浏览器对 ESNI 的支持情况**：当前只有 Firefox 支持 ESNI。
6. **浏览器对 DoH/DoT 的支持情况**：所有主流浏览器均支持。

## 参考

1. [https://www.infoblox.com/dns-security-resource-center/dns-security-faq/are-dns-requests-encrypted/](https://www.infoblox.com/dns-security-resource-center/dns-security-faq/are-dns-requests-encrypted/)
2. [https://zhufan.net/2022/06/18/tls%E6%8F%A1%E6%89%8B%E6%8C%87%E7%BA%B9%E6%A3%80%E6%B5%8B%E6%81%B6%E6%84%8F%E8%BD%AF%E4%BB%B6%E6%B5%81%E9%87%8F/](https://zhufan.net/2022/06/18/tls%E6%8F%A1%E6%89%8B%E6%8C%87%E7%BA%B9%E6%A3%80%E6%B5%8B%E6%81%B6%E6%84%8F%E8%BD%AF%E4%BB%B6%E6%B5%81%E9%87%8F/)
3. [https://blog.mozilla.org/security/2018/10/18/encrypted-sni-comes-to-firefox-nightly/](https://blog.mozilla.org/security/2018/10/18/encrypted-sni-comes-to-firefox-nightly/) 
4. [https://www.cloudflare.com/zh-cn/learning/dns/dns-over-tls/](https://www.cloudflare.com/zh-cn/learning/dns/dns-over-tls/)
5. [https://zinglix.xyz/2021/04/03/esni-ech/](https://zinglix.xyz/2021/04/03/esni-ech/)
6. [https://blog.cloudflare.com/encrypted-client-hello/](https://blog.cloudflare.com/encrypted-client-hello/)
7. [https://zinglix.xyz/2019/05/07/tls-handshake/](https://zinglix.xyz/2019/05/07/tls-handshake/)
8. [https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/)