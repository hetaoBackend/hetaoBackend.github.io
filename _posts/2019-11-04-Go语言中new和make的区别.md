---
layout:     post
title:      Golang中的概念
subtitle:   new和make的区别
date:       2019-11-04
author:     Walnut
header-img: img/post-bg-keyboard.jpg
catalog: true
tags:
    - Go
---

# Go语言中new和make的区别

Go语言中new和make是内建的两个函数，主要用来创建分配类型内存。在我们定义生成变量的时候，可能会觉得有点迷惑，其实他们的规则很简单，下面我们就通过一个示例说明他们的区别和使用。

## 变量的声明

```go

var i int
var s string

```

变量的声明我们可以通过`var`关键字，然后就可以在程序中使用。当我们不指定变量的默认值时，这些变量的默认值是他们的零值，比如`int`类型的零值是0，`string`类型的零值是`""`，引用类型的零值是`nil`。

对于例子中的两种类型的声明，我们可以直接使用，对其进行赋值输出。但是如果我们换成引用类型呢？

```go

package main

import (
    "fmt"
)

func main() {
    var i *int
    *i = 10
    fmt.Println(*i)
}

```

这个例子会打印出什么？`0`还是`10`？以上全错，运行的时候会panic，原因如下：
```text
panic: runtime error: invalid memory address or nil pointer dereference
```

从这个提示中可以看出，对于引用类型的变量，我们不光要声明它，还要为它分配内存空间，否则我们的值放到哪里去呢？这就是上面错误提示的原因。

对于值类型的声明不需要，是因为已经默认帮我们分配好了。

要分配内存，就引出今天的`new`和`make`。

## new

对于上面的问题我们如何解决呢？既然我们直到了没有为其分配内存，那么我们使用new分配一个吧。

```go

func main() {
    var i *int
    i = new(int)
    *i = 10
    fmt.Println(*i)
}

```

现在再运行程序，完美PASS，打印`10`。现在让我们看下`new`这个内置函数。

```go
// The new built-in function allocates memory. The first argument is a type,
// not a value, and the value returned is a pointer to a newly
// allocated zero value of that type.
func new(Type) *Type
```

它只接收一个参数，这个参数是一个类型，分配好内存后，返回一个指向该类型内存地址的指针。同时请注意它同时把分配的内存置为零，也就是类型的零值。

我们的例子中，如果没有`*i=10`，那么打印的就是0。这里体现不出来`new`函数这种内存置为零的好处，我们再看一个例子。

```go

func main(){
    u := new(user)
    u.lock.Lock()
    u.name = "张三"
    u.lock.Unlock()

    fmt.Println(u)

}

type user struct {
    lock sync.Mutex
    name string
    age int
}

```

示例中的`user`类型中的`lock`字段不用初始化，直接可以拿来用，不会有无效内存引用异常，因为它已经被零值了。

这就是`new`，它返回的永远是类型的指针，指向分配类型的内存地址。

## make

`make`也是用于内存分配的，但是和`new`不同，它只用于`chan`、`map`以及切片的内存创建，而且它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以没有必要返回他们的指针了。

注意，因为这三种类型是引用类型，所以必须得初始化，但是不是置为零值，这个和`new`是不一样的。

```go
func make(t Type, size ...IntegerType) Type
```
从函数声明中可以看到，返回的还是类型。

## 两者异同

从这里可以看的很明白了，两者都是内存的分配（堆上），但是`make`只用于slice、map以及channel的初始化（非零值）；而`new`用于类型的内存分配，并且内存置为零。所以我们编写程序的时候，就可以根据自己的需要做选择了。

`make`返回的还是这三个引用类型本身；而`new`返回的是指向类型的指针。

## 其实new不常用

`new`这个内置函数，可以给我们分配一块内存让我们使用，但是现实的编码中，它是不常用的。我们通常都是采用短语句声明以及结构体的字面量达到我们的目的，比如：

```go
i := 0
u := user{}
```

这样更简洁方便，而且不会涉及到指针这种比较麻烦的操作。

`make`函数是无可替代的，我们在使用slice、map以及channel的时候，还是要使用`make`进行初始化，然后才可以对他们进行操作。
