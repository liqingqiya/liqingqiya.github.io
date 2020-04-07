---
layout: post
title:  "一文读懂 对象存储S3访问姿势"
date:   2020-04-07 23:31:51 +0800
categories: 对象存储 云存储 存储 对象
---

S3支持三种资源访问方式：

- Path Style URL
- Virtual-hosted Style URL
- 自定义域名

---

本质上，对象存储使用bucket，key来唯一标识一个对象，所以只要你告诉对象存储服务端这两个东西，那么理论上就能定位到这个数据。以上三种方式，总归都是为了获取到（bucket，object_key）。

## Path Style URL

在Path Style URL中，bucket的名字紧跟在domain之后，成为URL path的一部分。
```
http://s3endpoint/BUCKET
```
比如，如果有一个photo.jpg存放在region为us-west-2，bucket为images的bucket中。可以用以下方式来访问：
```
http://s3-us-west-2.amazonaws.com/images/photo.jpg
```
**重点：**

- 所有用户请求Host相同（旁白：在鱼龙混杂的互联网环境下，这种方式有个坑，思考下？）
- bucket和key在URL里面：/ {bucket} / {key}

## Virtual-Hosted Style URL

在Virtual-Hosted Style URL 中，bucket的名称成了subdomain：

```
http://BUCKET.s3endpoint
```
比如，如果有一个photo.jpg存放在region为us-west-2，bucket为images的bucket中。可以用以下方式来访问：
```
http://images.s3-us-west-2.amazonaws.com/photo.jpg
```
推荐使用Virtual-Hosted Style的访问方式。因为这个可以提高访问性能，少一跳。

**重点：**

- bucket取自host一部分
- 通过泛域名解析到公有云厂商服务器上

## 自定义域名

这个是初学者最难理解的一种访问方式。先说一个具体的例子，如果你要使用自定义域名下载访问对象，怎么操作？

1. 首先，用户需要自己搞定一个能用的域名，并且把这个域名cname到你需要访问的S3 endpoint；
2. 其次，用户在厂商提供的对象存储的管理界面上配置绑定这个域名到某个bucket；（旁白：这个只是存储一个map映射：域名到bucket的映射）

准备好了前面两个步骤，你就可以用自定义域名来访问资源：

```
// 注意：这里不需要指定bucket，只需要指定对象key
http://${自定义域名}.com/photo.jpg
```

解释下这两个步骤的作用：

- 第一个步骤：用户负责S3请求发到S3的服务器上，用户负责这个路径的连通
- 第二个步骤：对象存储服务端 会创建一个map，负责解析这个域名到bucket的映射（旁白：对象存储服务器说，只要你请求发的过来，我就能找到这个域名对应的bucket）

---

AWS S3的厂商推荐使用的是Virtual-hosted style URL。提供这种方式访问的厂商，要能支持泛域名解析。其实对于公有云厂商，更愿意推荐你使用自定义域名的方式。为啥？可以思考下。

举个例子，如果用户使用的是Path Style访问，那么Host使用的就是厂商统一的域名，如果用户在云存储上放置了一些非法的内容，很有可能会连累存储厂商的自身域名被封，这样的影响是非常严重的，会导致整个存储服务挂掉，影响所有用户。

---

$\color{#ea4335}{坚持思考，方向比努力更重要。}$ 
**关注公众号：奇伢云存储**

![关注我，获取更多干货](https://upload-images.jianshu.io/upload_images/14414032-60dcb5bfc0c60873.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
