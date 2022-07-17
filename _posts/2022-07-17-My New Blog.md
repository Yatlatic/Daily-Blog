---
layout: post
toc: true
title: "Jekyll 搭建中遇到的一些问题"
categories: Jekyll
author:
  - Yatlatic
---

##  Question 1 : Cannot load such file --webrick

从 Ruby 3.0 开始 webrick 已经不在绑定到 Ruby 中了，需要手动进行添加。

添加的命令为：

```
  bundle add webrick
```
##  Question 2: Jekyll 服务器的启动
启动命令为：
```
  bundle exec jekyll serve --trace
```