---
title: ParallelChat：我觉得还可以再抢救一下
description: Electron 方案在 Gemini 登录与权限上的转机与取舍
date: 2025-11-17
categories:
    - coding-life
tags:
    - electron
    - 插件
    - Gemini
    - Google登录
    - UA
    - CDB
image: cover.png
---

# ParallelChat：我觉得还可以再抢救一下

本以为已经宣告死亡的项目，却在一次偶然的发现中找到了重生的可能。

前文说到，ParallelChat 发布即死亡，因为发现被浏览器插件吊着打。对于浏览器插件环境更正常，另外最致命的就是 Electron 环境无法登录 Gemini。

## 转机：一次偶然的发现

为什么说还可以抢救一下呢？因为看到一个 issue，说 Cherry Studio 里可以实现 Google 登录。于是我去看了一下源码，相关部分是伪装 UA，但伪装成的是 Safari，于是我也试了一下，成功！

## 重新审视：Electron 的隐藏优势

解决了登录问题后，和 Chrome 插件对比，Electron 方案可以说还是没有一点优点。没想到其实还有一个优势的，对于 Electron 来说，问题也是在 Gemini 上。

有个朋友和我聊天时提到，他用的浏览器插件没有文件上传功能。我去试了一下确实没有提供这个功能。我对 Chrome 插件不算熟悉，但是相对来说插件受到更严格的权限控制，所以实现上传我不太清楚插件容不容易实现。但是对于 Gemini插件应该很难实现。其他几个网站都是点击 input 来触发上传，所以很容易。但是 Gemini 有点特殊，不知道是不是因为它是基于 Angular 的原因，Gemini 网站的上传逻辑是点击上传按钮，动态创建 input，然后触发上传。并且这个点击按钮似乎必须在 isTrusted 为 true 的情况下才能触发，也就是说该事件是由用户的真实操作（如鼠标点击、键盘输入）触发。在 Chrome 插件里我觉得很难做到。但是在 Electron 可以通过 CDB 来实现。

也就是说，Electron 相对来说有更大的权限，更容易实现受限制功能。

## 结语：半死不活

ParallelChat 相比插件我觉得体验要差了不少，但好在有一些补充。实际上在我的计划里有提取内容做成 API 接口的打算，不过算了，这个项目就这样吧。