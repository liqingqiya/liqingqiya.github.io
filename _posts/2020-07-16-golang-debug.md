---

layout: post
title: "golang 调试高阶技巧"
date: 2020-7-16 1:44:09 +0800
categories: golang GC 垃圾回收

---

*   golang 高阶调试
    *   Golang tools
        *   nm
        *   compile
        *   objdump
        *   pprof
        *   trace
    *   单元测试
        *   执行单元测试
            *   go test 运行
            *   编译，运行
        *   统计代码覆盖率
    *   程序 Debug
        *   dlv 调试用法
            *   调试二进制
            *   调试进程
            *   调试 core 文件
            *   调试常用语法
                *   系统整理
                *   应用举例
        *   gdb 调试
    *   小技巧
        *   不知道怎么断点函数？
        *   不知道调用上下文？
        *   不知道怎么开启 pprof ？
        *   为什么有时候单点调试的时候，总是非预期的执行代码？
    *   总结

# golang 高阶调试

本文专注 golang debug 的一些技巧应用，以及相关工具的实用用法，再也不用怕 golang 怎么调试。golang 作为一门现代化语音，出生的时候就自带完整的 debug 手段：

*   golang tools 是直接集成在语言工具里，支持内存分析，cpu分析，阻塞锁分析等；
*   delve，gdb 作为最常用的 debug 工具，让你能够更深入的进入程序调试；
    *   delve 当前是最友好的 golang 调试程序，ide 调试其实也是调用 dlv 而已，比如 goland；
*   单元测试的设计深入到语言设计级别，可以非常方便执行单元测试并且生成代码覆盖率；

## Golang tools

golang 从语言原生层面就集成了大量的实用工具，这些都是 Robert Griesemer, Rob Pike, Ken Thompson 这几位大神经验沉淀下的精华。你安装好 golang 之后，执行 `go tool` 就能看到内置支持的所有工具了。

```
root@ubuntu:~# go tool
addr2line
asm
buildid
cgo
compile
cover
dist
doc
fix
link
nm
objdump
pack
pprof
test2json
trace
vet

```

我这里专注挑选几个 debug 常用的：

*   nm：查看符号表（等同于系统 `nm` 命令）
*   objdump：反汇编工具，分析二进制文件（等同于系统 `objdump` 命令）
*   pprof：指标，性能分析工具
*   cover：生成代码覆盖率
*   trace：采样一段时间，指标跟踪分析工具
*   compile：代码汇编

### nm

查看符号表的命令，等同于系统的 nm 命令，非常有用。在断点的时候，如果你不知道断点的函数符号，那么用这个命令查一下就知道了（命令处理的是二进制程序文件）。

```
# exmple 为你编译的二进制文件
go tool nm ./example

```

第一列是地址，第二列是类型，第三列是符号：

[图片上传失败...(image-1c9b7a-1594910164396)]

### compile

汇编某个文件

```
go tool compile -N -l -S example.go

```

你就能看到你 golang 语言对应的汇编代码了（注意了，命令处理的是 golang 代码文本），酷。

### objdump

反汇编二进制的工具，等同于系统 `objdump`（注意了，命令解析的是二进制格式的程序文件）。

```
go tool objdump example.o
go tool objdump -s DoFunc example.o  // 反汇编具体函数

```

汇编代码这个东西在 90% 的场景可能都用不上，但是如果你处理过 c 的程序，在某些特殊场景，通过反汇编一段逻辑来推断应用程序行为将是你唯一的出路。因为线上的代码一般都是会开启编译优化，所以这里会导致你的代码对不上。再者，线上不可能让你随意 attach 进程，很多时候都是出 core 了，你就只有一个 core 文件去排查。

### pprof

pprof 支持四种类型的分析：

*   CPU ：CPU 分析，采样消耗 cpu 的调用，这个一般用来定位排查程序里耗费计算资源的地方；
*   Memroy ：内存分析，一般用来排查内存占用，内存泄露等问题；
*   Block ：阻塞分析，会采样程序里阻塞的调用情况；
*   Mutex ：互斥锁分析，采样互斥锁的竞争情况；

我们这里详细以内存占用分析举例（其他的类似），pprof 这个是内存分析神器。基本上，golang 有了这个东西，99% 的内存问题（比如内存泄露，内存占用过大等等）都是可以非常快的定位出来的。首先，对于 golang 的内存分析（或者其他的锁消耗，cpu 消耗）我们明确几个重要的点：

*   golang 内存 pprof 是采样的，每 512KB 采样一次；
*   golang 的内存采样的是堆栈路径，而不是类型信息；
*   golang 的内存采样入口一定是通过`mProf_Malloc`，`mProf_Free` 这两个函数。所以，如果是 cgo 分配的内存，那么是没有机会调用到这两个函数的，所以如果是 cgo 导致的内存问题，go tool pprof 是分析不出来的；

