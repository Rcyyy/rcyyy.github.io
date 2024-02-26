---
title: Unix Domain Socket
# author: aaa
date: 2023-03-26 14:00:00 +0800
categories: [Linux, IPC]
tags: [IPC, Socket, UDS]
math: false
mermaid: true
# comments: true
image:
  path: /assets/img/posts/2023-03-19.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Socket
---

> 最近在工作中使用了 `Unix Domain Socket(简称UDS)`，开发了一个`local agent`采用**UPD模式**用来收集本机其他进程的发送的日志等数据，业务进程的数据通过`Unix Domain Socket`将数据发送给本机agent，由`local agent`处理数据、聚合数据等。在过程中踩了一些坑，以及对比网络协议栈中的`socket`，进行一些总结和思考。
<!-- {: .prompt-tip } -->

## UDS Usage
---
`Unix Domain Socket`是操作系统提供的进程间通信方法的其中一种，是POSIX操作系统标准的一部分，相比于其他IPC方法，UDS使用起来与常用的网络Socket（TCP/IP,address family:AF_INET）编程很相似，因此使用起来很方便，**UDS只是实现了Socket层的一些接口，并没有经过内核复杂协议栈来传递消息**，更像是一种进程之间的双向管道，传递效率也相当高。

因为平时对TCP/UDP的socket编程相对熟悉，因此本文就对比TCP/UDP的socket编程流程、特性进行对比书写。废话不多说，先看示例代码。

**1. Server端代码**
   
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <stdio.h>
#include <sys/un.h>
#include <iostream>
#include <unistd.h>
#include <string>

const char* SERVER_UNIX_FILE = "/tmp/test_server.sock";
char buff[1024*1024*4] {'0'};

int main(int argc,char** argv)
{
    // create socket
    int fd = socket(AF_UNIX,SOCK_DGRAM,0);
    if (fd < 0)
    {
        perror("socket");
        return -1;
    }

    // init socket address
    struct sockaddr_un serveraddr;
    memset(&serveraddr, 0, sizeof(serveraddr));
    serveraddr.sun_family = AF_UNIX;
    strncpy(serveraddr.sun_path, SERVER_UNIX_FILE, strlen(SERVER_UNIX_FILE));

    // check file existence
    if (!access(SERVER_UNIX_FILE, F_OK))
    {
        // if exist, delete
        remove(SERVER_UNIX_FILE);
    }

    // bind
    if (bind(fd, (struct sockaddr*)&serveraddr, sizeof(serveraddr)) != 0)
    {
        perror("bind");
        close(fd);
        return -1;
    }

    // set file permissions
    if (chmod(SERVER_UNIX_FILE, 0666) == -1)
    {
        perror("chmod");
        close(fd);
        remove(SERVER_UNIX_FILE);
        return -1;
    }

    // wait for message
    struct sockaddr_un clienSocket;
    socklen_t sock_size = sizeof(clienSocket);
    while(true)
    {
        int iRet = recvfrom(fd, buff, sizeof(buff), 0, (sockaddr*)(&clienSocket),
                            &sock_size);
        if (iRet == -1)
        {
            perror("recvfrom");
            break;
        }
        else if (iRet > 0)
        {
            std::cout << "Server received message! length:" << iRet << std::endl;
        }
    }

    close(fd);
    remove(SERVER_UNIX_FILE);
    return 0;
    
}
```

可以看到和网络协议socket使用非常相似，也可以选择协议类型为UDP还是TCP。区别在于，标识UDS的socket类型是**AF_UNIX(等同于AF_LOCAL)**，由于是本地通信，也就不需要什么IP的概念，因此绑定的socket地址是一个**文件**，也就是用文件来标识通信的一方，等同于**IP:PORT**。上面代码创建了一个监听UDP消息的server端。

成功运行之后，可以在文件系统上看到这个文件，是一个socket类型特殊文件，接下来就可以使用客户端进程来传递消息了:
![](/assets/img/posts/server-socket.png)


**2. Client端代码**

```c

#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <stdio.h>
#include <sys/un.h>
#include <iostream>
#include <unistd.h>
#include <string>

const char* SERVER_UNIX_FILE = "/tmp/test_server.sock";
char buff[1024*1024*4] {'0'};

