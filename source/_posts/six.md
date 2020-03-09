---
title: "前端国际化"
date: 2019-2-19 17:20:12
categories: 
    - "前端"
toc_number: false
tags:
	- i18n
	- js
---
公司需要开拓一个国外市场，很是看重，就给我们项目组提了些功能需求(包装美化)。其中就包括国际化问题，多语言切换是一个很重要的竞标资本。
国际化相关技术并不复杂，但涉及的点还是比较多的，前后端、数据、硬编码等等，作业量也不小。
这篇文章仅先探讨下前端国际化的细节。
<!--more-->
## i18n
i18n全称internationalization，是国际化的简称，因单词过长，而i和n之间有18个字符故缩写为i18n。
## 工具
js的国际化采用的市面上比较流行的jquery.i18n插件，轻量级，使用方法简单。
配置不同的语言(en、zh_CN等)的properties，每个文件一一对应相同的key(key不能重复)，value则是不同语言的释义。
简单的讲，就是获得切换语言的指令，在显示调用'key'的位置，置换成指定语言的value。
## 配置
### 资源文件
因为只是目前只是竞标阶段，所以暂时考虑中、英两种语言。
![littleCoder](i18n-web-1.png)
文件路径可自行定义
其中common.properties,可作为组件级或常用、通用的配置
![littleCoder](i18n-web-2.png)
![littleCoder](i18n-web-3.png)
中英文一一对应
### 代码配置
![littleCoder](i18n-web-4.png)
name是资源文件名称
path是资源文件路径
mode代表模式，有map、vars和both
language是从cookie中读取的指定语言，服务端读取后在jsp渲染时传入页面
简单的说就是加载path路径下name+language的文件
## 使用
js中使用i18n非常简单，在你需要国际化的位置用**$.i18n.prop("key")**替换掉原来的中文，如上面图中$.i18n.prop("calculate_add")
也可利用js原型的特性，给String加上i18n方法简化代码
![littleCoder](i18n-web-5.png)
使用则变成了**"key".i18n()**,如"calculate_add".i8n()
