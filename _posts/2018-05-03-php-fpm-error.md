---
layout: post
title:  "[proxy_fcgi:error]问题排查"
date:   2018-05-03 11:10:58 +0800
categories: php
tags: 方法
---

# php-fpm 问题排查

## 环境
1. web服务器 apache2.4 + php-fpm
2. php框架yii2
3. /etc/hosts 修改为
4. 系统ubuntu17.10
```
  127.0.0.1 backend.hyii2.com
  127.0.0.1 fronted.hyii2.com
```
5. apache2的vhosts 设置不同域名80端口的两个网站

## 故障
在访问网站时,会偶然出现卡死的状态,浏览器不能返回结果,要么超时之后出现Gateway Timeout 50X错误,要么很久之后才返回结果

## 排查1
  因为我这里使用的是本地域名,我怀疑解析的顺序出现故障
`
  ping backend.hyii2.com
  64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.036 ms
  //ping能够秒回
`

`
 traceroute backend.hyii2.com

 traceroute to fronted.hyii2.com (127.0.0.1), 30 hops max, 60 byte packets
 1  localhost (127.0.0.1)  0.497 ms  0.446 ms  0.421 ms
 `
    结果:ping 和traceroute 返回的信息证明是直接使用本地hosts的域名    

## 排查2
  一般问题都可以从日志和错误日志中查找答案,相关有一下日志:
  1. apache日志  access.log error.log
  2. php-fpm php-fpm.log
  3. yii2框架runtime日志

由于已经开启debug网页上面没有因为程序报错,那么框架日志无法记录错误信息

查找apache错误日志发现问题:
```
[proxy_fcgi:error][pid 29992] [client 127.0.0.1:38254] AH01067: Failed to read FastCGI header
[proxy_fcgi:error][pid 29992] [pid 29984] (104)Connection reset by peer:xxx
[proxy_fcgi:error] [pid 2575] (70007)The timeout specified has expired:
```
大概有3类问题,读取fastcgi头部失败,连接重置,连接超时.全部指向fcgi代理,因为这里的apache没有使用mod_php,使用php-fpm

根据提示,查找php-fpm日志,获得如下信息:
```
[03-May-2018 10:57:16] WARNING: [pool www] server reached pm.max_children setting (5), consider raising it
```

php-fpm警告最大子进程数过小需要提高

查看当前php-fpm状态
```
systemctl status php7.1-fpm
● php7.1-fpm.service - The PHP 7.1 FastCGI Process Manager
   Loaded: loaded (/lib/systemd/system/php7.1-fpm.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-05-03 10:56:45 CST; 56min ago
     Docs: man:php-fpm7.1(8)
 Main PID: 30269 (php-fpm7.1)
   Status: "Processes active: 5, idle: 0, Requests: 262, slow: 0, Traffic: 0req/sec"
    Tasks: 6 (limit: 4915)
   CGroup: /system.slice/php7.1-fpm.service
           ├─30269 php-fpm: master process (/etc/php/7.1/fpm/php-fpm.conf)
           ├─30291 php-fpm: pool www
           ├─30292 php-fpm: pool www
           ├─30299 php-fpm: pool www
           ├─30329 php-fpm: pool www
           └─30358 php-fpm: pool www

5月 03 10:56:44 wh-lenovo-g400 systemd[1]: Starting The PHP 7.1 FastCGI Process Manager...
5月 03 10:56:45 wh-lenovo-g400 systemd[1]: Started The PHP 7.1 FastCGI Process Manager.

```
状态信息显示 5个进程激活,空闲0,已经处理请求262

## 思考
开发环境需要的并发非常小,我只不过是手动点了记下,整体请求才不到300次,为什么会出现卡顿和进程不够用呢? 每个进程占用的资源不小,如果是进程的原因,那么在几千上万次请求时岂不是需要大量的资源?

## 调试
  我更改php-fpm.conf配置项
```
pm.max_children=50 # 最大进程数 50
pm.max_requests = 500 # 每个进程在重置之前要处理500次请求
```
重启php-fpm

解决问题,但是php-fpm内存占用的问题

```
pm.max_children=5
pm.max_requests=500
```
还是失败

## 查看所有php进程的内存占用
```
ps  aux | grep php-fpm | grep -v grep | awk '{ sum+=$6 } END { printf ("%d%s\n",sum/NR/1024,"M")}'
```
我这里php-fpm每个进程是20M左右的内存,如果最大进程数设置为50,那么跑满的最大内存消耗将达到1G

官方建议 最大进程数设定为 分配给web服务器的内存/最大进程的大小  

如我分给web 512内存,那么php进程ps查看最大为40M,那么max_children适合12
```
pm.max_children = Total RAM dedicated to the web server / Max child process size
```
默认安装的php会加载很多非必要的扩展,实际应用需要关闭这些扩展来降低内存使用

## 为什么选择php-fpm?
1. webserver 是只能处理动态内容的,当遇到动态请求,就会在后台执行php脚本程序并fork一个进程去完成任务,并返回结果. 即CGI程序
2. CGI协议,定义了web server 与cgi程序之间的通信,把context如path_info,script_path,request等如php $_SERVER,$_REQUESTS等全局变量的内容就是来自web server. 类似apache mod_php,就是一套cgi协议的扩展,和apache一起启动,每次有动态请求就去fork一个cgi程序.
3. fast-cgi使得cgi和webserver通过socket通信,cgi需要一个进程来管理,php-fpm就是一个主进程管理其他进程池
4. mod_php 和apache一起启动,修改php.ini配置需要重启apache
5. 性能差别,需要去测试验证
