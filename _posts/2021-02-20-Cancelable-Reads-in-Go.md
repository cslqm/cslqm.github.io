---
layout: post
title: "go中可以撤销的读"
subtitle: 'go read'
author: "cslqm"
header-style: text
tags:
  - Golang
---

https://benjamincongdon.me/blog/2020/04/23/Cancelable-Reads-in-Go/
作者：Ben Congdon

> 本文主要讲一个可以被抢占的异步读实现方案


io.Reader 接口，Read() 这个方法是阻塞的，开始操作后，就不能抢占。

io.ReadCloser 是 Go 的常用退出，允许抢占一个读取操作。但是，只能执行一次，当一个 reader 被关闭后，之后又来的读取操作会失败。

简单的实现一个异步操作

``` go
func asyncReader(r io.Reader) {
    ch := make(chan []byte)
    Go func() {
        defer close(ch)
        for {
            var b []byte
            // Point of no return
            if _, err := r.Read(b); err != nil {
                return
            }
            ch <- b
        }
    }()

    select {
    case data <- ch:
        doSomething(data)
    //...
    // Other asynchronous cases
    //...
    }
}
```

上边的代码有个问题，你只能等到 Read() 读完退出。
当你尝试清理整个系统时（中断任务），会遇到棘手的部分：如果我们想要发送结束该过程的信号，该怎么办？考虑一个改造过的例子：

``` go
func asyncReader(r io.Reader, doneCh <-chan struct{}) {
    ch := make(chan []byte)
    continueReading := false
    Go func() {
        defer close(ch)
        for !continueReading {
            var b []byte
            // Point of no return
            if _, err := r.Read(b); err != nil {
                return
            }
            ch <- b
        }
    }()

    select {
    case data <- ch:
        doSomething(data)
    case <-doneCh:
        continueReading = false
        return
    }
}
```

事实上，还是有问题，可能会泄露 goroutine。
当上面的 doneCh 被关闭时，asyncReader 方法将返回。我们创建的 Goroutine 也将在下次计算 for 循环中的条件时返回。但是，如果 Goroutine 阻塞在 r.Read() 该怎么办？那样的话，我们实质上泄露了一个 goroutine。在 reader 离开阻塞状态之前，我们将一直陷入困境。

Go 中常用的用来 goroutine 直接沟通的方法就是 context。传递信号。

如果把 Read() 方法给加上一个 context，通过 context 实现抢占不就行了。

``` go
type CancelableReader struct {
    ctx  context.Context
    data chan []byte
    err  error
    r    io.Reader
}

func (c *CancelableReader) begin() {
    buf := make([]byte, 1024)
    for {
        n, err := c.r.Read(buf)
        if err != nil {
            c.err = err
            close(c.data)
            return
        }
        tmp := make([]byte, n)
        copy(tmp, buf[:n])
        c.data <- tmp
    }
}

func (c *CancelableReader) Read(p []byte) (int, error) {
    select {
    case <-c.ctx.Done():
        return 0, c.ctx.Err()
    case d, ok := <-c.data:
        if !ok {
            return 0, c.err
        }
        copy(p, d)
        return len(d), nil
    }
}

func New(ctx context.Context, r io.Reader) *CancelableReader {
    c := &CancelableReader{
        r:    r,
        ctx:  ctx,
        data: make(chan []byte),
    }
    Go c.begin()
    return c
}
```

CancelableReader 包装器上有一个巨大的星号：它仍然存在 Goroutine 泄漏。如果底层的 io.Reader 永远不返回，那么 begin() 中的 Goroutine 将永远不会被清理。

至少，使用这种方法可以更清楚的知道泄漏发生的位置，你可以在 struct 上存储一些额外的状态来追踪 Goroutine 是否结束。或许，你可以将这些 CancelableReader 组成一个池，并在读取全部完成时回收它们。