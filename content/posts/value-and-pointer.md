---
title: 值与指针
date: 2023-04-15T14:11:51+08:00
lastmod: 2023-04-15T14:11:51+08:00
cover: https://img.sysummery.top/go-pointer.png
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - 后端
tags:
  - go
---

写这篇文章的目的是记录一下平时在使用golang的过程中对于`指针`和`值`这两个概念的理解。虽然golang是`值传递`，但是在使用的过程中还是会有需要注意的点。

<!--more-->

### 结构体的方法定义与结构体的初始化
2个问题
1. 给结构体定义方法的时候是使用指针还是使用值
2. 使用结构体的时候是初始化指针还是值

这2个问题经过叉乘就变成了4个问题
1. 使用值定义的结构体方法，初始化的时候也是使用的值
2. 使用值定义的结构体方法，初始化的时候使用的是指针
3. 使用指针定义的结构体方法，初始化的时候也是使用的指针
4. 使用指针定义的结构体方法，初始化的时候使用的是值

我们一个个看

#### 使用值定义的结构体方法，初始化的时候也是使用的值
```go
type Test1 struct {
	Val int
}

func (t Test1) Set(new int) {
	t.Val = new
}

func main() {
	test := Test1{Val: 1}
	test.Set(10)
	fmt.Println(test.Val)
}
```
结果
```
1
```
这种情况比较好理解，在调用`Set`的时候相当于把一个`Test1`的副本传给了函数，因此修改只在函数内生效，在函数外不生效。

#### 使用值定义的结构体方法，初始化的时候使用的是指针
```go
type Test2 struct {
	Val int
}

func (t Test2) Set(new int) {
	t.Val = new
}

func main() {
	test := &Test2{Val: 1}
	test.Set(10)
	fmt.Println(test.Val)
}
```
结果
```
1
```
从结果来看，调用`Set`的时候相当于把一个`Test2对应的结构体`的副本传给了函数，而不是把`Test2`这个指针的副本，为了验证这个猜想，我们修改一下代码，在`main`和`Set`里面打印出结构体的地址
```go
type Test2 struct {
	Val int
}

func (t Test2) Set(new int) {
	fmt.Printf("in set %p\n", &t)
	t.Val = new
}

func main() {
	test := &Test2{Val: 1}
	fmt.Printf("in main %p\n", &(*test))
	test.Set(10)
}

```
结果
```
in main 0xc0000120b8
in set 0xc0000120c8
```
我们的猜想是对的，虽然在外面定义的是指针，但是定义方法的时候使用的是值，**在调用的时候会复制指针对应的值复制给方法，而不是复制指针自身**

#### 使用指针定义的结构体方法，初始化的时候也是使用的指针
```go
package main

import "fmt"

type Test3 struct {
	Val int
}

func (t *Test3) Set(new int) {
	t.Val = new
}

func main() {
	test := &Test3{Val: 1}
	test.Set(10)
	fmt.Println(test.Val)
}
```
结果
```
10
```
但是有个疑问，传递给函数的是指针还是指针的副本呢？我觉得应该是指针的副本，还是验证一下把
```go
package main

import "fmt"

type Test3 struct {
	Val int
}

func (t *Test3) Set(new int) {
	fmt.Printf("in set %p\n", t)
	t.Val = new
}

func main() {
	test := &Test3{Val: 1}
	fmt.Printf("in main %p\n", test)
	test.Set(10)
}
```
```
in main 0xc000090020
in set 0xc000090020
```
我想当然了，并不会复制，而是直接使用的原指针。想想也合理，如果在这个地方复制一个指针没有任何意义。。。

