---
layout: post
title: "install jekyll 流程"
description: ""
category: 技术分享
tags: [install jekyll]
---


* 首先本机上要安装Ruby，Mac上是系统自带的，但是据说版本低的安装Jekyll无法安装，这个有带验证，因为之前本机已经升级过一次Ruby。

{% highlight ruby linenos %}
gem sources -l  #查看源，国外大多时候是无法使用的，我们把他换成淘宝的

gem sources --remove https://rubygems.org/ #一定要带上后面的/

gem sources -a http://ruby.taobao.org/
{% endhighlight %}
<!--break-->


* 安装jekyll

{% highlight ruby linenos %}
gem install jekyll 
{% endhighlight %}

安装完成后，cd到项目根目录，使用以下命令即可运行jekyll环境，通过 localhost:4000 即可访问

{% highlight perl %}
jekyll --server
{% endhighlight %}

 


