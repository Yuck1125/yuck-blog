---
title: mysql区间查询的注意点
date: 2019-06-17 11:47:25
tags: "mysql"
categories: "技术"
---
### 1. Description
&emsp;&emsp; 最近在使用mysql区间查询的时候遇到的一个问题。在此简单记录以下
``` 
SELECT * from table where  1 < id  <100 ;
```
这样查询会返回table表中所有的数据或者空数据，实际上的sql其实是
```
SELECT * from table where  1; 
SELECT * from table where  0;

```

### 2. Solution
``` 
SELECT * from table where  1 < id and id <100 ;
SELECT * from table where   id  between 1 and 100 ;
```
