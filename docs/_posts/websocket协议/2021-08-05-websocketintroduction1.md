---
layout: post
title: WebSocket协议浅析（1）- 概述
subheading: WebSocket协议简介
author: Colin Wei
categories: WebSocket
tags: [WebSocket, 网络协议]
---

前段时间，因公司项目需要实现单聊场景下的IM工具，笔者对目前业界比较通用的WebSocket协议进行了调研和学习。
WebSocket协议的出现，使浏览器和服务器具备了实时双向通信的能力。本文主要对WebSocket协议的使用场景、原理进行简要介绍。

## 1 什么是WebSocket?

借用维基百科的定义:
> WebSocket是一种网络传输协议，可在单个TCP连接上进行全双工通信，位于OSI模型的应用层。WebSocket协议在2011年由IETF标准化为RFC 6455，后由RFC 7936补充规范。<br/>
> WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。

上面的定义，知识点不太明显，我们简单总结一下WebSocket协议的要点：
1. 一种网络传输协议
2. 建立在TCP协议之上
3. 支持全双工通信
4. 兼容HTTP协议
5. 客户端和服务器端连接建立只需要一次握手

## 2 为什么需要WebSocket?

在WebSocket协议出现之前，在浏览器和服务器之前通信的时候，我们使用的一般都是HTTP协议。 但HTTP协议是单向通信，都是由客户端主动建立连接并发起请求，服务端只进行请求的响应，而不能主动发起请求。

这就好比我去一家自助餐厅点餐，我进入（客户端）进入餐馆（建立连接），点餐（发起请求），餐馆告诉你下单了（服务器响应），这样一次请求就结束了。
然后我就走了（连接断开）（虽然正常人都不会这么干）。人都走了，你说餐厅准备好餐品后，怎么给我送呢？那只能是我隔段时间再来看看订单好了没。

这也是使用http实现双向通信的实现方式——轮询：客户端定期请求服务器的数据。 

最简单的实现就是短轮询，即每次发送轮询请求都会重新建立连接，请求结束后，断开连接。

![短轮询流程](/assets/images/posts/WebSocket简介-HTTP短轮询.png)

这种实现方式的问题在于：
1. 带宽资源占用和浪费。在服务端任务完成期间，客户端会进行多次请求，其中大部分的请求并没有传输有用的数据，但对于带宽的占用是很明显的。
2. 服务器资源占用和浪费。服务器接收和处理请求也占用了服务器的资源。
3. 效率低下。每次都需要重新建立及销毁连接。

### 服务器推送技术