详细原理，可以复习另一篇文章：内存分析；

分析的形式有两种：

1.  如果是 `net/http/pporf` 方式开启的，那么可以直接在控制台上输入，浏览器就能看；
2.  另一种方式是先把信息 dump 到本地文件，然后用 `go tool` 去分析（我们以这个举例，因为这种方式才是生产环境通用的方式）

```
# 查看累计分配占用
go tool pprof -alloc_space ./29075_20190523_154406_heap
# 查看当前的分配占用
go tool pprof -inuse_space ./29075_20190523_154406_allocs

```

你也可以不指定类型，直接 `go tool pprof ./xxx` ，进入分析之后，调用 `o` 选项，指定类型：

我写了一个 demo 程序，然后 dump 出了一份 heap 的 pprof 采样文件，我们先通过这个 pprof 得出一些结论，最后我再贴出源代码，再品一品。

```
go tool pprof ./29075_20190523_154406_heap
(pprof) o              
...          
  sample_index              = inuse_space          //: [alloc_objects | alloc_space | inuse_objects | inuse_space]
...       
(pprof) alloc_space
(pprof) top
Showing nodes accounting for 290MB, 100% of 290MB total
      flat  flat%   sum%        cum   cum%
     140MB 48.28% 48.28%      140MB 48.28%  main.funcA (inline)
     100MB 34.48% 82.76%      190MB 65.52%  main.funcB (inline)
      50MB 17.24%   100%      140MB 48.28%  main.funcC (inline)
         0     0%   100%      290MB   100%  main.main
         0     0%   100%      290MB   100%  runtime.main

```

这个 top 信息表明了这么几点信息：

*   `main.funcA` 这个函数现场分配了 140M 的内存，`main.funcB` 这个函数现场分配了 100M 内存，`main.funcC` 现场分配了 50M 内存；
    *   现场的意思：纯粹自己函数直接分配的，而不是调用别的函数分配的；
    *   这些信息通过 flat 得知；
*   `main.funcA` 分配的 140M 内存纯粹是自己分配的，没有调用别的函数分配过内存；
    *   这个信息通过 `main.funcA` flat 和 cum 都为 140 M 得出；
*   `main.funcB` 自己分配了 100MB，并且还调用了别的函数，别的函数里面涉及了 90M 的内存分配；
    *   这个信息通过 `main.funcB` flat 和 cum 分别为 100 M，190M 得出；
*   `main.funcC` 自己分配了 50MB，并且还调用了别的函数，别的函数里面涉及了 90M 的内存分配；
    *   这个信息通过 `main.funcC` flat 和 cum 分别为 50 M，140 M 得出；
*   `main.main` ：所有分配内存的函数调用都是走这个函数出去的。main 函数本身没有函数分配，但是他调用的函数分配了 290M；

demo 的源代码：

```
package main

import (
	"net/http"
	_ "net/http/pprof"
)

func funcA() []byte {
	a := make([]byte, 10*1024*1024)
	return a
}

func funcB() ([]byte, []byte) {
	a := make([]byte, 10*1024*1024)
	b := funcA()
	return a, b
}

func funcC() ([]byte, []byte, []byte) {
	a := make([]byte, 10*1024*1024)
	b, c := funcB()
	return a, b, c
}

func main() {
	for i := 0; i < 5; i++ {
		funcA()
		funcB()
		funcC()
	}

	http.ListenAndServe("0.0.0.0:9999", nil)
}

```

**dump 命令**：

```
curl -sS 'http://127.0.0.1:9999/debug/pprof/heap?seconds=5' -o heap.pporf

```

对照着代码，再品一品。

### trace

程序 trace 调试

```
go tool trace -http=":6060" ./ssd_336959_20190704_105540_trace

```

trace 这个命令允许你跟踪采集一段时间的信息，然后 dump 成文件，最后调用 `go tool trace` 分析 dump 文件，并且以 web 的形式打开。

## 单元测试

单元测试的重要性就不再论述。golang 里面 `_test.go` 结尾的文件认为是测试文件，golang 作为现代化的语言，语言工具层面支持单元测试。

### 执行单元测试

执行单元测试有两种方式：

*   go test 直接运行，这个是最简单的；
*   先编译测试文件，再运行。这种方式更灵活；

#### go test 运行

```
// 直接在你项目目录里运行 go test .
go test .
// 指定运行函数
go test -run=TestPutAndGetKeyValue
// 打印详细信息
go test -v

```

#### 编译，运行

本质上，golang 跑单测是先编译 `*_test.go` 文件，编译成二进制后，再运行这个二进制文件。你执行 `go test` 的时候，工具帮你做好了，这些动作其实也是可以拆开来自己做的。

