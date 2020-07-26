---

layout: post
title:  "Golang 数据结构到底是怎么回事？gdb调一调？"
date:   2020-04-12 3:44:09 +0800
categories: Golang 并发 goroutine 数据结构

---


“ 不仅限于语法，使用gdb，dlv工具更深层的剖析golang的数据结构”



#  Golang数据结构

变量：有意义的一个数据块。
变量名：一个有意义的数据块的名字。

**为什么特意会有这个章节？**

golang本质是全局按照值传递的，也就是copy值，那么你必须知道你的内存对象是包含哪些内容，才能清楚掌握你在copy值的时候复制了哪些东西，这个很重要，第一部分的正文内容从这里开始。具体如下类型：

- num
- bool
- string
- array
- slice
- map
- channel
- interface 

这些结构实际在地址空间栈是什么形式的实现？这里直接看地址空间的内容，以下都是以这个例子进行分析：

```go
package main

func main () {
    var n int64 = 11
    var b bool = true

    var s string = "test-string-1"

    var a [3]bool = [3]bool{true, false, true}
    var sl []int = []int{1,2,3,4}

    var m map[int]string
    var c chan int
    var in interface{}

    _, _, _, _, _, _, _, _ = n, b, s, a, sl, m, c , in
}
```

## 数值类型
 n：n就是一个8字节的数据块。
## Bool类型
b：就是一个1字节的数据块。

## String类型
string类型在go里是一个复合类型，s变量是一个16字节的变量。其中str字段指向字符串存储的地方。

```sh
(gdb) pt s
type = struct string {
  uint8 *str;
  int len;
}
```
换句话说，s就是代表了一个16字节的数据块。所以我们每次定义一个string变量，除了字符序列，s的本身结构是分配16个字节在栈上。

赋值语句 
```sh
0x000000000044ebed <+61>:    lea    0x1e81d(%rip),%rax        # 0x46d411
0x000000000044ebf4 <+68>:    mov    %rax,0x48(%rsp)
0x000000000044ebf9 <+73>:    movq   $0xd,0x50(%rsp)
```

地址空间存储
```
(gdb) p/x uintptr(&s)
$9 = 0xc000030750
(gdb) x/2gx 0xc000030750
0xc000030750:    0x000000000046d411    0x000000000000000d
(gdb) x/s 0x000000000046d411
0x46d411:    "test-string-1triggerRatio=value method xadd64 failedxchg64 failed}\n\tsched={pc: but progSize  nmidlelocked= out of range  t.npagesKey=  untyped args -thread limit\nGC assist waitGC worker initMB; alloca"...
```
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-3651705a7bd71cb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 数组类型
数组类型，就是在地址空间中连续的一段内存块，和c一样（旁白：和c一样，都是平坦的内存结构）。

```sh
(gdb) p &a
$13 = ([3]bool *) 0xc00003070d
(gdb) x/6bx 0xc00003070d
0xc00003070d:    0x01    0x00    0x01    0x0b    0x00    0x00
```

## Slice类型
这是个复合类型，变量本身是一个管理结构（和string一样），这个管理结构管理着一段连续的内存。
```sh
(gdb) pt sl
type = struct []int {
  int *array;
  int len;
  int cap;
}
```

## map 类型 和 channel 类型
其中，map类型和channel类型特别提一点，变量本身本质上是一个指针类型。也就是说上面我们定义了两个变量m，c，从内存分配的角度来讲，只在栈上分配了一个指针变量，并且这个指针还是nil值，所以我们经常看到 go 的一个特殊说明：slice，map，channel 这三种类型必须使用make来创建，就是这个道理。因为如果仅仅定义了类型变量，那仅仅是代表了分配了这个变量本身的内存空间，并且初始化是nil，一旦你直接用，那么就会导致非法地址引用的问题。slice 的24个字节的管理空间，map和channel的一个指针8个字节的空间。那么如果是调用了make，其实就会把下面的结构分配并初始化出来。 

```sh
(gdb) pt m
type = struct hash<int, string> {
  int count;
  uint8 flags;
  uint8 B;
  uint16 noverflow;
  uint32 hash0;
  struct bucket<int, string> *buckets;
  struct bucket<int, string> *oldbuckets;
  uintptr nevacuate;  
  struct runtime.mapextra *extra;
} *

(gdb) pt c
type = struct hchan<int> {
  uint qcount;
  uint dataqsiz;
  void *buf;
  uint16 elemsize;
  uint32 closed;
  runtime._type *elemtype;
  uint sendx;
  uint recvx;
  struct waitq<int> recvq;
  struct waitq<int> sendq;
  runtime.mutex lock;
} *
```

那么make又做了什么，make 是 golang 的关键字，编译器会自动转变成函数调用，go 语言的关键字基本都是转换成内部的函数调用，梳理golang的关键字对应的转换表：
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-b3a56b6d30e6eb04?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
（旁白：编译器牛逼，golang 比c 高级就高级在编译器帮你做了非常多的事情）

