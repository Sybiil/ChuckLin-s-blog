# Python3.x、urllib、与HTTPCookieProcessor趣事


---

## 从最简单的post请求说起
python与urllib目前被广泛使用于网络蜘蛛的开发。
为了能够像浏览器一样获取到目标网站的信息，除了构建合理的request参数以外，cookie的处理也是必不可少的环节之一，而urllib中的HTTPCookieProcessor能够十分智能化的帮助我们解决这一问题。
```python
# 通过HTTPCookieProcessor()来获取一个能够处理cookie的
def get_opener(self, head):
        cookie = cookiejar.CookieJar()
        handler = urllib.request.HTTPCookieProcessor(cookie)
        opener = urllib.request.build_opener(handler)
        return opener

```
像通过上面的方式建立的openner，能够自动化处理请求过程中遇到的所有cookie数据，十分的方便。
接下来让我们尝试使用该openner发送一个post请求。
```python 
# 使用构建好的openner发送一个POST请求。
def go_login(self, openner):
        post_data = parse.urlencode(self.login_post_data).encode()
        # 通过openner发送请求
        response = openner.open(self.login_url, post_data)
```
请求发送成功，响应成功返回，大功告成。python真是一门短平快的语言！

## 伪装浏览器
仅仅成功发送一个请求是不够的，在实际项目中我们往往需要将蜘蛛伪装成
一个浏览器，从而达到欺骗目标服务器的目的。
```python
# 请求头参数
    head = {
        'Accept': '*/*',
        'Host': 'www.python.com',
        'Connection': 'keep-alive',
        'Origin': 'www.python.com',
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome'
                      '/51.0.2704.103 Safari/537.36',
        'Content-Type': 'application/json',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
        'Referer': 'your_url',
        'Accept-Language': 'zh-CN,zh;q=0.8',
        'Accept-Encoding': 'gzip, deflate'
    }
    
     def get_opener(self, head):
        cookie = cookiejar.CookieJar()
        handler = urllib.request.HTTPCookieProcessor(cookie)
        opener = urllib.request.build_opener(handler)
        header = []
        # 字典转换为truple集合.
        for key, value in head.items():
            elem = (key, value)
            header.append(elem)
        opener.addheaders = header
        return opener
```
在python的世界里一切就是这么简单，使用addheaders方法很容易就解决了问题。
现在，我们的openner不仅可以搞定cookie，还能在每次处理中带上请求头参数，它已经是一个具备最基本功能的浏览器了，真是方便。

现在，让我们尝试使用新的Openner发送一个请求看看！
```python
def search_region_info(self, openner, region_search_request_data):
        parsed_request_data = parse.urlencode(region_search_request_data).encode()
        response = openner.open(self.region_search_url, parsed_request_data)
        region_info_json = json.loads(response.read().decode())
        if len(region_info_json['Data']['Regions']) == 0:
            raise ValueError('%s地区无法查询到Region信息' % region_search_request_data['keyword'])
        return region_info_json
```
利用Charles抓包工具检查本次请求的请求头是否和我们预想的一致。
![request1.png-111.8kB][1]

## Content-Type带来的严重问题
charles抓包出来的结果与我们预想的完全不同。现在麻烦大了，不仅伪装浏览器失败，Content-Type的差异会直接造成服务器解析参数失败等严重问题。

### Content-Type:application/x-www-form-urlencoded
使用Content-Type:application/x-www-form-urlencoded提交的post请求一般是源生浏览器的表单数据提交的请求，他的请求参数会按照key1=val1&key2=val2 的方式进行编码。对于这样的数据，各服务器与控制层框架能够对请求参数进行很好的处理和映射。

### Content-Type:application/json
而提交json格式的数据，也就是Content-Type:application/json时，在服务端是无法将复杂json格式(如1个json想要映射成2个Bean这样的需求)的数据，直接映射成2个javaBean对象以后获取的。比如在springMvc中，往往需要这样进行解析：
```java
    @ResponseBody
   public Result updateCustomizedItineraryInfo(@RequestBody String requestBody) {
        Result result = new Result();
        JSONArray jsonArray = JSONArray.fromObject(requestBody);
        customizationService.updateCustomizedItineraryInfo(jsonArray);
        return result;
    }
    
    @ResponseBody
    public Result queryAirTicketList(@RequestBody SearchTicketParam searchTicketParam) {
        Result result = new Result();
        SearchTicketResult searchTicketResult = airTicketService.searchAirTickets(searchTicketParam);
        result.setData(searchTicketResult);
        return result;
    }
```


## 解决方案
笔者查阅了相关文档，但是没有找到为何cookie处理正常，而addheader没有正常工作的原因。（如果刚好你知道原因，请通过邮件也分享给我，谢谢！）
所以只好退而求其次使用下面这种方式发送Content-Type:application/json格式的post请求。
```python
def search_hotel_info(self, checkindate, checkoutdate, region_id, openner):
        self.hotel_search_request_data['data']['CheckInDate'] = checkindate
        self.hotel_search_request_data['data']['CheckOutDate'] = checkoutdate
        self.hotel_search_request_data['data']['RegionID'] = region_id
        parsed_request_data = json.dumps(self.hotel_search_request_data).encode()
        req = request.Request(self.hotel_search_url, data=parsed_request_data, headers=self.head)
        request.install_opener(openner)
        response = urllib.request.urlopen(req)
        return response.read().decode()

```

![request2.png-114.8kB][2]

现在请求头终于按照预想数据发送了！
最后，如果你对本项目有兴趣欢迎通过我的[github](https://github.com/mikumikulch/web-spider)查看与下载源代码。


## 总结
1. HTTPCookieProcessor能够自动化处理session。
2. 通过HTTPCookieProcessor创建的opener在发送请求时，无法按照预想的request header发送请求。
3. 我们通过install_opener与urlopen可以同时处理head与cookie。

  [1]: http://static.zybuluo.com/mikumikulch/3ui3k7w92v6b9ooq0vn8nf4a/request1.png
  [2]: http://static.zybuluo.com/mikumikulch/b2parqmc5oh5gjar7965nba5/request2.png