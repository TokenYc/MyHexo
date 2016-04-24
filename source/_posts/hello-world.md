title: Hexo的坑
---
终于完成了，折腾了几天，Hello World！

## 遇到的坑

### deploy失败

deploy后没有任何反应，一开始以为是正常的，git不就是这样嘛，然后到io上一看404，到github上一看，仓库没有任何变化。查了以后发现，犯了两个错。
 
- 在：之后没有加空格。（yaml必须这么做）
- 教程很多是2.+的hexo。3.+的模块已独立，需要自己添加模块（npm install hexo-deployer-git --save ）。
- git版本问题。

### cname

cname千万不要加拓展名，会无效的。一开始加了个.txt，喵了个咪。然后要放到theme下面的source文件夹下。(cname使你的域名指向github.io)

