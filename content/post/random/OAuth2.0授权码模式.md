---
title: "OAuth2.0授权码模式"
date: 2024-08-09T14:15:48+08:00
lastmod: 2024-08-09T14:15:48+08:00
draft: false
keywords: ["auth", "OAuth2.0", "授权码模式"]
description: "OAuth2.0授权码模式"
tags: []

categories: ["random"]
author: "Tangthinker"

# Uncomment to pin article to front page
# weight: 1
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false

# Uncomment to add to the homepage's dropdown menu; weight = order of article
# menu:
#   main:
#     parent: "docs"
#     weight: 1
---

<!--more-->


## OAuth2.0授权码模式

### 1. 概述

OAuth2.0是一种授权框架，它允许第三方应用程序获取有限的访问权限，而无需获取用户的用户名和密码。OAuth2.0授权码模式是OAuth2.0中最常用的授权模式之一，它通过一个授权码来获取访问令牌。

### 2. 授权码模式流程

授权码模式的基本流程如下：

1. 用户访问第三方应用程序，第三方应用程序将用户重定向到授权服务器。
2. 授权服务器要求用户登录并授权第三方应用程序访问其资源。
3. 用户同意授权，授权服务器将用户重定向回第三方应用程序，并附带一个授权码。
4. 第三方应用程序使用授权码向授权服务器请求访问令牌。
5. 授权服务器验证授权码，并返回访问令牌给第三方应用程序。
6. 第三方应用程序使用访问令牌访问用户的资源。

### 3. 重点

1. 为什么要有OAuth2.0

假设我是服务提供方 mine-service ，我还依赖于一个三方服务 third-part-service，
此时用户请求我，需要使用到 third-part-service 的资源，此时我需要向 third-part-service 请求授权，
获取授权码，然后使用授权码获取访问令牌，最后使用访问令牌访问 third-part-service 的资源。

2. 微信OAuth2.0授权码模式

![Wechat-OAuth2.0](/img/random/oauth-weixin.png)



