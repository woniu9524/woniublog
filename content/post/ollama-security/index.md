---
title: 搜索暴露在公网中的Ollama，以及Ollama安全部署建议
description: 探讨Ollama的安全部署问题，以及如何通过Fofa和Shodan发现公开的Ollama实例
date: 2025-02-14
image: cover.png
categories:
    - coding-life
tags:
    - ollama
    - security
    - fofa
    - shodan
---

> Ollama通常被设计为本地使用，但有时为了便于分享会直接部署在公网服务器上。这种做法可能导致任何互联网用户都能调用你的Ollama服务。本文将探讨这个安全隐患以及相关的防护建议。

## 一、通过Fofa搜索Ollama API

Fofa是一个强大的互联网设备和服务搜索引擎，可以帮助发现公开的API服务。通过特定的搜索语法，我们可以精准定位暴露在互联网上的Ollama API服务。

### Fofa搜索步骤

1. 访问Fofa官网，使用以下搜索语句：

```
app="Ollama" && is_domain=false
```

这个搜索语句包含两个关键条件：
- **app="Ollama"**：筛选Ollama相关服务
- **is_domain=false**：排除域名服务，只显示直接暴露的API端点

## 二、使用Shodan搜索Ollama API

Shodan是另一个专门用于发现互联网设备和服务的搜索引擎。它可以帮助我们找到运行中的Ollama实例。

### Shodan搜索方法

1. 访问[Shodan官网](https://shodan.io/)，输入以下搜索语句：

```
Ollama is running
```

搜索结果会返回包含Ollama API实例的IP地址和端口信息。


## 结语

不要把Ollama直接部署在公网上，另外部署在公网上的服务要考虑安全问题，很容易被扫出来。