## Interface
特别说明下interface，因为这个也是 golang 的一个特色点，你实现了interface定义的方法集，那么可以当作这个接口用，换句话说，你实现了这些行为，就是这个接口对象，怎么实现的？这个和c++的多态是很像的，和python的行为多态更像。c++是通过虚表结构来实现的多态。go实现的接口是通过interface结构来实现的。 

回忆下 c++ 的多态，c++ 是静态就定义好了继承或组合关系，虚表的个数是确定的：

1. 对象头部存在一个虚表指针
2. 虚表的内容是编译器在编译期间就定好了的
3. 每个类都是有自己的虚表的，不同类创建出来的对象虚表指针是指向自己类的虚表
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-a1dcf5e1ce1e640c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有继承覆盖的情况如下： 
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-ebea5579cdcfb51e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以 c++ 的多态就一目了然，虽然我们可能不知道当前对象是那个类，当时我们通过头部的虚表指针找到对应的虚表，按照一样的偏移offset去获取到函数指针，这个对象方法自然就会是我们对象正确的方法。 


```go
go interface

type iface struct {
  tab  *itab
  data unsafe.Pointer
}

type eface struct {
  _type *_type
  data  unsafe.Pointer
}
```

当我们把一个对象赋值给接口，调用方法的时候，接口怎么能获取到对象正确的方法？ iface 接口里 data 存放私有数据，一般是具体对象地址。关键就在 itab 结构，本质上是一个`pair（interface，concrete）`。 

### itab结构
这个结构会在两种情况下生成：

- 编译期间（ static in compile ）：这个是大部分情况，只要是编译器能分析出来的，就会尽量在编译期间生成itab结构，然后存放在.rodata只读数据区域。比如类型赋值到接口的场景。（旁白：绝大部分是这种情况，只要编译器能帮你干的就顺手帮你干了，这种情况的运行时开销几乎没有，就是一个间接调用的开销）
- 运行期间（runtime）：有些场景是在编译期间无法确认itab的，这个只能等到runtime期间，动态的去查找、获取、生成。比如接口和接口直接相互转换的场景，这个就只能在运行期间才能确定

这种情况下，带来的开销是很大的，所以内部这种情况下是有对itab做一个全局缓存的，优化（ interface 类型，具体类型）查找itab的性能 

```go
type itab struct {
   inter *interfacetype
   _type *_type
   hash  uint32 // copy of _type.hash. Used for type switches.
   _     [4]byte
  fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

字段含义：

1. inter是描述接口类型的变量。
2. _type是描述具体对象的类型对象。
3. fun是一个可变数组，依次存放着这个pair（interface，concrete）实际的方法地址。

内存布局：
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-7e178397f7853507.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这么个结构如果是编译器生成，是存在.rodata里。 

#### 编译器静态生成 itab
举个例子：
```go
package main

type Node interface {
    Add(a, b int32) int32
    Sub(a, b int64) int64
}

type SObj struct{ id int32 }
func (adder SObj) Add(a, b int32) int32 { return a + b }
func (adder SObj) Sub(a, b int64) int64 { return a - b }

func main() {
    m := Node(SObj{id: 6754})
    m.Add(10, 32)
}
```

这里通过 SObj 变量类型赋值到接口 Node. 首先呢，这个在编译期就会检查是否能够这样赋值，允许赋值的满足的要求就是：SObj实现了Node的声明的两个方法。编译通过之后呢，会生成一个二元组对pair（Node，SObj）。调用过程呢，就是先获取itab，在获取fun，根据偏移获取方法地址。看下汇编代码： 

```sh
(gdb) disassemble
Dump of assembler code for function main.main:
  => 0x000000000044ec70 <+0>:    mov    %fs:0xfffffffffffffff8,%rcx
  0x000000000044ec79 <+9>:    cmp    0x10(%rcx),%rsp
  0x000000000044ec7d <+13>:    jbe    0x44ecf8 <main.main+136>  
  0x000000000044ec7f <+15>:    sub    $0x40,%rsp
  0x000000000044ec83 <+19>:    mov    %rbp,0x38(%rsp)
  0x000000000044ec88 <+24>:    lea    0x38(%rsp),%rbp
  0x000000000044ec8d <+29>:    movl   $0x0,0x24(%rsp)

  // 根据itab, val 构造出 interface结构 (这个是)
  0x000000000044ec95 <+37>:    movl   $0x1a62,0x24(%rsp)
  0x000000000044ec9d <+45>:    lea    0x2993c(%rip),%rax        # 0x4785e0 <go.itab.main.SObj,main.Node> // itab的地址;编译期确定
  0x000000000044eca4 <+52>:    mov    %rax,(%rsp)
  0x000000000044eca8 <+56>:    mov    0x24(%rsp),%eax           // value的值6754
  0x000000000044ecac <+60>:    mov    %eax,0x8(%rsp)
  0x000000000044ecb0 <+64>:    callq  0x407fd0 <runtime.convT2I32>
  0x000000000044ecb5 <+69>:    mov    0x18(%rsp),%rax
  0x000000000044ecba <+74>:    mov    0x10(%rsp),%rcx
  0x000000000044ecbf <+79>:    mov    %rcx,0x28(%rsp)           // interface m的地址
  0x000000000044ecc4 <+84>:    mov    %rax,0x30(%rsp)

  0x000000000044ecc9 <+89>:    mov    0x28(%rsp),%rax
  0x000000000044ecce <+94>:    test   %al,(%rax)
  0x000000000044ecd0 <+96>:    mov    0x18(%rax),%rax           // 获取到SObj.Add的地址
  0x000000000044ecd4 <+100>:    mov    0x30(%rsp),%rcx
  // 传参数 10，20
  0x000000000044ecd9 <+105>:    movabs $0x200000000a,%rdx
  0x000000000044ece3 <+115>:    mov    %rdx,0x8(%rsp)
  0x000000000044ece8 <+120>:    mov    %rcx,(%rsp)
  // 调用到 SObj.Add 方法
  0x000000000044ecec <+124>:    callq  *%rax
  0x000000000044ecee <+126>:    mov    0x38(%rsp),%rbp
  0x000000000044ecf3 <+131>:    add    $0x40,%rsp
  0x000000000044ecf7 <+135>:    retq
  0x000000000044ecf8 <+136>:    callq  0x446fb0 <runtime.morestack_noctxt>
  0x000000000044ecfd <+141>:    jmpq   0x44ec70 <main.main>
  End of assembler dump.

