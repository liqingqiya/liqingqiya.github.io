---
layout: post
title:  "python 怎么走 socks5 代理？"
date:   2020-4-12 2:44:09 +0800
categories: python socks5 代理 透明
---

[TOC]

# Python 怎么使用 Socks5 协议？

Python 有一个库 PySocks ，这个库非常方便你使用 socks 代理协议，比如有些时候，你的 Python 程序需要发送一个 HTTP 请求到某个机器，但是网络不能直接连接，需要走跳板机，走代理，那么就可以使用这个库让你偷偷的走 Socks 代理，业务完全无感知的，请求就发往了机器（但其实是走了代理）。

## Pip 安装

```sh
pip install PySocks
```

## 使用场景

### 官方例子

**官方使用例子**：

```python
import socks

# 建立一个操作句柄；
s = socks.socksocket() # Same API as socket.socket in the standard lib

# 指明代理服务器和端口
s.set_proxy(socks.SOCKS5, "localhost", 8888) 

# 走代理发 HTTP 请求
s.connect(("www.somesite.com", 80))
s.sendall("GET / HTTP/1.1 ...")
print s.recv(4096)
```

上面的例子，还是不够优美，你仔细看看，业务的逻辑和代理直接耦合在一起了。所以这种是侵入式的，适用场景有限。

### 更通用的场景

更多的场景是，我业务代理已经有了，配置什么的都是直接的 target ，这样直接通信。线上跑当然没问题，但是如果我是在本地电脑上调试，如果网络不能直接连通，只能通过跳板机，那 Python 程序在本地便跑不起来。

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-06-03-python-socks5/C5483538-5A6B-4E70-912C-3635ED3B7E54.png)

这个时候，就可以用到 Pysocks 的 Monkeypatching 功能，就可以业务无感知的使用到代理，什么叫做业务无感知？就是业务完全不改代码，自己都不知道，就走了代理了。

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-06-03-python-socks5/F71F94FB-B458-441F-994A-F33B1201EB57.png)

**举例**：

```python
import urllib2
import socket
import socks

# socks5 猴子补丁
socks.set_default_proxy(socks.SOCKS5, "localhost", 8888)
socket.socket = socks.socksocket

# 业务代码（完全无感知）
urllib2.urlopen("http://www.somesite.com/")
```

## PySocks 库原理

PySocks 整个库就两个文件：

1. socks.py
2. sockshandler.py 

其中最核心的就是 socks.py 文件里，最核心的实现。sockshandler.py 只是一个简单的封装。

我们取一些代码片段，来看看最核心的原理。

### 设置 Proxy 

```python
def set_default_proxy(proxy_type=None, addr=None, port=None, rdns=True,
                      username=None, password=None):
    """Sets a default proxy.
    All further socksocket objects will use the default unless explicitly
    changed. All parameters are as for socket.set_proxy()."""
    socksocket.default_proxy = (proxy_type, addr, port, rdns,
                                username.encode() if username else None,
                                password.encode() if password else None)
```

这里只是创建了一个元组，把代理服务器地址，端口，鉴权等信息保存下来，以待后用。

### socksocket 类

最核心的实现就是这个类了，connect，bind 的实现就不说了，是 patch 的实现。

```python
class socksocket(_BaseSocket):
    """
    代理协议的实现 + money patch 的实现
    """
    
    default_proxy = None

    def bind(self, *pos, **kw):
        """Implements proxy connection for UDP sockets.
        Happens during the bind() phase."""
        
    @set_self_blocking
    def connect(self, dest_pair, catch_errors=None):
        """
        Connects to the specified destination through a proxy.
        Uses the same API as socket's connect().
        To select the proxy server, use set_proxy().
        dest_pair - 2-tuple of (IP/hostname, port).
        """    
        
    def _negotiate_SOCKS5(self, *dest_addr):
        """Negotiates a stream connection through a SOCKS5 server."""

    def _negotiate_SOCKS4(self, dest_addr, dest_port):
        """Negotiates a connection through a SOCKS4 server."""

    def _negotiate_HTTP(self, dest_addr, dest_port):
        """Negotiates a connection through an HTTP server.
```

其中，`_negotiate_SOCKS5`，`_negotiate_SOCKS4`，`_negotiate_HTTP` 这三个函数是代理实现，从名字上也能看出来，分别是 Socks5，Socks4，HTTP 的代理实现。就以 Socks5 的实现是 `_SOCKS5_request` 函数。

