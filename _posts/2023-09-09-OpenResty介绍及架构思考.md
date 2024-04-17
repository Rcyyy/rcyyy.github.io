---
title: OpenResty介绍及架构思考
date: 2023-09-09 09:00:00 +0800
categories: [OpenResty]
tags: [OpenResty,Coroutine]
math: false
mermaid: true
image:
  path: /assets/img/posts/2023-09-09.jpeg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: PP美照
---

> 开启第一篇关于`OpenResty`的记录，关于Openresty的介绍及入门见这里：[OpenRest最佳实践](https://moonbingbing.gitbooks.io/openresty-best-practices/content/)。

## 前置知识
### 1.事件驱动架构(Event-driven architecture)
事件驱动架构是一种优秀的软件(系统)设计模式，以事件为核心驱动整个系统的运转。事件驱动架构通常包括**事件分发者、生产者、消费者**。事件消费者在事件分发者注册自己感兴趣的事件，生产者产生事件给分发者，分发者收到事件后分发给已注册的消费者进行消费。
![Event-driven architecture](/assets/img/posts/event_driven.svg)

图中的Emitter、Event-Loop、Handler对应生产者、分发者、消费者。分发者**循环**的收集、检测生产者发来的事件，当有事件到来时，根据事件类型并分发(**callback**)对该事件感兴趣的消费者(**handler**)。

而在定义也说了，事件是核心，是该系统需要处理的问题。拿一个Web服务来说，作为网络服务器需要关注的是来自用户的请求，用户的请求通过TCP/UPD网络协议发送到服务端，服务端进程检测到IO可读事件，便分发给HTTP协议处理模块，接着再检测下一个用户请求。

**事件驱动架构的优点有分工明确（解耦，各个模块之前通过事件交流，模块之间解耦，模块内专注自己的处理逻辑，架构整体清晰整洁）、高效（使用异步回调的处理方式能够基本无阻塞的运行，充分利用CPU）。**

### 2.同步(synchronous)与异步(asynchronous)
> Synchronous uses the Greek syn-, meaning “together.” The middle part of the word comes from the Greek chron(os), meaning “time.” The ending -ous is used to form adjectives. Based on its word parts, synchronous basically means “happening at the same time.” Asynchronous uses the prefix a-, meaning “not,” making it the opposite: “not happening at the same time.”

同步用来描述多件事发生在同一时间、同一背景，强调的是一个整体、同步流程，异步则是反意，指可以发生在不同时间、不同背景下，强调的是流程间独立，看下面一个请求的例子：
- 同步处理：发起请求，等待结果，得到结果并处理，继续运行；发起请求与得到结果并处理发生在同一时期、同一背景（client中间什么都没干，就等上一步的返回结果）。
- 异步处理：发起请求，不等待结果，继续运行，当结果返回时再处理。发起请求与得到结果并处理发生在不同时间。（client发起请求后就继续忙别的工作了，等待结果有了再来处理（回调））。
![sync_vs_async](/assets/img/posts/sync_vs_async.png)
优缺点：
- 同步处理：逻辑清晰，一次处理一条请求，但当前流程需要等待，不能同时处理多个请求。
- 异步处理：不等待结果，而是等结果就绪后回调，可以同时处理多条请求，一条请求可能多次回调，回调及上下文状态导致处理逻辑复杂。

**异步处理往往是与事件驱动架构是相辅相成的**，事件驱动往往需要异步处理的方式，当消费者消费事件遇到需要等待的资源时，就注册相应的事件到分发者并等待回调，同时这种无阻塞的编程方式也是事件驱动架构高性能的原因。


### 3.Nginx与传统Web服务器
Nginx与传统Web服务器是当前流行的Web服务器的两类代表。从架构上区分，可以分为**完全事件驱动架构**与**不完全事件驱动架构**。从编程方式与执行流程上，可以分为**同步处理**与**异步回调处理**。

Nginx采用完全的事件驱动架构来处理业务，处理过程全程无阻塞，处理过程遇到资源需要等待的情况，**消费模块会注册相应的事件及回调函数，将运行权交还给事件驱动模块，等待再次被分发激活**。

传统的Web服务器(如Apache)采用的所谓事件驱动往往局限在TCP连接建立、关闭事件上，这种传统Web服务器往往把一个进程或线程作为事件消费者，一旦建立连接后整个处理流程都交给一个线程采用同步处理的方式直到结束。
![Nginx vs Other](/assets/img/posts/nginx.png)

从上面的内容可以看出传统Web服务器与Nginx间的重要差别，前者一个请求对应一个线程，请求的并发数对应于线程的数量，虽然在操作系统的支持下，阻塞的线程会被挂起，等待阶段不会浪费CPU资源，但是当线程数增长到一定数量后会导致两个副作用无法被忽视（瓶颈）：
 - 第一：虽然线程是轻量级进程，但还是会占用一定量的内存与进程号等资源。
 - 第二：系统上可运行的线程数过多时，由于内核公平调度的进程调度策略，频繁的上下文切换会大幅度降低系统的整体性能，这也是主要原因。
  
而Nginx使用单进程单线程就可以处理数十万、百万的并发请求（理论上仅受限于内存），而一个请求在Nginx中占用的内存是极少的（一个不活跃的连接只有几k大小）。但是由于Nginx异步非阻塞的处理方式，这加大了事件消费程序的开发者的编程难度，因此，这也导致了Nginx的模块开发相对于Apache来说复杂不少。这也是为什么Nginx经常被用作反向代理、负载均衡器，而请求业务逻辑由上游服务器真正处理（比如传统http服务器）。


## Openresty
### 协程
协程的概念最早在1958年就被提出，但一直没有被大家重视。随着某些技术架构高并发的需求，需要异步回调的编程方式，但异步回调的编码方式太过复杂，同步多线程的方式也太昂贵，协程又重新回到了人们的视角。协程又被称为用户级线程，协程对内核来说完全是透明的，是完完全全用户空间的概念，协程像线程一样具有独立的执行流，协程可以在用户空间主动让出执行权，并等待被唤醒继续执行。

在Lua语言中就实现了协程，Lua语言是一门小巧、高性能的基于虚拟机解释执行的脚本语言，天生就能方便的嵌入C语言，后来又有了实时编译(jit)版本的实现，看下面Lua协程这个例子：
```lua
local function dosomething(num)
    print("coroutine" .. num .. " start!")

    print("coroutine" .. num .. " stop!")
    coroutine.yield()
    print("coroutine" .. num .. " finish!")
end
local coroutine1 = coroutine.create(dosomething)
local coroutine2 = coroutine.create(dosomething)

coroutine.resume(coroutine1, 1)
coroutine.resume(coroutine2, 2)
coroutine.resume(coroutine1)
coroutine.resume(coroutine2)
```
运行结果：
```
coroutine1 start!
coroutine1 stop!
coroutine2 start!
coroutine2 stop!
coroutine1 finish!
coroutine2 finish!
```
从用户空间来看两个执行流交替执行，但在内核来看只有一个线程顺序的执行所有代码。协程具有主动让出执行权、并恢复到中断点的功能，如果协程没有让出，上面的协程其实跟一个普通的用户函数没什么区别。

### OpenResty
那么协程有什么用呢？**协程最大的作用就是搭配异步回调来使用，以同步编程的方式编码，底层仍然是非阻塞的异步回调。**将请求的一段执行流作为一个协程，在需要请求其他资源的时候注册回调并把当前协程挂起让出执行权，当资源就位，通过回调函数唤醒协程继续协程的执行流程。协程与异步回调的事件驱动架构相互配置，就兼具了可编程性以及高性能。

OpenResty就是在Nginx的基础上，进行二次开发，嵌入了Lua语言，提供接口可以让用户编写Lua代码处理用户请求，**使得OpenResty不仅具有Nginx架构的高性能，又兼具了动态性、可编程性**，可以通过Lua代码使用OpenResty封装好的接口(cosocket)进行同步编程，犹如老虎插上了翅膀。
![](/assets/img/posts/openresty.png)

OpenResty在每个Worker中创建一个Lua虚拟机，对于每一个请求，当执行到一段Lua代码时(`set/access/rewrite/access_by_lua_*`)，便创建一个协程执行这段Lua代码。

计算机世界中一个重要的概念是抽象，它能够屏蔽底层细节，对上提供一个清晰、统一的视角。如果我们把Nginx抽象为OpenResty的底层"操作系统"、"调度器"，再来看OpenResty的架构就可以是这个样子：

![](/assets/img/posts/openresty_vs_other.png)

OpenResty上运行了一个个的协程，Nginx就像左边的操作系统来调度进程一样，Nginx的事件驱动架构就像作为OpenResty的底层核心来调度Lua协程。

最后我们再来看OpenResty的使用场景：
- OpenResty继承了Nginx的高性能、高并发特性，又嵌入Lua拥有了动态性，OpenResty一大应用点是**网关**，高并发的能力可以使它作为接入层处理大量请求，同时可动态修改、实现丰富的功能，如限流、安全校验、转发等等。
- OpenResty也可以用来处理复杂的业务逻辑，像传统的Web服务器那样作为一个动态HTTP服务器。但是注意与多线程的传统的Web服务器不同，多线程会在操作系统的调度下公平占用CPU时间，而OpenResty不会对协程进行抢占式调度，如果业务逻辑复杂，又没有主动让出的动作，协程就会长时间占用CPU，会导致其他请求迟迟得不到处理，“就像是事件驱动架构遇到了阻塞，无法及时响应新的事件”。


