---
layout: post
title:  "记一次DNS引发的问题"
date:   2015-08-26  16:14:47
categories: python
tags: python elasticsearch
---


公司有一个邮件群发的服务，近期需要将邮件群发的服务日志接入到公司的Hadoop日志收集平台，

Hadoop已经有一个RESTful的HTTP封装接口，
通过调用这个接口，可以将日志写入到flume中去，再由flume写入HDFS.

我的业务代码需要频繁的调用这个HTTP接口。
当时的想法是，反正HTTP接口本身是带异步队列的，
我就算同步阻塞的调用它，也不会怎么样，
因为毕竟服务是在内网中，调用起来也应该非常快。

早上邮件服务公司打电话过来，
说他们的消息队列已经快满了，
原因是我们这个接口调用返回的太慢。

查看日志发现，我的服务的响应时间超过*10s*...OMG..!

我调用的那个接口时而300ms返回，时而5s才返回，时而10s才返回，导致整个服务被远程的并发连接直接打死.


赶紧联系同事，发现他的接口在别处调用都没问题，我甚至在公司的跳板机中开2000个并发都打不慢他的服务。


这什么情况!

算了！改变一下实现方式，使用异步+HTTP连接池的方式来调用HTTP接口。

发现处理时间缩短为3ms，

看起来没啥问题了.

不过还是可以看到每隔一段时间，就会有偶尔一个请求几秒才返回，可大部分都是3ms左右返回.


难道是DNS问题？(我通过域名来调用HTTP接口)

最后，发现Linux的DNS配置为8.8.8.8 而由于你懂的的原因，8.8.8.8的返回常被干扰。

改为114.114.114.114，问题解决。

真是X了.


总结: 

*  linux服务器是不缓存DNS结果的！每一次使用到域名的情况，都会重新请求DNS！

*  永远不要在生产环境下部署8.8.8.8 作为DNS ... 党国实在...