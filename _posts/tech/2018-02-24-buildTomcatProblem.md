---
layout: post
title: eclipse集成tomcat遇到的问题
category: tech
description: Die Erkenntnis.
tags:
    - eclipse
    - tomcat
    - 问题
---

## 问题一：缺失类库  
`The superclass "javax.servlet.http.HttpServlet" was not found on the Java Build Path`  
解决方案：  
1、右击web工程-》属性或Build Path-》Java Build Path->Libraries-> Add Libray...->Server Runtime -》Tomcat Server
2、切换到Java Build Path界面中的Orader and Export，选择Tomcat。

