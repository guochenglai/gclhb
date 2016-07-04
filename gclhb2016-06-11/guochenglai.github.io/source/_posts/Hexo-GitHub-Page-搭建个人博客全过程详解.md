---
title: Hexo-GitHub-Page-搭建个人博客全过程详解
toc: true
mathjax: true
date: 2016-05-31 19:59:54
categories: 杂谈
tags:
- Hexo
- GitHub Pages
- 搭建博客
description: 本站博客是基于Hexo和Maupassant搭建的，代码服务托管在GitHub Pages上。所有的代码均来自网络。感谢那些乐于奉献的大神将自己开发的工具分享到网络并让我们能够自由使用和传播。才使得我能在如此之小的时间成本和金钱成本之内完成自己的博客搭建。在之前我也层用Java编写过自己的博客，最后考虑到各种成本，还是切换到本文所提的方法。在我的第一篇博客中提到过，当我觉得我的博客已经开发完成之后，就公开我的博客的搭建方法，并开放源代码。今天终于付诸实现了。分享真是给人带来快乐的一件事情...
---
# 综述
## 框架的选取
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上大学的时候，自己曾经折腾过一个网站，采用MySQL+Java+JS编写的，开发成本很大，而且还要买服务器，运营成本也很大，后来就下线了。然后开始在CSDN上面写博客，但是写了一段时间发现CSDN的用户体验非常不好。比如：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1 文章需要审核，而且我猜测还是人工审核，写一篇文章发布上线之后少则几分钟可以看到，多则几小时才能看到。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2 文章不能在线预览。500的错误特别多。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3 会在你的文章上面添加广告。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以又准备自己搭建一个博客系统。希望能够开发成本和运营成本都比较小。最后终于找到了Hexo+GitHub Pages的搭建方式。他的优点如下：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1 不需要你编写后端代码。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2 只需要你略懂html即可。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3 在GitHub Pages上面有免费的300M空间。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4 代码开源，并且有各路大神提供自己的开源样式。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5 支持MarkDown语法。
## 整体运行原理
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hexo是基于node.js的，所有的文章最后都被编译成了静态的HTML，所以你搭建一个博客系统并发布到线上从整体来看只需要如下三步：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1 在本地搭建一个博客系统。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2 在本地编写博客并编译
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3 将本地编译的文件提交到GitHub Pages上。
     
## 需要注意的问题
<font color=red>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**因为Hexo所有的博客编写都是在本地完成，然后编译提交到线上，所以你所有的劳动成果都在本地机器，请记住在你每次比较大的改动网站之前进行备份。最好能够定时进行备份。**</font>
    
# 本地博客搭建
1 下载并按照node.js，这个可以直接去官网下载，然后双击按照，一路点击“下一步”最后到安装完成即可。
2 使用node.js安装hexo
   npm是node的包管理工具，所以所有按照node插件的地方都是用npm命令来按照。如果你按照的插件被墙了，请切换到淘宝的数据源cnpm。
```shell
npm install hexo-cli -g 或者 npm install -g hexo 
```
3 在你的本地初始化一个hexo工程
```shell
hexo init guochenglai.github.io
```
4 进入hex工程
```shell
cd guochenglai.github.io
```
6 安装hexo的依赖
```shell 
npm install
```
7 编译整个项目
```shell
hexo generate 
```
8 运行整个项目
```
hexo server
```
快捷命令：
```shell
    hexo g == hexo generate
    hexo d == hexo deploy
    hexo s == hexo server
    hexo n == hexo new
    还能组合使用，如：
    hexo d -g
```
9 浏览器测试
    http://localhost:4000