int main(int argc,char** argv)
{
    // create socket
    int fd = socket(AF_UNIX,SOCK_DGRAM,0);
    if (fd < 0)
    {
        perror("socket");
        return -1;
    }

    // init socket address
    struct sockaddr_un serveraddr;
    memset(&serveraddr, 0, sizeof(serveraddr));
    serveraddr.sun_family = AF_UNIX;
    strncpy(serveraddr.sun_path, SERVER_UNIX_FILE, strlen(SERVER_UNIX_FILE));
    int len = sizeof(serveraddr);

    // send message
    std::string msg(100, 'a');
    int sendSize = sendto(fd, msg.c_str(), msg.size(), 0, (sockaddr*)&serveraddr, len);
    if (sendSize < 0)
    {
        perror("sendto");
        std::cout << "Client send fail" << std::endl; 
    }
    else
    {
        std::cout << "Client send success! message length:" << sendSize << std::endl; 
    }
   
    return 0;
    
}

```
client端代码也很简单，创建socket，初始化socket地址（文件路径就是server端绑定的那一个），然后就可以发消息了。

![](/assets/img/posts/server-100.png)
![](/assets/img/posts/client-100.png)

## Problem-1:Message too long
> 以为搞好了就万事大吉，开始收发消息了，为了提高通信效率，我在客户端将消息合并累积发送，随后就遇到了这个错误： `sendto: Message too long` 。
{: .prompt-warning }

这个错误出在client端的sendto函数，字面意思来看就是消息长度太长，超过限制。那么这个限制是什么呢？**在网络协议的UDP协议中，一个UDP包的最大长度为65535**（UPD协议16位标识长度）。但UDS并不经过网络协议栈，理论上不会受这个限制，与这个无关，那么由什么控制呢？So check the source code.

我参考的内核版本是linux-5.4.236,UDS相关的实现源码都在`linux-5.4.236/net/unix`目录下，最终找到`sendto`调用的函数是

```c

// linux-5.4.236/net/unix/af_unix.c
static int unix_dgram_sendmsg(struct socket *sock, struct msghdr *msg,
			      size_t len)
{
    struct sock *sk = sock->sk;
	struct net *net = sock_net(sk);
	struct unix_sock *u = unix_sk(sk);
    // ...
    err = -EMSGSIZE;
	if (len > sk->sk_sndbuf - 32)
		goto out;
    // ...
}

```

定位到`Message too long`的错误码`EMSGSIZE`在这里出现，当发送消息的长度`len > sk->sk_sndbuf - 32`就会报错`EMSGSIZE`。
其中sock是内核中socket编程中非常核心的一个结构体，再来看`sk_sndbuf`是什么，可以在`sock`结构体定义中看到：

```c
/**
  *	struct sock - network layer representation of sockets
  ...
  *	@sk_sndbuf: size of send buffer in bytes
  ...
  **/
//linux-5.4.236/include/net/sock.h
struct sock {
}
```
在UDS代码中创建`socket`的时候就会对`sock结构体`进行初始化，初始化的相关代码如下：

```c
// linux-5.4.236/net/unix/af_unix.c
static struct sock *unix_create1(struct net *net, struct socket *sock, int kern)
{
	struct sock *sk = NULL;
	struct unix_sock *u;

    // 分配空间
	sk = sk_alloc(net, PF_UNIX, GFP_KERNEL, &unix_proto, kern);
	if (!sk)
		goto out;

    // 初始化
	sock_init_data(sock, sk);
    ...
}

//最终调用到这里
//linux-5.4.236/net/core/sock.c
void sock_init_data_uid(struct socket *sock, struct sock *sk, kuid_t uid)
{
	sk_init_common(sk);
	sk->sk_send_head	=	NULL;

	timer_setup(&sk->sk_timer, NULL, 0);

	sk->sk_allocation	=	GFP_KERNEL;
	sk->sk_rcvbuf		=	sysctl_rmem_default;
    // here 最终答案！
	sk->sk_sndbuf		=	sysctl_wmem_default;
	sk->sk_state		=	TCP_CLOSE;
}

```
最终看到要找的长度限制在哪里了，`sysctl_wmem_default`这个是内核参数，在这个文件里`/etc/sysctl.conf`可以看到实验机器的参数为：

```yaml
#per socket, byte
net.core.optmem_max = 40960
net.core.rmem_default = 65536
net.core.rmem_max = 33554432
net.core.wmem_default = 65536
net.core.wmem_max = 33554432
```

所以最大长度的限制就是`65536-32=65504`，用代码验证一下：
![](/assets/img/posts/send-65536.png)

可以看到问题确实就在这里，怎么改呢？可以直接修改内核参数对所有进程生效，也可以通过`setsockopt`系统调用设置某个socket的`SO_SNDBUF`，同样也有`getsockopt`系统调用获取当前socket某个选项的值。这里我直接通过修改系统参数，然后`sysctl -p`生效。修改后参数如下：

```yaml
#per socket, byte
net.core.optmem_max = 40960
net.core.rmem_default = 8388608 
net.core.rmem_max = 33554432
# 8M
net.core.wmem_default = 8388608
net.core.wmem_max = 33554432
```

改为了8M，再测试下：

![](/assets/img/posts/clien-1m.png)
![](/assets/img/posts/server-1m.png)

## Problem-1 Summary
- UDS(UDP)发送的大小受`net.core.wmem_default`限制，这个参数不仅对UDS的socket生效，它是socket全局参数，不管什么socket。
- 最大值为`net.core.wmem_default-32`，至于为什么减32，可能是在设计的时候考虑加一些包头之类的，但在UDS场景下毫无意义。
- 可以通过修改内核参数或者`setsockopt`修改，但不得大于`net.core.wmem_max`。
  
## Problem-2: No buffer space available

> 在上面将`net.core.wmem_default`改为8M大小后，随着我调大发送数据的大小，当增大为5M的时候，又在发送端的sendto函数出现了错误： `sendto: No buffer space available` 。
{: .prompt-warning }

这个错误最开始字面理解的话，就是内存buffer空间不足了。进程间通信总要使用一段共享内存，UPS也是类似的。但我只发了一个消息，难道buffer大小只有不到5M吗？

修改一下server端代码，只监听不收取消息，然后客户端来发包，测试发现发送几十M都没问题，所以这个错误**并不是总buffer的限制，还是对单个消息大小的限制**，继续看源码找原因。还是`unix_dgram_sendmsg`这个函数，先了解一下大致的发送流程。


```c
// linux-5.4.236/net/unix/af_unix.c
static int unix_dgram_sendmsg(struct socket *sock, struct msghdr *msg,
			      size_t len)
{
    struct sock *sk = sock->sk;
    // 对端sock(对本文而言就是server端的sock)
    struct sock *other = NULL;
    // 内核中用来存放消息的结构题
    struct sk_buff *skb;
    int data_len = 0;

    // problem-1 已经讨论过的
    err = -EMSGSIZE;
	if (len > sk->sk_sndbuf - 32)
		goto out;

    // 根据要发送的数据大小，分配sk_buff用来放数据
    {
        if (len > SKB_MAX_ALLOC) {
            data_len = min_t(size_t,
                    len - SKB_MAX_ALLOC,
                    MAX_SKB_FRAGS * PAGE_SIZE);
            data_len = PAGE_ALIGN(data_len);

            BUILD_BUG_ON(SKB_MAX_ALLOC < PAGE_SIZE);
        }
        skb = sock_alloc_send_pskb(sk, len - data_len, data_len,
                    msg->msg_flags & MSG_DONTWAIT, &err,
                    PAGE_ALLOC_COSTLY_ORDER);
        if (skb == NULL)
		    goto out;
    }

    // 根据文件名找到对端的地址
    other = unix_find_other(net, sunaddr, namelen, sk->sk_type,
					hash, &err);
    // 将sk_buf挂到对方的接受队列上
    skb_queue_tail(&other->sk_receive_queue, skb);
	return len;
}
```
在上面代码中可以看到UDS的大致逻辑，当然我省略了很多很多的代码（我看不懂的和无关的），就是在内核空间申请一段内存`sk_buff`，将用户要发送的数据拷贝进去，然后找到`对方的sock`，将`sk_buff`放到对方的接受队列中，over。

这里看一下`sk_buff`，这个结构体也是内核网络栈中通用的一个非常重要的结构体，在发送数据，经过不同网络协议层一层一层包装，或者接受数据，一层一层解协议，都是用`sk_buff`这个结构体来传递和容纳数据包的，它里面有两类存放数据的地方，一块是线性区，内存上是连续的，用于存放协议头之类的，长度为`header_len`；另一块是非线性区，将大块内存用链表结构组织起来，用来存放大块数据，长度为`data_len`。

那么`No buffer space available`这个错误就是发生在了创建`sk_buff`这一步，我在上面代码也用大括号括了起来，第一步判断`len > SKB_MAX_ALLOC`，`SKB_MAX_ALLOC`这个宏定义在：


```c