编译生成单元测试可执行文件：

```
// 先编译出 .test 文件
$ go test -c 

```

```
// 指定跑某一个文件
$ ./raftexample.test -test.timeout=10m0s -test.v=true -test.run=TestPutAndGetKeyValue

```

这种方式通常会出现在以下几种场景：

1.  这台机器上编译，另一个地方跑单测；
2.  debug 单测程序；

### 统计代码覆盖率

golang 的代码覆盖率是基于单测的，由单测作为出发点，来看你的业务代码覆盖率。

操作很简单：

1.  加一个 `-coverprofile` 的参数，声明在跑单测的时候，记录代码覆盖率；
2.  使用 `go tool cover` 命令分析，得出覆盖率报告；

```
go test -coverprofile=coverage.out
go tool cover -func=coverage.out

```

类似如下：

```
root@ubuntu:~/opensource/readcode-etcd-master/src/go.etcd.io/etcd/contrib/raftexample# go tool cover -func=coverage.out
go.etcd.io/etcd/v3/contrib/raftexample/httpapi.go:33:	ServeHTTP		25.0%
go.etcd.io/etcd/v3/contrib/raftexample/httpapi.go:108:	serveHttpKVAPI		0.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:41:	newKVStore		100.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:50:	Lookup			100.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:57:	Propose			75.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:71:	readCommits		55.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:107:	getSnapshot		100.0%
go.etcd.io/etcd/v3/contrib/raftexample/kvstore.go:113:	recoverFromSnapshot	85.7%
go.etcd.io/etcd/v3/contrib/raftexample/listener.go:30:	newStoppableListener	75.0%
go.etcd.io/etcd/v3/contrib/raftexample/listener.go:38:	Accept			92.9%
go.etcd.io/etcd/v3/contrib/raftexample/main.go:24:	main			0.0%
total:							(statements)		57.1%

```

这样的话，你就知道每个函数的代码覆盖率。

## 程序 Debug

程序的调试主要由两个工具：

1.  dlv
2.  gdb

这里推荐 dlv，因为 gdb 功能实在是有限，gdb 不理解 golang 的业务类型和协程。但是 gdb 有一个功能是无法替代的，就是 gcore 的功能。

### dlv 调试用法

#### 调试二进制

```
dlv exec <path/to/binary> [flags]

```

举例：

```
dlv exec ./example

```

dlv 调试二进制，并带参数

```
dlv exec ./example -- --audit=./d

```

#### 调试进程

```
dlv attach ${pid} [executable] [flags]

```

进程号是必选的。

举例：

```
dlv attach 12808 ./example

```

#### 调试 core 文件

dlv 调试core文件；并且标准输出导出到文件

```
dlv core <executable> <core> [flags]

```

```
dlv core ./example core.277282

```

#### 调试常用语法

##### 系统整理

**程序运行**

1.  call ：call 函数（注意了，这个会导致整个程序运行的）
2.  continue ：往下运行
3.  next ：单步调试
4.  restart ：重启
5.  step ：单步调试，某个函数
6.  step-instruction ：单步调试某个汇编指令
7.  stepout ：从当前函数跳出

**断点相关**

1.  break (alias: b) ：设置断点
2.  breakpoints (alias: bp) ：打印所有的断点信息
3.  clear ：清理断点
4.  clearall ：清理所有的断点
5.  condition (alias: cond) ：设置条件断点
6.  on ：设置一段命令，当断点命中的时候
7.  trace (alias: t) ：设置一个跟踪点，这个跟踪点也是一个断点，只不过运行道德时候不会断住程序，只是打印一行信息，这个命令在某些场景是很有用的，比如你断住程序就会影响逻辑（业务有超时），而你仅仅是想打印某个变量而已，那么用这种类型的断点就行；；

**信息打印**

*   args : 打印程序的传参
*   examinemem (alias: x) ：这个是神器，解析内存用的，和 gdb 的 x 命令一样；
*   locals ：打印本地变量
*   print (alias: p) ：打印一个表达式，或者变量
*   regs ：打印寄存器的信息
*   set ：set 赋值
*   vars ：打印全局变量（包变量）
*   whatis ：打印类型信息

**协程相关**

*   goroutine (alias: gr) ：打印某个特定协程的信息
*   goroutines (alias: grs) ：列举所有的协程
*   thread (alias: tr) ：切换到某个线程
*   threads ：打印所有的线程信息

**栈相关**

*   deferred ：在 defer 函数上下文里执行命令
*   down ：上堆栈
*   frame ：跳到某个具体的堆栈
*   stack (alias: bt) ：打印堆栈信息
*   up ：下堆栈

