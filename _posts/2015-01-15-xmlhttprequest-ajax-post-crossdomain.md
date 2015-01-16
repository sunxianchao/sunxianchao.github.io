---
layout: post
title: 记一次ajax(XMLHttpRequest) post跨域请求  
description: "XMLHttpRequest 这货是浏览器内置的对象，早期没有js框架的时候使用ajax都是用这个原始的对象发送ajax请求，需要写很长的代码。"  
keywords: ajax, 跨域, XMLHttpRequest
category: 工作记录
tags: [ajax, 跨域]  

---


### 1. XMLHttpRequest对象介绍  

>XMLHttpRequest 对象用于在后台与服务器交换数据。
XMLHttpRequest 对象是开发者的梦想，因为您能够：
在不重新加载页面的情况下更新网页
在页面已加载后从服务器请求数据
在页面已加载后从服务器接收数据
在后台向服务器发送数据
所有现代的浏览器都支持 XMLHttpRequest 对象  

XMLHttpRequest 这货是浏览器内置的对象，早期没有js框架的时候使用ajax都是用这个原始的对象发送ajax请求，需要写很长的代码。  

<!--break-->

### 2. jsonp解决使用get方式的跨域请求  

jsonp通过请求后带上callback函数在返回时直接执行这个callback 函数即可实现跨域，那么问题来了，很多时候我们会选择post请求，比如文件上传等该如何解决呢？  
![同源策略](/images/ajax-post-cross-domain-error.png)


### 3. 解决post跨域请求  

实际情况如下一个静态页面  

<code> 
http://127.0.0.1:9999/static/cors.html
</code> 

在页面中需要ajax跨域访问我的另外一个应用地址  

<code> 
http://127.0.0.1:8080/game/remoting/test.html
</code>

如果不做任何设置那么在浏览器console中会得到如下提示：    
>
已阻止交叉源请求：同源策略不允许读取 http://127.0.0.1:8080/game/remoting/test.html 上的远程资源。可以将资源移动到相同的域名上或者启用 CORS 来解决这个问题  

* 那么为什么会出现这个提示呢？如果两个页面的协议、域名和端口是完全相同的，那么它们就是同源的。同源策略是为了防止从一个地址加载的文档或脚本访问从另外一个地址加载的文档的属性。说白了就是为了安全考虑。此外如果两个页面的主域名相同，则还可以通过设置 document.domain 属性将它们认为是同源的。  

在发送的请求头信息中，浏览器使用 Origin 这个 HTTP 头来标识该请求来自于 哪个；在返回的响应信息中，使用 Access-Control-Allow-Origin 头来控制哪些域名的脚本可以访问该资源。如果设置 Access-Control-Allow-Origin:*，则允许所有域名的脚本访问该资源。如果有多个，则只需要使用逗号分隔开即可。  

主要是在服务端response上加上需要跨域的域名就可以了。  

{% highlight java %}
response.setHeader("Access-Control-Allow-Origin", "*");
{% endhighlight %}

* cors.html 这个页面上发起ajax需要使用XMLHttpRequest对象

{% highlight javascript %}
var xhr = new XMLHttpRequest();
    var url = 'http://127.0.0.1:8080/game/remoting/test.html';
    function doCrossDomainRequest() {
      if (xhr) {
        xhr.open('POST', url, true);
        xhr.onreadystatechange = handler;
        xhr.send();
      } else {
        document.getElementById("content").innerHTML = "不能创建 XMLHttpRequest";
      }
    }
    function handler(evtXHR) {
      if (xhr.readyState == 4) {
        if (xhr.status == 200) {
          var response = xhr.responseText;
          document.getElementById("content").innerHTML = "结果：" + response;
        } else {
          document.getElementById("content").innerHTML = "不允许跨域请求。";
        }
      }
      else {
        document.getElementById("content").innerHTML += "<br/>执行状态 readyState：" + xhr.readyState;
      }
    }
    
{% endhighlight %}

具体要post什么请求信息可以查阅XMLHttpRequest对象的[详细用法](http://www.w3school.com.cn/xml/xml_http.asp)



 
