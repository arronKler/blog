

# 事件循环


# 浏览器中的事件循环


# Nodejs中的事件循环

Nodejs启动的时候，会初始化一个 **事件循环(event loop)** ， 处理传入的可能会触发异步API调用、timer、process.nextTick()的script脚本，然后再处理这个事件循环

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```
每一个盒子都被看做是事件循环的一个 *阶段(phase)* 。每个阶段都有一个先进先出的等着被执行的回调函数队列。每个阶段都有它特殊的地方，不过通常来说，当事件循环进入一个阶段的时候，它将会执行那个阶段的特定操作，然后执行那个阶段的回调函数队列，直至队列清空或者达到能够执行的最大回调数，事件循环就会移动到下一个阶段，如此下去。

## 阶段(phase)概览

- timers：执行setTimeout和setInterval设定的回调函数
- pending callbacks: 执行延迟到下一个循环的IO回调
- idle, prepare: 仅仅用于内部使用
- poll: 获取新的IO事件；执行IO相关回调；恰当的时候node会在这里进行阻塞
- check: 调用setImmediate设定的回调函数
- close callbacks: 调用一些关闭回调. （e.g. socket.on('close', ...)）

## 阶段(phase)详解

### timers
一个设定的timer设定的时间只是一个阈值，超过这个阈值之后回调函数会被执行，但是也不是准确的时间去执行，而是取决于其他回调是否会延迟这个过程。

> 技术上来说，poll阶段控制了timer什么时候会被执行(poll阶段可能会阻塞导致延迟)

### pending callbacks
这个阶段会处理一些系统操作的回调，比如TCP的一些错误类型

### poll阶段
这个阶段有两个主要的功能：
1. 计算需要进行IO阻塞和轮询多久，然后
2. 处理轮询队列中的事件

如果事件循环进入了poll阶段，并且没有timer被设定，以下两件事中的一个会发生：
- 如果poll队列不是空的。事件循环会迭代这个回调队列并同步执行，直到队列清空或者达到了系统设定的极限
- 如果poll队列是空的。会这样进行：
  - 如果脚本代码被setImmediate设定了，事件循环就会终结poll阶段然后继续进入check阶段去执行被设定的脚本
  - 如果脚本代码没有被setImmediate设定，事件循环就会等待回调被添加到队列中，然后立刻执行它

一旦poll队列是空的时候，事件循环就会检查有哪些timer已经抵达设定的时间阈值了，如果有准备好的timer，事件循环就会绕回到timers阶段并执行

### check
这个阶段可以允许你在poll阶段结束之后立刻执行一个回调函数。当poll阶段空闲下来了，某些脚本代码也被setImmediate设定到队列里了，事件循环就会进入check阶段而不是干等

### close callbacks
If a socket or handle is closed abruptly (e.g. socket.destroy()), the 'close' event will be emitted in this phase. Otherwise it will be emitted via process.nextTick().

## setImmediate() vs setTimout()

- setImmediate() 被设计成当当前poll阶段结束执行执行一段脚本
- setTimout() 是预设一段需要被执行的脚本在一个最小时间阈值之后执行

这两个函数在同一个模块中被调用时，其执行先后根据上下文环境是不一定谁先谁后的。比如当这两个函数的执行都不是在一个IO操作内进行调用的时候，谁先谁后取决于处理的性能；当他们在一个IO操作中被调用，那么这个时候 setImmediate 就一定优先于setTimeout被调用了。

所以应用setImmediate的主要优势在于，在一个IO循环中不管调用了多少次setTimeout，你都可以使用setImmediate确保某些代码在其之前被执行

## process.nextTick()

### 理解 process.nextTick()
尽管process.nextTick() 是异步API的一部分，但是因为技术上来说，它并非是事件循环的一部分。而是，`nextTickQueue` 将会在每个阶段执行结束的时候被一个一个执行直至清空后才会进入下一个阶段。