(gdb) p m
$3 = {tab = 0x4785e0 <SObj,main.Node>, data = 0xc00005e000}
(gdb) p &m
$4 = (main.Node *) 0xc000030778
(gdb) p $rsp + 0x28
$5 = (void *) 0xc000030778
(gdb) x/1gx 0x28 + $rsp
0xc000030778:    0x00000000004785e0
(gdb) x/1gx 0x00000000004785e0 + 0x18
0x4785f8 <go.itab.main.SObj,main.Node+24>:    0x000000000044ed70
(gdb) info symbol 0x000000000044ed70
main.(*SObj).Add in section .text of /root/go-proj/test_interface_1
```

查看二进制 .rodata 段
```sh
[root@bogon go-proj]# objdump -xt -j .rodata ./test_interface_1|grep SObj
00000000004785e0 g     O .rodata    0000000000000028 go.itab.main.SObj,main.Node
```
这里还要提一点，这里定义的两个方法reciver都是变量值，而不是指针，但其实golang默认的都是按照引用操作的，reciver为指针的版本一定会生成，取值则是按照反引用取值的。所以编译器其实生成了四个函数。 
```
000000000044ec30 g     F .text    0000000000000015 main.SObj.Add
000000000044ed70 g     F .text    000000000000008c main.(*SObj).Add
000000000044ec50 g     F .text    0000000000000019 main.SObj.Sub
000000000044ee00 g     F .text    0000000000000097 main.(*SObj).Sub
```

业务逻辑是在 main.(*SObj).Add和main.(*SObj).Sub 里面，另外两个是包装函数。所以，当reciver为值的时候，你传指针或者值都可以。编译期可以帮你判定搞定。当reciver为引用的时候，你只能传引用，因为编译期没法搞定，因为这种情况就是假定我们可能要修改原对象，你如果传值，值copy之后，就不是原来的值了，这就违背了基本语义。

#### 动态生成 itab

这个主要用在接口<->接口的场景。这种情况的itab表是动态生成的，编译期是没有静态生成的。这种两个都是接口的，你就必须是动态的去找，因为只有在runtime的时候才能知道里面赋值的到底是什么具体类型。对于itab会有一个全局hash缓存表，缓存优化性能。

举个例子： 

```go
package main

type Student struct { name string }

type Ione interface { getName() string }
type Itwo interface { getName() string }

func (s Student) getName() string { return s.name }
func main() {
var i Ione
var t Itwo

    s := Student{ name:"concrete obj" }

    i = s
    t = i // interface 2 interface

    _ = t
}
```

其中由于 i = s 是concrete类型到interface的赋值，这里编译器可以直接生成静态的itab结构。而由于 t = i 是接口到接口的赋值，这个编译器是无法去生成静态itab的（不要看到这里的简单例子，觉得自己肉眼可以在编译期间确定静态itab表。如果放到项目的大环境，比如纯操作接口的一个函数里面，你是无法分析出interface是否绑定到某个对象）。
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-6008d90969731b05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
（旁白：动态的有点是更灵活，缺点是现场性能损耗）


t = i 是调用 convI2I 转换函数生成 itab <Itwo, Student>。 
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-b01775b5019fcd87?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关键路径：
```
convI2I -> getitab
            -> new itab & itab.init
            -> itabAdd
```

---

坚持思考，方向比努力更重要。关注我：奇伢云存储

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)