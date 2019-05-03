---
title: LAMP学习之Apache添加SSL证书
date: 2019-04-07 09:25:00
author: Kyle Liu
<!-- img: /source/images/xxx.jpg -->
top: false
cover: false
<!-- coverImg: /images/1.jpg -->
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: true
summary: 这是你自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要
categories: Linux
tags:
  - Linux
  - Apache
  - SSL
---


## LAMP学习之Apache添加SSL证书

## 环境装备
- 已经安装好了的一套Lamp环境，和
- 阅读过Httpd所有官方文档的的自信。

## 开始安装
### 1.下载证书

> 进入阿里云域名管理，再点击右边的SSL

![进入SSL](http://oss.anonycurse.cn/article/images/20181016/mYcEkWLm1bhlJeFDGxXcsBsci3tMKWLhNFEvhBRs.png "进入SSL")

> 申请完证书后记得点击左上角返回列表下载证书
![申请SSL](http://oss.anonycurse.cn/article/images/20181016/LA9MaQ3IqD4QMrB5OhAhPxXPG1PUYssGOfOh5qcn.png "申请SSL")



![下载证书](http://oss.anonycurse.cn/article/images/20181016/W9fdbI03VOMYjSJszyHRM0bGlwKA9jT4ukwV2jkZ.png "下载证书")


![下载证书2](http://oss.anonycurse.cn/article/images/20181016/muxi4oUZfjctSdoYw4CiLr9z2eRbBzv6uuksYZCR.png "下载证书2")

### 上传证书
> 我在网上看见有人把SSL证书直接放在项目里面，对于这种举动我感觉还是有点小怕，
最后我觉得证书放在在apache 的conf目录下

![证书路径](http://oss.anonycurse.cn/article/images/20181016/1njDCGjbCdHEEuBHLTpxbZHshGTQuuCNRuu1TNbQ.png "证书路径")

### 修改配置
- 1.httpd.conf

```apache
#1. 开启必要的module
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
LoadModule ssl_module modules/mod_ssl.so

#2.开启SSL配置文件
Include conf/extra/httpd-ssl.conf

#3.导入外部主机配置文件
#注意，我安装Apache的时候是给每个网站添加一个配置文件，所以会有一个vhosts专门存放网站主机配置文件
# Virtual hosts
Include conf/vhosts/*.conf
```
- conf/extra/httpd-ssl.conf
> 这个配置文件和httpd.conf有点类型，存放了一些关于HTTPS的配置，包括一些全局配置和一个默认的虚拟主机配置，这个我喜欢删掉默认想虚拟主机，只保留一些全局配置就好

```apache
Listen 443

SSLCipherSuite HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM
SSLHonorCipherOrder on

SSLProtocol all -SSLv2 -SSLv3
SSLProxyProtocol all -SSLv3

SSLPassPhraseDialog  builtin

SSLSessionCache        "shmcb:/web/bin/apache/logs/ssl_scache(512000)"
SSLSessionCacheTimeout  300
```
- conf/vhosts/www.angexinjia.com.conf

```apache
#默认HTTP主机
<VirtualHost *:80>
    DocumentRoot "/web/www/www.anonycurse.com/public"
    ServerName www.anonycurse.com
    ServerAlias www.anonycurse.com www.anonycurse.com

#    ErrorLog logs/www.anonycurse.com.error.log
    CustomLog logs/www.anonycurse.com.access.log common

# Laravel 项目配置【开启HTTPS后可以注释掉】
#    <Directory "/web/www/www.anonycurse.com/public">
#        AllowOverride All
#        Require all granted
#        DirectoryIndex index.php
#    </Directory>

# 开启后HTTP跳转HTTPS配置
    RewriteEngine on
    RewriteCond   %{HTTPS} !=on
    RewriteRule   ^(.*)  https://www.anonycurse.com$1 [R=permanent,L]


</VirtualHost>

<VirtualHost *:443>
    DocumentRoot "/web/www/www.anonycurse.com/public"
#  因为是HTTPS单域名所以不需要太多Alias    
	ServerName www.anonycurse.com
    ServerAlias www.anonycurse.com

    ErrorLog logs/www.anonycurse.com.ssl.error.log
    CustomLog logs/www.anonycurse.com.access.ssl.log common

#	HTTPS Laravel 配置
    <Directory "/web/www/www.anonycurse.com/public">
        AllowOverride All
        Require all granted
        DirectoryIndex index.php
        SSLOptions +StdEnvVars
    </Directory>
#	证书配置
    SSLEngine on
    SSLCertificateFile /web/bin/apache/conf/ssl-key/www.anonycurse.com/public.pem
    SSLCertificateKeyFile /web/bin/apache/conf/ssl-key/www.anonycurse.com/1541114560394.key
    SSLCertificateChainFile /web/bin/apache/conf/ssl-key/www.anonycurse.com/chain.pem

    <FilesMatch "\.(cgi|shtml|phtml|php)$">
        SSLOptions +StdEnvVars
    </FilesMatch>

    BrowserMatch "MSIE [2-5]" \
         nokeepalive ssl-unclean-shutdown \
         downgrade-1.0 force-response-1.0

    CustomLog "/web/bin/apache/logs/ssl_request_log" \
          "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"

</VirtualHost>
```