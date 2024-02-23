---
title: OpenResty Lua全局变量
date: 2024-01-08 09:00:00 +0800
categories: [OpenResty]
tags: [OpenResty]
math: false
mermaid: true
image:
  path: /assets/img/posts/2024-01-08.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: 
---
这篇文章讨论下OpenResty框架下编码时，主要指`access/set/rewrite/content_by_lua_*`上下文中Lua变量的**使用、作用域或者说可见性**。

OpenResty使用的是LuaJIT，语法上兼容Lua5.1，对Lua之后版本的语言特性和修改不一定支持，因此这篇文章的讨论也是在`LuaJIT2.1`以及`Lua5.1`的背景下。

## Lua中的全局变量
在通过Lua虚拟机单独运行一段Lua代码时，代码中声明一个全局变量非常简单，不带任何关键字的变量名就会被Lua认作全局变量。要定义一个局部变量需要使用`local`关键字。

全局变量的作用域是全局的，定义以后作用域贯穿整个运行生命周期，**在不同文件、函数、coroutine中都是可见的**。
``` lua
--global variable, value:nil
name = nil

--global variable, value:18
age = 18

--local variable, value:20
local age = 20
```
默认情况下全局变量是存在一个全局环境表(`global environment`)中，它就是一个常规的Lua Table，里面的k/v保存了全局变量，其中有一个key `_G`指向全局环境表自身，因此我们可以通过`_G`来操作全局变量。可以看到平时使用的Lua标准库函数`print`、`require`等也是存在全局变量中的，我们也可以通过`_G`修改全局表中的内容，比如把`print`函数删除，删除后就再也无法获取和使用print函数，如下示例代码：

``` lua
name = nil
age = 18
local age = 20
for k in pairs(_G) do
  print(k)
end

--[[result:
_G
math
assert
module
require
table
...
name
...
age
...
]]

_G.print = nil
-- or print = nil
print(1)
--result:
--attempt to call global 'print' (a nil value)
```
可以看到在Lua中全局变量的使用非常的简单和随意，而Lua作为一门胶水语言，使用场景经常需要加载调用外部代码，需要对全局变量提供某种保护或者隔离机制。在Lua中提供了两个操作环境变量的函数，分别是`getfenv/setfenv`，为函数设置新的环境从而实现运行环境的隔离。
``` lua
local function exec()
    print("exec in")
    print = nil
end

local newGlobal = {}
local mt = {__index = _G}
setmetatable(newGlobal, mt)

setfenv(exec, newGlobal)
exec()

print("main done")

--[[result:
  exec in
  main done
]]
```
上述代码是一种经典`sand box`实现，创建一个新的环境表并设置一个元表，元方法`__index`指向原全局环境表`_G`，最后将新的环境表**设置为某个函数执行的环境**，在这个函数执行的过程中既可以访问到调用者提供的全局变量，在运行中定义和修改的全局变量也不会污染原环境表(当然这个简单的实现还是有办法可以修改的，这里不展开了)。

## OpenResty下Lua运行环境
简单介绍完Lua中的全局变量，我们再回到OpenResty中，OpenResty会创建一个新的coroutine来运行定义的一段lua handler代码(指OpenResty提供的set/rewrite/access/content_by_lua*命令，下文不再赘述)，**从上文我们知道Lua全局变量默认是coroutine共享的，所以在OpenResty中是这样吗？有没有设置`sand box`环境?**

