---
title: nginx-http-slice-module解析
date: 2024-03-14 09:00:00 +0800
categories: [OpenResty]
tags: [OpenResty,CDN]
math: false
mermaid: true
image:
  path: /assets/img/posts/2024-03-14-title.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: versions
---

nginx官方http模块下有一个`ngx_http_slice_filter_module`模块，作用是将一个大文件请求分片成一个一个小的HTTP range请求，有利于提升缓存效果以及控制流量/内存，是个非常常用的模块，模块代码非常简洁，这篇博客对slice模块做一个简单的源码解析。

## 使用
模块只定义了**一个配置指令`slice`**和**一个变量`slice_range`**，模块指令`slice`定义了切片的大小，可用于所有上下文中：
```
Syntax:	slice size;
Default: slice 0;
Context: http, server, location
```
但是如果只配置了`slice`是不会使模块生效的，同时需要配合使用模块定义的变量`slice_range`，变量的值为**当前分片下HTTP range的取值**，使用该变量的方法是结合`proxy_set_header`指令给回源请求添加`HTTP Range Header`，官方文档给出的使用该模块的配置示例如下：
``` nginx
location / {
    slice             1m;
    proxy_cache       cache;
    proxy_cache_key   $uri$is_args$args$slice_range;
    proxy_set_header  Range $slice_range;
    proxy_cache_valid 200 206 1h;
    proxy_pass        http://localhost:8000;
}
```
**注意：一定要同时有`slice             1m;`和`proxy_set_header  Range $slice_range;这两条配置`**
这样配置之后，比如有一个4m大小的资源请求过来，在nginx内部就会转化为4个1m的HTTP Range请求，逐个向源站请求并逐个返回给用户。

![](/assets/img/posts/slice.jpg)

模块的作用（优点）：

-  有效控制峰值带宽和内存的使用，比如某一时刻有并发的100个大请求，经过slice模块，就会转化为100个小请求，在高并发下文件大小对于机器峰值的内存和带宽的使用都有非常明显的影响。
  
