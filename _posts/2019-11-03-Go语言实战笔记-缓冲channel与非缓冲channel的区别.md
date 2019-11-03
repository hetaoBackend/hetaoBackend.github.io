---
layout:     post
title:      Go语言实战笔记
subtitle:   Go缓冲channel和非缓冲channel的区别
date:       2019-11-03
author:     Walnut
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Go
    - Go In Action
---

# Go缓冲channel和非缓冲channel的区别

在看本篇文章之前，我们需要了解阻塞的概念。

- 在执行过程中暂停，以等待某个条件的触发，我们就称之为**阻塞**

## channel类型

在Go中我们make一个channel有两种方式，分别是有缓冲的和没缓冲的
- 缓冲`channel`，即`buffer channel`创建方式为`make(chan TYPE, SIZE)`
    - 如`make(chan int, 3)`就是创建一个`int`类型，缓冲大小为3的channel

- 非缓冲`channel`即`unbuffer channel`创建方式为`make(chan TYPE)`
    - 如`make(chan int)`就是创建一个`int`类型的非缓冲`channel`

## 两种channel的区别

- 非缓冲`channel`，`channel`发送和接收动作是同时发生的
- 例如`ch := make(chan int)`，如果没`goroutine`读取接收者`<-ch`，那么发送者`ch<-`就会一直阻塞
- 缓冲`channel`类似一个队列，只有队列满了才可能发送阻塞

## 代码演示

### 非缓冲channel

```go

package main

import (
    "fmt"
    "time"
)

func loop(ch chan int) {
    for {
        select {
            case i := <-ch:
                fmt.Println("this value of unbuffer channel", i)
        }
    }
}

func main() {
    ch := make(chan int)
    ch <- 1
    go loop(ch)
    time.Sleep(1 * time.Millisecond)
}

```

这是会报错`fatal error: all goroutines are asleep - deadlock!`，就是因为`ch<-1`发送了，但是同时没有接收者，所以发生了阻塞。

但是如果我们把`ch <- 1`放到`go loop(ch)`下面，程序就会正常运行。

### 缓冲channel

缓冲`channel`的阻塞只会发生在`channel`的缓冲使用完的情况下

```go

package main

import (
    "fmt"
    "time"
)

func loop(ch chan int) {
    for {
        select {
            case i := <-ch:
                fmt.Println("this value of unbuffer channel", i)
        }
    }
}

func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    ch <- 4
    go loop(ch)
    time.Sleep(1 * time.Millisecond)
}

```

- 这里会报`fatal error: all goroutines are asleep - deadlock!`，这是因为`channel`的大小为3，而我们要往里面塞4个数据，所以就会阻塞住

- 解决的办法有两个
    - 把`channel`开大一点
    - 把`channel`的消息发送者`ch <- 1` 这些代码移动到`go loop(ch)`下面，让`channel`实时消费就不会导致阻塞了