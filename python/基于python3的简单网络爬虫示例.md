# 基于python3的简单网络爬虫示例


---

网上用python写爬虫的示例大多数是基于python2.x版本的。由于爬虫需要的库在python3版本大都进行了重写，所以各种包的位置和用法都有所改变。
笔者参照网上2.x版本的代码，写了这个基于python3.x版本的简单爬虫示例，供大家参考。


```python
# -*- coding:utf-8 -*-

import re
import urllib.error
import urllib.parse
import urllib.request


class QSBKSpider:

    # 初始化返回页面用正则表达式
    pattern = re.compile(r'.<h2>(.*?)</h2>.*?<div class="content">(.*?)<!--(\d*?)-->.*?</div>'
                         r'(.*?)<div class="stats.*?class="number">(\d*?)</i>', re.S)
    # 初始化图片正则表达式(过滤用)
    img_pattern = re.compile(r'<img.*?/>')

    # 初始化http请求头,页数等信息
    def __init__(self):
        self.pageIndex = 1
        self.header = {
            'Connection': 'Keep-Alive',
            'Content-Type': 'application/x-www-form-urlencoded',
            'User-Agent': 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)',
            'Accept-Encoding': 'utf-8, deflate',
            'Host': 'www.qiushibaike.com',
        }

    # 取得页面数据
    def get_page_content(self, page_index):
        try:
            self.pageIndex = page_index
            url = 'http://www.qiushibaike.com/textnew/' + self.pageIndex
            # 构建reqeust
            req = urllib.request.Request(url, None, self.header)
            # 发起请求
            rep = urllib.request.urlopen(req)
            # 读取返回数据
            content_bytes = rep.read()
            page_content = content_bytes.decode('utf-8')
            return page_content
        # 打印通信错误原因
        except urllib.error.HTTPError as e:
            print(e.code), (e.reason)
        except urllib.error.URLError as e:
            print(e.reason)
        finally:
            print('页面获取完毕')

    # 获取当前页面的所有段子
    def get_page_stories(self, content):
        # 对文本进行匹配
        re_results = re.findall(QSBKSpider.pattern, content)
        # 存放过滤掉图片段子后的段子集合
        page_stories = []
        for result in re_results:
            if not QSBKSpider.img_pattern.search(result[3]):
                page_stories.append(result)
        return page_stories

    # 段子生成器.按需返回所有的段子
    def get_one_stroy(self, page_stories):
        for story in page_stories:
            print(story[1])
            yield story
        return 'done'

    # 主处理函数
    def app_start(self):
        page_index = input('请输入页码')
        page_stories = self.get_page_stories(self.get_page_content(page_index))
        g = self.get_one_stroy(page_stories)
        while input('输入回车读取段子.停止读取请输入EOF') != 'EOF':
            try:
                next(g)
            except StopIteration as e:
                print('第%s页故事读取结束' % self.pageIndex)
                break
        print('读取完毕,程序结束')

# start
qs = QSBKSpider()
qs.app_start()


```
该项目现在通过[github](https://github.com/mikumikulch/web-spider)进行管理。
你可以通过我的[github个人页面](https://github.com/mikumikulch/)来获取更多的关于python3.x版本的项目示例。欢迎follow。


