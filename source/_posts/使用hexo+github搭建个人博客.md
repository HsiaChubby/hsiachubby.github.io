---
title: 使用hexo+github搭建个人博客
tags: Hexo
categories: 前端
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## 创建GitHub仓库
1. 登录https://github.com/
2. 注册账号
3. 创建一个新Repositorie(仓库名必须要为以下格式：**xxx.github.io**，其中，xxx为用户名小写)
## hexo部署
### 什么是hexo?
hexo是一个简单但强大的博客系统，Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
### 部署
1. 安装node.js
```
<!--MacOS-->
<!--安装了node之后，默认也会安装npm-->
brew install node
```
2. 安装hexo
```
npm install hexo-cli
```
3.创建本地项目
```
hexo init xxx.github.io
cd xxx.github.io
<!--修改项目配置文件中的deploy配置-->
	deploy:
	  type: git
  repo: https://github.com/HsiaChubby/hsiachubby.github.io.git,master
```
<!-- more -->
## 修改配置
### 设置主题
1. 下载next主题
```
git clone https://github.com/iissnan/hexo-theme-next themes/next
```
2. 配置项目的主题
打开项目配置文件_config.yml，设置theme如下：
```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
```
3. 配置主站相关信息
打开项目配置文件_config.yml，设置基础如下:
```
# Site
title: Chubby的博客
subtitle:
description:
keywords:
author: Chubby Hsia
language: zh-Hans
timezone:
```
4. 配置首页支持分类、标签、关于、归档
分别运行一下命令
```
<!--创建分类页面-->
hexo new page categories
<!--创建标签页面-->
hexo new page tags
<!--创建归档页面-->
hexo new page archives
<!--创建关于页面-->
hexo new page about
```
创建好上述页面后，会在source目录下，分别生成categories/tags/archives/about四个文件夹，然后分别进入相应的文件夹中，修改各自的index.md文件，如下：
```
<!--categories的index.md-->
---
title: categories
date: 2019-01-30 14:57:39
type: "categories"
---
<!--tags的index.md-->
---
title: tags
date: 2019-01-30 15:02:24
type: "tags"
---
<!--archives的index.md-->
title: archives
date: 2019-01-30 15:02:33
type: "archives"
---
<!--about的index.md-->
---
title: about
date: 2019-01-30 15:02:41
type: "archives"
---
```
然后，进入theme/next主题下，打开主题的配置文件_config.yml，修改menu配置，如下：
```
menu:
  home: / || home
  about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```
## 启动服务
```
hexo clean
hexo g -d
```
然后，稍等一会，直接访问https://hsiachubby.github.io/

## 提交代码
新建hexo分支，将本地代码提交至hexo分支，该分支用来维护具体的文档信息；master分支是用来发布hexo的。

## 相关问题
* [ ] 设置”全文阅读“，在具体的md文件中，选择合适的地方，插入**<!-- more -->**
* [ ] 文章插入图片，在source目录中，创建images目录，该目录用来维护图片信息，当Hexo项目中只用到少量图片时，可以将图片统一放在source/images文件夹中，通过markdown语法访问它们。



