---
title: hexo博客自动化部署到阿里云tips
date: 2017-09-01 00:08:06
tags:
categories: 学
---

最近想了解点系统集成nginx dubbo什么的，就整了个阿里云主机
虽然这之间好像没什么联系。。。
然后顺手把blog搬进去
过程就是本地git push到阿里云，然后钩子脚本自动拷贝一份到网站目录，再nginx设置跳转一下
打开冰箱门，把大象塞进去，关上冰箱门，就这么简单
就这么简单我捣了一晚上。。。

<!--more-->
主要参考这位兄弟的
https://segmentfault.com/a/1190000005723321

-------------------------------------------------------

putty，用阿里云主机实例的公共ip 和22端口连
话说root密码还是复杂点好，我刚开实例不到2个小时就被绿了（异地登陆）
然后操作能用git用户就别用root
对于git钩子实现自动部署有的用复制操作，有的用checkout
不管用哪种，操作涉及到哪些目录，git用户就需要它的权限

-------------------------------------------------------

FATAL remote: error: refusing to update checked out branch: refs/heads/master
本地提交到阿里云上出现这个的错
是由于git默认拒绝了push操作，需要进行设置，修改.git/config文件后面添加如下代码：
```
[receive]
denyCurrentBranch = ignore
```
-------------------------------------------------------

我部署的路径信息：
git库：/home/git/blog.git
网站目录：/home/git/blog/deigo42
git钩子下面两种都可以用，随意
没效果的话可以跑到post-receive所在目录自己跑一遍脚本./post-receive看下错误在那
##### 1.复制到网站目录
```
#!/bin/bash
GIT_REPO=/home/git/blog.git
TMP_GIT_CLONE=/home/git/cptemp
PUBLIC_WWW=/home/git/blog/deigo42
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE
rm -rf ${PUBLIC_WWW}/*
cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
```
这里涉及到的三个目录都用git用户创建吧
##### 2.checkout到网站目录
```
#!/bin/sh
git --work-tree=/home/git/blog/deigo42 --git-dir=/home/git/blog.git checkout -f
```
checkout要注意先随便提交个readme，不然会报还没有master分支的错误


-------------------------------------------------------

nginx配置
```
server {
    listen         80 ;
    root /home/git/blog/deigo42;
    server_name www.deigo.win deigo.win;
    location /deigo42 {
        root /home/git/blog;
     }
}
```
nginx在这作用就是跳转下
意思就是域名解析那里填阿里云实例的公共ip，然后浏览器输入deigo.win的时候来到填的ip也就是云主机这
因为没带端口就默认80，所以nginx这里监听80端口

话又说，因为博客之前是部署在github的page服务，放github里面的东西路径前头都带项目前缀，也就是deigo42/
又要兼顾baidusitemap.xml的生成，于是hexo的config.yml里的最后这样配置才过关
```
url: http://deigo.win
root: /deigo42
```
然后这次生成的静态网站，一开始是放在home/git/blog的，nginx的root配的自然也就是home/git/blog
结果当然还是没样式，链接和图片全失效，因为它们url全是/deigo42/*******这种
最后捣腾半天，变成这样
网站丢到了home/git/blog/deigo42/里
因为所有链接和图片因为都带/deigo42，经过nginx都跳到home/git/blog目录
再拼接它们各自带/deigo42的url，也就找得到了

-------------------------------------------------------

其实写这个主要就是因为这个破路径问题
因为我也没见别人有这个问题，就我有，我哪里错了，太费解了





