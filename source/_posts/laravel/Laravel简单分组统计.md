---
title: Laravel简单的分组统计
date: 2018-09-07 09:25:00
author: Kyle Liu
<!-- img: /source/images/xxx.jpg -->
top: false
<!-- cover: false -->
<!-- coverImg: /images/1.jpg -->
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: true
summary: 这是你自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要
categories: Laravel
tags:
  - PHP
  - Laravel
  - MySQL
---


## 问题描述

> 最近在开发博客的时候需要对分类进行一个简单的数量统计，以此片文章来记录学习过程

![博客分类](http://oss.anonycurse.cn/article/images/20180926/YdmZ31Y85Y9WKqz7mhe8SLIh8hVxrDecB4hRPVzj.png)

## 解决方法
> laravel 的DB 支持各种查询，最开始不怎么会，最后各种百度还是搞定了

### 数据库表
![文章表](http://oss.anonycurse.cn/article/images/20180926/l8LWYNe2oPrQb6HNUPDbuK971OM8jKApPDEDUfZm.png)
![分类表](http://oss.anonycurse.cn/article/images/20180926/RxBuEJN65Yy63SijYlkgoez0wrZUjyf8gsbBobO1.png)

### 查询语句
```PHP
$article_groups = DB::table('article_group')
	->select([
		'article_group.name as name',
		DB::raw('count(`blog_article`.`id`) as count'),

	])
	->leftJoin('article','article_group.id','=','article.g_id')
	->groupBy('article_group.id')
	->get();
dump($article_groups);
```
### 查询结果
![查询结果](http://oss.anonycurse.cn/article/images/20180926/qb06QCqFjjYOiBwOyjhWNDFSoroU71UyPVNcAOTi.png)

### SQL
```SQL
SELECT 
	blog_article_group.id as id,
	blog_article_group.name as name,
	count(blog_article.id) AS count
FROM blog_article_group
LEFT JOIN blog_article ON blog_article_group.id = blog_article.g_id
GROUP BY blog_article_group.id
```

## 总结讨论

每种语言都会遇到类似的情况，在学习Golang的时候也会遇到类似的情况

## 相关资源
[Laravel 一条 SQL 如何 count 多个字段，Laravel 一条 sql 查询每个分类的数量](https://laravel-china.org/articles/4692/laravel-a-sql-how-to-count-multiple-fields-laravel-a-sql-query-the-number-of-each-category)