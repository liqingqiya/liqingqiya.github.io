---
layout: post
title:  "对象存储5个好用S3客户端，推荐给你"
date:   2020-04-11 23:44:09 +0800
categories: 对象存储 S3 客户端
---

有5个非常简单好用 S3 客户端工具，可以方便接入对象存储，让你昂你领成本的上手对象存储，还能够抓一抓 S3 协议的包。

## s3curl

s3curl 是命令行工具，开源免费使用，非常轻量，也是我平时用的最多的一个工具。s3curl 是 perl 写的逻辑脚本， 本质上，就是帮你构造一个合法的 S3 请求，通过 curl 工具发出去。所以你能做到非常基础的行为，了解到 S3 请求的本质。

### 安装

在ubuntu系统上，你直接安装即可：
```sh
apt-get install s3curl
```
安装好之后，需要配置两个东西：你 S3 的 endpoint（旁白：你发给谁，默认是发给AWS的），你账号的 ak/sk。


### 配置

**1. endpoint在那里配置？**

修改 s3curl 这个脚本内容，添加你自己的 endpoint ：

![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-09ad5c33ae651d1a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
s3curl 在那里？`which s3curl` 看下呗。我这里随便填了两个地址，因为我在自己本地搭建了一个 S3 对象存储服务后端，所以这里填写了 `127.0.0.1` ，注意，这里不需要写上端口。

**2. ak/sk在那里配置？**

```sh
cat ~/.s3curl 
%awsSecretAccessKeys = (
    # 你的账号配置；名字可以随便起，待会是要用的； 
    mock => {
        id => '{你的ak}',
        key => '{你的sk}',
    }
);
```

列举对象：
```sh
 s3curl --id=mock  -- http://127.0.0.1:20000/qiya-bucket-1
```

举例：

```sh
hostname$ s3curl --id=mock  -- http://127.0.0.1:20000/qiya-bucket-1 -v|xmllint -format -
*   Trying 127.0.0.1...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 127.0.0.1 (127.0.0.1) port 20000 (#0)
> GET /qiya-bucket-1 HTTP/1.1
> Host: 127.0.0.1:20000
> User-Agent: curl/7.47.0
> Accept: */*
> Date: Sat, 11 Apr 2020 15:22:07 +0000
> Authorization: AWS {你的AK}:{V2签名}
> 
< HTTP/1.1 200 OK
< Vary: Origin,Access-Control-Request-Method,Access-Control-Request-Headers
< x-amz-request-id: VngAAFaVJyUlzQQW
< Date: Sat, 11 Apr 2020 15:22:08 GMT
< Content-Length: 496
< Content-Type: text/plain; charset=utf-8
< 
{ [496 bytes data]
100   496  100   496    0     0   1157      0 --:--:-- --:--:-- --:--:--  1158
* Connection #0 to host 127.0.0.1 left intact
<?xml version="1.0"?>
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Name>qiya-bucket-1</Name>
  <Delimiter/>
  <Prefix/>
  <Marker/>
  <MaxKeys>1000</MaxKeys>
  <IsTruncated>false</IsTruncated>
  <Contents>
    <Key>code.data</Key>
    <ETag>"81c6873073ddf75c5766a60a75b002de"</ETag>
    <Size>13811</Size>
    <LastModified>2020-04-10T10:47:31.000Z</LastModified>
    <StorageClass>STANDARD</StorageClass>
    <Owner>
      <ID>260637563</ID>
      <DisplayName>260637563</DisplayName>
    </Owner>
  </Contents>
</ListBucketResult>
```

你看这个S3协议包都打印出来了，并且你可以配合 fiddler 等工具抓包，甚至改包头等操作，调试S3协议。

## s3cmd

s3cmd 也是一个 S3 客户端工具，命令行式，python 语言开发的，使用起来比 s3cmd 自然丝滑一些，于人的交互性更自然。

### 安装

```sh
apt-get install s3cmd
```

### 配置

调用 `s3cmd --configure` 进行配置，配置完成之后会在主目录生成一个 .s3cfg 文件，你 `cat ~/.s3cfg` 就能看到了。

```sh
s3cmd --configure
```

其中最关键的几个配置：

```sh
access_key = 你的 ak
secret_key = 你的 sk
host_base = localhost:20000         // 你的 S3 服务器 endpoint
host_bucket = localhost:20000/%(bucket)  // path 模式
```

其他的自行斟酌配置，配置项还挺多的，s3cmd 的功能还挺多。

```sh
s3cmd ls  # 列举所有的桶
```

输出例子：

```
hostname$ s3cmd ls 
2020-04-10 10:47  s3://qiya-bucket-1
```

## CloudBerry

免费的开源工具，这个是可视化的，比较容易入门，更人性化一些，同样的，还是两件事：

1. 配置好 endpoint
2. 配置好 ak/sk

配置好账号：
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-9a5f72d6d5d7416e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就能显示所有的 bucket

这个操作就简单了，可视化操作，双击能打开目录，两边拖放就能传输文件，下载文件之类的。并且这个还能指定同步目录，把本地的某个目录自动同步到某个 bucket 。

工具链接：http://www.cloudberrylab.com/

## s3 brower

这个也是可以免费使用的 s3 客户端工具，非常方便能够通过 s3 协议接入对象存储后端。

### 配置

![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-83f255e13b249422?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这就非常方便的能够通过这个工具进行对象存储的测试，比如你可以捕获这个 s3 请求，研究或者修改包：

![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/14414032-e7c71476ceb063c0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这种可视化工具，非常方便让你拖放的上传下载文件，并且该工具会自动帮你做分片上传等操作，这个是好处，也是不方便的地方。这也是我更喜欢使用 s3curl 的原因；

工具链接：http://s3browser.com/


## s3fs

本质这个不算一个客户端工具，而是针对s3协议封装的文件网关，本质上是一个 c/c++ 实现的 fuse 项目，对外提供 posix 文件系统，后接的通过 s3 协议连接的对象存储，s3fs 只是做了一个协议转换。

### 安装

```sh
apt-get install s3fs-fuse
```

### 配置

创建 ak/sk 配置

```sh
echo AK:SK > ~/.passwd-s3fs
```

挂载到操作系统：

```sh
s3fs qiya-bucket-1 /data/s3fs -o passwd_file=/root/.passwd-s3fs -o url=http://s3.example.xxx.com -o use_path_request_style
```

代码链接地址：https://github.com/s3fs-fuse/s3fs-fuse

---
关注我，获取更多干货
![qrcode_for_gh_77bf4987514d_430.jpg](https://upload-images.jianshu.io/upload_images/14414032-75b2140619d8e1fb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
