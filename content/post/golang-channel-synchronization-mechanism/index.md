---
title: Golang 协程同步机制
description: 通道不仅是用来传递数据的，更重要的是，它提供了一种让两个协程能够 安全地同步 的方式。
date: 2023-10-13T13:37:32+08:00
lastmod: 2023-10-13T13:37:32+08:00
slug: golang-channel-synchronization-mechanism

tags:
  - goroutine
  - channel
categories:
  - Golang
---

先来通过一个例子来观察一下协程的运行机制：

```go
package main

import (
	"fmt"
	"time"
)

func reportNap(name string, delay int) {
	for i := 0; i < delay; i++ {
		fmt.Println(name, "等待")
		time.Sleep(1 * time.Second)
	}
	fmt.Println(name, "等待结束!")
}
func send(myChannel chan string) {
	reportNap("第二goroutine", 2)
	fmt.Println("***发送到channel***")
	myChannel <- "a"
	fmt.Println("***发送到channel***")
	myChannel <- "b"
}

func main() {
	myChannel := make(chan string)
	go send(myChannel)
	reportNap("主程序 goroutne", 5)
	fmt.Println("接收" + <-myChannel)
	fmt.Println("接收" + <-myChannel)
}
/*
输出结果：
主程序 goroutne 等待
第二goroutine 等待
主程序 goroutne 等待
第二goroutine 等待
主程序 goroutne 等待
第二goroutine 等待结束!
***发送到channel***
主程序 goroutne 等待
主程序 goroutne 等待
主程序 goroutne 等待结束!
接收a
***发送到channel***
接收b
*/
```

## 并发启动

程序一开始，在 main 函数里做了三件事：

- 创建一个通道： myChannel := make(chan string) 创建了一个无缓冲区的通道。这意味着它没有容量，只有在发送和接收操作都准备好时，数据才能通过。如果发送方准备好了但没有接收方，发送操作就会阻塞。

- 启动一个新协程： go send(myChannel) 启动了一个新的、独立的执行线程，也就是 goroutine，去执行 send 函数。此时，main 函数和这个新启动的协程是并行运行的。

- 主程序开始等待： main 函数调用 reportNap("主程序 goroutne", 5)，开始自己的 5 秒钟等待。

## 交错执行与阻塞

### 第一秒片：

```go
主程序 goroutne 等待
第二 goroutine 等待
```

### 第二秒片：

```go
主程序 goroutne 等待
第二 goroutine 等待
```

### 第三秒片：

```go
主程序 goroutne 等待
第二 goroutine 等待结束!
***发送到 channel***
```

### 第四秒片：

```go
主程序 goroutne 等待
```

### 第五秒片：

```go
主程序 goroutne 等待
```

### 第六秒片：

```go
主程序 goroutne 等待结束!
接收 a
***发送到 channel***
接收 b
```

在上面这个输出过程中，我们首先看到两个协程在前二秒都开始 sleep 等待。然后，  
第三秒主程序还在 sleep 等待，第二个协程结束了自己的 sleep 等待，并开始向通道发送数据 a。然后被 channel 阻塞开始等待。  
第四秒，第五秒属于主程序 sleep 等待。  
第六秒，主程序的 sleep 等待结束，并开始接收 channel 数据，并打印出来。然后第二个协程的 channel 阻塞结束。开始发送数据 b。主程序开始接收数据 b，并打印出来。

> **注意：**  
> 协程之间是并发执行的，而不是顺序执行的。  
> 所以同一时间片内的协程可能会被交替执行，而不是顺序执行。

## 总结

这个过程的核心是 通道的同步机制。通道不仅是用来传递数据的，更重要的是，它提供了一种让两个协程能够 安全地同步 的方式。通过通道，一个协程可以等待另一个协程完成它的任务，从而避免了数据竞争和不确定性。
