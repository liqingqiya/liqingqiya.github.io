---
layout: post
title:  "同步框架异步化改造—任务协程化 （二）"
date:   2020-04-04 00:31:51 +0800
categories: 并发 coroutine python
---

![微信搜索框](/images/wechat_search_box.png)


## Gevent 库的使用

手动从0开始写一个协程调度功能，考虑的东西很多。对于当前的项目，时间不允许。考虑到B项目是python代码，所以就使用了gevent调度。gevent的patch功能，patch掉底层的io接口。这样就能做到不改业务一行代码，化同步为异步调用。

首先提一下 greenlet库。这个库才是真正提供协程切换的接口库。其实，我们只需要greenlet库就可以，按照之前的ucontext协程调度的方式，用greenlet提供的switch接口进行切换。可以理解为，greenlet和ucontext协程库提供的功能是一样的：都是提供最基本的切换协程功能。怎么调度？还得自己去封装。而gevent库就提供了这一层封装。

### gevent的两个核心组件

1. greenlet：

   提供协程对象的封装和协程切换的接口，使用这个接口，用户可以自己进行调度，无论是对称的，还是非对称的

2. libev：

   事件反应堆，封装使用了原生的epoll池，抽象了事件的概念。比如最常用的事件：io事件，定时事件

## gevent的架构原理

gevent是严格的非对称调度方式。有一个hub协程，其他的都是任务协程。

![中心调度](/images/posts/coroutine/ED20A951-0A2F-46B8-A0EC-3B6DAB6EAA96.jpeg){:height="50%" width="50%"}

gevent严格遵循中心调度原则：

- 任何一次切换都必须是和hub协程之间的切换
- hub运行的逻辑很简单，就是运行epoll池，从池里面取出事件执行

举一个简单的生产协程的例子：

- import greenlet的时候，模块初始化会初始化出一个main 协程，这个可以认为是root协程。这个是greenlet模块创建的，但是调度协程是hub。

使用姿势：
```python
import gevent

# 封装的业务逻辑
def test_wrap():  
  pass
# 生成一个协程任务  
A = gevent.spawn(test_wrap)
gevent.joinall([A])
```

**生产协程：spawn**
解释：

1. 这个接口为了满足中心化调度，所以只要是gevent创建的协程parent都是指向hub协程的。这样保证切换时在hub和普通协程之间
2. 在start方法里，调用了 loop->run_callback方法把协程切入的协程注册到loop事件池的prepare事件，这样就能保证下一次切入hub执行的时候，hub能够调用回调 A->switch，切入协程执行

```python
   class Greenlet(greenlet):
     def __init__(self, run=None, *args, **kwargs):    
       hub = get_hub()    
       greenlet.__init__(self, parent=hub)    
     @classmethod
     def spawn(cls, *args, **kwargs):        
         g = cls(*args, **kwargs)          
         g.start()
         return g
     def start(self):
         if self._start_event is None:
         self._start_event = self.parent.loop.run_callback(self.switch)
```

**注册prepare事件，切换到hub里，准备调度执行**
总的来说，做了两件事情：

1. 封装了一个waiter对象，该对象用于保存上下文，设置切回的路径，然后切到hub协程里。那么保存的上下文是哪个上下文呢？main协程。

2. hub执行prepare回调函数的时候，切到协程任务里执行业务代码。执行完之后，会重新切回到main

3. main把自己从注册的池子取出来unlink掉

```python
   def joinall(greenlets, timeout=None, raise_error=False, count=None):
     if not raise_error:
       # 注册事件，切到hub执行        
       wait(greenlets, timeout=timeout)
   def wait(objects=None, timeout=None, count=None):   
     result = []
     if count is None:
       return list(iwait(objects, timeout))
   def iwait(objects, timeout=None):    
     waiter = Waiter()    
     switch = waiter.switch
     try:        
       count = len(objects)
       for obj in objects:
         # 注册事件到prepare的事件回调中            
         obj.rawlink(switch)
       for _ in xrange(count):
         # 保存上下文，设置好回来的路径，然后从这里切到hub协程            
         item = waiter.get()
     finally:
       # 执行到这里的时候，就说明协程任务执行完成了
       for obj in objects:            
         unlink = getattr(obj, 'unlink', None)
         if unlink:
           try:
             # 从prepare事件回调里，取出来该协程                    
             unlink(switch)
             ...
```

### gevent 踩到的坑

1. gevent有一个黑魔法就是patch。这个既是优点也是缺点。如果使用gevent 的patch，一定要记得把gevent的patch放到代码的最顶部。一定要patch完全。不然很容易出现一个问题就是：重入断言的问题。举个例子，socket是需要patch的，如果多个协程并发用到了同一个socket，那么是不安全的，会出现问题的。所以这个必须断言。比如A协程执行的时候，由于socket读事件还没有ok，就切到hub了。然后B协程执行的时候，又用到了同一个socket，那么就会出现重入断言。或者说，hub切回A的执行点不在刚切走的点，也会出现断言。这个都是patch的不完全导致的奇奇怪怪的问题
2. gevent和peewee配合的问题，由于项目中使用到了mysql数据库，使用了pymysql引擎。peewee不会主动关闭socket连接，并且peewee为了解决socket在多协程中的使用问题，使用了协程local变量。也就是说，同一个数据库，在不同的协程中个，socket是不同的。如果在协程销毁的时候，没有关闭连接，就会出现句柄泄露
3. gevent需要配合pymysql才能发挥作用，因为这样才能在数据库socket io的时候，让出cpu。mysqldb库是c库，patch不到。

### 总结

1. 使用协程方案来提高并发吞吐处理能力最核心是因为：业务的代码过于复杂，直接异步化不现实。非侵入式的改动，即不改动业务代码，就可以将架构变成全程异步非阻塞的架构。从而大大提高并发能力
2. gevent刚好提供了一个比较好的非对称调度框架，和patch方案，但是要小心使用

---

![关注我公众号](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)