// linux-5.4.236/include/unix/sk_buff.h
#define SKB_DATA_ALIGN(X)	ALIGN(X, SMP_CACHE_BYTES)
#define SKB_WITH_OVERHEAD(X)	\
	((X) - SKB_DATA_ALIGN(sizeof(struct skb_shared_info)))
#define SKB_MAX_ORDER(X, ORDER) \
	SKB_WITH_OVERHEAD((PAGE_SIZE << (ORDER)) - (X))
#define SKB_MAX_HEAD(X)		(SKB_MAX_ORDER((X), 0))
#define SKB_MAX_ALLOC		(SKB_MAX_ORDER(0, 2))

#if (65536/PAGE_SIZE + 1) < 16
#define MAX_SKB_FRAGS 16UL
#else
#define MAX_SKB_FRAGS (65536/PAGE_SIZE + 1)
#endif


// linux-5.4.236/arch/x86/include/asm/page_types.h
/* PAGE_SHIFT determines the page size */
#define PAGE_SHIFT		12
#define PAGE_SIZE		(_AC(1,UL) << PAGE_SHIFT)
#define PAGE_MASK		(~(PAGE_SIZE-1))

```

经过上面的宏定义可以看到内存页大小为`4kb`，`SKB_MAX_ALLOC`大小为4倍内存页大小，就是`16kb`。因此大于`16kb`的数据会走到`date_len`的赋值计算里来。`data_len`赋值为`len - SKB_MAX_ALLOC`和`MAX_SKB_FRAGS * PAGE_SIZE`较小值，`SKB_MAX_ALLOC`已经计算出来了为`16kb`，`MAX_SKB_FRAGS`的宏也在上面，`(65536/PAGE_SIZE + 1)`也就是`17kb`。那么`data_len=min(len - 16kb, 17*4kb)`。

这一步的意义是什么呢？没有找到准确的相关资料，这里应该是对数据大小进行一个限制，防止内存占用的过大，同时对`header_len`和`data_len`都进行了一定的限制。

计算完后就进入到`sk_buff`的分配函数中，代码如下：

```c
/*
 *	Generic send/receive buffer handlers
 */
// linux-5.4.236/net/core/sock.c
struct sk_buff *sock_alloc_send_pskb(struct sock *sk, unsigned long header_len,
				     unsigned long data_len, int noblock,
				     int *errcode, int max_page_order)
{
    struct sk_buff *skb;
    //...
    skb = alloc_skb_with_frags(header_len, data_len, max_page_order,
				   errcode, sk->sk_allocation);
    //...
    return skb;
}

/**
 * alloc_skb_with_frags - allocate skb with page frags
 *
 * @header_len: size of linear part
 * @data_len: needed length in frags
 * @max_page_order: max page order desired.
 * @errcode: pointer to error code if any
 * @gfp_mask: allocation mask
 *
 * This can be used to allocate a paged skb, given a maximal order for frags.
 */
 // linux-5.4.236/net/core/sk_buff.c
 struct sk_buff *alloc_skb_with_frags(unsigned long header_len,
				     unsigned long data_len,
				     int max_page_order,
				     int *errcode,
				     gfp_t gfp_mask)
{
    int npages = (data_len + (PAGE_SIZE - 1)) >> PAGE_SHIFT;
	struct sk_buff *skb;
	struct page *page;
    //...

	*errcode = -EMSGSIZE;
	/* Note this test could be relaxed, if we succeed to allocate
	 * high order pages...
	 */
	if (npages > MAX_SKB_FRAGS)
		return NULL;

	*errcode = -ENOBUFS;
	skb = alloc_skb(header_len, gfp_mask);
	if (!skb)
		return NULL;
    for (i = 0; npages > 0; i++) {
        while (order) {
            //...
            page = alloc_pages()
            //...
		}
        page = alloc_page(gfp_mask);
        skb_fill_page_desc(skb, i, page, 0, chunk);
        //...
    }
    return skb;
}
```
在`alloc_skb_with_frags`这个函数里面终于看到了`ENOBUFS`这个错误码，不出意外的话问题应该就出在`alloc_skb`这个函数里。同时在上面这个函数也可以了解到`sk_buff`的创建过程，`header_len`部分直接申请一段连续空间，`data_len`部分计算需要多少内存页，然后申请n块内存页并组织起来。

```c
/*
 * Allocate a new &sk_buff. The returned buffer has no headroom and a
 *	tail room of at least size bytes. The object has a reference count
 *	of one. The return is the buffer. On a failure the return is %NULL.
 */
