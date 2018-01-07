---
title: reactor
tags: reactor, netty
grammar_cjkRuby: true
---

这篇文章不错，
https://my.oschina.net/andylucc/blog/618179


http://www.infoq.com/cn/articles/netty-threading-model
http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf

上面这两篇对reactor在细节方面的理解有点不一样。
Doug认为在多线程Reactor中，只有一个Reactor，Reactor要负责读写io操作，其他线程负责Handler中。
另一位作者认为，有多个Reactor，每个Reactor在一个线程中，负责读写，编解码等其他操作
个人觉得，netty EventLoopGroup更像是第二种。

主从Reactor的关键点是，主Reactor在一个线程池中，他有负责认证等功能