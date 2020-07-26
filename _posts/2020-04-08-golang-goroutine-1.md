---
layout: post
title:  "Golang 协程Goroutine到底是怎么回事？（一）"
date:   2020-04-08 23:44:09 +0800
categories: Golang 并发 goroutine
---

Golang号称云计算时代的C语言，是非常值得研究的一门语言

本文是笔者在初学Golang的时候，学习的一些新的分享。现在开一个系列，Golang究竟怎么回事系列？谈Goroutine，谈数据结构，不仅语言语义理解，还要更深入的，更本质的看到，Golang的数据结构到底是怎么回事？

其中，使用到gdb，dlv等调试工具，有此经验的更佳。（旁白：这也是我更喜欢Golang的原因，可以使用gdb拨开云雾，看到最本质的东西）

Goroutine思考几个问题

1.  协程是什么，协程应用场景？

2.  协程的调度实现有哪几种样式？有哪些常见的协程实现？

3.  实现一个简易协程调度

4.  协程上最重要的准则是什么？

5.  有了协程要配套哪些东西？ 

前面有两篇介绍协程的文章： 

1.  [同步框架异步化改造—任务协程化 （一）](http://mp.weixin.qq.com/s?__biz=MzU2MDcwNTg3OA==&mid=2247483673&idx=1&sn=372586f35bb60e2c183db7820efee896&chksm=fc02b920cb753036fd0b4d9daed7dbea08ba2be7ba0340fcaa47a0026fd43a4b9aaa5b5f1c8d&scene=21#wechat_redirect)

2.  [同步框架异步化改造—任务协程化 （二）](http://mp.weixin.qq.com/s?__biz=MzU2MDcwNTg3OA==&mid=2247483671&idx=1&sn=9a1a7717fda4298977f1d0c16dd34c16&chksm=fc02b92ecb7530382f291e2911adeb1f2675f992df264f7e150fa7f1248df07acd8c36ad309b&scene=21#wechat_redirect)

从简单的讲起，协程是什么？

## 协程是什么？

协程是什么？ 协程就是用户态的最小调度执行单位，类比理解就是用户态线程，本质就是用户态自己切换cpu，在协程这一层我们基本可以把线程和cpu等同起来。（旁白：协程这个执行体操作系统是不认识的，只有用户自己认识，所以你用pstack看线程的工具是看不了协程的）

协程应用场景？

1.  IO密集型：IO密集型程序，cpu利用率低，使用协程，可以让用户按照实际情况调度，充分利用cpu，在当前多核cpu的架构中非常重要

2.  框架改造：原本项目全是同步调用，cpu利用率低。直接改成异步回调不现实，通过实现协程，达到非侵入式的框架异步改造

3.  协程的实现使用会使得全异步框架代码的编写简单，可维护性好

（旁白：协程两个用法：1）框架同步改造异步    2）异步代码写成同步样子）

协程的调度实现一般有哪几种样式？

协程最根本的就两种类型：

1.  对称的切换调度方式

2.  非对称的切换调度方式

**对称的调度方式**

每个协程任务都是一样的，不存在主次，都可以相互切换。这类调度类型看着美观，但是实现起来会非常复杂，如果加上一些协程锁，异步io切换逻辑之后，而且极容易出错。不容易实现时序的串行化。 

![image](https://upload-images.jianshu.io/upload_images/14414032-9b1806614617d792?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**非对称的调度方式**

最典型的就是有一个中心调度任务。主要角色分为：

1.  主协程：负责所有的协程调度

2.  任务协程：执行具体的业务逻辑代码的协程任务 

**基本原则：**

1.  严格保证所有的协程切换都必须且只能在 "主协程"<-> "任务协程” 之间进行

2.  存在串行逻辑的时候，必须保证严格的串行时序（这个会在协程锁的实现里讲） 

![image](https://upload-images.jianshu.io/upload_images/14414032-a34f0297163a33d7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有哪些常见的协程实现？

Linux提供了协程库，可以基于以下这四个调用实现协程切换

```
#include <ucontext.h>
void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
int  swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
int  getcontext(ucontext_t *ucp);
int  setcontext(const ucontext_t *ucp);
```

1.  glusterfs

2.  qemu，等

或者你可以自己保存，交换寄存器栈环境：

1.  libco （C++）

2.  greenlet（Python），等

怎么实现一个简易的协程调度？

![image](https://upload-images.jianshu.io/upload_images/14414032-6972caebbe651c00?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是一个比较完整的切换示意图：

1.  主协程（调度协程）总是从协程队列中取出协程任务执行

2.  协程任务执行过程中，遇到等待事件，需要保存好上下文，设置好唤醒路径之后，切回调度

3.  切回调度之后，CPU就让出来了，就可以执行其他的任务，从而实现了并发

4.  等待事件到来之后，按照之前设置好的环境路径，把协程任务再次投入到协程队列尾端，等待执行

5.  等重新取到协程的时候，主协程切入，从之前切出的地方开始执行 

以上就是实现的一个简单的协程调度的原理。当然具体细节会有很多，状态修改，协程生命周期，校验逻辑。比如必须：

1.  加入爆栈的校验（支不支持栈的自动扩容）

2.  协程的生命周期的校验

3.  可能还需要做一些调试工具，比如查看某个协程的协程函数调用栈

4.  死锁检查

1.  比如，某个协程加了mutex阻塞锁，走到后面代码，就直接切到调度，那么后面一旦有协程任务来加同一把mutex锁，就会导致死锁问题 

协程上最重要的准则是什么？

协程任务上一定不能跑阻塞的任务调用。一定要确保cpu不停的转。因为所有的协程当前本质上是不支持抢占任务的，因为没有时间片的概念。一旦阻塞，会导致这个线程执行所有的任务阻塞。 

协程要配套哪些东西？

1.  协程锁，条件变量，sleep，或者其他一切和阻塞有关的调用。 

## Goroutine的设计

前面复习完了协程通用的知识，下面终于到了重点戏码——Golang的协程是怎么回事？ （旁白：协程实现很简单，就四板斧：任务，队列，切换上下文的手段，代码执行者）
**G-P-M的数据结构**

![image](https://upload-images.jianshu.io/upload_images/14414032-18911edf1edda19e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

作为Go的最大宣传特点，来看看goroutine的协程实现。goroutine本质上和上面我实现的协程是一样的。但是由于做了一些层次抽象，更具灵活性。

*   G：Goroutine，一个G就是我们协程任务，是调度执行的单位。所以最重要的就是栈结构了（旁白：四板斧之一：任务）

*   M：Machine，这是一个抽象出来的数据结构，可以认为就是执行体，就是线程，就是cpu，每个M都代表一个线程（旁白：四板斧之一：执行者）

*   P：processor。处理器，这个可以认为就是代表一个硬件cpu核心。通常这个数量也就是和cpu核数相同（旁白：四板斧之一：队列，Golang的设计就是得P者得天下，得队列者得天下）

其中启动开始P就是固定的，M是会增长的，M执行任务必须是绑定到一个P（也就是说，一定要有一个队列），没有绑定到P的M就是空闲的，或者游离态的。这样数据结构（P）和执行（M）分离增加了扩展性。 

举两个例子：

1.  如果M被阻塞，这个时候，队列里面所有的G都是要移交出去的，之前会存在比较复杂的操作。GMP架构，只需要M释放P，空闲的M去接管P就行了。

2.  如果当前M执行完了P队列的所有任务，那么也不会空闲等待，而是会尝试去steal其他的G。先尝试从全局队列里获取，没有获取到，那么再去随机挑选一个P队列，拿走部分的G。（worke-steal） 

这个GMP的设计是在Go1.1之后加入的：https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw

提一下：go里面实现一些并发同步操作的时候，很多都是使用原子操作来替代锁，从而减少消耗，这个值得我们学习。

有些特殊的M，比如sysmon是不绑定P的。这个用于监控一些阻塞的异常情况，比如一个M长时间阻塞超过10ms，那么强制把M-P解绑，把M游离出去，P绑定到一个空闲的M上，继续执行队列里的G任务。

## Go程序启动

```
// The bootstrap sequence is:
//
// call osinit
// call schedinit
// make & queue new G
// call runtime·mstart
//
// The new G calls runtime·main.
```

1.  做一些初始化的操作

2.  创建出一个goroutine结构 runtime.main 函数

3.  执行runtime.mstart 函数

4.  汇编引导结束，之后就由golang的函数main入口运行 

![image.gif](https://upload-images.jianshu.io/upload_images/14414032-41538b29c629d1e1.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

初始化的时候，会创建几个线程（M）

1.  sysmon特殊线程

2.  垃圾回收的线程 

（旁白：goroutine有runtime的运行逻辑）

## Goroutine调度

![image](https://upload-images.jianshu.io/upload_images/14414032-c84d3f612263907a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

创建goroutine

接口

```
newproc
```

goroutine的调度跟之前我实现的协程调度核心是一致的，但是由于是多了一个抽象层（GPM），灵活性和扩展性大大提高。

1.  go语言里面go关键字用于创建goroutine（协程），实际调用的是newproc函数

2.  newproc创建出一个goroutine结构体：G，分配2kb的协程栈（在systemstack环境下调用）

3.  然后把G加入P队列中，等待执行

4.  切回原来的goroutine执行指令

**步骤一：**用来创建goroutine的结构

```
type funcval struct {
   fn uintptr
   // variable-size, fn-specific data here
}
```

![image](https://upload-images.jianshu.io/upload_images/14414032-417cae1005a47e8a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：特意标红的地方，这里是goroutine调度的一个关键。在goroutine执行完fn函数之后，在执行ret汇编指令的时候，会把这个地址取出来放到指令计数器（pc）去执行，而这个地址恰好是goexit的地址。这个赋值就是在newproc的时候赋值的。执行了goexit，你才能切回调度里（非对称中心化调度）。

```
newproc -> newproc1 -> gostartcallfn
```

**步骤二： **newproc是在systemstack的包装下调用的，这个调用保证newproc的函数执行是在调度协程的栈里面（M.g0栈）

```

// func systemstack(fn func())
TEXT runtime·systemstack(SB), NOSPLIT, $0-8
...
// 切换到调度: switch to g0
MOVQ   DX, g(CX)
MOVQ   (g_sched+gobuf_sp)(DX), BX
SUBQ   $8, BX
MOVQ   $runtime·mstart(SB), DX
MOVQ   DX, 0(BX)
MOVQ   BX, SP

// 执行函数：call target function
MOVQ   DI, DX
MOVQ   0(DI), DI
CALL   DI

// 切回原来的协程：switch back to g
MOVQ   g(CX), AX
MOVQ   g_m(AX), BX
MOVQ   m_curg(BX), AX
MOVQ   AX, g(CX)
MOVQ   (g_sched+gobuf_sp)(AX), SP
MOVQ   $0, (g_sched+gobuf_sp)(AX)
```

这个就符合中心调度的设计思想。解释几个函数调用

```
runqget  // goroutine 出队
runqput  // goroutine 入队
runqgrab // goroutine 抢占
```

G入队的几个优先级：

1.  runqput

1.  _p_.runqnext （第一优先级）

2.  _p_.(runqhead, runqtail) 双端队列 

3.  runqputslow

1.  sched.runq 全局队列 （p队列满了就会溢出到全局队列，p队列256个槽位） 

```
newproc -> newproc1 -> systemstack ( runqput )
```

步骤三：执行goroutine

调度接口入口

```
schedule
```

流程就是： 

1.  从队列里获取到G

1.  从P队列里获取G任务

2.  第二优先级从其他地方获取

3.  切入执行

这里提到一点细节就是：go的调度机制是，当执行了n（61）个任务之后，必须要去全局列表获取G任务，保证公平执行。

具体切入执行某个G

```
execute -> gogo
```

其中gogo的代码 

![image](https://upload-images.jianshu.io/upload_images/14414032-cba90f0cb55b737e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

goroutine的抢占调度

goroutine本质上是没有抢占式的调用，只是会在goroutine结构体上加上一个标记。因为没有时间片。只有当有机会调用到特定的调用的时候，才可能发生切出。

goroutine的自动扩容

1.  编译器分析判断是否可能会导致2kb的栈溢出，如果可能，那么就会在函数的汇编代码前后加上指令代码

1.  前面——判断是否栈溢出

2.  后——栈扩容调用morestack

（旁白：自动扩容的触发机制也被复用在抢占调度了） 

goroutine的主动切出

1.  Gosched : 把当前G放入到队列中，然后切出

2.  gopark/goparkunlock ： 保存上下文，直接切出

3.  goready ： 唤醒G（把G重新入队） 

![image](https://upload-images.jianshu.io/upload_images/14414032-b790d6ee0f0f378e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

坚持思考，方向比努力更重要。
**关注我：奇伢云存储**
![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)



