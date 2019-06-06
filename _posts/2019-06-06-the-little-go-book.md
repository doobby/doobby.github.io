---
title: The little Go Book 笔记 
layout: post
tags: code
---


目录
====

-   [前言](#前言)
-   [代码结构](#代码结构)
-   [Basic](#basic)
-   [Structure](#structure)
-   [Maps, Arrays and Slices](#maps-arrays-and-slices)
-   [Code Organization and
    Interfaces](#code-organization-and-interfaces)
-   [Tidbits](#tidbits)
-   [Concurrency](#concurrency)

前言
====

本笔记整理自 [The Little Go
Book](https://www.openmymind.net/The-Little-Go-Book/) 一书。原始笔记在 http://blog.doobby.org/diary/programming/golang/the-little-go-book.html ，更新不会同步到本文。

为什么不愿意学习新的语言
------------------------

1.  语言是开发的基础，语言特性影响设计模式乃至上层架构
2.  语言细节多，学习涉及方方面面，付出与收效比例不相当

为什么要学习 go
---------------

-   语法和标准库相对简单
-   社区活跃
-   静态语言
-   适合于系统级或大型应用级别的开发

代码结构
========

-   `${GOPATH}` 环境变量定义 workspace
-   workspace 下保存目录代码和生成结果，如 `src`, `bin`, `pkg` 等

Basic
=====

特点
----

### 编译型

-   编译速度快，能更快迭代
-   运行速度相对解释型快
-   无依赖

### 静态类型

-   可以通过手动指定类型，或由编译器推导类型
-   编译时可以检查出很多错误，增强可靠性

### 类 C 语法

-   去掉了 semi-colon terminated
-   去掉了 parenthese around conditions
-   在表示优先级时可以使用括号

### GC

-   keep track of values
-   and free them when they're no longer used.

running
-------

-   使用 go run 来执行
-   使用 go build 来编译，生成可执行文件
-   入口是 main 包的 main 函数
-   非 main 包不能执行
-   非 main 包能 build，但不会生成可执行文件
-   build 生成的可执行文件与源码同名

Import
------

-   有一些默认的函数不需要导入，如 `len`, `cap`, `println`
-   `import` 用于导入指定字符串对应的包，可以一次导入多个
-   不允许未被使用的包的导入（会降低编译速度）
-   使用 godoc 来查看包及函数、类型的使用手册

变量与申明
----------

-   使用 `var` 定义，类型在后
-   或者使用 `:=` 来定义局部变量，类型由值推导而来
-   默认会赋予 zero value
-   可以多变量赋值
-   不允许存在未被使用的变量

### 注意

-   `:=` 会阻止重复定义
-   但只要左值有一个是新的变量， `:=` 将允许赋值

函数声明
--------

-   可以返回多个值
-   使用 blank identifier ( "\_" ) 来忽略赋值
-   相同参数类型时，可以简写类型到最后（ shorter syntax ）
-   Named return values

Structure
=========

Overview
--------

-   Go 不支持继承 (inheritance), 不支持多态 (polymorphism) 和重载
    (overloading)
-   Go 提供 composition 功能
-   可以为结构体添加方法

声明与初始化
------------

-   `Struct{Key: value, Key2: value2,}` 构造结构体
-   也可以忽略键名，顺序传入参数构造结构体: `Struct{value, value2}`
-   Go 默认以拷贝形式传参，通过指针避免拷贝结构体
-   使用 `&` 符号来通变量值的地址，使用 `*` 来取指针指向的对象
-   指针拷贝开销低，并且拷贝指针指向的内容还是原地址
-   指针是一个内存地址，指向实际变量值

方法
----

-   `func(s *Struct) Func() {...}` 为结构体定义一个方法
-   其中 `*Struct` 被称为 `Func` 的接收者 (receiver)

构造器
------

-   Go 中结构体没有构造器
-   实现一个普通函数来生成并返回结构体即可

new 方法
--------

-   `new(X)` 等价于 `&X{}`
-   更习惯于使用后者

结构体成员
----------

-   结构体成员称为 field
-   可以是任何类型：结构体，array, map, interface 或者 function

Composition
-----------

-   将一个结构体包含于另一个
-   也称为 trait 或者 mixin
-   避免手工封闭子结构体及其方法
-   外部结构体可以复写(overwrite) 内部结构体的方法
-   Composition 较之 inheritance 更健壮
-   inheritance 方式下我们将更多关注继承关系，而非子类的行为

指针
----

-   时刻关注应该用“值”还是“指针”
-   值传递更安全，开销也更大

Maps, Arrays and Slices
=======================

Arrays
------

-   固定长度，申明时须指定长度
-   下标从 0 开始，越界访问会报错
-   使用 `len` 获取长度
-   通过 `for index, value := range scores` 来遍历

Slices
------

-   Go 中很少直接使用 array，而是使用 slices
-   Slice 是基于 Array 的一个轻量封装
-   可以用 `[]int{}` 或 `make([]int, 10)` 来构造
-   `make` 不同于 `new` ，后者仅建立了对象，而前者还要申请底层的数组空间
-   slices 有两个长度： `cap` 表示底层数组的容量， `len` 表示 slice
    的实际长度
-   `make([]int, length, capacity)` 可以同时指定 len 和 cap
-   访问 len 范围之外的元素用报错
-   `append` 方法可以安全的在 slice 中添加元素，必要时会申请新的 array
    并将数据拷贝过去
-   `scores[0:8]` 来 re-slice，重新定义 slice 的 length，容量不变
-   Ruby 或者 javascript 中，slice 方法将生成一个新的数组（拷贝出）;
    但在 Go 中 Slice 共享底层数据
-   不同于 Python，Go 不支持负索引
-   `copy(dst, src)` 函数复制 slice 的数据，注意层级数据的覆盖

Maps
----

-   使用 `make(map[string]int)` 或 `map[string]int{}` 构造
-   `len` 求得内容数量
-   `delete(m, key)` 删除键值对
-   可以指定初始容量 `make(map[string]int, 100)`
-   用 `for key, value := range lookup` 遍历所有键值对，不应假设遍历顺序

值类型
------

-   slices 是引用类型
-   map 是引用类型

Code Organization and Interfaces
================================

Packages
--------

-   通常包名与目录名一致，使用 `package` 关键字申明
-   导入时使用完整路径，使用 `import` 关键字导入
-   可执行命令通常放在 `main` 子目录中， `main.go` 文件中定义 `main`
    方法

### 循环依赖 (Cyclical Imports)

-   不允许两个 package 相互引用: *import cycle not allowed*
-   可以抽象出一个公用 package 为两者导入使用

### 可见性

-   首字母小写包外不可见
-   没有文件级的访问控制

### 包管理

-   `go get` 命令，支持多种协议，包括 github
-   同时会下载依赖
-   `-u` 参数更新包

Interfaces
----------

-   定义接口，没有具体实现
-   用于解耦具体实现与标准接口，可以用于避免循环导入
-   不同于 C\# 或者 Java，实现接口时不需要显式地声明
-   满足接口的实现将自动被认为实现了该接口
-   习惯将接口设计的比较简单、单一、少量
-   接口也支持 composition，可以与其它接口混合，例如 `io.ReadCloser`
    混合了 `io.Reader` 和 `io.Closer`

Tidbits
=======

Error handling
--------------

-   Go 通过返回 error 值来表示错误
-   error 接口中只有一个方法 `Error() string` ，可以自定义 Error 类型
-   `errors.New()` 函数生成一个新的 Error 对象
-   Go 提供了 `panic` 和 `recover`
    函数，前者抛出异常，后者捕获异常。但很少用

Defer
-----

-   Go 带有 GC （垃圾回收），但有时我们需要显式地关闭资源（如关闭文件）
-   `defer`
    语句保证在函数结束前被调用，在有多返回点时可以避免忘记显式关闭资源
-   可以用于结束时打印函数用时

go fmt
------

-   `go fmt` 格式化代码成标准格式
-   `go fmt ./...` 来格式化当前所有的包

Initialized if
--------------

-   Go 的 `if` 语句支持变量构造: `if err := process(); err != nil {...}`
-   构造的变量可以在配对的 `else` 和 `else if` 访问，但不能在外部访问

Empty Interface and Conversions
-------------------------------

-   Go 没有继承，也就不存在 `object` 对象做为所有对象的基类对象
-   所有的对象都可以被转换为 `interface{}` 类型
-   通过 type assert 来转换类型，例如 `a.(int)`
-   switch 语句支持类型分支，如
    `switch a.(type) {case int: ...; default: ...}`
-   谨慎使用 `interface{}`
    来表示抽象类型，丢失类型对静态语言不是个好事情

Strings and Byte Arrays
-----------------------

-   字符串类型和 byte 数组可以相互转换： `[]byte(s)`, `string(byts)`
-   Go 的字符串类型是不可变类型， `[]byte(s)` 会产生数据拷贝
-   `len(s)` 表示字节数，而非字符 (rune) 数
-   对 string 类型进行 for 遍历时，访问的是 rune 元素

Function Type
-------------

-   类似 C 的函数指针，定义一个函数 `type Add func(a int, b int) int`
-   函数类型是第一类型，可以做为其它函数的参数或返回值
-   如同 interface, 可以用于解耦函数定义和函数实现

Concurrency
===========

Goroutines
----------

-   类似于线程 (thread)，但由 Go 核心管理，而非操作系统
-   开销较线程小，可以开很多个。多个 goroutines 可以共享于同一个 thread
    中
-   并发 (concurrency) 而非并行：有独立的上下文，但不保证同时被调度
-   `go` 关键字使后面的函数调用在新的 goroutine 中执行

### Synchronization

-   主协程退出将导致其它协程一并退出
-   协程间访问共享数据可能会出现抢占错误，需要考虑对共享数据进行加锁保护
    (`sync.Mutex`)
-   加锁可能降低性能，可能造成死锁 (deadlock)
-   `sync.RWMutex` 实现读写锁

Channels
--------

-   erlang 的思路，消息传递方式实现多协程间的通信，避免在协程间共享数据
-   channel 用于在 goroutines 之间传递数据，同一时刻只能有一个 goroutine
    来访问 channel
-   channel 指定传输数据的类型， `make(chan int)` 创建一个 int 类型的
    channel，其类型为 `chan int`
-   `ch <- data` 写数据，channel 没有容量写数据，将导致写 channel 的
    goroutine 被阻塞
-   `v := <-ch` 读数据，channel 中没有数据，将导致读 channel 的
    goroutine 被阻塞
-   创建时可以指定 channel 的容量 `make(chan int, 100)`
-   `len(ch)` 可以检查 channel 中的元素数量

### select

-   语法类似于 `switch` , `default` 入口用于处理 channel
    资源不可得的情况
-   通常用于同时监听多个 channel，随机选中可以访问的 channel
-   没有 `default` 分支，则在无数据就绪时将被阻塞

### timeout

``` {.go}
for {
        select {
        case c <- rand.Int():
        case <-time.After(time.Millisecond * 100):
                fmt.Println("timed out")
        }
        time.Sleep(time.Millisecond * 50)
}
```

-   `time.After()` 创建一个 channel，并在 timeout 时间后向其中传递一个值
-   若在指定时间没有其它数据到达， `select` 将进入超时处理流程
