# CORS 跨域资源访问


---
在介绍常见的针对 web 应用的攻击手法之前，我们先复习一下下面这些与 web 安全息息相关的知识点。理解掌握这些知识点，是进一步学习 web 安全的基础。

##跨域
当一个资源请求一个其它域名或者另外一个端口的资源时会产生一个跨域 HTTP 请求(cross-origin HTTP request)。比如说，http://domaina.example 的某HTML页面通过 ```<img>``` 的 src 请求 http://domainb.foo/image.jpg。在当今的 Web 开发中，许多页面都会从另外一个站点加载各类资源（包括CSS、图片、JavaScript 脚本以及其它类资源）。
针对不同的来源的各类资源的交互，各大浏览器往往遵循**同源策略**，对交互进行处理。

##同源策略
同源策略限制从一个源加载的文档或脚本如何与来自另一个源的资源进行交互。如果协议，端口（如果指定了一个）和主机对于两个页面是相同的，则两个页面具有相同的源。
下表给出了相对 http://store.company.com/dir/page.html 同源检测的示例:
![](/assets/屏幕快照 2017-04-03 下午11.10.26.png)

###常见跨源网络访问
同源策略控制了不同源之间的交互，例如在使用 XMLHttpRequest 或 ```<img>``` 标签时则会受到同源策略的约束。交互通常分为三类：

- 通常允许进行跨域写操作（Cross-origin writes）。例如链接（links），重定向以及表单提交。特定少数的HTTP请求需要添加 preflight。
- 通常允许跨域资源嵌入（Cross-origin embedding）。之后下面会举例说明。
- 通常不允许跨域读操作（Cross-origin reads）。但常可以通过内嵌资源来巧妙的进行读取访问。例如可以读取嵌入图片的高度和宽度，调用内嵌脚本的方法，或availability of an embedded resource.
###常见跨域资源嵌入示例
- ```<script src="..."></script>```标签嵌入跨域脚本。语法错误信息只能在同源脚本中捕捉到。
- ```<img>```嵌入图片。支持的图片格式包括PNG,JPEG,GIF,BMP,SVG,...
- ```<video>``` 和 ```<audio>```嵌入多媒体资源。

可见，在通常情况下使用 XMLHttpRequest 的跨域读取操作将会受到限制，而``` <img> ```的资源嵌入操作则可以执行。不过，若当前环境希望用户能够自由请求服务器，不受同源策略限制应该怎么办？这样的问题，就需要引入 CORS。

##CORS
是一种跨域资源共享（Cross-origin resource sharing）的解决方案。它定义了一种浏览器和服务器交互的方式来确定是否允许跨域请求。
它是一个妥协，有更大的灵活性，但比起简单地允许所有这些的要求来说更加安全。简言之，CORS就是为了让AJAX可以实现可控的跨域访问而生的。
通常我们采用配置 http header 的方式就可以启用 CORS
```Access–Control-Allow-Origin: * ```

###启用 CORS 带来的安全隐患

- 恶意跨域请求 
即便页面只允许来自某个信任网站的请求，但是它也会收到大量来自其他域的跨域请求。.这些请求有时可能会被用于执行应用层面的DDOS攻击，并不应该被应用来处理。

- 假定一个内部站点开启了CORS，如果内部网络的用户访问了恶意网站，恶意网站可以通过COR（跨域请求）来获取到内部站点的内容。

##总结
一句话概括跨域、同源策略、CORS
: 属于不同源的资源之间互相请求时会产生**跨域访问**的问题。各大浏览器通常基于**同源策略**对跨域访问进行处理。而 **CORS** 则提供了跨域资源共享的解决方案，使的同源策略能够通过 CORS 变得更加的灵活，以满足不同场景的需要。

最后，下一章我们将学习 CSRF 跨站请求伪造（Cross-Site Request Forgery），来加深对跨域、同源、CORS 运行原理的理解。

---
Copyright 2017/08/15 by Chuck Lin



若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！
![alipay.jpg-17.7kB][1]![wechat.jpg-16.7kB][2]


[1]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[2]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg















