---
title: 记MultipartException一次导致nginx500错误
date: 2019-08-04 21:14:50
tags: ["nginx","zull","Exception"]
categories: "技术"
---
### 前言
最近写了一个上传文件的接口，在小程序访问的时候,nginx报了500。看了nginx的 `error log` 发现并没有相关的错误日志。看了后台日志后，发现请求也没有进来。最后发现在zuul报了下面的错误...

```
MultipartException: Could not parse multipart servlet request; 
nested exception is java.io.IOException: The temporary upload location
[/tmp/tomcat.691065484080156170.9002/work/Tomcat/localhost/ROOT] is not valid
```
### 解决
找到问题就好解决了，在zuul的配置文件中配置tomcat的临时文件目录既可
**server.tomcat.basedir**
