---
layout: post
title: "nginx 做静态分离后html5中字体不显示"
description: ""
category: nginx
tags: [nginx]
---


本地访问跨域的静态资源字体文件不能够显示，字体文件可以下载，需要在nginx上添加header Access-Control-Allow-Origin

Access-Control-Allow-Origin是HTML5中定义的一种服务器端返回Response header，用来解决资源（比如字体）的跨域权限问题。

ngixn 在server上添加如下：

{% highlight python %}
location ~* \.(html|htm|gif|jpg|jpeg|bmp|png|ico|txt|js|css|woff|ttf|eot|svg|otf)$ {
           add_header Access-Control-Allow-Origin *;
 }
{%endhighlight%}