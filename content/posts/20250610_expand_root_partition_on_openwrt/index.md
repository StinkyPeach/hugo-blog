---
title: "Expand Root Partition On OpenWrt"
subtitle: ""
description: "OpenWrt扩容根分区"
date: 2025-06-10T21:08:38+08:00
author: "Devin"
tags: 
    - OpenWrt
draft: true
---

<h1 align="center">	
    expand root partition on OpenWrt
</h1>

由于OpenWrt默认镜像的分区空间一般只有100M左右，无法满足一些日常操作，比如运行容器和一些web服务等。本期我们尝试在成功刷入OpenWrt后，如何对根分区进行扩容。

OpenWrt镜像一般分为2个大类