-  提高缓存效果。单从Nginx缓存的使用可以参考[这篇文章](https://www.nginx.com/blog/smart-efficient-byte-range-caching-nginx/#cache-slice)。如果从CDN的架构来讲，Nginx作为接入层，首先回源到下一层缓存服务器，缓存服务器会缓存每个Range请求，slice模块还具有尺寸规整的功能，可以有效提高对于原本为Range请求（视频文件非常常见）的缓存命中，同时缓存驱逐也会被划分为小块驱逐，并且小请求将有利于缓存服务器的合并回源。

缺点：

- 代价是一定的CPU内存消耗和整体请求延时的一定增加。但是介于slice模块的优良实现和Nginx优秀的架构，CPU内存的增加基本可以忽略不计，同时对于大文件而言，增加的耗时对于整体请求的影响也微乎其微。

- 缓存清理会增加难度，一个资源需要清理其所有分片。

因此在我们实际使用过程中，会建议对视频文件(mp4等类型)、大文件(apk等类型)开启`slice`，而其他图片类型，以及不是特别大且没有Range请求场景的不开启。

## 原理解析

**参考源码版本为openresty-1.21.4.2，**
模块代码位于`src/http/modules/ngx_http_slice_filter_module.c`，看名字可以知道这是一个过滤模块，nginx在发送请求头和请求体之前会调用各个模块注册的过滤函数，经过过滤链之后发送给用户，与HTTP协议对应，过滤链也有两条，分别是**HTTP Header过滤和Body过滤**。由于Nginx是非阻塞回调的架构，一条请求下Header回调只会被激活一次，但是**Body回调有可能会被激活多次**。

### 模块定义
首先看模块定义：
``` c
static ngx_http_module_t  ngx_http_slice_filter_module_ctx = {
    ngx_http_slice_add_variables,          /* preconfiguration */
    ngx_http_slice_init,                   /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_slice_create_loc_conf,        /* create location configuration */
    ngx_http_slice_merge_loc_conf          /* merge location configuration */
};
```
其中需要关注的就是前两个函数`ngx_http_slice_add_variables`以及`ngx_http_slice_init`，分别在`preconfiguration`和`postconfiguration`阶段回调，其中`ngx_http_slice_init`很简单，就是分别在两个过滤链上注册了自己的filter回调函数：
``` c
static ngx_int_t
ngx_http_slice_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_slice_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_slice_body_filter;

    return NGX_OK;
}
```
请求在向用户发送Header之前会调用`ngx_http_slice_header_filter`，每次发送Body之前会调用`ngx_http_slice_body_filter`，这两个函数具体内容等下再看。

`ngx_http_slice_add_variables`函数也非常简短，顾名思义，模块定义了一个变量添加到Nginx中：
``` c
static ngx_str_t  ngx_http_slice_range_name = ngx_string("slice_range");
...
static ngx_int_t
ngx_http_slice_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var;

    var = ngx_http_add_variable(cf, &ngx_http_slice_range_name, 0);
    if (var == NULL) {
        return NGX_ERROR;
    }

    var->get_handler = ngx_http_slice_range_variable;

    return NGX_OK;
}
```

变量名为`slice_range`，变量对应一个回调函数，当获取变量值的时候会调用回调函数`ngx_http_slice_range_variable`。


### 模块指令

整个模块只定义了一个指令：
``` c
static ngx_command_t  ngx_http_slice_filter_commands[] = {

    { ngx_string("slice"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_size_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_slice_loc_conf_t, size),
      NULL },

      ngx_null_command
};

typedef struct {
    size_t               size;
} ngx_http_slice_loc_conf_t;

```
nginx解析到`slice`指令时会调用对应的函数`ngx_conf_set_size_slot`，函数作用是`slice`对应的值解析并存放到配置`ngx_http_slice_loc_conf_t`的唯一成员`size`。

比如配置`slice 4m;`解析之后变成`ngx_http_slice_loc_conf_t.size=4*1024*1024`。


### 模块解析

可以看到模块一共就定一个一条指令`slice`，指令只是定义了分片的大小并解析到配置文件中，此外还定义了一个变量`slice_range`以及注册了两个过滤链的回调函数。

过滤链是在请求结束发送字节流给客户端之前才会被调用的，那么这个模块是如何完成将一个大请求划分成多个小请求的呢，答案就在`slice_range`，最开始的配置示例说了，只有添加了`proxy_set_header  Range $slice_range;`才能使slice起作用。

假设配置了这样一个nginx服务：
``` nginx
server {
  server_name     a.b.c;
  location / {
    slice             1m;
    proxy_set_header  Range $slice_range;
    proxy_pass        http://localhost:8000;
  }
}
```
当使用到`slice_range`时，会调用变量注册的回调函数`ngx_http_slice_range_variable`:
``` c
static ngx_int_t
ngx_http_slice_range_variable(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    u_char                     *p;
    ngx_http_slice_ctx_t       *ctx;
    ngx_http_slice_loc_conf_t  *slcf;

    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
    // 第一次调用该回调函数（第一次请求上游服务器）
    if (ctx == NULL) {
        // 设置slice模块上下文
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_slice_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }
        ngx_http_set_ctx(r, ctx, ngx_http_slice_filter_module);

        p = ngx_pnalloc(r->pool, sizeof("bytes=-") - 1 + 2 * NGX_OFF_T_LEN);
        if (p == NULL) {
            return NGX_ERROR;
        }
        // 计算Range参数，存储到上下文结构体中
        ctx->start = slcf->size * (ngx_http_slice_get_start(r) / slcf->size);

        ctx->range.data = p;
        // slcf->size即slice的大小 1m = 1048576
        // 计算第一次请求值为：bytes=0-1048576
        ctx->range.len = ngx_sprintf(p, "bytes=%O-%O", ctx->start,
                                     ctx->start + (off_t) slcf->size - 1)
                         - p;
    }

    // 已经有上下文了，直接从上下文中取出range参数的值返回(后续子请求的Range参数更新将都在ngx_http_slice_header_filter中进行)
    v->data = ctx->range.data;
    v->valid = 1;
    v->not_found = 0;
    v->no_cacheable = 1;
    v->len = ctx->range.len;
    return NGX_OK;
}
```

此时来了一个请求`http://a.b.c/test.mp4`，且`/test.mp4`完整大小为4m。请求需要根据`proxy_set_header`添加请求Header `Range`，需要拿`slice_range`的值，根据上述代码可知，**此时设置了模块上下文，并取得`slice_range`值为`bytes=0-1048576`**，由此原本的请求就变成了一个Range请求，大小为1m，发送给上游服务器，这也是第一片请求（**主请求**）。

第一片请求也就是主请求返回后，发送给用户之前调用Header的回调函数，此时会来到slice模块注册的函数`ngx_http_slice_header_filter`:


``` c
static ngx_int_t
ngx_http_slice_header_filter(ngx_http_request_t *r)
{

    // 检查当前请求有模块上下文吗？ 没有则表示没有使用slice模块 什么都不做
    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
    if (ctx == NULL) {
        return ngx_http_next_header_filter(r);
    }

   // 检查返回码是否为206
    if (r->headers_out.status != NGX_HTTP_PARTIAL_CONTENT) {
        // 不是206且为主请求，表示上游服务器不支持Range，那么直接返回，不进行下一步分片
        if (r == r->main) {
            ngx_http_set_ctx(r, NULL, ngx_http_slice_filter_module);
            return ngx_http_next_header_filter(r);
        }
        // 不是206且不是子请求，错误！
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "unexpected status code %ui in slice response",
                      r->headers_out.status);
        return NGX_ERROR;
    }

    // 检查每次请求的etag是否一致
    h = r->headers_out.etag;
    if (ctx->etag.len) {
        if (h == NULL
            || h->value.len != ctx->etag.len
            || ngx_strncmp(h->value.data, ctx->etag.data, ctx->etag.len)
               != 0)
        {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "etag mismatch in slice response");
            return NGX_ERROR;
        }
    }

    ngx_http_slice_content_range_t   cr;
    // 根据此次请求结果，设置下一次请求的Range参数范围，更新到上下文中
    if (ngx_http_slice_parse_content_range(r, &cr) != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid range in slice response");
        return NGX_ERROR;
    }
    slcf = ngx_http_get_module_loc_conf(r, ngx_http_slice_filter_module);
    end = ngx_min(cr.start + (off_t) slcf->size, cr.complete_length);
    if (cr.start != ctx->start || cr.end != end) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "unexpected range in slice response: %O-%O",
                      cr.start, cr.end);
        return NGX_ERROR;
    }
    ctx->start = end;
    ...
    ...
}

typedef struct {
    off_t                start; // range起始字节
    off_t                end; // range末尾字节
    off_t                complete_length; // 总字节
} ngx_http_slice_content_range_t;
```
关键点是：如果没有模块上下文且不是range请求则什么都不做，如果发现有模块上下文，说明配置了slice模块，**则首先检查此时这一片请求的返回码、etag，并计算更新下一片请求的范围，更新到上下文**。

接下来发送请求体之前会调用slice模块注册到Body过滤链的回调函数`ngx_http_slice_body_filter`:

``` c
static ngx_int_t
ngx_http_slice_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    // 没有模块上下文或者不是主请求，忽略
    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);

    if (ctx == NULL || r != r->main) {
        return ngx_http_next_body_filter(r, in);
    }

    //=========以下都是主请求中执行=========

    // 用于判断第一分片是否结束，结束后标记到ctx->last
    for (cl = in; cl; cl = cl->next) {
        if (cl->buf->last_buf) {
            cl->buf->last_buf = 0;
            cl->buf->last_in_chain = 1;
            cl->buf->sync = 1;
            ctx->last = 1;
        }
    }

    // 第一片没结束，什么都不做
    if (rc == NGX_ERROR || !ctx->last) {
        return rc;
    }

    // 第一片已经结束了，说明此时上下文是后序分片，后序分片结束使用子请求的done字段判断

    // 当前分片子请求是否已经传输完成，没完成什么都不做
    if (ctx->sr && !ctx->sr->done) {
        return rc;
    }

    //当前分片子请求已经传输完成

    //判断全部内容是否都已经传输完成？ 传输完成则删除模块上下文
    if (ctx->start >= ctx->end) {
        ngx_http_set_ctx(r, NULL, ngx_http_slice_filter_module);
        ngx_http_send_special(r, NGX_HTTP_LAST);
        return rc;
    }

    // 上一分片已完成，且需要发起下一分片
    // 关键点：注册一个子请求，即发起下一片请求
    if (ngx_http_subrequest(r, &r->uri, &r->args, &ctx->sr, NULL,
                            NGX_HTTP_SUBREQUEST_CLONE)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 更新range参数
    ngx_http_set_ctx(ctx->sr, ctx, ngx_http_slice_filter_module);
    slcf = ngx_http_get_module_loc_conf(r, ngx_http_slice_filter_module);
    ctx->range.len = ngx_sprintf(ctx->range.data, "bytes=%O-%O", ctx->start,
                                 ctx->start + (off_t) slcf->size - 1)
                     - ctx->range.data;
}
```

Body回调函数关键点是**检测上一分片是否传输完毕，如果结束则并利用子请求发送下一分片，直到全部内容都发送完毕**。

### 总结

slice模块利用HTTP协议中的Range请求，通过模块变量的方式开启分片，在请求的Filter阶段检测当前分片并使用子请求的方式发起后序分片请求，从而将大请求串行化成一个一个的小请求。当然这篇文章只是说明了一个大体流程，还有很多细节问题没有搞清楚，包括主请求、子请求的关系、互相回调，客户端原本就是Range请求，返回码的处理等细节。






## 参考

[Smart and Efficient Byte-Range Caching with NGINX & NGINX Plus](https://www.nginx.com/blog/smart-efficient-byte-range-caching-nginx/#cache-slice)

[nginx slice模块的使用和源码分析](https://blog.csdn.net/bluestn/article/details/136029381?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-136029381-blog-129917821.235%5Ev43%5Epc_blog_bottom_relevance_base3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-136029381-blog-129917821.235%5Ev43%5Epc_blog_bottom_relevance_base3)

[nginx分片模块流程分析](https://blog.csdn.net/linux_cwg/article/details/125074021)

