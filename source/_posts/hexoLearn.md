---
title: Hexo搭建博客
date: 2018-11-1 20:11:42
tags:
---
# 使用Hexo搭建博客

------
# Hexo搭建
&ensp;&ensp;&ensp;&ensp;Hexo是一个快速、简洁且高效的博客框架。Hexo使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。Hexo是一个采用nodejs的静态博客，类似的博客也有很多，比较有名的Jekyll，Octopress等。Hexo官网https://hexo.io/zh-cn/。
&ensp;&ensp;&ensp;&ensp;1、按照官网教程首先安装nodejs，下载地址：
```text
http://nodejs.cn/download/
```
&ensp;&ensp;&ensp;&ensp;2、安装git，下载地址：
```text
https://git-scm.com/downloads
```
&ensp;&ensp;&ensp;&ensp;3、使用npm安装hexo
```bash
 npm install -g hexo-cli
```
&ensp;&ensp;&ensp;&ensp;4、创建一个文件夹作为博客的运行目录，
```bash
hexo i myblog //i是init的缩写 myblog是项目名
cd myblog //切换到站点根目录
hexo g   //生成静态文件，g即generetor
hexo s   //启动hexo服务，s即server
```
浏览器访问http://localhots:4000预览效果

&ensp;&ensp;&ensp;&ensp;5、创建第一篇博客，进入hexo根目录下输入 ：
```bash
hexo new "myFirstPost"
```
"myFirtPost" 为博客的名字，会在根路径/source/_posts下创建myFirstPost.md。在myFirstPost.md文件里编辑文章，也可以本地通过markdown在线编辑工具编辑完后同步到服务器上去替换_posts目录下的md文件
执行如下命令来发布：
```bash
hexo g //生成静态页面
hexo d //发布
```
&ensp;&ensp;&ensp;&ensp;更新服务：
```bash
hexo clean //清除已生成的静态文件
hexo g     //重新生成文件
```
也可以直接github上建一个git仓库，本地搞定了推到github，hexo服务器端pull github上的代码就可以了。

&ensp;&ensp;&ensp;&ensp;6、选择博客主题样式，可以使用hexo插件Next来定制自己喜欢的主题，参见Next官网
```text
http://theme-next.iissnan.com/getting-started.html
```


# Hexo进程守护
&ensp;&ensp;&ensp;&ensp;ssh登陆服务器，启动hexo服务，Ctrl+C或者ssh断开连接，服务中断，网站无法访问。Node进程守护有很多工具，Forever，PM2等，这里讲一下用forever解决Hexo进程守护的问题，首先安装forever:
```bash
npm install forever -g
```
&ensp;&ensp;&ensp;&ensp;在Hexo根路径下新建一个app.js,写入下面代码：
```js
var spawn = require('child_process').spawn;
free = spawn('hexo', ['server', '-p 4000']);/* 其实就是等于执行hexo server -p 4000*/

free.stdout.on('data', function (data) {
console.log('standard output:\n' + data);
});

free.stderr.on('data', function (data) { 
console.log('standard error output:\n' + data);
});

free.on('exit', function (code, signal) {
console.log('child process eixt ,exit:' + code);
});
```
&ensp;&ensp;&ensp;&ensp;**启动服务**
&ensp;&ensp;&ensp;&ensp;其实思路也很简单，大致意思就是node启动一个子进程，用forever 守护 hexo sever -p 4000这条命令（4000代表端口），关于node的child_process的相关知识，请自行baidu、google,或者去查nodejs的文档。
执行forever命令：
```bash
forever --minUptime 10000 --spinSleepTime 26000 start app.js
```
&ensp;&ensp;&ensp;&ensp;**停止服务**
&ensp;&ensp;&ensp;&ensp;这里值得注意的是你拿forever启动的服务，通过forever stopall是根本停不掉的，因为其实你执行的是hexo sever，可以通过下面的办法：
```bash
forever stopall  //先停掉守护进程
ps aux|grep hexo
kill -9 pid    //pid是hexo进程id
```