为了优化上述问题，人们提出了[服务器推送技术](https://zh.wikipedia.org/wiki/%E6%8E%A8%E9%80%81%E6%8A%80%E6%9C%AF)。

使用HTTP协议的服务器推送技术的实现，主要思路是HTTP长轮询和HTTP流机制。

#### HTTP长轮询

HTTP长轮询的思路是，当客户端请求后，服务端不立即返回，而是保持该连接的状态。在发生某个事件、达到某个状态或者超时的时候，返回给客户端。客户端重新建立连接发起请求，并以此循环直到任务或会话完成。

相比于短轮询： 长轮询在通信的时候，没有发送空信息，减少了对带宽的占用。并且，服务端在处理结束后，会立即返回给客户端，这样也减小了客户端发送接受消息的时延。

![短轮询流程](/assets/images/posts/WebSocket简介-http长轮询.png)

但长轮询的方案也有很多问题，例如：
1. 请求包头开销大。长轮询中的每个请求和响应都包含了完整的http请求头，对于请求信息内容小且频繁的场景，请求包头的开销可能远大于实际报文的开销，这对于带宽也是一种浪费。
2. 单个消息延迟增加。对于服务端来说，在它发送消息之后，如果想再次发送消息，必须断开连接，并等待客户端重新建立长轮询连接才可以。
3. 资源浪费。对于每个长轮询请求，它大部分的时间都是被服务端挂起的，并没有传递信息。但在此期间，该连接无法被其他客户端占用。但服务端资源有限，如果请求量激增，对于服务的影响无法预估。

#### HTTP流机制

HTTP流机制的思路是，在一个连接建立后，一直保持该连接为打开的状态，不会断开。当服务端触发某个时间或状态的时候，发送响应给客户端，但该连接仍然保持。只有当任务完成或会话结束，才会去断开连接。

![短轮询流程](/assets/images/posts/WebSocket简介-http流机制.png)

与长轮询相比，流机制减少了连接建立和销毁的次数，使占用的带宽和整体的消息延迟进一步减小。

流机制也引入了新的问题，例如：
1. 代理的问题。http协议允许使用代理进行请求的转发和响应，但代理的机制一般是在客户端请求的时候才能知道把服务端的响应发给哪个客户端。流机制与这样的代理是不兼容的，因此，需要额外的机制来处理代理的问题。
2. 消息延迟的问题。虽然理论上流机制可以只建立一次连接，但实际上，由于客户端或服务端的连接数或内存等的限制，往往对于建立的连接会进行断开并重新连接，来降低资源的占用。这样的话，该机制和长轮询基本就是一样了。
3. 额外的机制和协议。因为需要一直维护连接的状态，并实现由服务器主动推送给客户端，因此需要在原有的HTTP协议上增加额外的机制和协议来实现该功能。

## 3 WebSocket的原理

无论是轮询还是流机制，都是使用的http协议，每次请求都包含了完整的http请求头，并且，需要多次请求来获取完整的响应，开销大。

问题来了，难道不能使用单个TCP连接解决双向通信的问题吗？答案就是WebSocket协议。

下面，我们看下WebSocket协议的基本原理。
WebSocket协议主要包含两部分，建立连接和数据传输。

### 3.1 建立连接
WebSocket协议复用了http协议的握手通道来建立连接。

首先，客户端发起http升级请求，请求示例如下：
```http request
GET /chat HTTP/1.1
Host: server.example.com
Origin: http://example.com
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```
`Connection: Upgrade`：表示要升级协议<br/>
`Upgrade: websocket`：表示要升级成websocket协议<br/>
`Sec-WebSocket-Protocol`：表示客户端可以使用的websocket协议。服务端需要从中选择一个协议，并在握手时返回给客户端。<br/>
`Sec-WebSocket-Version`：表示客户端使用的websocket的版本。如果服务端不支持该版本，需要返回一个Sec-WebSocket-Version header，里面包含服务端支持的版本号。根据RFC6455，该值不能小于13，低版本的已经被废废弃了。<br/>
`Sec-WebSocket-Key`：该字段值是由客户端生成的base64编码的字符串，客户端通过对字段和服务端响应头的Sec-WebSocket-Accept字段的值进行校验，保护连接不会被窃取。

客户端响应示例如下：
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Protocol: chat
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
服务端的响应头比客户端的简单很多。
`101 Switching Protocols`: 101状态码表示websocket协议握手完成。除101之外的状态码都表示websocket协议握手未完成，此时协议仍为http协议。<br/>
`Connection`和`Upgrade`：表示将协议升级为了websocket。<br/>
`Sec-WebSocket-Protocol`：表示服务端选择的websocket协议。<br/>
`Sec-WebSocket-Accept`：由Sec-WebSocket-Key计算得到，用于向客户端表明身份：我接收到了你的请求。

服务端生成Sec-WebSocket-Accept的算法如下：
1. 拼接Sec-WebSocket-Key字段的值和"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"（Globally Unique Identifier(GUID，[RFC4122](https://www.rfc-editor.org/rfc/rfc4122))），得到字符串A;
2. 对字符串A进行SHA1 hash，得到字符串B；
3. 对字符串B进行base64编码得到Sec-WebSocket-Accept的值。




## 参考文献

- 维基百科：https://zh.wikipedia.org/wiki/WebSocket
- https://www.cnblogs.com/chyingp/p/websocket-deep-in.html
- websocket RFC: https://www.rfc-editor.org/rfc/rfc6455.html#page-5
- http rfc: https://datatracker.ietf.org/doc/html/rfc2616#page-144
- http双向通信实践 rfc：https://datatracker.ietf.org/doc/html/rfc6202

未完待续。。。
