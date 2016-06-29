# 关于 Golang 的一些坑和说明

这里记录 Golang 学习过程中的一些意想不到的特点，如果不注意，按照平时其它语言的习惯来，很可能会踩坑。

## 堆、栈
- Golang 不区分堆空间和栈空间，希望程序员不需要关心这些东西。于是：
  - Golang 的栈空间是可以自动扩容的，不用担心爆栈了。但是经过测试，我本机使用了 1GB 的栈空间的时候，爆炸了。（毕竟要照顾像我这样容易写出无限递归的程序员）
  - Golang 的栈空间是引用安全的。引用来自一个函数内部的变量，函数退出之后，继续引用，也不会产生错误。因为 Golang 发现这个变量被拿走地址，就会把这个变量放在堆里。

## 指针
- Golang 的指针共享实体的成员函数，所以没有 -> 操作符，统一使用 .（点）操作符。
  - 但是这只包括实体和一级引用，涉及到更高级别的引用，仍然需要 explicit dereference。
  - 见下面的代码：
```go
package main

import "fmt"

type T struct {
	i int
}

func (t T) f() {
	fmt.Println(t.i)
}
func (t *T) g() {
	fmt.Println(t.i)
}
func main() {
	t := T{1}
	tt := &t
	ttt := &tt
	tttt := &ttt
	fmt.Println(t, tt, ttt, tttt)
	fmt.Println(t.i, tt.i) // compile error: ttt.i, tttt.i
	t.f()
	tt.f()
	// compile error: ttt.f(), tttt.f()
	t.g()
	tt.g()
	// compile error: ttt.g(), tttt.g()
}
```

- Value Type 和 Ref Type
  - Golang 同时有 Value Type 和 Ref Type。
  - Value Type 可以直接放在栈空间里，新建和回收的效率更高；Ref Type 的实体放在堆上，拿到的是指针，新建和回收的代价更大。
  - Value Type 进行变量赋值的时候，需要完全拷贝一份，而 Ref Type 只需要拷贝一个指针。
  - 所以可以发现，对 Value Type 的修改，不影响它的副本，而 Ref Type 的所有副本指向同一个实体，永远相同。
  - Value Type 包括：struct Type
  - Ref Type 包括：[]Type、map Type、chan Type
  - 不可变类型无需关心

-  下面的代码包含一处运行时错误
```go
package main

import "fmt"

func main() {
	const n = 10
	ch := [n]chan int{}
	for i := 0; i < n; i++ {
		if i == 0 {
			ch[i] = make(chan int)
		} else {
			ch[i] = ch[i-1]
		}
		go func() {
			ch[i] <- i
		}()
	}
	v := [n]int{}
	for i := 0; i < n; i++ {
		v[i] = <-ch[i]
	}
	fmt.Println(v)
}
```

## 并发控制

- 阅读列表
  - [Go并发模式：管道和取消](https://segmentfault.com/a/1190000000437463)
	- [goroutine背后的系统知识](http://www.infoq.com/cn/articles/knowledge-behind-goroutine)

- `chan Type`
  - `chan Type` 是可读可写的，`<-chan Type` 是只读的，`chan<- Type` 是只写的。
	- `make(chan bool, 3)` 带有长度为 3 的缓冲区，读未准备好时，可以一直写，直到缓冲区满；写为准备好时，读会阻塞。
	- `make(chan bool)` 没有缓冲区，读和写必须都准备好，否则就被阻塞。
	- Close 后的 chan 也是可以读的，读出的是默认值。

- Goroutine
  - Go 的 main 线程结束的时候，即便有仍在 block 的其它线程，也会一并都退出。
	- Goroutine 最终会落到运行时的线程池中。
