---
layout: post
title: jekyll搭建博客的时候遇到的问题
category: tech
description: 三个问题，部分问题没有记录.
tags:
    - jekyll
    - 问题
---
## 1.安装Ruby出现问题
安装2.3以下版本的Ruby要安装devkit  
安装2.3以上版本的Ruby要安装安装结束后的sys2插件
## 2.安装插件rediscount出现问题
**问题描述**  
`ERROR: Failed to build gem native extension`   
根据错误提示，要求去阅读日志文件，发现某个函数再传参的时候出现了中文路径/？？？/无法识别，所以出现问题，使用Administrator账号重新安装了sys2即可
## 3.安装jekyll-paginate出现问题提示证书错误
**问题描述**   
` SSL_connect returned=1 errno=0 state=error: certificate verify failed`  
下载cacert.pem证书  
`set SSL_CERT_FILE=E:\cacert.pem`