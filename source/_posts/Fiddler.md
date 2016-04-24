title: 【日常】使用Fiddler抓包
---
知乎上看到一个问题是，现在有什么有趣的api，赞最多的答案是自己去抓 =。=  那就抓一下吧

### 原理
Fiddler使用本地127.0.0.1代理。可以设置代理的浏览器和应用都可以监测。
### 准备
 - 安装Fiddler -。-废话
 - 设置可以远程连接(为了抓app的包)
 - 设置https(如果要抓取https的话)

### 安装

官方下载地址： [下载地址](http://www.telerik.com/fiddler)

下一步------下一步------下一步------........

### 抓取浏览器请求
进入后打开某个网页。
![进入程序就可以了](http://7xpp4m.com1.z0.glb.clouddn.com/blogfiddler1.png)

### 设置远程连接

目的是让手机可以使用fiddler的127.0.0.1：8888代理。手机和电脑必须在同一网络下才行。
最上面一栏 Tools-->Fiddler options

![](http://7xpp4m.com1.z0.glb.clouddn.com/blogfiddler2.png)

手机上要设置wifi代理

![](http://7xpp4m.com1.z0.glb.clouddn.com/blogfiddler4.png)

设置好后进入某一app就会抓取到请求了

### 设置https

![](http://7xpp4m.com1.z0.glb.clouddn.com/blogfiddler3.png)

手机上用浏览器访问: 主机名+端口  下载证书

![](http://7xpp4m.com1.z0.glb.clouddn.com/blogfiddler5.png)

