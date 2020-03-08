---
title: 「译文」深入探索现代 web 浏览器（第一部分）
author: 陈加菲
date: 2019-02-03 13:51:47
tags:
- 翻译
---

> 原文链接：https://developers.google.com/web/updates/2018/09/inside-browser-part1  
>
> 在这篇译文中，我对 web, Chrome 等常见单词作了保留，一些术语的中文翻译参考了维基百科，而一些句子的翻译则参考了 Google 翻译。 

## CPU, GPU, 内存和多进程架构  

在这个由四部分组成的博客系列中，我们会从高层架构到渲染器管道的细节来深入探索 Chrome 浏览器。如果你对浏览器如何将你的代码变成一个功能性网站的过程感到好奇，或者你困惑于为什么要使用某种特定技术来提高性能，便值一读该系列文章。  

在这个系列的第一部分，我们会来看一下计算机的核心术语以及 Chrome 的多进程架构。  

> 提示：如果你熟悉 CPU/GPU 和 进程/线程 的概念，你可以直接跳到浏览器架构部分阅读  

## 计算机的核心是 CPU 和 GPU  

为了理解浏览器运行的环境，我们需要了解一些计算机的组成部分以及它们负责的工作。  

### CPU  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/CPU.png" alt="图 1：CPU 芯片像坐在办公室的上班族，处理到来的任务" title="图 1：CPU 芯片像坐在办公室的上班族，处理到来的任务" style="zoom:100%;" />
图 1：CPU 芯片像坐在办公室的上班族，处理到来的任务  

首先是中央处理器（Central Processing Unit），即 CPU。CPU 可以认为是你计算机的大脑。一个 CPU 核心，如图所示如同一个在办公室里的上班族，可以一个接一个地处理到达面前的不同的任务。它不仅可以处理从数学到艺术的每一件事，且同时知道如何响应用户的调用。过去大多数 CPU 是单核心的，一个核心就如同在同一芯片里的另一个 CPU 。在现代硬件中，你通常可以得到多于一个核心的 CPU，以便为你的手机和笔记本电脑提供更多计算能力。  

### GPU  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/GPU.png" alt="图 2：许多拿着扳手的 GPU 芯片表明他们能处理有限的任务" title="图 2：许多拿着扳手的 GPU 芯片表明他们能处理有限的任务" style="zoom:100%;" />
图 2：许多拿着扳手的 GPU 芯片表明他们能处理有限的任务  

图形处理器(Graphics Processing Unit)，即 GPU，是计算机的另一个组成部分。不同于 CPU，GPU 擅长于处理同时跨越多个芯片的简单的任务。正如它的名字所示，GPU 最初被开发用来处理图形，这也是为什么在图形环境中 “使用 GPU” 或者 “GPU 支持” 与快速渲染和平滑交互相关联。近几年，随着 GPU 计算能力的提升，将使越来越多的计算单独运算在 GPU 上成为可能。  

当你在计算机或手机上开启一个应用程序时，CPU 和 GPU 是为这些应用程序提供动力的部件。通常，应用程序使用操作系统提供的机制在 CPU 和 GPU 上运行。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/hw-os-app.png" alt="图 3：计算机的三层架构。机器硬件位于底层，操作系统处于中间，上层是应用程序" title="图 3：计算机的三层架构。机器硬件位于底层，操作系统处于中间，上层是应用程序" style="zoom:100%;" />
图 3：计算机的三层架构。机器硬件位于底层，操作系统处于中间，上层是应用程序。  

## 在进程和线程上执行程序  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/process-thread.png" alt="图 4：进程就像一个有边界的盒子，而线程好比在一个进程里面游泳的抽象的鱼" title="图 4：进程就像一个有边界的盒子，而线程好比在一个进程里面游泳的抽象的鱼" style="zoom:100%;" />
图 4：进程就像一个有边界的盒子，而线程好比在一个进程里面游泳的抽象化的鱼  

进程和线程是深入探索浏览器架构之前需要掌握的另一个概念。一个进程可以描述为一个应用的执行程序，一个线程是存在于进程内部，并执行进程任一程序的部分。  

当你开启一个应用程序时，将会创建一个进程，该程序可能会创建一个（些）线程来协助其完成工作，但这是可选的。操作系统会分配一块 “平板” 似的内存给进程来工作，且所有应用程序的状态将维持在这块私有的内存空间中。当你关闭应用程序，这些进程将同时消失，而操作系统将会释放这些内存。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/memory.png" alt="图 5：一个进程使用内存空间以及存储应用程序数据的图解" title="图 5：一个进程使用内存空间以及存储应用程序数据的图解" style="zoom:100%;" />
图 5：一个进程使用内存空间以及存储应用程序数据的动图  

