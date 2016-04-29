title: 【日常】 列表页优化
---
### 记录一下项目中列表页优化过程。
 
#### 页面布局
- 第一个关注的是列表页中item的布局层数。布局的层数越多，视图树的深度就越大，会增加测量和布局的时间。具体优化的方法是：使用RelativeLayout代替LinearLayout布局，在使用include来引入布局的时候，一定的情况下可以使用merge标签来减少一层viewgroup。
- 使用viewstub，有些布局可能暂时被隐藏，可以使用setVisiable来控制可见性，但是相对于viewstub来占位，这样的性能消耗会大一些。
- 在项目中有一个九宫格的布局，里面每次都会判断图片的个数，把图片加载进这个布局中，每次setData的时候都会inflate一个布局，然后findview，inflate是比较耗时的，我采用了列表页用viewholder做缓存的方法，这样就不用每次都inflate了。

#### 使用工具
- 在做完上面这些事后，列表页卡顿还是比较严重，但是没有找到特别明显的造成卡顿的原因，于是是时候用一波工具了。我使用了两个工具。第一个是traceview，一个用来检测每个方法占用cpu时间的工具。第二个工具是hierarchyviewer，这是一个检测布局，控件测量布局与绘制时间，以及层级关系的工具。
- 在使用traceview后，我发现pattern.compile这个方法占用了很多的cpu时间。因为列表中需要匹配表情，话题等，所以用到了正则表达时，而且每次都去调用pattern.compile这个方法，造成了很大的开销，解决方案是pattern将作为全局变量，而不是局部变量，这样就不用每次都去pattern.compile了。
- 还发现了另一个问题，textview的append（spannableString）这个方法占用了很大的cpu时间。查找资料后发现有spannableStringBuilder这个类，可以直接调用append进行拼接，cpu占用时间大大减小，大约只使用了text的append方法的1/4。
- 在获取一张本地图片并裁剪的时候，使用ContextCompat获取drawable后转成bitmap比直接decode出一个bitmap效率要高。
- 使用traceview的时候发现列表页中一张图的绘制时间是2.7ms而超过16.7毫秒，就会掉帧，大约5张图（加上别的地方的耗时）就必然会造成卡顿。因此有必要在滑动的时候不去加载图片，监听滑动事件。现在很多主流app都是这么干的。