```python
    def _SOCKS5_request(self, conn, cmd, dst):
        """
        Send SOCKS5 request with given command (CMD field) and
        address (DST field). Returns resolved DST address that was used.
        """
        proxy_type, addr, port, rdns, username, password = self.proxy

        writer = conn.makefile("wb")
        reader = conn.makefile("rb", 0)  # buffering=0 renamed in Python 3
        try:
            # 按照数据格式鉴权
            
                # 认证成功

            # 如果没有鉴权，那么代理服务器返回的是 0x00
            elif chosen_auth[1:2] != b"\x00":
                # 如果鉴权失败，那么返回的是 0xff
                if chosen_auth[1:2] == b"\xFF":
                    raise SOCKS5AuthError()
                else:
                    raise GeneralProxyError()

            # 鉴权成功，则代理服务器可以和后端建立连接了，那么则可以把一些信息发给代理服务器了 
            writer.write(b"\x05" + cmd + b"\x00")
            resolved = self._write_SOCKS5_address(dst, writer)

            # 获取到代理服务器的响应
            resp = self._readall(reader, 3)
            if resp[0:1] != b"\x05":
                raise GeneralProxyError()
                
            # 获取代理服务器返回的 处理IP和端口
            bnd = self._read_SOCKS5_address(reader)

            # 这个搞完，就可以传输数据了
```


# 什么是 Socks5 协议？

Socks5 协议是基于 TCP/IP 之上的代理协议。按照之前5层协议的说法应该也算是应用层协议，但更准确的，按照 OSI 7层模型，Socks 属于会话层协议，在表示层和传输层之间，具体定义可以查看 Wiki 。我们经常看见 Socks 协议可以作为一个透明协议，上面可以跑其他的的应用层协议，比如最常见的 HTTP 协议就可以跑在 Socks 协议上面。

思考下，HTTP 本身也有代理协议的实现，那么和 Socks 代理协议对比，有什么不一样？最关键的还是，HTTP 的代理实现，是 HTTP 应用协议本身实现的代理协议，是业务本身要强感知的，比如我们经常看到 HTTP 代理的时候，去掉一些 HTTP Header ，不像 Socks 是透明的代理协议，是完全不感知自己传输的数据的。

用简单的话说下，这个代理协议到底是个什么代理协议？

**举个例子**：

有 A，B，C 三个人，A 要给 C 说句话，但两人天各一方，B 神通广大，可以联通两人，A 就把要说的话，写在纸上，用信封装好，把信给到 B，B 怎么才知道这个信是给 C 的呢？那么 A 就要明明白白的在信封上把情况在信封上说好，给谁的，这个格式就是 Socks 协议格式（旁白：B的心里话，只要给我的信上把事情说明白了，那我就知道怎么办了 ）；

在这个例子里面，A 就是 Client，B 就是 Proxy（或者也是 Socks Server），C 事 Target 
。

## Socks5 协议

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-06-03-python-socks5/62069882-4477-4343-AF96-056B7741287F.png)

Socks5 协议本身非常简单，分为传输数据之前，会经历两次 RTT ，四个请求：

1. 协商鉴权；
2. 响应鉴权结果，是否支持代理，就在此响应了；
3. 客户端上报目标信息（发给谁的）
4. 代理服务器回应

### 鉴权查询

数据包格式：

```
+----+----------+----------+
|VER | NMETHODS | METHODS |
+----+----------+----------+
| 1　| 　　1　　| 1 to 255 |
+----+----------+----------+
```

字段解析：

- VER：Socks版本（在socks5中是0x05）；
- NMETHODS：在METHODS字段中出现的方法的数目；
- METHODS：客户端支持的认证方式列表，每个方法占1字节。

`METHODS` 字段指明类型，0x00 标示不鉴权。如果代理服务器不支持的，可以回复 `0xff` ，如果一切Ok，则可以恢复 `0x00` 。

这个阶段核心在于协商，客户端告诉代理服务器他想要的是什么，需要知道服务端支持什么。

### 鉴权响应

数据包格式：

```
+----+----------+
|VER | NMETHOD |
+----+----------+
| 1　| 　　1　　| 
+----+----------+
```

字段解析：

- VER：socks版本（在Socks5中是0x05）；
- METHOD：服务端选中的方法（若返回0xFF表示没有方法被选中，客户端需要关闭连接）；

METHOD字段的值可以取如下值：

- 0x00: NO AUTHENTICATION REQUIRED
- 0x01: GSSAPI
- 0x02: USERNAME/PASSWORD
- 0x03: to X’7F’ IANA ASSIGNED
- 0x80: to X’FE’ RESERVED FOR PRIVATE METHODS
- 0xFF: NO ACCEPTABLE METHODS