一个进程能够要求操作系统启动另一个进程来运行不同的任务。当这个情况发生时，其他部分的内存会被分配给新的进程。如果两个进程间需要通信，它们可以通过进程间通信（Inter Process Communication, IPC）来实现。许多应用程序被设计成这种方式来进行工作，因此，假如一个运行中的进程没有了响应，可以在不停止其他正在运行不同应用程序的进程情况下重启该进程。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/workerprocess.png" alt="图 6：隔离的进程间通过 IPC 进行通信的图解" title="图 6：隔离的进程间通过 IPC 进行通信的图解" style="zoom:100%;" />
图 6：隔离的进程间通过 IPC 进行通信的动图  

## 浏览器架构  

所以，一个 web 浏览器是如何利用进程和线程来构建的？好吧，它可能是一个具有许多不同线程的进程或者许多不同进程，且只有少数线程通过 IPC 来进行通信。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/browser-arch.png" alt="图 7：不同浏览器架构中的进程/线程图" title="图 7：不同浏览器架构中的进程/线程图" style="zoom:100%;" />
图 7：不同浏览器架构中的进程/线程图  

这里要注意的地方是这些不同的架构是实现的细节，关于如何构建一个 web 浏览器是没有标准规范的。一个浏览器的实现方法可能与另一个浏览器完全不同。  

在这个博客系列，我们将使用下图中描述的 Chrome 最新架构。  

