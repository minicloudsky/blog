---
title: 深入解析Golang异常机制：Panic 与 Recover 的原理与最佳实践
date: 2024-12-28 20:53:21
categories:
- golang
tags:
- golang


---

# 一、前言

在 Go 中，`panic` 是一种用于处理异常的机制。`panic` 能够改变程序的控制流，调用 `panic` 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 `defer`；当程序遇到不可恢复的错误或异常状态时，可以通过调用 `panic` 来中止当前函数的执行，触发递归的函数栈退出，最终程序崩溃并打印错误信息。

`panic` 常用于以下场景：

- 程序进入了不可能出现的逻辑分支。
- 遇到无法继续运行的致命错误，例如读取配置文件失败、数据库连接失败等。
- 需要强制中止程序运行并提供调试信息。

```go
// Bind is a helper function for given interface object and returns a Gin middleware.
func Bind(val any) HandlerFunc {
	value := reflect.ValueOf(val)
	if value.Kind() == reflect.Ptr {
		panic(`Bind struct can not be a pointer. Example:
	Use: gin.Bind(Struct{}) instead of gin.Bind(&Struct{})
`)
	}
	typ := value.Type()

	return func(c *Context) {
		obj := reflect.New(typ).Interface()
		if c.Bind(obj) == nil {
			c.Set(BindKey, obj)
		}
	}
}
```

`recover` 是 Go 提供的另一种异常处理机制，用于从 `panic` 中恢复程序执行。它通常与 `defer` 一起使用，可以捕获 `panic` 的信息，从而避免程序直接崩溃，它是一个只能在 `defer` 中发挥作用的函数，在其他作用域中调用不会发挥作用。

```go
package main

import (
	"log"
)

func main() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("Recovered from panic: %v", r)
		}
	}()

	log.Println("Before panic")
	panic("An error occurred")
	log.Println("Unreachable code  after panic.")
}
```

通过 `recover`，程序能够在 `panic` 后继续运行。

# 二、现象

我们先通过下面几个例子了解一下使用 `panic` 和 `recover` 时会遇到的现象：

- `panic` 只会触发当前 Goroutine 的 `defer`；
- `recover` 只有在 `defer` 中调用才会生效；
- `panic` 允许在 `defer` 中嵌套多次调用；

## 跨协程失效

`panic` 只会触发当前 Goroutine 的延迟函数调用

```go
package main

import "time"

func main() {
	defer println("running in main func")
	go func() {
		defer println("running in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

// 运行结果
panic: 

goroutine 17 [running]:
main.main.func1()
        /main.go:9 +0x3e
created by main.main in goroutine 1
        /main.go:7 +0x37
```

当我们运行这段代码时，会发现 `main` 函数中的 `defer` 语句并没有执行，执行的只有当前 Goroutine 中的 `defer`，多个 Goroutine 之间是没有关联的，一个 Goroutine 在 `panic` 时，不应该执行其他 Goroutine 的延迟函数。

## 崩溃恢复失效

`recover` 只有在发生 `panic` 之后调用才会生效，下面的 recover ` 是在 `panic` 之前调用的，并不满足生效的条件，所以我们需要在 `defer` 中使用 ` recover` 关键字

```go
package main

import (
	"fmt"
)

func main() {
	defer fmt.Println("running in main func")
	if err := recover(); err != nil {
		fmt.Println(err)
	}

	panic("unknown err happened")
}

// 执行结果
running in main func
panic: unknown err happened

goroutine 1 [running]:
main.main()
        /main.go:13 +0x85
```

## 嵌套崩溃

Go 语言中的 `panic` 是可以多次嵌套调用的，下面的代码展示了如何在 `defer` 函数中多次调用 `panic`：

```go
package main

import (
	"fmt"
)

func main() {
	defer fmt.Println("running in main")
	defer func() {
		defer func() {
			panic("panic inside defer defer")
		}()
		panic("panic inside defer")
	}()

	panic("panic in main")
}

// 执行结果
panic: panic in main
        panic: panic inside defer
        panic: panic inside defer defer

goroutine 1 [running]:
main.main.func1.1()
        /main.go:11 +0x25
panic({0x108e5a0?, 0x10c0f00?})
        /Users/admin/go/go1.21.1/src/runtime/panic.go:914 +0x21f
main.main.func1()
        /main.go:13 +0x3e
panic({0x108e5a0?, 0x10c0ee0?})
        /Users/admin/go/go1.21.1/src/runtime/panic.go:914 +0x21f
main.main()
        /main.go:16 +0x78
```

从上面程序输出的结果，我们可以确定程序多次调用 `panic` 也不会影响 `defer` 函数的正常执行，所以使用 `defer` 进行收尾工作一般都是安全的。

# 三、执行流程

#### `panic` 内部工作机制

- `panic` 会向调用栈上传递一个值，这个值通常是一个字符串或 `error` 类型，表示异常的原因。
- 运行时会记录当前调用栈的信息（类似堆栈跟踪），并将其打印到标准错误输出中。
- 如果一个函数中没有被 `recover` 捕获到的 `panic`，则继续向上层函数传播，直至程序退出。

#### `recover` 的执行流程

1. **捕获 `panic` 信息**：当调用 `recover` 时，如果当前调用栈处于 `panic` 状态，`recover` 会返回传递给 `panic` 的值。
2. **停止调用栈回溯**：调用 `recover` 后，Go 运行时会停止继续回溯调用栈，程序从捕获 `panic` 的位置继续执行。
3. **正常退出函数**：通过 `recover` 成功捕获 `panic` 后，程序会正常退出当前函数。

![](http://img.minicloudsky.cn/imgs/panic.png)



# 四、常见的panic场景

* **数组越界**：访问不存在的数组索引。

  ```go
  userIds := []int{1, 2, 3}
  fmt.Println(userIds[5]) // panic: runtime error: index out of range [5] with length 3
  ```

  在访问数组或切片时，需要进行边界检查，确保索引合法。

* **空指针引用**：对未初始化的指针或接口调用方法。

  ```go
  var p *int
  fmt.Println(*p) // panic: runtime error: invalid memory address or nil pointer dereference
  ```

​		在使用指针或接口前，请检查指针或接口是否为 `nil`

* **显式调用 `panic`**：当程序出现无法继续运行的致命错误时，手动调用 `panic`。

  ```go
  package main
  
  func main() {
  	panic("panic happened")
  }
  ```

* **类型断言失败**

  ```go
  var i interface{} = "hello"
  fmt.Println(i.(int)) // panic: interface conversion: interface {} is string, not int
  ```

​		使用类型断言时，可以通过 `comma-ok` 语法避免 `panic`。

        ```go
        var i interface{} = "hello"
        	if v, ok := i.(int); ok {
        		fmt.Println(v)
        	} else {
        		fmt.Println("Type assertion failed")
        	}
        ```

# 五、总结

`panic` 和 `recover` 是 Go 中强大的异常处理工具，在使用时需要谨慎，`recover`通常只在以下场景下适合:

* 在 Web 服务或者守护进程中，使用 `recover` 防止单个请求的异常导致整个服务崩溃。
* 在程序崩溃前释放资源，例如关闭文件或释放锁。
* 在程序崩溃前，记录详细日志便于排查问题。

不推荐在下面的场景中滥用 `recover`:

- 代替正常的错误处理：优先考虑返回 `error`,而不是用 `recover` 兜底。
- 在非必要情况下捕获所有异常：这可能会掩盖问题的根本原因，导致问题更难排查定位。

在Go程序开发中，推荐优先考虑 Go 的惯用错误处理方式（`error`）来设计更健壮的代码，同时在发生意外时尽量减少影响。