这个阶段核心在于：告诉客户端自己的选择。

### 代理信息

这个请求是由 Client 发往代理服务器的，明白告诉代理，我要把信息发给谁（ip:port），这个阶段一般来讲，代理就会和 Target 建立网络连接。

数据包格式：

```
+----+-----+-------+------+----------+----------+
|VER | CMD |　RSV　| ATYP | DST.ADDR | DST.PORT |
+----+-----+-------+------+----------+----------+
| 1　| 　1 | X'00' | 　1　| Variable |　　 2　　|
+----+-----+-------+------+----------+----------+      
```

字段解析：

- VER 版本号，指明 Socks 协议版本，Socks5 即是 0x05
- CMD 命令类型
    - CONNECT X'01'
    - BIND X'02'
    - UDP ASSOCIATE X'03'
- RSV 保留字段
- ATYP 地址类型
    - IP V4 address: X'01'
    - DOMAINNAME: X'03'
    - IP V6 address: X'04'
- DST.ADDR 目标地址（告诉代理服务器发给谁）
- DST.PORT 目标端口

服务器把客户端收到的这个数据包收到之后，鉴权，版本，等等都做好了。就该处理数据了。


### 代理建立响应

代理服务器发给 Client 的响应，告诉自己已经准备好了，自己的服务IP地址等。这个请求之后，客户端就可以发数据了，代理服务器直接做转发。

数据包格式：

```
+----+-----+-------+------+----------+----------+
|VER | REP |　RSV　| ATYP | BND.ADDR | BND.PORT |
+----+-----+-------+------+----------+----------+
| 1　|　1　 | X'00' |　1 　| Variable | 　　2　 　|
+----+-----+-------+------+----------+----------+
```

字段解析：

- VER 版本号
- REP 响应字段（reply）
    -  X'00' succeeded
    -  X'01' general SOCKS server failure
    -  X'02' connection not allowed by ruleset
    -  X'03' Network unreachable
    -  X'04' Host unreachable
    -  X'05' Connection refused
    -  X'06' TTL expired
    -  X'07' Command not supported
    -  X'08' Address type not supported
    -  X'09' to X'FF' unassigned
- RSV 保留字段
- ATYP 地址类型
    - IP V4 address: X'01'
    - DOMAINNAME: X'03'
    - IP V6 address: X'04'             
- BND.ADDR 代理服务器地址
- BND.PORT 代理服务器端口

# 分布式系统里要用 Socks 协议？

公有云厂商考虑到机房故障域，会有多个机房，会在多个地域部署数据中心，这样线上一个机房故障，甚至某整个地域故障也能保证可用性。但这机房之间，地域之间是要通信的，中间隔着公网，一般有几个方法：

1. 要么就是走公司的专门的 vpn 网络；
2. 要么走专线网络，两个机房相当于全都在局域网中；
3. 要么就走网络的代理，这 Socks 代理就是一个选择。能够让两个机房的内网组件穿透公网进行通信；

**比如下图**：

1. 组件1 要发消息给 组件5；
    1. 组件1 配置填的了组件5的ip，注意了是内网ip
2. 组件1 发消息到 Socks 代理服务器 A 上，代理服务器一看地址，判断了下，发现本机房没有这个 ip ，于是发给机房2的 Socks 代理服务器 B；
3. Socks 代理服务器 B 一看ip，确认组件5在自己机房，于是把消息发给组件5；
4. 组件5抽到消息之后，响应原路返回；

![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-06-03-python-socks5/914E0C33-D88E-4EF8-9AF6-857064DE5A36.png)

这里有个点，希望大家能想通：

1. Socks 代理服务器 A 和Socks 代理服务器 B 既要和内网组件通信，又要过外网，所以要绑定两个 ip，内网 ip 和外网 ip ；
2. 那是否有人有疑问，这不说明有能通外网吗？那让组件1和组件5直接外网通信不就行了。想想，这个是不行的，这个相当于把内网的所有组件全都暴露到外网了，这个有严重的安全问题。但凡公网通信，必要严格鉴权；

这个内网组件和 Socks A，B 的通信协议格式就可使用我们说的 Socks 协议，当前最常用的就是 Socks5 协议。

Socks 协议是一个极简的协议，用于透明的数据代理，配合端口转发，有很多有意思的用法。之后再分享。

---
坚持思考，方向比努力更重要。微信公众号关注我：奇伢云存储
![关注我公众号, 获取更多干货](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/wechat_public_no.png)

