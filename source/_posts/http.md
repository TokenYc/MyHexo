title: 【日常】网络请求总结
---
看完一篇文章[Android网络请求心路历程](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0105/3828.html)后总结一下

#### http请求
 - 请求行：请求方式，url，协议版本
 - 请求头
 - 请求体

#### http响应
 - 状态行：协议版本，状态码，状态码说明
 - 响应头
 - 响应体
 
#### get请求和post请求的区别
 - get的参数放在请求行中(get参数中文需要编码)
 - post的参数经过编码后放在请求体中

#### 数据在客户端和服务端的传递
 - 对象（客户端）<--> 文本（http传输）<--> 对象（服务器）

#### Json解析库
 - Google：GSON
 - 阿里：FastJson

#### HttpURLConnection
 - POST：从URL获取HttpURLConnection对象，设置请求方式，获取输出流。调用connect()方法，获取输入流。
 - GET：和POST类似，把获取输入流去掉。

#### 同步和异步
 - 同步：在UI线程执行操作
 - 异步：在非UI线程完成操作后通过handler在UI线程更新UI

#### Etag
 - 客户端Etag<-->服务端Etag
服务端根据Etag判断服务端数据是否修改过。未修改返回304.实际上就是客户端用来询问服务器，客户端的缓存是否有效。

#### 图片加载
 - 特点：url不变，占内存很大，加载时间长，需要占位图。
 - 通常两级缓存：内存，硬盘。（lrucache）
 - 优秀的库：fresco，glide

#### 图片管理方案
 - 七牛，阿里云。可以在传输图片url上添加一些参数来处理图片