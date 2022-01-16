---
title: "Unix IO models"
slug: ""
date: 2021-12-11T15:40:33+08:00
lastmod: 2022-01-16T13:50:00+08:00
draft: false
toc: true
weight: false
categories: ["操作系统"]
tags: ["IO"]
---

想起前同事有一天问我的一个问题，blocking|unblocking I/O 跟 sync|async I/O 有什么不同？  
前几天带着这个问题，读了《Unix Network Programming - Volume 1》。本文准备学以致用，把 Unix 系统上 5 种可用的 I/O 模型用自己的话讲清楚，并把前面这个问题给解答了。<!--more-->

正常情况下，一次在 socket 上进行的输入操作包含两个清楚的阶段：

1. 等待数据在内核中就绪  
   等待数据到达网卡，当数据包到达时，它们被拷贝到内核中的缓冲区。
2. 将数据从内核的缓冲区拷贝至应用的缓冲区中

Unix 提供了 5 种可用的 I/O 模型，它们的差异主要在与这两个阶段中应用跟内核之间的交互方式：

- blocking I/O
- nonblocking I/O
- I/O multiplexing (select and poll)
- signal driven I/O (SIGIO)
- asynchronous I/O (the POSIX aio_functions)

## Blocking I/O Model

最常见的是 blocking I/O model。默认情况下，所有的 socket 都是阻塞的（blocking）。下图以 UDP datagram socket 为例，描述这种模型的工作方式：

{{< figure src="/images/blocking-model.png" title="图 1" >}}

上图中发生两次上下文切换，应用调用 recvfrom 时从运行在应用中（running in the application）切换至运行在内核中（running in the kernel）。成功返回时，从内核切换回应用。

## Nonblocking I/O Model

当我们将 socket 设置为 nonblocking 时，我们正在告诉内核 "如果不让进程休眠，我请求的操作就无法完成，那就别让进程休眠，而是立即返回一个错误"。

{{< figure src="/images/nonblocking-model.png" title="图 2" >}}

前三次我们调用 `recvfrom`，都没有数据可返回，因此内核立即返回一个 `EWOULDBLOCK` 错误。第四次尝试时，一个数据报（datagram）就绪了，它被拷贝至应用缓冲区。在拷贝期间，应用进程一直阻塞。

应用在一个循环中在一个 nonblocking descriptor 上调用 `recvfrom`，这就是轮询（polling）。这往往会浪费 CPU 时间，还好偶尔会遇到这个模型，通常在专注于单一功能的系统上。

## I/O Multiplexing Model

对 I/O Multiplexing 来说，我们调用 `select` 或 `poll` 并阻塞在这两个系统调用之一上，而非阻塞在**实际的 I/O 系统调用**上。

{{< figure src="/images/multiplexing-model.png" title="图 3" >}}

跟 blocking model 比较，这个模型好像没有任何优势，反而还有轻微的劣势，因为它有两次系统调用，而非一次。但它的优势在于，我们一次可以等待多个描述符就绪（ready）。

另一个紧密相关的 I/O 模型是将多线程跟 blocking I/O 一起使用。该模型非常像 multiplexing model，除了它不是阻塞在多个文件描述符上，而是使用多个线程（每个文件描述符一个）自由地调用像 `recvfrom` 这样的阻塞系统调用（blocking system call）。

## Signal-Driven Model

我们也可以告诉内核，当描述符就绪的时候，请它使用 `SIGIO` 信号通知我们。

{{< figure src="/images/signal-driven-model.png" title="图 4" >}}

为了启用 signal-driven I/O，我们先对 socket 做一些设置。然后使用 `sigaction` 系统调用安装一个信号处理器。安装完信号处理器后，进程就继续去做其他事情了，无需阻塞等待数据报（datagram）达到。当信号处理器收到信号时，要么它先调用 `recvfrom` 将数据报读到应用缓冲区再通知 main loop 数据报可处理了，要么它通知 main loop 数据报可读取了并让 main loop 来读取数据。

