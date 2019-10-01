---
title: Hexo和GitPages搭建博客并自动发布
date: 2019-10-01 00:06:20
tags: 
- 博客搭建
categories:
- 博客搭建
---

现有的博客网站用着都不太满意，思来想去还是决定自己搭一个。按照惯例，第一篇文章记录下整个博客的搭建过程。整个博客基于[Hexo](https://hexo.io/zh-cn/docs/)和[Github Pages](https://pages.github.com)构建，并使用[Travis CI](https://www.travis-ci.org)持续集成，自动发布。

<!-- more -->

## 搭建过程

### 创建Github Pages

1. 注册并登陆你的Github
2. 创建一个仓库，名为 username.github.io ，username 就是你的 Github 用户名，提交后就可以通过http://username.github.io/访问了

### 环境配置
#### 安装Node.js和Git

程序员基本都会用到的模块，安装可参考[Hexo文档](https://hexo.io/zh-cn/docs/)。

#### 安装Hexo

cmd/gitbash执行如下命令

```shell
npm install hexo-cli -g # 安装hexo
hexo init blog # 初始化blog文件夹
cd blog
npm install # 安装博客所需npm包
hexo g # 生成静态文件
hexo s # 启动本地web服务，可在http://localhost:4000/ 预览博客
```

### 域名绑定

#### 购买域名

域名绑定第一步当然是先去买一个域名，不然拿头绑定。有了域名就可以用自己的域名，比如说www.kosho.tech访问了，比http://username.github.io/舒服多了。

那么，在哪里才能买得到呢？

其实阿里云就不错，价格便宜，配置方便，自带DNS解析。

#### DNS解析

买完在域名解析里增加两条记录：

| 主机记录 | 记录类型 | 线路类型 |      记录值      |
| :------: | :------: | :------: | :--------------: |
|   www    |    A     |   默认   | 185.199.110.153  |
|    @     |  CNAME   |   默认   | koshox.github.io |

然后记得在hexo博客的source文件夹下加一个CNAME文件，内容是你的域名：

```
kosho.tech
```

这样访问你的域名时，浏览器会自动跳转到Github Pages。

### 自动部署

一顿操作以后，怎么把生成的博客内容部署上去呢？很多人会使用[hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)插件部署。

在站点配置文件 _config.yml 中修改设置：

```yaml
deploy:  
	type: git
	repo: <repository url>
	branch: [branch]
```

然后执行

```shell
hexo d
```

但是这样还是稍微有点麻烦，每次得输入用户名密码。

官方推荐的方式是利用Travis CI持续集成：https://hexo.io/zh-cn/docs/github-pages
注意不配置分支的话，默认会把部署分支设置为gh-pages，可以像我一样，把源码分支设置为hexo，部署分支设置为master，那.travis.yml这么配置：

```yaml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - hexo
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  target-branch: master
  on:
    branch: hexo
  local-dir: public
```
这样向hexo分支提交文章，CI会帮你自动部署上去，顺滑就完事了。

### 继续折腾

搭建完了还有一堆的事需要折腾，包括但不限于：

1. 折腾主题
2. 增加评论
3. 添加搜索功能
4. 阅读次数统计
5. 添加社交链接
6. 提交百度和谷歌收录

等等...

Hexo和Next主题默认给我们集成了很多功能，网上也有大把的文章，可以折腾出很炫酷的效果。