出现以下界面则表示本地站点搭建成功。
![](http://7xutce.com1.z0.glb.clouddn.com/hexo%2Fnew_hexo_server.png)
# 配置maupassant样式
1 安装maupassant的主题和渲染器
```shell
git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant

npm install hexo-renderer-jade --save
npm install hexo-renderer-sass --save
```
2  将<font color=red>Hexo</font>目录下的_config.yml的 theme字段由landscape改为maupassant然后重新执行 hexo server会发现页面的样式变为如下图所示：
![](http://7xutce.com1.z0.glb.clouddn.com/hexo%2Fhexo_maupassant_init.png)
3  将<font color=red>Hexo</font>目录下的 _config.yml的 language 字段设置为 zh-CN（<font color=red>注意要有空格</font>）然后重新执行hexo server会发现页面会变成中文。

至此你的博客基本功能已经完整了。如果你想将就着直接用，就可以直接查看“部署到GitHub Pages”如果你想丰富一下网站的功能可以继续查看“定制maupassant”
# 定制maupassant
1 为网站添加评论功能
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本网站的所有评论功能都托管给了“多说”，要想添加评论需要如下的几步:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.1 注册多说 [http://duoshuo.com/](http://duoshuo.com/)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.2 将多说分配给你的用户名添加到<font color=red>maupassant</font>配置目录下的_config.yml中，如下：
```
duoshuo: guochenglai ##这里“guochenglai”是我申请的多说用户名，你改成你自己的即可。
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.3 重新编译之后重启服务就可以使用评论功能了。
2 为搜索框添加本地搜索功能
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.1 修改<font color=red>maupassant</font>配置目录下的_config.yml如下：
```shell
self_search: true ## Use a jQuery-based local search engine, true/false.
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.2 安装  [hexo-generator-search](https://github.com/PaicHyperionDev/hexo-generator-search) 执行如下命令：
```shell
npm install hexo-generator-search --save
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3 重启服务然后在搜索框就可以搜索本站点的数据了。
3 添加关于我的页面
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.1 执行如下命令创建页面
```shell
hexo  new page “about"
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.2 页面的路径会在 ~/hexo/guochenglai.github.io/source/about/index.md
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3 修改页面到自己喜欢的样式
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.4 重新编译运行项目就可以看到关于我的页面了
4 添加谷歌分析
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.1 使用 Google 帐号登录 [Google Analytics](https://accounts.google.com/ServiceLogin?service=analytics&passive=1209600&continue=https://www.google.com/analytics/web/provision?authuser%3D0%23provision%2FSignUp%2F&followup=https://www.google.com/analytics/web/provision?authuser%3D0#identifier)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.2 根据提示进行注册操作，完成注册后通过 <font color=red>管理 - 选择用户 - 选择媒体 - 跟踪信息 - 跟踪代码</font> 找到谷歌给你分配的"跟踪 ID"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.3 修改 <font color=red>Hexo</font> 下的_config.yml
配置文件，设置 google_analytics如下：
```shell
google_analytics: UA-7263826 ## 这里填上你的谷歌ID
```
5 404页面定位到腾讯公益
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 <font color=red>hexo/source</font> 目录下创建 <font color=red>404.html</font> 文件或者 <font color=red>404.md</font> 文件。内容如下：
```html
---
layout: false
title: "页面不存在"
---

<!DOCTYPE html>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
        <script type="text/javascript" src="http://www.qq.com/404/search_children.js" charset="utf-8" homePageUrl="http://guochenglai.com" homePageName="回到首页"></script>
    </head>
    <body>
    </body>
</html>
```
6 支持Rss订阅
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;安装hexo-generator-fee
```
 npm install hexo-generator-feed --save
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;修改 hexo 配置 #RSS订阅
```
plugin:
- hexo-generator-feed
#Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20
其中，feed 配置是可选项
```
7 博客模板配置
 模板文件的路径如下：
 ```
 ~/hexo/guochenglai.github.io/scaffolds/post.md
 ```
 修改内容如下：
 ```
---
title: {{ title }}
date: {{ date }}
toc: true
categories:
tags:
description:
---
 ```
 
# 部署到GitHub Pages
1 在 [GitHub](https://github.com/) 上面创建一个项目
2 创建项目如下：
![](http://7xutce.com1.z0.glb.clouddn.com/hexo%2Fhexo_init_github.png)
<font color=red>注意:
&nbsp;&nbsp;&nbsp;&nbsp;1 项目的名字必须为：xxx.github.io
&nbsp;&nbsp;&nbsp;&nbsp;2 必须选择public   
</font>
3 安装并配置Git这个可以在网上搜索
4 安装 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)，安装命令如下：
```
npm install hexo-deployer-git --save
```
5 配置<font color=red>Hexo</font>目录下的_config.yml如下：
```
deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  message: [message]
```
6 在本地运行编译，并提交，命令如下：
```
hexo g -d
```
7 工程提交到github之后可以在浏览器输入：test.github.io就可以在线访问自己的项目了。


# 配置阿里云域名

# 七牛云托管图片

