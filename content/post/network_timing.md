---
title: "网络资源请求流程及优化"
date: 2021-01-29T21:18:29+08:00
draft: false
ShowToc: true
TocOpen: true
---


浏览器在获取网络资源的时候，会经过一系列的网络连接建立、请求数据、接收数据等过程。整个过程中的每个阶段都会存在时间消耗。有的时间消耗是不可避免的，但有的消耗是可以被进一步优化的。

本文想传递的内容就是：网络请求的流程是怎样的，以及如何通过 devtool 查看时间消耗，和如何进行优化。


# 一个完整的资源请求流程
我们先看一下完整的资源请求的流程是如何进行的

<style>
table tr > td:first-child {
  width: 10em;
}
</style>

步骤|描述
-- | :--
1️⃣ 准备请求|构建请求行信息
2️⃣ 查找缓存|这个比较好理解，要请求之前先查看要请求的资源是否已经缓存过了，并且还处在有效期内。如果存在有效的缓存，那就不需要再去发送网络请求了，直接就可以返回给浏览器处理。
3️⃣ 准备IP|一般来说，我们给到的资源都是一个URL，其域名需要先转换为IP地址才能拿来使用。将域名转换为IP地址的过程，首先是会看DNS缓存是否存在，存在就返回IP，不存在就去 DNS 服务器查这个域名对应的IP。IP准备好之后就可以进行下一步了。
4️⃣ TCP排队|TCP连接不是随意建立的，对于 Chrome 浏览器来说，<u>**针对每一个域名，同时最多建立 6 个TCP连接**。</u> 所以如果当前已经存在了6个对同一个域名的TCP连接，那当前的请求只能先到TCP请求队列里面去排队，等待有连接被释放了，再从TCP请求队列里拿出来建立新的连接。
5️⃣ 建立TCP连接|三次握手🤝，两台计算机之间建立连接。TCP是可靠的面向连接的传输层的协议，建立了TCP连接，上层数据就能完整的送达对方计算机了。
6️⃣ 发送HTTP请求|给服务端发送基于HTTP协议的请求，然后会有一小段时间等待服务端的响应
7️⃣ 接收HTTP响应|服务端处理完请求，给出响应数据（🔆 分两次, 先给响应头的数据，再给响应体的数据）
8️⃣ 断开TCP连接|数据交互结束，四次挥手👋断开TCP连接


# 用 Devtool 查看时间消耗
![timing](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/dHO1r5_11_10-11-10.png)

操作步骤:
1. 打开Chrome开发者工具
2. 在Network面板下选择某个资源请求
3. 切换到 Timing 子面板

# 关键时间消耗和优化策略
Timing面板中展示了整个资源请求所消耗的时间, 其中的时间消耗和我们上面所提及的一些资源请求的流程是对应的。关于Timing面板下的数据，完整的解释可以看下图（取自Chrome Devtools官方文档 [https://developers.google.com/web/tools/chrome-devtools/network/reference](https://developers.google.com/web/tools/chrome-devtools/network/reference)）
![Timing消耗](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/3UKAFq_11_11-10-23.png)

其中有三个阶段我们需要特别关注一下：
1. Queueing : 排队时间，也即 TCP 建立连接的请求发起前的排队时间
2. Waiting(TTFB) : Time To First Byte ,发送请求到接收到第一个字节的时间
3. Content Download : 资源下载时间，服务端从返回第一个字节到返回完所有数据所花费的时间

## Queueing
排队阶段，这个阶段是指请求已经做好准备了，但是因为一些原因需要先排队等待。排队的原因主要有下面三种
1. 有其它更高优先级的资源要请求（比如html、css 就比 image资源优先级更高）
2. 同域名下已经有了6个TCP连接请求了（最多只让有6个）
3. 分配磁盘空间用来存储请求下来的数据（这个阶段一般很快）

优化策略：
- 移除或者延缓不紧急重要的资源，让紧急的资源更先下载（比如将不重要的脚本放最底部）
- 使用 **域名分片** 从更多域名下下载资源（每个域名最多6个TCP，比如多了两个CDN域名，那就可以同时发起18个连接了）
- 升级为HTTP/2，可以链路复用，一个连接通道同时发起多个请求（升级之后就别再同时使用域名分片了）
  

## Waiting(TTFB)
TTFB 是指从发送请求到收到服务端返回的第一个字节的一个时间，导致这个时间长的因素，可能为：
1. 网络本身慢
2. 服务端处理请求太耗时

优化策略：
1. 部署CDN，让内容分发到距离用户最近的主机。
2. 服务端优化数据库查询操作，增加缓存机制等


## Content Download
从TTFB得到第一个从服务端返回的字节之后，就开始了整个返回内容的下载了，Content Download表示的就是整个资源下载下来所需要的时间，这个时间显然是受到资源自身的大小的限制。所以优化策略也比较明显。

优化策略：
1. 移除注释、压缩代码、移除无效代码内容
2. 启用资源压缩 (gzip)


# 参考
- Network Reference. Chrome DevTools: [https://developers.google.com/web/tools/chrome-devtools/network/reference](https://developers.google.com/web/tools/chrome-devtools/network/reference)
- Network Issues Guide. Chrome DevTools: [https://developers.google.com/web/tools/chrome-devtools/network/issues](https://developers.google.com/web/tools/chrome-devtools/network/issues)
- 极客时间《浏览器工作原理与实践》—— HTTP请求流程

---
