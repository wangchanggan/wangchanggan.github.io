---
layout:     post
title:      Golang defer、panic和recover
date:       2023-01-26
catalog: true
tags:
    - Go语言底层原理剖析
---

# defer
defer是 Go语言中的关键字，也是Go语言的重要特性之一。defer 的语法形式为 defer Expression，其后必须紧跟一个函数调用或者方法调用。在很多时候，defer后的函数以匿名函数或闭包的形式呈现，例如：
```
defer func(...){
//实际处理
}()
```
defer将其后的函数推迟到了其所在函数返回之前执行。例如在运行如下代码后，将首先打印出下方的"normal func"，接着打印出"defer func"。
```
func main() {
	defer fmt.Println("defer func")
	fmt.Println("normal func")
}
```
不管defer函数后的执行路径如何，最终都将被执行。在Go语言中，defer一般被用于资源的释放及异常panic的处理。

程序中可以有多个 defer、defer的调用可以存在于函数的任何位置等。defer可能不会被执行，例如，如果判断条件不成立则放置在if语句中的 defer可能不会被执行。
## 使用
### 资源释放
不管defer后面的执行路径如何，defer中的语句都将执行。在每个资源后都立即加入defer file.Close函数，保证函数在任意路径执行结束后都能够关闭资源。defer是一种优雅的关闭资源的方式，能减少大量元余的代码并避免由于忘记释放资源而产生的错误。
```
func CopyFile(dstName, srcName string) (written int64, err error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}
	defer src.Close()

	dst, err := os.Create(dstName)
	if err != nil {
		return
	}
	defer dst.Close()
	return io.Copy(dst, src)
}
```

### 异常捕获
defer的特性是无论后续函数的执行路径如何以及是否发生了panic，在函数结束后一定会得到执行，这为异常捕获提供了很好的时机。异常捕获通常结合recover函数一起使用。

如下所示，当在defer函数中使用recover进行异常捕获后，程序将不会异常退出，并且能够执行正常的函数流程。
```
func executePanic() {
	defer func() {
		if errMsg := recover(); errMsg != nil {
			fmt.Println(errMsg)
			fmt.Println("This is recovery function... ")
		}
	}()
	panic("This is Panic Situation")
	fmt.Println("The function executes Completely")
}

func main() {
	executePanic()
	fmt.Println("Main block is executed completely...")
}
```
如下输出表明，尽管有panic，main函数仍然在正常执行后退出。
```
This is Panic Situation
This is recovery function...
Main block is executed completely...
```

## 特性
### 延迟执行
defer后的函数并不会立即执行，而是推迟到了函数结束之后执行，这一特性一般用于资源的释放和异常捕获。
### 参数预计算
当函数到达defer语句时，延迟调用的参数将立即求值，传递到defer函数中的参数将预先被固定，而不会等到函数执行完成后再传递参数到defer中。

如下例所示，defer 后的函数需要传递int 参数，首先将a赋值为1，接着defer函数的参数传递为a+1，最后，在函数返回前a被赋值为99。那么最后defer函数打印出的b值是多少呢?答案是2。原因是传递到defer的参数是预执行的，因此在执行到defer语句时，执行了a+1并将其保留了起来，直到函数执行完成后才执行defer函数体内的语句。
```
func main() {
	a := 1
	defer func(b int) {
		fmt.Println("defer b", b)
	}(a + 1)
	a = 99
}
```

### defer多次执行与LIFO执行顺序
在函数体内部，可能出现多个defer函数。这些defer函数将按照后入先出(last-in-first-out，LIFO)的顺序执行，这与栈的执行顺序是相同的，这也意味着后申请的资源将会先得到释放。

## 返回值陷阱
```
func main() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```
程序输出结果如下：
```
1
```

```
func c() (i int) {
	defer func() {
		i++
	}()
    // i = 1
	return 1
}

func main() {
	fmt.Print(c())
}
```
程序输出结果如下：
```
2
```