#### 使用指针定义的结构体方法，初始化的时候使用的是值
```go
package main

import "fmt"

type Test4 struct {
	Val int
}

func (t *Test4) Set(new int) {
	t.Val = new
}

func main() {
	test := Test4{Val: 1}
	test.Set(10)
	fmt.Println(test.Val)
}
```
结果
```
10
```
之前有看过别人的blog，说是这种情况会`panic`，但是我得到的结果是修改成功了，可能是`go`的版本不一致导致的把，我使用的是`1.20版本`。从结果看函数获取的应该是外面结构体的地址，我们修改代码来验证一下
```go
package main

import "fmt"

type Test4 struct {
	Val int
}

func (t *Test4) Set(new int) {
	fmt.Printf("in set %p\n", t)
	t.Val = new
}

func main() {
	test := Test4{Val: 1}
	fmt.Printf("in main %p\n", &test)
	test.Set(10)
}
```
结果
```
in main 0xc000094020
in set 0xc000094020
```
证明我们的猜想是对的，这种情况会把结构体的地址传递给函数。

我们可以得出结论是：对于结构体内部的数据修改是否成功取决于定义函数的方式是使用的指针还是值，使用指针定义才会修改成功，与初始化结构体的方式没有关系。初始化的方式决定了下层函数是否与当前函数使用的是同一个数值，我们再看个例子
```go
package main

import (
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	wg.Add(2)

	for i := 0; i < 2; i++ {
		go func(wg sync.WaitGroup) {
			defer wg.Done()
		}(wg)
	}

	wg.Wait()
}
```
结果
```
fatal error: all goroutines are asleep - deadlock!
```
因为初始化是使用的是值，所以传递给协程的是值得副本，`wg.Add(2)`与`wg.Done()`这两个wg是不一样的。要想改很简单，改成指针就好了
```go
package main

import (
	"sync"
)

func main() {
	wg := &sync.WaitGroup{}
	wg.Add(2)

	for i := 0; i < 2; i++ {
		go func(wg *sync.WaitGroup) {
			defer wg.Done()
		}(wg)
	}

	wg.Wait()
}
```
或者干脆作为闭包使用外面的公共变量
```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(2)

	for i := 0; i < 2; i++ {
		go func() {
			defer wg.Done()
		}()
	}

	wg.Wait()
}
```
也可以使用指针的方式初始化`wg := &sync.WaitGroup{}`， 因为`sync.WaitGroup`的所有方法都是使用指针定义的。

### 闭包的引用
一个函数内引用了外部的局部变量，这种现象，就称之为闭包。

```go
package main

import "fmt"

func toSum() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	f := toSum()
	fmt.Println(f(2))
	fmt.Println(f(2))
}
```
结果
```go
2
4
```
首先编译器肯定会把`sum`这个变量分配到堆上。直观上来看`sum`是个具体的值，每一次调用闭包函数，`sum`的值都应该是0。但是从运行结果看，闭包对外部变量的引用存的是这个变量的`指针`而不是`值`。我们修改代码验证一下
```go
package main

import "fmt"

func toSum() func(int) int {
	sum := 0
	return func(x int) int {
		fmt.Printf("%p\n", &sum)
		sum += x
		return sum
	}
}

func main() {
	f := toSum()
	f(2)
	f(2)
}
```
结果
```
0xc000090020
0xc000090020
```
验证了猜想，闭包对外部的变量是通过指针引用的方式建立关系的，即使变量定义的时候是个值

### defer的引用
其实与上面的闭包的引用很相似，只不过之前在开发的时候在这里遇到过问题所以想记录一下，上代码
```go
package main

import "fmt"

func toSum() int {
	sum := 0
	defer fmt.Println(sum)

	sum = 100
	return sum
}

func main() {
	toSum()
}
```
结果
```
0
```
如果改成
```go
package main

import "fmt"

func toSum() int {
	sum := 0
	defer func() {
		fmt.Println(sum)
	}()

	sum = 100
	return sum
}

func main() {
	toSum()
}
```
结果
```
100
```
以为第一个例子是一个语句，所以在执行`defer fmt.Println(sum)`的时候直接复制的sum的值给`fmt.Println`，当时值是0，因此最终打印的也是0。第二个例子是使用了闭包，其内部保留的是对sum的应用，也就是存放的是指针，因此最终打印的是100。
