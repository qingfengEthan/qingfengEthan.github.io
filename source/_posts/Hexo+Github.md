---
title: Hexo+GitHub 搭建个人博客
date: 2019-12-11 21:37:39
tags: hexo
---

------
#### 一、Hexo发布到Github Pages
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Github Pages服务可以给我们提供一个静态网页的托管，以便远程浏览我们的博客内容。Github Pages是给开发者建立的私人页面，免费且没有空间流量限制。每个github账号都可以创建一个Github Pages项目。 

##### 1、创建Github Pages项目
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在github上新建一个项目，项目的名称必须是（你的用户名.github.io)才行 
![cmd-markdown-logo](http://139.224.113.197/20191211212249.jpg)

##### 2、Hexo和Github通过ssh通信

###### 2.1 设置Git的user name和email(第一次使用时)

```
git config --global user.name "qingfengEthan"
git config --global user.email "136937****@qq.com"
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;查看是否配置成功

```
git config --global user.name
git config --global user.email
```

###### 2.2 生成ssh密钥
###### 2.2.1 检查SSH keys是否存在Github

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;需要在本机生成一个公钥以便跟github建立安全连接，执行如下命令，检查SSH keys是否存在。如果有文件id_rsa.pub或id_dsa.pub，则直接进入步骤2.2.3将SSH key添加到Github中，否则进入下一步生成SSH key。

```
ls -al ~/.ssh
```
###### 2.2.2 生成新的ssh key
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;执行如下命令生成public/private rsa key pair，注意将邮箱地址换成你自己注册Github的邮箱地址。
```
ssh-keygen -t rsa -C "136937****@qq.com"
cat id_rsa.pub >> ~/.ssh/authorized_keys
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;默认会在相应路径下（~/.ssh/id_rsa.pub）生成id_rsa和id_rsa.pub两个文件

###### 2.2.3 将ssh key添加到Github中
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;打开id_rsa.pub文件，里面的信息即为ssh key，将这些信息复制到Github的Add SSH key页面即可。

进入Github --> Settings --> SSH and GPG keys --> new SSH key：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;输入如下命令测试添加ssh是否成功。如果看到Hi后面是你的用户名，就说明成功了。
```
ssh -T git@github.com

Hi qingfengEthan! You've successfully authenticated, but GitHub does not provide shell access.
```


##### 3、Hexo 与 GitHub Pages关联 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在hexo blog项目根目录下里找到_config.yml文件，找到deploy，然后做如下修改：

```
deploy:
  type: git
  repo: git@github.com:qingfengEthan/qingfengEthan.github.io.git
  branch: master
```

##### 4、安装 hexo-deployer-git自动部署发布工具
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hexo blog部署到git我们需要安装hexo-deployer-git插件,在blog目录下运行一下命令进行安装:
```
npm install hexo-deployer-git --save
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果安装失败，可以使用淘宝NPM镜像

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm install hexo-deployer-git --save
```



##### 5、生成静态文件部署到github

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过命令hexo clean && hexo g && hexo d，发布到github


```
hexo clean && hexo g && hexo d
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过链接就可以进行访问：qingfengEthan.github.io


#### 二、Hexo源文件保存到Github
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;github远程仓库只会保存hexo发布后的静态HTML文件，hexo md源文件、主题配置等还在本地，一旦电脑磁盘坏了或者换了电脑，就无法在之前仓库的基础上继续写博客。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;解决办法：github master分支来保存hexo生成的静态网页，对于博客源码，可以新建一个source分支来存储。

##### 1、新建远程分支source
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;新建远程sourc分支，并把source分支设为仓库的默认分支。

##### 2、将本地hexo目录与远程仓库关联
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;进入到本地hexo工程目录，也就是我们通常执行hexo new post等命令的目录，执行如下操作：

```
git init
git remote add origin https://github.com/qingfengEthan/qingfengEthan.github.io
```

##### 3、推送博客源码
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将本地的hexo md源文件、站点配置文件等推送到source分支。
因为我们只需要保留博客源码，其他无关的文件并不希望推送，需要配置.gitignore文件，通常如下：

```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;然后依次执行如下命令：

```
git add .
git commit -m 'hexo source post'
git push origin source
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;执行完之后，仓库目录如下：

![cmd-markdown-logo](http://139.224.113.197/20191211180305.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;确保hexo deploy推送的是master分支，hexo目录下的_config.yml文件deploy配置如下：

```
deploy:
  type: git
  repo: git@github.com:qingfengEthan/qingfengEthan.github.io.git
  branch: master
```





