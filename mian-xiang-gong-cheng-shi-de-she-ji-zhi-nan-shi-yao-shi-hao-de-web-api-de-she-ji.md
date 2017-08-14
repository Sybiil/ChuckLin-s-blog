# 面向工程师的设计指南 - 什么是好的 Web Api 的设计

---

##Web Api 的重要性
web api 就像一张名片一样，专业的名片可以迅速帮助你与客户之间建立信任感，也可能让你的产品在被使用前，就给客户留下业余，糟糕的负面印象。而一旦客户对你的产品产生负面情绪，这种情绪就会蔓延到产品生态圈甚至于相关公司上。

所以，优雅的 api 设计是成功的产品不可或缺的一部分。

回归主题，对后端工程师来讲，web api 的设计是每天都会接触的工作。在枯燥单调的重复性劳作中，你是否曾经认真思考过好的 web api 设计应该是什么样子的？

别担心。本篇文章围绕 web api 的设计，作者与大家分享一些 api 设计的套路与规范，让你和你的团队也能设计出优雅、健壮、扩展性强的应用程序接口。

##什么是优雅的 Web Api
回顾曾经提到过的关于设计模式的内容：设计模式的精髓在于设计原则，而非模式的生搬硬套。与此类似，优雅的 Web Api 有两个重要原则、如下所示：

- 设计规范明确的内容必须遵守相关规范
- 没有设计规范的内容必须遵守相关事实标准

好了，在长篇大论之前，先通过国际大厂的 web api 示例来观赏一下国际范儿的 web api 长啥样。

| 厂名 | uri | 备注 | 
| -- |-- | -- |
| Twitter | api.twitter.com/1.1 | "治国理政平台" |
| Google Calendar | www.googleapis.com/calendar/v3 | "大厂" |
| instagram | api.instagram.com/v1/usrs/search?q=jack | "宠物自拍网站" |
| linkedin | api.linkedin.com/v1/people-search   | "招工" |
| Tumblr | api.tumblr.com/v2 | "真的不是成人网站" |
| foursquare|  api.foursquare.com/v2/venues/search?q=apple&categoryId=asad123456 | "基于地理位置射交" |

为节约篇幅举例就到此了。如果你觉得示例较少，可自行搜索其他大厂的 Api 示例。通过分析大厂 Api 端点的设计可以揣摩总结出各种套路规范，供我们模仿学习。

##如何设计出国际范儿的 Api 端点？
###背口诀
补充一下，端点就是指 uri，应用通过 uri 访问 api 提供的功能。国际大厂的设计往往都具备以下几个特点：

- 短小便于输入的 URI。
>没人喜欢复杂的单词。

- 人可以读懂的 URI。
>名片是拿给人读的。不要把机器码和16进制写到 URI 里。

- 没有大小写混用的 URI。
>实际上一般的事实规范是建议全部小写。

- 修改方便的 URI。
> api.sample.com/user/:id 傻子都看的出来通过变量 id可以访问不同的用户信息。

- 不会暴露服务端架构的 URI。
>api.sample.com/servlet/login.do 这样的代码傻子都知道后端是用面向环境编程的语言写的。

- 规则统一的 URI。
> api.com.cn/Api/user-InfoById/info.json

- 当端点里出现两个以上的单词时，使用脊柱法（连词符号），如：profile-image

大部分工程师都很聪明，所以根本没有必要对所有的特点进行说明。你只要把以上特点都背下来然后记在脑子里。


###合理利用 REST API

谈到 WEB API 离不开的概念就是 REST API。那么优雅的 API 是不是一定要采用 REST API 来设计呢？
要理清这个问题首先要搞明白 REST 的准确含义到底是什么。REST API 的概念首先出现在 Roy Fielding 的论文中。

>狭义上的 REST API 实际上就是指符合 Fielding 的 REST 架构风格的 Web 服务系统。REST API 的设计风格严谨，考虑周全，结构优美。但过于苛刻的标准一直是狭义 REST API 推广壮大的绊脚石。

>广义上的 REST API 指符合 RPC 风格的 JSON + Http 的接口的系统。这样的 REST API 却又过于粗犷和随意了。

于是，针对 WEB API 的发展以及广义 REST 与 狭义 REST 的特点， Martin Fowler 的提出在达到完美的 REST API 之前有以下几种 API 的设计级别。

- REST LEVEL0: 使用 HTTP
- REST LEVEL1: 引入资源的概念
- REST LEVEL2: 引入 HTTP 动词
- REST LEVEL3: 引入 HATEOAS 概念

按照 HTTP 协议中，URI 代表资源，而 HTTP method 代表对资源的操作的理念，再参照上述等级，我设计了一个符合 REST LEVEL2 的设计示例如下所示

| 目的 | 端点 | 方法
| -- | -- |
| 获取用户信息列表 | api.example.com/v1/users | GET |
| 新用户注册| api.example.com/v1/users | POST |
| 获取特定用户信息 | api.example.com/v1/users/:id | GET |
| 更新用户信息  | api.example.com/v1/users/:id/ |PUT/PATCH |
| 删除用户信息 | api.example.com/v1/users/:id | DELETE |

虽然 REST Level3 描绘了一副美妙的蓝图，但截止本篇文章发表之前互联网上的大部分项目都只是朝向 Level2 的标准在努力，距离 LEVEL3 还有十分遥远的距离。故本篇文章就不在此展开对 LEVEL3 的讨论了。如果你有兴趣请自行参考相关资料了解。

##总结
本篇文章是本系列的第一个篇章。着重围绕各大厂的 API 示例与设计行业的通用理念，归纳总结了若干设计国际范儿 api 的套路与规则。熟读这些套路，遵守这些规则是你领会 web api 设计的一小步。后期的文章中我们将逐步介绍各种设计细节，完善你的网络应用程序接口。


---

参考书籍
《Web Api 的设计与开发》

**如果你觉得这篇文章帮到了你，欢迎你请我喝咖啡以鼓励我写出更多更棒的作品。
支付宝：mikumiku.lch@homtail.com**

Copyright 2017/08/15 by Chuck Lin









