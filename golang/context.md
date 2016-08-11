# 关于 Golang 中的 context 包的介绍

包路径：golang.org/x/net/context
阅读资源：[go程序包源码解读](http://studygolang.com/articles/5131)

context 包是用来解决 Goroutine 控制问题的。context 包由于被广泛使用，现已被加入 Go 1.7 标准库。

## 问题

考虑 agent Watch 一个 ETCD server，这个 Watch 肯定需要一个 Goroutine 来做，以免 block 了当前线程。当我们想取消 Watch 的时候，执行 Watch 的 Goroutine 必需收到通知，从而结束 Watch。否则 Watch 过程可能由于没有监管导致 Goroutine 泄露。而这个通知如何发呢？

## trivial 的解决方法

如果整套代码都是我们自己写的，可以这样做：直接给 Watch 函数一个 chan struct{} 变量，当这个变量收到信号的时候，就结束 Watch。

## context 的解决方案

可惜的是，这个 Watch 的函数是 coreOS 团队写的，而他们选择的是 context 包，而不是裸露的一个 chan 变量。之所以还要在 chan 上封装，是因为 context 能解决更多问题，包括回收一个 Goroutine 树，超时回收等。

## context 包简介

context 包的使用非常简单。

Context 是一个 interface，有四个需要实现的函数：
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```
我们不需要手动实现这个接口，context 包已经给我们提供了两个，一个是 `Background()`，一个是 `TODO()`，这两个函数都会返回一个 Context 的实例。

一共有四个主要的包函数：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```
其中，WithCancel 返回一个新的 Context，同时返回一个 CancelFunc，CancelFunc 是 context 包中定义的一个函数类型：`type CancelFunc func()`。当我调用这个 CancelFunc 的时候，执行 Watch 的 Goroutine 就会退出。当然，这也依赖这个 Watch 函数被正确实现了。如果 Watch 实现得不对，仍然无法收拾残局。

WithDeadline 和 WithTimeout 是相似的，WithDeadline 是设置具体的 deadline 时间，到达 deadline 的时候，Watch 会退出，而 WithTimeout 简单粗暴，直接 `return WithDeadline(parent, time.Now().Add(timeout))`。

WithValue 是在 Context 中设置一个 map，拿到这个 Context 以及它的后代的 Goroutine 都可以拿到 map 里的值。

## context 使用示例

### 使用使用了 context 包的包 :P

考虑 etcd 中的 KeysAPI 的 Get 方法。在 agent 的 config.go 中，使用了这个方法：
```go
ctx, _ := context.WithTimeout(context.Background(), 5*time.Second)
if resp, err := s.keysAPI.Get(ctx, key, nil); err != nil {
    return fmt.Errorf("get %s: %v", key, err)
} else {
    doSomething(resp)
}
```
这里在 context 中设置了 5 秒超时，同时直接忽略了返回的 CancelFunc，因为不需要在中途调用它。ctx 被当做第一个参数传入 Get 方法。注意，context 包推荐所有使用了 context 包的接口，都将 context 设置为第一个参数。

考虑我们需要 Set 方法，同样 5s 退出，但是，同时当前 Goroutine 不能 block，而且我还能在中途 Cancel 掉这个 Set。直接给出代码：
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
go s.keysAPI.Set(ctx, key, value, nil)
err := doSomething()
if err != nil {
    cancel() // 一言不合就 cancel 掉执行 Set 的 Goroutine。
}
```

### 写一个这样的 API

```go
package main

import (
	"fmt"
	"time"
	"golang.org/x/net/context"
)

// 模拟一个最小执行时间的阻塞函数
func inc(a int) int {
	res := a + 1                // 虽然我只做了一次简单的 +1 的运算,
	time.Sleep(1 * time.Second) // 但是由于我的机器指令集中没有这条指令,
	// 所以在我执行了 1000000000 条机器指令, 续了 1s 之后, 我才终于得到结果。B)
	return res
}

// 向外部提供的阻塞接口
// 计算 a + b, 注意 a, b 均不能为负
// 如果计算被中断, 则返回 -1
func Add(ctx context.Context, a, b int) int {
	res := 0
	for i := 0; i < a; i++ {
		res = inc(res)
		select {
		case <-ctx.Done():
			return -1
		default:
		}
	}
	for i := 0; i < b; i++ {
		res = inc(res)
		select {
		case <-ctx.Done():
			return -1
		default:
		}
	}
	return res
}

// ----------------- API 构造完毕 -----------------

func main() {
	{
		// 使用开放的 API 计算 a+b
		a := 1
		b := 2
		timeout := 2 * time.Second
		ctx, _ := context.WithTimeout(context.Background(), timeout)
		res := Add(ctx, 1, 2)
		fmt.Printf("Compute: %d+%d, result: %d\n", a, b, res)
	}
	{
		// 手动取消
		a := 1
		b := 2
		ctx, cancel := context.WithCancel(context.Background())
		go func() {
			time.Sleep(2 * time.Second)
			cancel() // 在调用处主动取消
		}()
		res := Add(ctx, 1, 2)
		fmt.Printf("Compute: %d+%d, result: %d\n", a, b, res)
	}
}
```

## 最后，最后，最后，但不能相爱

context 包其实是很自然的一种解决方案。问题来自于 Goroutine 的并发模型，Goroutine 是没有父子关系的。对于有父子关系的进程模型来说，结束一个进程树的操作是十分自然的，但是在 Go 中，它们只能由程序员主动设置通信方案和父子关系算法，才能模拟出有父子关系的进程模型。context 包就是 Go 中模拟进程父子关系的成熟方案。

所以 context 这个词，直译是“上下文”，应该就是进程树中的上下文环境的意思，子进程能访问父进程的 Value map。

context 包现在越来越通用了，Go 1.6 http 包中的 cancel 方案，已经不推荐使用了。context 包里封装了一份 http 的包，使用的是 context 的解决方案。未来应该会纳入标准库。

更多的 context 的内容，可以直接读 context 源码。如果得到你的反馈，我会很开心 =)。