---
title: "Timestamp Transformation in Uniapp"
date: 2019-08-22 00:14:53
tags: [小程序]
category: [Debug]
---

## 记一次 uniapp 上的时间格式转换

**背景**：当前项目聊天业务需要显示聊天消息的创建时间。在网上搜索一番之后，发现很多都是答非所问，所以决定自己探索一番，并记录下这个有趣的过程。

<!-- more -->

**业务流程**：
	1. 客户端发消息时，本地创建聊天对象加到缓存中并渲染给页面
	2. 将聊天对象发送给服务器
	3. 服务器将聊天对象转发给对方用户

为了实现该功能，我先分别打印了来自 java 后端和 uni-app 获取时间方法的结果。发现他们的格式不一样，如图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190821231519466.png)

从后端拿到的是 java.util New Date() 之后的原始长整数（我称之为原始时间），
而在 uni-app 里直接 new Date() 后，得到是一个经格式化后的长字符串。

我想，按理来说，考虑到客户端时区的差异，我们最好在客户端对原始时间进行解析。
而两个用户可能处在不同的时区，所以唯一的方法就是传递原始时间。

所以在uni-app如何获取未经过格式化的原始时间，已经如何解析呢？

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190821235003442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODM1NDk2,size_16,color_FFFFFF,t_70)
可以看到这边 uni-app 其实给出了很多方法，其中 getTime() 可以获取一个 numeric universal time (我所指的原始时间)。也就是说我们储存和传输这个时间就可以了。
那么我们如何解析它呢？
经过一段时间的尝试... 我直接提供简单的示范代码吧。

```
var time = new Date().getTime() // 获取原始时间
// 通过服务器接收后...
var newTime = new Date(time) // 把接收到的时间转换为 Date 类型
// 其实这样就可以了 可以直接通过 Date 类型自带的 get 方法解析，比如：
console.log(newTime.getDate());
console.log(newTime.getDay());
```
这是我 getDate() 之后的结果，这样就成功获取并解析到啦。

![在这里插入图片描述啊](https://img-blog.csdnimg.cn/20190822001210866.png)
**END...**