OpenResty(ngx_lua module)官方文档中[Lua Variable Scope](https://github.com/openresty/lua-nginx-module#lua-variable-scope)中提到，在设计上，每个请求的lua handler拥有独立的运行环境，在各个lua handlerOpenResty会创建一个新的coroutine和新的environment用于执行lua代码。

> Here is the reason: by design, the global environment has exactly the same lifetime as the Nginx request handler associated with it. Each request handler has its own set of Lua global variables and that is the idea of request isolation. 

> Acts as a "content handler" and executes Lua code string specified in { lua-script } for every request. The Lua code may make API calls and is executed as a new spawned coroutine in an independent global environment (i.e. a sandbox).

那么每个handler中的lua代码应该是拥有独立的全局环境表的，并且在执行中定义和修改的全局变量不会干扰到其他阶段以及其他请求，但是我在测试的时候却发现并不是这样，每个worker所有Lua代码使用的是同一个全局环境表！测试方式如下，一个location设置全局变量，再通过另一个location看能否取到，同时打印全局环境表的地址。

``` nginx
worker_processes  1;
server {
...
  location = /set {
    content_by_lua_block { key = 1 ngx.say("set global done!") ngx.say(tostring(_G)) }
  }

  location = /get {
    content_by_lua_block { ngx.say(key or "nil") ngx.say(tostring(_G)) }
  }
}

```

感兴趣的可以自己测一下，我下载了`openresty-1.21.4.2`源码，分别使用OpenResty自己的LuaJIT以及官方版本的LuaJIT2.1编译两个OpenResty进行测试，结果就是：
- **OpenResty LuaJIT版本在多次请求下使用的都是同一个全局变量表**
- **官方LuaJIT版本如文档所述，每个请求的全局表进行了隔离**
  


之后就是阅读源码，在OpenResty执行每个handler之前创建coroutine处有如下代码：
``` c
// ngx_stream_lua_util.c
lua_State *
ngx_stream_lua_new_thread(ngx_stream_lua_request_t *r, lua_State *L, int *ref)
{
    ...
    co = lua_newthread(L);

#ifndef OPENRESTY_LUAJIT
    /*  new globals table for coroutine */
    ngx_stream_lua_create_new_globals_table(co, 0, 0);

    lua_createtable(co, 0, 1);
    ngx_stream_lua_get_globals_table(co);
    lua_setfield(co, -2, "__index");
    lua_setmetatable(co, -2);

    ngx_stream_lua_set_globals_table(co);
#endif /* OPENRESTY_LUAJIT */
    return co;
}
```
看到如果没有`OPENRESTY_LUAJIT`宏定义，就给新创建的coroutine创建一个新的全局表，隔离方法与前文Lua代码设置隔离环境方法一样。如果有定义`OPENRESTY_LUAJIT`就直接返回并使用了，此时这个coroutine的全局环境表与`lua vm`是一样的。

`OPENRESTY_LUAJIT`宏定义在OpenResty自己维护的LuaJIT下的`luajit.h`文件中，也就是说编译OpenResty时，**如果使用的是OpenResty自己版本的LuaJIT，则不需要设置新的环境表。这个问题困扰了我好几天，官方文档并未提及LuaJIT版本不同会导致Lua全局环境表的设置不同，同时也带来一个问题：为什么OPENRESTY_LUAJIT不设置隔离环境？为什么其他版本LuaJIT需要设置隔离环境？**

在进行了多次搜索后，发现了ngx_lua模块下非常接近这个问题的两次[PR](https://github.com/openresty/lua-nginx-module/pull/1379)，`OPENRESTY_LUAJIT`这个宏的添加好像也是在这次合进去的，两次PR信息如下：
> commit 7286812116940216344ade33722c49ae47037605  
> Author: doujiang24 <doujiang24@gmail.com>
> Date:   Wed Jul 26 15:51:38 2017 +0800
>
> feature: added support for ARM64 (Aarch64).
> 
> On architectures other than ARM64, the thread.exdata API also saves the
per-request global env table and closure objects for each light thread,
which gives a nice ~10% speedup for the simplest Lua handler location
loaded by wrk over HTTP 1.1.

> commit 3754757be7acd4d8118bbfa0d10a334e4f45875a  
> Author: Thibault Charbonnier <thibaultcha@users.noreply.github.com>  
> Date:   Thu Nov 8 17:12:31 2018 -0800
>
> change: we now print an alert when a non openresty-specific version of LuaJIT is detected since many optimizations would be missing.  
> 
> Right now whether OpenResty's LuaJIT is used in this module makes a difference for many use cases 
> (like the new table.clone API and the new closure-factorary-less and **global-env-less** request handlers). 
> Let's issue a warning to the user when OpenResty's LuaJIT is not used during the nginx startup.


因此我猜测导致这个的原因是：OpenResty为了支持ARM64架构，通过修改LuaJIT的实现达成，同时呢也刚好可以去掉全局表的重新设置以及隔离，因此也在ARM64以外原有支持的架构上取得了~10%性能的提升。

具体细节以及原因有待考证，但是我觉得官方文档对此没有说明也是有问题的，有可能导致用户的使用错误，也提了一个issue看有没有人解答下。



## 结论

首先在OpenResty环境下编码应避免使用全局变量，尽量使用`require`的方式。如果有全局变量的使用，应当注意你所使用的OpenResty版本，官方版本同一个worker使用的是同一个全局环境表，有可能导致问题或难以定位的bug。



## 参考

[Lua 5.1 Reference Manual](https://www.lua.org/manual/5.1/manual.html)

[ngx_lua](https://github.com/openresty/lua-nginx-module)

OpenResty-1.21.4.2


