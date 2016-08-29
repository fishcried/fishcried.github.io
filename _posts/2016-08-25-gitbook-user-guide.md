---
layout: post
title: 使用gitbook进行个人知识管理
subtitle:
description:
category: 高效工作
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - 高效工作
    - gitbook
    - wiki
---

# 使用gitbook进行个人知识管理

每个程序员都会进行个人知识管理，我自己在个人wiki上经历过几个阶段:

> 1. 最开始使用在线笔记,后来不用.一是习惯了vim,可视化后按键别扭; 二是使用笔记容易copy/paste,自己整理的少，懒.
> 2. 使用vim+vimwiki插件.后来不用,一是那时候vimwiki还不支持markdown,而是使用特定语法;二是vimwiki生成html页面,效果看个人的css,而前端正式我的弱项,所以我的页面很难看.
> 3. 使用gitpage+jekyll+markdown.很多人用这个承载blog,开始的时候是把wiki直接放到这个上，但是很多杂乱的草稿直接公开不是太好。而且jekyll没有toc,搜索上不是很好。最后只有自己的blog还用这个.就是现在这个网站。
> 4. 现阶段, gitbook来维护自己的wiki。实际上gitbook就是用git+markdown来编写书籍的解决方案。效果是这个样子:

---

![](/img/gitbook-overview.png)


**为什么选择gitbook做wiki管理**

三个理由:

1. 系统性。gitbook是适合写书的，写书需要系统性，所以gitbook有左侧的目录栏。wiki往往是片段的，所以系统性非常重要。TOC是个人选择gitbook的最重要的因素。
1. 书写上,markdown+git天生适合程序员,高效便利。
2. gitbook展示非常好,html,pdf都可以，上面的图就可以看出来，而且还提供搜索。


## 本地gitbook-server安装

gitbook-server可以将文档目录转换成html网站,或者pdf等.

**软件包方式安装**

On Ubuntu14.04

```
sudo apt-get install nodejs
sudo apt-get install npm
sudo npm install  -g gitbook-cli
apt-get install nodejs-legacy
```

**通过docker使用gitbook-server**

```
docker pull billryan/gitbook
docker run -d -v "$PWD:/gitbook" -p 4000:4000 billryan/gitbook gitbook  serve --watch
```

### gitbook-server使用

```
gitbook serve .

# 更多信息查看帮助信息
gitbook --help

```

## gitbook使用举例

**准备文件和目录**

```
mkdir gitbook-demo
cd gitbook-demo
mkdir chapter1
touch  chapter1/{readme,1,2}.md
touch README.md
echo test > README.md
```

**编写目录**

```
vim SUMMARY.md
* [前言](README.md)
* [第一章](chapter1/readme.md)
  * [第一节](chapter/1.md)
  * [第二节](chapter/2.md)
```


> SUMMARY.md是目录文件,gitbook生成网页左侧的目录就是SUMMARY.md文件中记录的.

**运行gitbook server**

```
gitbook serve . --port 8888
```

![](/img/gitbook-demo.png)


非常简单吧，这样我们就可以专注于内容了。


## 几点说明

1. 上面介绍的是本地使用gitbook-server。还可以使用gitbook.com的服务，支持一本私有图书，无限公开图书，可以和github联动，只要将代码提交到github,gitbook网站会自动的生成，可是国内的访问速度基本无法使用。
2. 还有个工具是gitbook-editor,可视化的markdown编辑工具，gitbook.com提供，做了深度集成。不怎么推荐使用，目前不太好用，而且markdown就是为了编写时不考虑展示，用了可视化工具，感觉效率不怎么高了。
3. 使用docker运行gitbook-server非常方便。gitlab+jenkins+docker(gitbook-server)可以非常便捷地提供研发文档解决方案。只要开发人员想代码一样提交变更,文档就自动生成了，目前我们组就是采用了这套方案，效果很好。形成了文档的CICD,质量很好。