// linux-5.4.236/net/core/sk_buff.c
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
			    int flags, int node)
{
    struct sk_buff *skb;
    //...
    /* We do our best to align skb_shared_info on a separate cache
	 * line. It usually works because kmalloc(X > SMP_CACHE_BYTES) gives
	 * aligned memory blocks, unless SLUB/SLAB debug is enabled.
	 * Both skb->head and skb_shared_info are cache line aligned.
	 */
    size = SKB_DATA_ALIGN(size);
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);
	if (!data)
		goto nodata;
    //...
}
```
``__alloc_skb``这个函数调用`kmalloc`来分配一段连续的内存空间，在申请之前加上了一块`struct skb_shared_info`（这个暂时没有了解）。

`kmalloc`就是最终的内存分配函数了，用来分配一块连续的内存空间，就像我们在用户空间使用的`malloc`似的。在内核中，对一次可分配的最大值当然是有限制的，这个限制首先要看内核使用的内存管理器`SLAB/SLOB/SLUB`,这个可以通过`cat /boot/config-xxx`系统配置中看到，实验机器用的是`SLUB`

```yaml
CONFIG_SLUB_DEBUG=y
CONFIG_SLUB_MEMCG_SYSFS_ON=y
CONFIG_SLUB=y
CONFIG_SLUB_CPU_PARTIAL=y
```

再找`SLUB`对于最大内存的限制，在`linux-5.4.236/include/linux/slab.h`:

```c
/* Maximum allocatable size */
#define KMALLOC_MAX_SIZE	(1UL << KMALLOC_SHIFT_MAX)
/* Maximum size for which we actually use a slab cache */
#define KMALLOC_MAX_CACHE_SIZE	(1UL << KMALLOC_SHIFT_HIGH)
/* Maximum order allocatable via the slab allocagtor */
#define KMALLOC_MAX_ORDER	(KMALLOC_SHIFT_MAX - PAGE_SHIFT)

#ifdef CONFIG_SLUB
/*
 * SLUB directly allocates requests fitting in to an order-1 page
 * (PAGE_SIZE*2).  Larger requests are passed to the page allocator.
 */
#define KMALLOC_SHIFT_HIGH	(PAGE_SHIFT + 1)
#define KMALLOC_SHIFT_MAX	(MAX_ORDER + PAGE_SHIFT - 1)
#ifndef KMALLOC_SHIFT_LOW
#define KMALLOC_SHIFT_LOW	3
#endif

//linux-5.4.236/include/linux/mmzone.h
#ifndef CONFIG_FORCE_MAX_ZONEORDER
#define MAX_ORDER 11
#else
#define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
#endif
```
从宏定义中可以看到，最大值为`MAX_ORDER-1=10`个内存页大小(`PAGE_SHIFT`),等于`4mb`。
到这里就可以倒推我们在**UDS中UDP模式下单次可以发送的最大数据大小**(前提是没有问题1的干涉)，

```yaml
KMALLOC_MAX = 4*1024*1024
sizeof(skb_shared_info) = 320
```
所以可以分配的最大连续空间，也就是`sk_buff`的`header_len`长度为`1024*1024*4-320=4193984`，`sk_buff`的`data_len`部分为`17*4kb=69632`,所以最大可单次发送的数据大小，也就是最大可分配的`sk_buff`大小等于`header_len+data_len=4263616`。验证一下：

![](/assets/img/posts/client-4263616.png)

所以问题解决了！

## Problem-2 Summary
- 单次发送的数据大小受`sk_buff`可申请大小的限制
- 分别对`sk_buff`的`header_len`和`data_len`最大值有限制
- `header_len`最大约为内核内存分配器可分配的最大值`kmalloc`
- `data_len`最大由宏确定，本实验为`17个内存页大小`



## Summary
___
   本文的探讨都集中在发送端，而且是UPD模式下。TCP模式下(`SOCK_STREAM`)的UDS不一定适用。在UDS场景下选用`UPD`还是`TCP`更多是使用方式的不同，因为本机通信不会存在**乱序**、**丢包**等网络问题的存在，之后有用到的话在研究吧。

## Update
后续在使用的时候有的机器偶现NO BUFFER错误。后续已解决，见[这篇博客](../unix-domain-socket)
   
