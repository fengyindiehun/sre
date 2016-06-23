# 关于 Golang 中的 context 模块的介绍

包路径：golang.org/x/net/contex
阅读资源：[go程序包源码解读](http://studygolang.com/articles/5131)

context 包是用来配合 Golang 中并发编程的一种模式的，主要解决的问题是 Go routine 的回收难。context 包由于包广泛使用，现已被加入 Go 1.7 标准库。

## 问题

考虑 agent Watch 一个 ETCD server，这个 Watch 肯定需要一个 Go routine 来做，以免 block 了当前线程。当我们想取消 Watch 的时候，这个 Go routine 必需收到通知，从而结束 Watch。而这个通知如何发呢？

## trivial 的解决方法

如果整个代码都是我们自己写的，可以这样做：直接给 Watch 函数一个 chan struct{} 变量，当这个变量收到信号的时候，就结束 Watch。

## context 的解决方案

可惜的是，这个 Watch 的函数是 coreOS 团队写的，而他们选择的是 context 包，而不是裸露的一个 chan 变量。之所以还要在 chan 上封装，是因为 context 能解决更多问题，包括回收一个 routine 树，超时回收等。

## context 包简介

## context 使用示例