## Async I/O Model

asynchronous I/O 由 POSIX 规范定义。async func 告诉内核，开始期望的操作并在整个操作完成（包括将数据报从内核拷贝至应用缓冲区）时通知我们。这个模型跟前一节描述的信号驱动的模型之间主要的不同之处在于，使用信号驱动，当一个 I/O 操作**可以启动（can be initiated）时**内核就通知我们；而使用 async I/O，当一个 I/O 操作**完成时**内核才通知我们。

{{< figure src="/images/async-io.png" title="图 4" >}}

我们调用 `aio_read` 函数（POSIX 异步 I/O 函数的名称以 aio_ 或 lio 开始），并将 descriptor、buffer pointer、buffer size、file offset (类似于 lseek)、当整个操作完成时如何通知我们 这 5 个参数传给内核。

## I/O 模型之间的比较

前 4 个模型之间的主要不同在第一阶段，因为它们的第二阶段是相同的：在将数据从内核拷贝至应用缓冲区期间进程阻塞在 `recvfrom` 系统调用上。然而，async I/O 跟前 4 个不同，一次异步函数调用处理这两个阶段，在此期间进程继续处理其他事情。

{{< figure src="/images/cmp-io-models.png" title="图 5" >}}

## Sync I/O vs Async I/O

POSIX 定义了如下两个术语：

- 一个 **sync I/O 操作**会导致请求进程阻塞，直到 I/O 操作完成。
- 一个 **async I/O 操作**不会导致请求进程阻塞。

根据这个定义，前 4 个模型都是同步的，因为使用它们时 **实际的 I/O 操作（`recvfrom`）** 都会阻塞进程。只有 async I/O 模型满足 async I/O 的定义。

{{< admonition type=note title="注意" open=true >}}
sync 和 async I/O 并不是具体的 I/O model，只是根据 **请求进程是否被阻塞直到 I/O 操作完成** 这一点对 I/O model 的分类而已。因此它们不像 I/O model 那样描述 I/O 操作的具体工作方式。
{{< /admonition >}}

{{< admonition type=note title="注意" open=true >}}
本文多次提及 **I/O 操作** 这个概念。在大多数 I/O model 中，应用执行的操作不止一个，到底哪个操作是 I/O 操作呢？*UNP-Vol1, I/O Models* 章节中对比 sync 和 async I/O 时，有这么一句话：

> Using these definitions, the first four I/O models—blocking, nonblocking, I/O multiplexing, and signal-driven I/O—are all synchronous because **the actual I/O operation (recvfrom)** blocks the process.

根据粗体部分可知，I/O 操作指的是 `recvfrom`。虽然在不同的 I/O model 中，`recvfrom` 做得事情可能不同，但都包括将数据从内核空间拷贝至应用空间。从应用的角度看，I/O 操作应该是指应用接收来自内核的数据，换句话说，就是将数据从内核空间拷贝至应用空间。
{{< /admonition >}}

## 常见问题与错误理解

- blocking/non-blocking I/O 与 sync/async I/O 之间有什么区别？

  我们先看问题本身，blocking/non-blocking I/O 是具体的 I/O model。但如果 sync/async I/O 指的是 I/O model，那么问题描述本身就有错误，因为没有一个具体的 I/O model 叫 sync I/O；如果指的是上一节中讲述的对 I/O model 的分类，也需要对问题描述做些调整，以便说清楚要对比的两个具体对象。

- 有人认为，I/O 有阻塞/非阻塞、同步/异步这分，这两个维度是正交的，两两组合就得到四种 I/O model。
  
  这个理解错在，这两个维度并非是正交的，因此不能两两组合。根据上一节内容可知，sync I/O 这个分类包含了所有在 I/O 操作完成前会阻塞请求进程的 I/O model，因此这两个维度不是正交的。

## 参考资料

- UNP-Vol1, I/O Models: *UNIX Network Programming Volume 1, Third Edition* "6.2 I/O Models"
























