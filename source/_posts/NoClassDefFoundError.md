---
title: 'A java.lang.NoClassDefFoundError: ch/qos/logback/classic/spi/ThrowableProxy'
date: 2019-06-13 12:19:17
tags: "Exception"
categories: "Summary"
---
### Describtion
&emsp;&emsp;今天打完jar包，上传服务器时，执行脚本时报错。
### Solution
&emsp;&emsp;上传jar包前，把需要更新的服务先停了，不然可能就会抛出此错误。