```
var g = 100

func f() (r int) {
	defer func() {
		g = 200
	}()
	fmt.Printf("f: g = %d\n", g)
    // r = g
	return g
}

func main() {
	i := f()
	fmt.Printf("main: i = %d, g = %d\n", i, g)
}
```
程序输出结果如下：
```
f: g = 100
main: i = 100, g = 200
```
从输出结果可以推测出，在return之后，执行了defer函数。
```
var g = 100

func f() (r int) {
	r = g
	defer func() {
		r = 200
	}()
	r = 0
	return r
}

func main() {
	i := f()
	fmt.Printf("main: i = %d, g = %d\n", i, g)
}
```
程序输出结果如下：
```
main: i = 200, g = 100
```
总结：将返回值保存在栈上->执行defer函数->函数返回。
# panic
```
func panic(interface{})
```
panic函数传递的参数为空接口interface{}，其可以存储任何形式的错误信息并进行传递。在异常退出时会打印出来。

Go程序在panic时并不会像大多数人想象的一样导致程序异常退出，而是会终止当前函数的正常执行，执行defer 函数并逐级返回。例如，对于函数调用链a()->b()->c()，当函数c发生panic后，会返回函数b。此时，函数b也像发生了panic一样，返回函数a。在函数c、b、a中的defer函数都将正常执行。
```
func a() {
	defer fmt.Println("defer a")
	b()
	fmt.Println("after a")
}

func b() {
	defer fmt.Println("defer b")
	c()
	fmt.Println("after b")
}

func c() {
	defer fmt.Println("defer c")
	panic("this is a panic")
	fmt.Println("after c")
}

func main() {
	a()
}
```
如下所示，当函数c触发了panic后，所有函数中的defer语句都将被正常调用，并且在panic
时打印出堆栈信息。
```
defer c
defer b
defer a
panic: this is a panic
goroutine 1 [running]:
main.c()
```

```
func a() {
	defer b()
	panic("a panic")
}

func b() {
	defer c()
	panic("b panic")
}

func c() {
	panic("c panic")
}

func main() {
	a()
}
```
程序输出结果如下：
```
panic: a panic
panic: b panic
panic: c panic
```
先打印最早出现的panic，在打印其他的panic。
# recover
```
func recover() interface{}
```
为了让程序在panic时仍然能够执行后续的流程，Go语言提供了内置的recover函数用于异常恢复。recover函数一般与defer函数结合使用才有意义，其返回值是panic中传递的参数。由于panic会调用defer函数，因此，在defer函数中可以加入rover起到让函数恢复正常执行的作用。
```
func a() {
	defer fmt.Println("defer a")
	b()
	fmt.Println("after a")
}

func b() {
	defer func() {
		fmt.Println("defer after b")
		if x := recover(); x != nil {
			fmt.Printf("run time panic: %v\n", x)
		}
	}()
	c()
	fmt.Println("after b")
}

func c() {
	defer fmt.Println("defer c")
	panic("this is a panic")
	fmt.Println("after c")
}

func main() {
	a()
}
```
程序输出结果如下：
```
defer c
defer after b
run time panic: this is a panic
after a
defer a
```

```
func a() {
	defer b()
	panic("a panic")
}

func b() {
	defer c()
	panic("b panic")
}

func c() {
	panic("c panic")
}

func catch(funcname string) {
	if r := recover(); r != nil {
		fmt.Println(funcname, "recover:", r)
	}
}

func main() {
	defer catch("main")
	a()
}
```

程序输出结果如下：
```
main recover: c panic
```

recover函数最终捕获的是最近发生的panic，即便有多个panic函数，在最上层的函数也只需要一个recover函数就能让函数按照正常的流程执行。

```
func a() {
	defer b()
	panic("a panic")
}

func b() {
	defer c()
	panic("b panic")
}

func c() {
	defer catch("c")
	panic("c panic")
}

func catch(funcname string) {
	if r := recover(); r != nil {
		fmt.Println(funcname, "recover:", r)
	}
}

func main() {
	a()
}
```
程序输出结果如下：
```
c recover: c panic
panic: a panic
panic: b panic
goroutine 1 [running]:
main.b()
```
内部的recover只能捕获由当前函数或其子函数触发的panic，而不能触发上层的panic。