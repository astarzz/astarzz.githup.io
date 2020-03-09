---
title: cnpm
date: 2019-06-23 09:50:22
categories: 
    - "node"
toc_number: false
tags:
	- npm
	- cnpm
---
## npm
Node Package Manager，是nodejs的包管理器，用于node插件管理。

## cnpm
是淘宝团队提供的免费国内npm下载镜像源。
<!--more-->


npm同maven类似，默认的仓库地址在国外(<https://registry.npmjs.org>)，因网络问题经常性的出现下载慢、出错等问题。
如果使用国内镜像，就可以很好的避免问题，目前有2种实现方式。
- 安装cnpm

<table><tr><td bgcolor=#F5F8FA>npm install cnpm -g --registry=https://registry.npm.taobao.org</td></tr></table>

后续需要使用**cnpm**命令来安装模块

<table><tr><td bgcolor=#F5F8FA>cnpm install hexo</td></tr></table>

- 修改默认仓库地址

<table><tr><td bgcolor=#F5F8FA>npm config set registry https://registry.npm.taobao.org</td></tr></table>


