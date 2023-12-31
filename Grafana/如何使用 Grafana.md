---
layout: post
author: sjf0115
title: 如何使用 Grafana
date: 2019-08-04 13:05:34
tags:
  - Grafana

categories: Grafana
permalink: how-to-use-grafana
---

这篇文章将会帮助你开始并熟悉 `Grafana`。我们假设你已安装并启动了 `Grafana` 服务器。如果没有，请先阅读[安装指南](http://smartsi.club/how-to-install-and-run-grafana.html)。

### 1. 登录

要运行 `Grafana`，请打开浏览器并转到 http://localhost:3000/。如果你尚未配置其他端口，则 `Grafana` 侦听的默认 Http 端口为 3000。

![](https://github.com/sjf0115/ImageBucket/blob/main/Grafana/how-to-install-and-run-grafana-1.png?raw=true)

> 默认登录账号与密码为: admin/ admin。当你第一次登录时，系统会要求你更改密码。

![](https://github.com/sjf0115/ImageBucket/blob/main/Grafana/how-to-install-and-run-grafana-2.png?raw=true)

### 2. 添加数据源

在创建第一个仪表盘之前，我们需要添加数据源。

![](https://github.com/sjf0115/ImageBucket/blob/main/Grafana/how-to-use-grafana-1.png?raw=true)

首先将光标移动到侧面菜单上的齿轮菜单。配置菜单上的第一项是数据源，单击它，我们将进入数据源页面，可以在其中添加和编辑数据源。单击添加数据源，我们将进入新数据源的设置页面：

![](https://github.com/sjf0115/ImageBucket/blob/main/Grafana/how-to-use-grafana-2.png?raw=true)

首先，为数据源指定一个名称，然后选择要创建的数据源类型，有关详细信息以及如何配置数据源，请参阅支持的数据源。










原文:[Getting started](https://grafana.com/docs/guides/getting_started/)
