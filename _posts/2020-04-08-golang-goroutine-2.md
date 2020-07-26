---
layout: post
title:  "Golang 协程Goroutine到底是怎么回事？（二）"
date:   2020-04-09 23:44:09 +0800
categories: Golang 并发 goroutine
---

上一篇从协程的通用原理讲起，讲了通Golang的协程，使用一个完成的协程，必须要配合完善的配套设备，协程锁，定时器等，这篇文章就是描述于此。 

# Go 协程配套设备

Golang 协程锁，定时器，是怎么回事？系统调用又有什么特殊，G-M锁定是什么？

## 协程锁

之前提到，协程使用之后，是必须配套实现一些配件的。关键就是要保证在执行goroutine的时候不阻塞。最典型的的就是锁、timer、系统调用这三个方面。其中锁必须要是协程锁。

举例：某个场景，任务A需要修改Z，任务B也需要修改Z。如果是串行系统，A执行完了，再执行B，那么不会有问题。A -> B 。现在A，B是goroutine，可以并发执行，那么在操作Z的时候我们必须要有保证串行化的机制。

```
CO_LOCK
{
    #处理逻辑
}
CO_UNLOCK
```

现在的关键点就是，我们不能直接用之前的mutex锁，或者是自旋锁。这样会严重影响并发，或者导致死锁。而必须配套实现协程锁。

```
sync.Mutex.Lock 
-> runtime_SemacquireMutex
    -> sync_runtime_SemacquireMutex
        -> semacquire1 // runtime/sema.go
```

1.  当加锁失败，则保存上下文，把自己赋值到一个sudog结构里
2.  挂接到锁内部相关队列里（semaRoot），root.queue() 。
3.  调用goparkunlock主动切走，切到调度协程

```
sync.Mutex.Unlock
-> runtime_Semrelease
    -> sync_runtime_Semrelease
        -> semrelease1
```

1.  解锁
2.  取出这个锁内部等待队列的一个元素（g）
3.  调用goready唤醒goroutine，投入队列中，等待执行 

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-04-08-golang-goroutine-2/1240.png)

现在就以A, B任务同时处理Z来举例：

1.  A因为要修改Z，所以加了协程锁
2.  加锁之后，由于处理一些其他的逻辑，因为某些等待事件，又把cpu切到M.g0调度了 （yield）；注意了还没有放锁
3.  这个时候M把B拿过来执行，yield to B
4.  B也要修改Z，这个时候发现锁已经被加上了，于是把自己挂到锁结构里面去
5.  然后B直接切走，yield to M.g0
6.  现在A的事件满足了，M.g0 重新调度到A执行，yield to A
7.  A 从刚刚切走的地方开始执行，然后放锁
    1.  注意了，放锁这里就会把B这个协程任务从锁队列中摘除，加到调度队列中，
9.  A执行完成之后，M.g0 调度B执行
10.  B从刚刚加锁的地方唤醒，于是加上锁了。然后走锁内逻辑，走完就放锁

以上就是协程锁的实现原理。保证A,B在修改Z的时候必须串行化。（旁白：加锁其实就是入队，串行入队，解锁就是出队，串行出队唤醒）

## timer

time的实现原理：

1.  time.Sleep()的时候先创建好timer结构体，挂到哈希表
2.  确保创建了一个goroutine（timeproc），这个会不断检查超时的timer
3.  调用gopark保存栈，切到调度
4.  timeproc循环检查，当发现有超时的timer的时候，调用goready，把这个挂到运行队列里，等待运行

## 系统调用 

对于某些系统调用，可能是会导致阻塞的，所以这个也必须封装才能让goroutine有让出cpu的机会。go内部实现系统调用会在前后包装两个函数：

```
entersyscall
exitsyscall
```

解决syscall可能导致的问题关键就在这两个函数。这两个函数主要做了这些事情

**entersyscall**

1.  设置p的状态为 _Psyscall

2.  暂时解除P->M的绑定。但是M是有路径找到P的。并且虽然解除了P->M的绑定，但是这里并不会把P绑定到其他的M

**exitsyscall**

1.  先尝试绑定到之前P

2.  如果之前的P已经被sysmon处理掉了，那么则挑选一个空闲的P

3.  如果还不行，则挂到全局队列sched里面去

（旁白：封装这两个函数，就是为了监控，不能让这一个系统调用阻塞了队列里所有的任务。你不能执行P了，就让给别人，就是这个思路） 

sysmon线程就是处理_Psyscall状态的P，发现有超时的，则把P找个空闲的绑定，去执行P队列里的协程任务。 

## G-M锁定 

golang支持了一个G-M锁定的功能，通过lockOSThread和unlockOSThread来实现。主要是用于一些cgo调用，或者一些特殊的库，有些库是要求固定在一个线程上跑。

1.  G_a锁定M0 lockOSThread

2.  G_a调用gosched切走，投入P1队列

3.  M0调度，发现是lockedm，于是让出P0，自己调用notesleep睡眠

4.  M1取出G_a，发现是lockedg，于是让出P1给M0，并且唤醒M0. 自己变idle，stopm休眠

5.  M0继续执行G_a

你可以发现，G_a只在M0上运行，锁定这段期间，M0也只执行了G_a任务。 

## 当前go有哪些问题 

当前go没有实现异步io。换句话说，如果在一个goroutine里面使用read/write io的系统调用，这些都是同步的io调用。会实实在在的阻塞M的调度，在遇到io延迟慢的时候，会导致sysmon检查到M-P超时（10ms），那么就会把M-P解绑，M游离出去执行阻塞任务，分配一个新的M来绑定P执行队列里的任务。

那么这种情况，虽然没有完全阻塞死P任务的执行，但是代价非常大，而且可能会导致M的数量一直飙升。就算没有这些极限情况，IO的并发能力相较于aio也是不行的。（旁白：Golang能切走的当前只有网络IO，磁盘io走的是系统调用，协程切不走）

当前net库是已经实现了底层的patch，aio还没有实现关键还是aio的复杂性导致的。 其实很多的工程实践是通过libaio来实现磁盘io的异步，配合协程一起使用。

* * *

坚持思考，方向比努力更重要。**关注我：奇伢云存储**
![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)

