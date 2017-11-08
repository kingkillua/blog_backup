---
title: hexo+yilia博客搭建并托管至github+coding绑定域名
date: 2015-03-09 22:33:01
tags: 
- hexo
- blog
categories: 学
---

因为闲暇时间断断续续弄的，所以就不全文细讲
把还记得的一些坑说一下

<!-- more -->
## 1.root模式进行各种操作- -！

## 2.yml配置文件包括md文章的书写都要记得有空格的带空格

## 3.页面没有样式
弄这个博客出问题最多就是这路径设置
```
url: http://deigo.win
root: /deigo42
permalink: :year/:month/:day/:title/
permalink_defaults:
```
上面是我的配置，几经波折
一开始样式加载不出来
然后文章图片显示不了
然后绑定域名又有问题
然后百度收录不了github，于是同时托管github和coding,又有问题
然后百度收录好了，回头发现图片又不行了
真的是无语了
大概总结下
没买域名并绑定的话，就把URL项注释掉，root项填写项目名就好
绑了域名，就填上去
然后因为这个项目目录名其实就是新建仓库的时候填的嘛
所以如果要同时托管到github和coding上并且绑定域名的话，
必须github和coding的项目名（仓库名）要一样，不然你填其中一个，另外一个就加载不了js没样式

## 4.实现标签（tags），分类（categories）功能主要是3个地方做好
1. 配置好blog的_config.yml
```
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:
```
上面是我的配置，tag_dir,category_dir这两项的值决定你hexo生成的博客目录里面public生成的文件夹名称
关系到后面的theme的_config.yml的配置

2. 配置好theme的_config.yml
```
menu:
主页: /
学: /categories/学/
想: /categories/想/
唠: /categories/唠/
光影: /categories/光影/
```
上面的是我的配置，theme就是根据上面的路径这个找到分类/标签页的

3. md文章书写按规范来，文章有标签，hexo g的时候就根据tag_dir生成对应文件目录和里面的页面
然后tags可以多个，互相不影响，但是categories多个的话生成的文件就是嵌套的，然后theme里面就配置不了路径了
没找到怎么弄

## 5.同时托管两个地方的写法网上过时了，官网使用手册更新如下
```
deploy:
- type: git
  repo: git@github.com:kingkillua/xxx.git,master
- type: git
  repo: git@git.coding.net:deigo42/xxx.git,master
```
另外如果后期作了项目备份，又down下来的时候可能需要hexo 重新装一遍，也可能出现deploy的问题
可以尝试删掉隐藏的.deploy_git文件，然后重新装一边deploy.git

## 6.如果同时托管到github&coding并且绑定自己的域名的话，必须把两个地方项目名都改成你coding的用户名
解析的时候，github的用ip（这个ip ping你的项目域名得到）
conding就用项目域名就行了，去掉后面的目录，比如xxx.coding.me/blog ，直接用xxx.coding.me就可以

最后，弄出来效果还是可以的，只是没技术含量，发表什么的也不太方便
下一步准备写个爬虫整个相册功能，熟悉nodejs先