**其他命令**

*   config ：配置变更
*   disassemble (alias: disass) ：反汇编
*   edit (alias: ed) ：略
*   exit (alias: quit | q) ：略
*   funcs ：打印所有函数符号
*   libraries ：打印所有加载的动态库
*   list (alias: ls | l) ：显示源码
*   source ：加载命令
*   sources ：打印源码
*   types ：打印所有类型信息

以上就是完整的 dlv 的支持的命令，从这个来看，是完全满足我们的调试需求的（有的只适用于开发调试环节，比如线上的程序不可能让你随意单步调试的，有的使用于线上生产环节）。

##### 应用举例

**打印全局变量**

```
(dlv) vars

```

这个非常有用，帮助你看一些全局变量。

**条件断点**

```
# 先断点
(dlv) b 

# 查看断点信息
(dlv) bp

# 然后定制条件
(dlv) condition 2 i==2 && j==7 && z==32

```

**查看堆栈**

```
# 展示所有堆栈
(dlv) goroutines
# 所有堆栈展开
(dlv) goroutines -t

```

**解析内存**

```
(dlv) x -fmt hex -len 20 0xc00008af38

```

`x` 命令和 gdb 的 `x` 是一样的。

### gdb 调试

gdb 对 golang 的调试支持是通过一个 python 脚本文件 `src/runtime/runtime-gdb.py` 来扩展的，所以功能非常有限。gdb 只能做到最基本的变量打印，却理解不了 golang 的一些特殊类型，比如 channel，map，slice 等，gdb 原生是无法调适 goroutine 协程的，因为这个是用户态的调度单位，gdb 只能理解线程。所以只能通过 python 脚本的扩展，把协程结构按照链表输出出来，支持的命令：

[图片上传失败...(image-c8e3d1-1594910164394)]

gdb当前只支持6个命令：

**3个 cmd 命令**

1.  info goroutines；打印所有的goroutines
2.  goroutine ${id} bt；打印一个goroutine的堆栈
3.  iface；打印静态或者动态的接口类型

**3个函数**

1.  len；打印string，slices，map，channels 这四种类型的长度
2.  cap；打印slices，channels 这两种类型的cap
3.  dtype；强制转换接口到动态类型。

**打印全局变量** (注意单引号)

```
(gdb) p 'runtime.firstmoduledata'

```

由于 gdb 不理解 golang 的一些类型系统，所以调试打印的时候经常打印不出来，这个要注意下。

**打印数组变量长度**

```
(gdb) p $len(xxx)

```

所以，我一般只用 gdb 来 gcore 而已。

## 小技巧

### 不知道怎么断点函数？

有时候不知道怎么断点函数：可以通过nm查询下，然后再断点，就一定能断到了。

[图片上传失败...(image-f2bd4b-1594910164394)]

[图片上传失败...(image-e94d0b-1594910164394)]

### 不知道调用上下文？

在你的代码里添加一行：

```
debug.PrintStack()

```

这样就能当前代码位置的堆栈给打印出来，这样你就直到怎么函数的调用路径了。

### 不知道怎么开启 pprof ？

pprof 功能有两种开启方式，对应两种包：

*   net/http/pprof ： 使用在 web 服务器的场景；
*   runtime/pprof ：使用在非服务器应用程序的场景；

这两个本质上是一致的，`net/http/pporf` 也只是在 `runtime/pprof` 上的一层 web 封装。

**`net/http/pprof` 方式**

```
import _ "net/http/pprof"

```

**`runtime/pprof` 方式**

这种通常用于程序调优的场景，程序只是一个应用程序，跑一次就结束，你想找到瓶颈点，那么通常会使用到这个方式。

```
	// cpu pprof 文件路径
    f, err := os.Create("cpufile.pprof")
	if err != nil {
		log.Fatal(err)
	}
    // 开启 cpu pprof
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()

```

### 为什么有时候单点调试的时候，总是非预期的执行代码？

这种情况一般是被编译器优化了，比如函数内联了，编译出的二进制删减了无效逻辑、无效参数。这种情况就会导致你 dlv 单步调试的时候，总是非预期的执行，或者打印某些变量打印不出来。这种情况解决方法就是：禁止编译优化。

```
go build -gcflags "-N -l"

```

## 总结

该篇文章系统的分享了 golang 程序调试的技巧和用法：

1.  语言工具包里内置 tool 工具，支持汇编，反汇编，pprof 分析，符号表查询等实用功能；
2.  语言工具包集成单元测试，代码覆盖率依赖于单元测试的触发；
3.  常用 dlv/gdb 这两个工具作为大杀器，可以分析二进制，进程，core 文件；

---
坚持思考，方向比努力更重要。微信公众号关注我：奇伢云存储

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)

