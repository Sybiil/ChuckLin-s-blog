# CSRF/XSRF 跨站请求伪造


---

CSRF（Cross Site Request Forgery, 跨站域请求伪造）也称 XSRF， 是一种网络的攻击方式，它在 2007 年曾被列为互联网 20 大安全隐患之一。其他安全隐患，比如 SQL 脚本注入，跨站域脚本攻击等在近年来已经逐渐为众人熟知，很多网站也都针对他们进行了防御。然而，对于大多数人来说，CSRF 却依然是一个陌生的概念。即便是大名鼎鼎的 Gmail, 在 2007 年底也存在着 CSRF 漏洞，从而被黑客攻击而使 Gmail 的用户造成巨大的损失。

##同源策略带来的安全隐患
回顾上一章节的内容我们有提到，在对于跨域资源交互的处理约束中，【通常允许跨域资源嵌入（Cross-origin embedding）】
常见跨域资源嵌入示例：

- ```<script src="..."></script>```标签嵌入跨域脚本。语法错误信息只能在同源脚本中捕捉到。
- ```<img>```嵌入图片。支持的图片格式包括PNG,JPEG,GIF,BMP,SVG,...
- ```<video>``` 和 ```<audio>```嵌入多媒体资源。

CSRF 攻击的原理，就是利用由于浏览器的同源策略对以上嵌入资源不做限制的行为进行跨站请求伪造的。

##CSRF 攻击原理
![屏幕快照 2017-04-03 下午12.11.03.png-62.4kB][1]


1. 用户浏览位于目标服务器 A 的网站。并通过登录验证。
2. 获取到 cookie_session_id，保存到浏览器 cookie 中。
3. 在未登出服务器 A ，并在 session_id 失效前用户浏览位于 hacked server B 上的网站。
4. server B 网站中的```<img src = "http://www.altoromutual.com/bank/transfer.aspx?creditAccount=1001160141&transferAmount=1000">```嵌入资源起了作用，迫使用户访问目标服务器 A
5. 由于用户未登出服务器 A 并且 sessionId 未失效，请求通过验证，非法请求被执行。

##如何防御 CSRF
- referer 验证解决方案。
最简单的方法依赖于浏览器引用页头部。大多数浏览器会告诉 Web 服务器，哪个页面发送了请求。如：
```
POST /bank/transfer.aspx HTTP/1.1
Referer: http://evilsite.com/myevilblog
User-Agent: Mozilla/4....
Host: www.altoromutual.com
Content-Length: 42
Cookie: SessionId=x3q2v0qpjc0n1c55mf35fxid;
```
creditAccount=1001160141&transferAmount=10不少站点通过 referer 验证来防止盗链本站图片资源。
不过由于 http 头在某些版本的浏览器上存在可被篡改的可能性，所以这个解决方案并不完善。

- Token 解决方案
令牌解决方案向表单添加一个参数，让表单在用户注销时或一个超时期限结束后过期
```
<form id="transferForm" action="https://www.altoromutual.com/bank/transfer.aspx" method="post">

Enter the credit account:
<input type="text" name="creditAccount" value="">
Enter the transfer amount:
<input type="text" name="transferAmount" value="">

<input type="hidden" name="xsrftoken" value="JKBS38633jjhg0987PPll">

<input type="submit" value="Submit">

</form>
```
或者将服务端动态生成的 Token 加入到 自定义 http 请求头参数中
```
POST /bank/transfer.aspx HTTP/1.1
Referer: https://www.altoromutual.com/bank
xsrftoken: JKBS38633jjhg0987PPll
User-Agent: Mozilla/4....
Host: www.altoromutual.com
Content-Length: 42
Cookie: SessionId=x3q2v0qpjc0n1c55mf35fxid;

creditAccount=1001160141&transferAmount=10
```
token 解决方案的问题在于前后端代码的巨大变更。并且每一步都动态生成 token 并且对 token 进行验证的话，也会造成额外的资源开销。可以尝试在关键性操作的地方再加上 token 验证逻辑。但是，token 验证锁带来的前后端代码的变动所带来的消耗，则需要慎重考虑。

##总结
要记住 CSRF 不是黑客唯一的攻击手段，无论你 CSRF 防范有多么严密，如果你系统有其他安全漏洞，比如跨站域脚本攻击 XSS，那么黑客就可以绕过你的安全防护，展开包括 CSRF 在内的各种攻击，你的防线将如同虚设。

CSRF 是一种危害非常大的攻击，又很难以防范。目前几种防御策略虽然可以很大程度上抵御 CSRF 的攻击，但并没有一种完美的解决方案。一些新的方案正在研究之中，比如对于每次请求都使用不同的动态口令，把 Referer 和 token 方案结合起来，甚至尝试修改 HTTP 规范，但是这些新的方案尚不成熟，要正式投入使用并被业界广为接受还需时日。在这之前，我们只有充分重视 CSRF，根据系统的实际情况选择最合适的策略，这样才能把 CSRF 的危害降到最低。





#####参考链接
- [避免 SQL 注入](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/09.4.md), by wuyuanwei
- [Web 攻击与防护](http://liuwanlin.info/webgong-ji-yu-fang-hu/), by liuwulin
- [浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [总结 XSS 与 CSRF 两种跨站攻击](https://blog.tonyseek.com/post/introduce-to-xss-and-csrf/)
- [XSS 攻击入门](http://www.cnblogs.com/bangerlee/archive/2013/04/06/3002142.html)
- [关于Web安全，99%的网站都忽略了这些](https://blog.wilddog.com/?p=290), by wilddog
- [预防跨站点请求伪造：了解浏览器选项卡中的隐藏危险](https://www.ibm.com/developerworks/cn/web/se-appscan-detect-csrf-xsrf/)
- [CSRF 攻击的应对之道](http://www.importnew.com/5839.html)

  [1]: http://static.zybuluo.com/mikumikulch/ta5h6dxooi0wzkkqq1st8y7n/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-03%20%E4%B8%8B%E5%8D%8812.11.03.png
  
  
---

Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！
![alipay.jpg-17.7kB][99]
![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg









