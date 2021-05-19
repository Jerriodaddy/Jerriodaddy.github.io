---
title: "Url Parsing Error: 小程序页面跳转传参解析错误 —— Unexpected end of JSON input"
date: "2019-11-27 18:09:34"
tags: [小程序, Error] 
category: Debug
---
![截图1](https://img-blog.csdnimg.cn/20191127172333593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODM1NDk2,size_16,color_FFFFFF,t_70)

在小程序 A->B 页面跳转时，B页面对A界面传入参数解析失败, 打印传参结果发现...
<!-- more -->

今天发现一个问题，记录一下。
在小程序 A->B 页面跳转时，B页面对A界面传入参数解析失败, 打印传参结果发现

![截图1](https://img-blog.csdnimg.cn/20191127172333593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODM1NDk2,size_16,color_FFFFFF,t_70)

数据在 faceImg 处被截断。 faceImg 原数据为 https://timgsa.baidu.com/timg?image&quality=80&size... .jpeg(太长了我就截取了部分)
首先排除字符串过长溢出的情况（因为这段输出太短不至于溢出）
可以注意到字符串在 timg? 问号处被截断，
因此，判断出错原因是图片的url中的特殊字符“？”被识别导致错误的解析。
根据此原因对代码进行修改：

A页跳转原代码：
```javascript
uni.navigateTo({
	url: "../chatpage/chatpage?friendInfo=" + JSON.stringify(friendInfo),
});
```
B页接收原代码：
```javascript
onLoad(opt) {
	this.friendInfo = JSON.parse(opt.friendInfo); //解码
}
```
修改后代码：
A页：
```javascript
var encodeData = encodeURIComponent(JSON.stringify(friendInfo)); // 对数据字符串化并转码，防止特殊字符影响传参
// console.log(decodeURIComponent(encodeData));
uni.navigateTo({
	url: "../chatpage/chatpage?friendInfo=" + JSON.stringify(friendInfo),
});
```
B页：
```javascript
onLoad(opt) {
	this.friendInfo = JSON.parse(decodeURIComponent(opt.friendInfo)); //解码
}
```

我在传参前使用 encodeURIComponent() 了对传输字符串做了转码，并在对应接收处解码。
修改过后，成功获取到正确参数。
转码后字符串如下图所示：

![转码后字符串](https://img-blog.csdnimg.cn/20191127180708629.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODM1NDk2,size_16,color_FFFFFF,t_70)

附上 encodeURI() 和 encodeURIComponent() 区别：
https://blog.csdn.net/qq_34629352/article/details/78959707