最上方是浏览器进程与其他负责应用程序不同部分的进程间的协作。对于渲染器进程，多个进程被创建和分配给每一个 tab。直到最近，Chrome 尽可能为每一个 tab 分配一个进程，如今它尝试为每一个站点分配属于自己的进程，包括 iframes（见 [Site Isolation](https://developers.google.com/web/updates/2018/09/inside-browser-part1?utm_source=wanqu.co&utm_campaign=Wanqu+Daily&utm_medium=website#site-isolation)）  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/browser-arch2.png" alt="图 8：Chrome 的多进程架构图。渲染器进程下显示多个图层，表示 Chrome 为每一个 tab 运行多个渲染器进程" title="图 8：Chrome 的多进程架构图。渲染器进程下显示多个图层，表示 Chrome 为每一个 tab 运行多个渲染器进程" style="zoom:100%;" />
图 8：Chrome 的多进程架构图。渲染器进程下显示多个图层，表示 Chrome 为每一个 tab 运行多个渲染器进程。  

## 每个进程掌控什么？  

下面的表格介绍了每个 Chrome 进程以及它们掌控什么：  


|该进程|其掌控的内容|
|:------|:------|
|浏览器|控制应用程序的 "chrome" 部分，包括地址栏、书签和前进后退按钮|
|渲染器|控制显示网站的 tab 内的任何内容|
|插件|控制被网站使用的所有插件，例如 flash|
|GPU|从其他进程中独立地处理 GPU 任务。它从不同的进程中分离开来，是因为 GPU 处理来自多个应用程序的请求并在同一的界面绘制它们|

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/browserui.png" alt="图 9：不同的进程指向浏览器用户界面中的不同部分" title="图 9：不同的进程指向浏览器用户界面中的不同部分" style="zoom:100%;" />
图 9：不同的进程指向浏览器用户界面中的不同部分  

此外还有像扩展进程和实用进程（utility processes） 等更多的进程，如果你想看看你的 Chrome 中运行着多少进程，点击右上方的 ┋ 图标，选择更多工具，然后选中任务管理器，这将会打开一个窗口，显示当前正在运行的进程列表以及它们所使用的 CPU 和内存。  

## Chrome 中多进程架构的好处  

刚才，我提到 Chrome 使用多个渲染器进程，在最简单的例子中，你可以想象每个 tab 拥有自己的渲染器进程。比如你打开了 3 个 tab，每个 tab 在独立的渲染器进程中运行，如果其中一个 tab 无响应，你可以关闭该没有响应的 tab 并且继续保持其他 tab 的运行正常。如果所有的 tab 都运行在同一个进程中，当其中一个 tab 无响应时，所有的 tab 都会无响应。这很不好。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/tabs.png" alt="图 10：该图显示每个 tab 运行着多个进程" title="图 10：该图显示每个 tab 运行着多个进程" style="zoom:100%;" />
图 10：该图显示每个 tab 运行着多个进程  

将浏览器的工作分离成多个进程的另一好处是安全性和沙盒性（[sandboxing](https://en.wikipedia.org/wiki/Sandbox_(computer_security))）。自从操作系统提供一种方法来限制进程的特权，浏览器便可以从某些特定功能中沙盒化某些特定的进程。例如，Chrome 浏览器限制处理任意用户输入（如浏览器进程）的进程，对任意文件的访问。  

因为进程拥有自己的私有内存空间，它们通常会包括共同基础设施的副本（例如  Chrome 浏览器的 JavaScript 引擎 V8）。这意味着更多的内存使用，因为如果它们是同一进程内的线程，则无法以它们的方式来共享数据。为了节省内存，Chrome 限制了它可以启动的进程数量。这种限制依赖于你的设备拥有多少内存以及 CPU 运算能力，但是当 Chrome 达到其限制的数量时，它会在一个进程中开始在同一站点中运行多个 tab。  

## 节省更多的内存 - Chrome 的 Servicification  

同样的方法适用于浏览器进程。Chrome 正在进行架构的更改，以便将浏览器的每一部分作为一项服务运行，从而可以轻松地拆分为不同的进程或汇总为一个进程。  

一般的做法是，当 Chrome 运行在强大的硬件上时，它可以为每项服务拆分进不同的进程来提高稳定性，但如果它运行在资源受限的设备上时，Chrome 会将服务整合到一个进程中，以节省内存。在此更改之前，在类似 Android 的平台上使用了相同的方法来整合进程以减少内存的使用。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/servicfication.png" alt="图 11：Chrome 的 servicification 将不同的服务转移到多进程中和一个浏览器进程中" title="图 11：Chrome 的 servicification 将不同的服务转移到多进程中和一个浏览器进程中" style="zoom:100%;" />
图 11：Chrome 的 servicification 将不同的服务转移到多进程中和一个浏览器进程中  

## 每个 frame（Per-frame）的渲染器进程 - 站点隔离  

站点隔离（[Site Isolation](https://developers.google.com/web/updates/2018/07/site-isolation)）是 Chrome 最近推出的一项功能，可以为每个跨站的 iframe 运行一个单独的渲染器进程。我们已经讨论过每个 tab 一个渲染器进程的模型，它允许跨站的 iframe 运行在一个单一的渲染器进程中，并在不同站点间共享内存空间。在相同的渲染器进程中运行 a.com 和 b.com 似乎没问题，同源策略（[Same Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)）是 web 的核心安全模型，它确保一个站点不能未经同意访问另一个站点站的数据。绕过此策略是安全攻击的主要目标。进程隔离是隔离站点的最有效方法。因为 [Meltdown and Spectre](https://developers.google.com/web/updates/2018/02/meltdown-spectre) 漏洞，我们需要利用进程来隔离站点将变得更加明显。自 Chrome 67 以来默认在桌面上启用了站点隔离，一个 tab 内的每个跨站 iframe 会得到一个单独的渲染器进程。  

<img src="http://q6q6q9uu8.bkt.clouddn.com/images/translate-inside-look-at-modern-web-browser-part-1/isolation.png" alt="图 12：站点隔离图，多个渲染器进程指向 一个站点内的 iframe" title="图 12：站点隔离图，多个渲染器进程指向 一个站点内的 iframe" style="zoom:100%;" />
图 12：站点隔离图，多个渲染器进程指向一个站点内的 iframe  

使站点能够隔离是一个多年的工程努力。站点隔离并不像分配不同的渲染器进程那样简单，它从根本上改变了 iframe 间通信的方法。在具有运行着不同进程的 iframe 的页面上打开开发者工具，意味着开发者工具必须实现幕后的工作，以使该 iframe 看似是运行在同一进程中的。即使在一个页面中运行简单的 Ctrl+F 来搜索单词，也意味着将从不同的渲染器进程中进行搜索。你可以看到这就是为什么浏览器工程师将站点隔离的发布作为一个重要里程碑的原因！  

## 总结  

在这篇文章中，我们从高层角度介绍了浏览器架构和多进程架构的好处。我们也介绍了 Chrome 中与多进程架构密切相关的 Servicification 和站点隔离。在下一篇文章中，我们将开始深入研究这些进程和线程间发生的事情，以此来展示一个网站。  

**END**

