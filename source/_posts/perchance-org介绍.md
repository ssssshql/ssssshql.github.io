---
title: Perchance.org 介绍 — 免费在线随机生成器平台
date: 2026-08-12 09:40:59
tags: [Perchance, 随机生成, 工具]
categories: 工具
description: 介绍 perchance.org 这个免费在线随机生成器平台：核心概念、基本语法、用例，以及如何把生成的 App 嵌入自己的网站
---

Perchance.org 是一个免费、无需编程基础就能创建随机生成器（generator）的在线平台。最近用它做了一个「像素场景生成器」，顺便把整个平台研究了一遍，记录一下。

## 简介

简单说，Perchance 让你用类似"列表套列表"的方式定义一个随机生成器：你写一小段模板文本，里面用 `[方括号]` 引用不同的列表，每次刷新就会从列表里随机挑选内容填进去，生成一个不重复的结果。

它适合做这类东西：

- 随机名字、随机文案、随机标题
- 游戏里的物品掉落、剧情灵感、NPC 生成
- 随机菜谱、随机任务、随机事件
- 甚至组合输出"看起来像 AI 生成"的短文本

底层是 HTML/CSS + JavaScript，本质上是一套**随机文本模板引擎**，不是大模型那种 AI，但经常被大家习惯性地叫"AI 生成器"。Perchance 近两年也接入了真正的 AI 功能（文本生成、角色对话等）。

## 核心概念：列表引用列表

官方欢迎页给了一个最经典的最小例子：

```
output
  Your [pack] contains [item], [item] and [item].

item
  a few coins
  an old {silver|bronze} ring
  a handkerchief
  a shard of bone
  some lint
  a tin of tea leaves

pack
  purse
  backpack
  bag
  pack
  knapsack
  rucksack
```

只要在模板里写 `[item]`，就会从 `item` 列表随机挑一个填进去；`[pack]` 同理。所以两次生成的结果可能是：

```text
Your backpack contains a few coins, an old silver ring and a tin of tea leaves.
Your purse contains some lint, a handkerchief and a shard of bone.
```

## 常用语法速览

### 权重

在条目后加 `^2`，该条目被选中的概率就是普通的 2 倍：

```
pack
  purse
  backpack ^2
  bag
```

### 导入他人生成器

可以 `import` 别人的生成器，直接复用里边的列表：

```
sentence
  I need a new {import:noun}.
```

### 修饰符

对单词做大小写、复数、时态等处理：

```
sentence
  [name.titleCase], can you hear me?
  HELLO, [name.upperCase]!

name
  patricia
  khalid
```

还有 `pluralForm`（复数）、`pastTense`（过去式）、`capitalize` 等一大堆修饰符。

### 随机数字范围

`{15-20}` 表示 15~20 的随机整数：

```
sentence
  {She|He} was about {15-20}0cm tall and was carrying {1-3} things.
```

更完整的上手建议直接看官方的 [tutorial](https://perchance.org/tutorial)。

## 分享与分发方式

创建完生成器之后，Perchance 提供了多种分发途径：

- **直接分享 URL**：链接即生成器，别人打开就能用（URL 可以在设置菜单里自定义）
- **下载为单个 HTML 文件**：可以完全离线运行
- **嵌入自己网站**：用一行 `<iframe>` 即可
- **变成机器人**：可以做成 Discord bot、Twitter bot、Tumblr bot
- **配置 API 服务**：给会写代码的人准备的，可以自己搭 API server 调接口

## 嵌入到自己的网站

官方推荐就是这个 iframe 写法：

```html
<iframe src="https://perchance.org/my-generator-name" style="width:100%; height:600px; border:none;"></iframe>
```

我在这个博客里就用它嵌入了自己做的「像素场景生成器」，直接放出来体验一下：

<iframe src="https://perchance.org/3i9jdvdh74" style="width:100%; height:600px; border:none;"></iframe>

要点：

- `src` 填生成器的 URL，**域名用 `perchance.org` 或 `null.perchance.org` 均可**
- `height` 根据生成器的内容量调合适的高度，一般 `500~800px`
- `border:none` 去掉边框，视觉上更干净
- 因为是 iframe，所以 Hexo / WordPress / 任意静态站都能直接用

## 它在哪些场景好用

- **快速原型**：不需要搭服务器，不需要写后端，几分钟做一个随机内容工具
- **游戏辅助**：跑团 GM、游戏随机事件、装备掉落表
- **内容创作**：起名、写标题、找灵感、占位文案
- **教学入门**：官方说它的隐藏目标是让人对"写代码"产生兴趣，语法比真编程语言简单得多

如果只是想要一个"随机抽一个条目"的工具，用 Perchance 可能有点大材小用；但当你需要权重、嵌套、导入、修饰符这些进阶能力时，它依然不需要任何代码基础，就很有优势了。

相关可替代/同类产物：Chartopia（同类型随机生成器网站）、neocities（重 HTML/JS 方向）。

## 总结

Perchance 的核心就一句话：**列表引用列表，方括号随机填充**。免费、免登录即可创建、无需编程知识、支持 iframe 嵌入任意网站 —— 很适合随手做一些小型随机生成工具，也适合嵌入到自己的博客/作品集里当个小玩具。

我的「像素场景生成器」就嵌在上面，点开就能用。

## 参考链接

- [Perchance 官网](https://perchance.org/)
- [官方教程 Tutorial](https://perchance.org/tutorial)
- [示例生成器 Examples](https://perchance.org/examples)
- [社区论坛 Lemmy Community](https://lemmy.world/c/perchance)