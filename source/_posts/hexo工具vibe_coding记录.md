---
title: hexo工具vibe coding记录
date: 2026-03-27 22:30:00
tags: [hexo]
categories: [hexo]
---

最近公司比较闲，又折腾起 hexo 来了。发现在 Android 上没有什么好用的写博客工具，索性用 Flutter 撸了一个，支持仓库的推拉和文章的增删改查。

![](https://picgo.19991029.xyz/%E8%90%BD%E5%9C%B0%E9%A1%B5.png)

## 使用演示

### 配置仓库

需要配置三项信息：
- 仓库地址
- 用户名
- 访问令牌

访问令牌获取：https://github.com/settings/personal-access-tokens

配置完成后即可 clone 仓库，之后可以进行文章的查看、删除、新增、修改等操作。

### 文章操作

新增、修改、删除文章后推送，等待 GitHub Actions 自动部署即可。

<img src="https://picgo.19991029.xyz/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20260327222012_53_2.jpg" alt="微信图片_20260327222012_53_2" style="zoom:15%;" /> <img src="https://picgo.19991029.xyz/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20260327222013_54_2.jpg" alt="微信图片_20260327222013_54_2" style="zoom:15%;" /> <img src="https://picgo.19991029.xyz/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20260327222013_55_2.jpg" alt="微信图片_20260327222013_55_2" style="zoom:15%;" />  

## 关于项目

目前功能比较简单，后续会慢慢补充。

项目已开源在 GitHub，欢迎提建议：[blog.19991029.xyz/open_hexo](https://blog.19991029.xyz/open_hexo)

软件支持应用内更新。

