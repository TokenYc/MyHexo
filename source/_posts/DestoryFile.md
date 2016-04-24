title: 【黑科技】让qq，微信无法读取图片和文件
---

![日常](http://img4.duitang.com/uploads/item/201301/30/20130130010434_jy3hH.thumb.600_0.jpeg)

#### 事件的经过

在开发程序的时候碰到一个问题：无法在自己手机的SD卡中创建应用的文件夹。别的测试机就可以。而且改个名字就可以创建了。看了半天没找到原因，后来终于发现在SD卡下有个和应用同名的文件，注意，是文件，不是文件夹。我把它删了，果然就可以创建同名的文件夹了。

#### 结论

如果目录中存在同名的文件，则同名的文件夹无法创建。

#### 尝试

就想了下，要是把qq的文件夹删了，再创建个同名文件会不会出现同样的问题。发现只要tencent这个文件夹被删，立马会重新创建。腾讯应该是通过什么方法对这个事件进行了监听，被删除了就重新创建文件夹。看来，手动的不行，那就写代码吧。在代码里，顺序执行，删除后立马创建tencent文件。运行一下，果然，tencent文件夹不能创建了。然后打开QQ，出现了如图的情况。
![截图](http://7xpp4m.com1.z0.glb.clouddn.com/destoryfile.png)

既然提示重启了，那就重启吧。重启过后，发现tencent文件夹又出现了，当然，tencent文件已经被删除。既然如此，那就监听手机开机事件，然后在tencent文件夹创建之后再把它删了，创建我们的tencent文件。最终成功了。问题是自启动需要手动打开，miui的安全中心默认是关闭你的应用的自启动的。
然后就无限读取不了图片和文件了。

[源码github地址](https://github.com/TokenYc/DestoryFile)

请勿用作不良用途，仅供娱乐。
