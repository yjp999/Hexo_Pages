---
title: 爬虫系列（1）——解析JS
date: 2016-04-21 21:50:18
tags: [爬虫,Javascript]
categories: [编程]
---


**网络爬虫（Web Crawler）**也叫网络蜘蛛（Web Spider），蚂蚁，自动检索工具，是一种自动浏览网络的程序，也可称为网络机器人。爬虫主要分为两大类：
 
- **广度搜索** ：例如一些著名搜索引擎Google、百度、必应、雅虎等，它们都属于广度搜索爬虫，原理就是每将一个页面所有的链接拿下来后，去遍历所有的链接，再按照上述步骤不断抓取页面直到找到相关的关键词，并按相关度对页面进行排序

- **垂直搜索** ：简单来说就是抓取特定的数据，如京东上面的所有书籍信息（包括书名、作者、出版社、语种、价格、链接等等），将这些特定数据（也称之为结构化数据）序列化（JSON或XML）之后存储到数据库或者文件里

		因为实习期间主要工作就是垂直搜索这块儿，所以在接下来的爬虫系列里，我就垂直搜索爬虫方面的一些难点和新手经常会踩的几个坑做一个简要的总结，既是对自己实习经历的一个回顾加深，也是希望能够帮到即将或者正在踩坑的那些童鞋。
		
-------------------

## Ajax+Javascript生成动态信息

> 目前Web前端技术越来越成熟，许多网页不再是简单的静态网页，而是动态网页，这些动态网页中许多信息都是通过Ajax请求从服务端动态获取的，因此要想抓取那些源代码中不包含的信息，就必须通过一些技术手段来得到它们。

我在分析页面时主要使用Chrome浏览器的F12（开发者工具），很强大，当然也可以使用FireFox浏览器的插件Firebug来分析一个较为复杂的网站或者使用抓包软件如[Fiddler](http://www.telerik.com/fiddler)。这些工具的使用方法和一些技巧可以自行Google，百度。

### 方法1：手动解析JS
这种方法比较耗时同时难度最大，但是一旦分析成功，抓取速度会远远快于方法2，具体原因我会在方法2中说明。下面我会演示一个例子如何去解析动态信息。

拿抓取[高圆圆怀抱干女儿](http://slide.ent.sina.com.cn/star/w/slide_4_704_137965.html#p=1)这条新浪新闻举个栗子，这条新闻的评论在源代码中是直接拿不到的，其实新浪所有新闻网页都是不能通过网页源码拿到评论内容的，此时就需要我们去分析网络请求并找出评论的来源。
1. 得到评论链接

{% asset_img review.jpg comments_url %}
打开F12功能后，点击图中数字1 处的“Network”，查看所有请求信息，这其中包括图片和css文件、Img、JS、以及Doc等所有请求的url，通过图中数字2 处的过滤功能，在数字3 处输入"comment"信息，因为要抓取的是评论内容，所以输入的是comment（仅仅是一种基于经验的猜测），当然也可以下拉评论内容，最后在network中发现评论的来源。回车后发现只剩下了3条Type类型均为script的url，每条都试过之后，发现第二条的url可以打开而且页面内容确实是评论信息。
2. 分析评论链接
根据步骤1，我们得到了评论的链接：
http://comment5.news.sina.com.cn/page/info?version=1&format=js&channel=yl&newsid=slidenews-album-704-137965&group=1&compress=1&ie=gbk&oe=gbk&page=1&page_size=100&jsvar=requestId_6363077
可以看到url后面有许多参数，其中比较重要的有:
**format**: js [也可以写成json，即返回的数据格式]
**newsid**: slidenews-ablum-704-137965 [新闻id，每条新闻的唯一标识]
**page**: 1 [评论内容的页码，2，3，4...]
即我们只要知道每条新闻的id，就可以通过上面这条url得到每条新闻的所有评论内容，在我将format赋值为json后，得到的评论内容如下：

{% asset_img comments.jpg comments_content %}
可以通过上图发现，评论在cmntlist这个列表里，每一条评论的详情又是一个字典，在字典中key为content，对应的value则是评论内容。上图中的json内容之所以能够在浏览器中结构化显示，是因为我使用了一个chrome插件：JSONView。谁用谁知道...

3. 小结
其实对于有些安全性较高的网站的请求分析起来还是相当复杂和麻烦的，这里的例子比较简单，我也简略了许多内容，所以对于刚刚入手爬虫的同学来说可能还是有点费解的。这些技巧有时候靠直觉和经验，所以只有多多尝试才能掌握这些技能。

### 方法2：用mini浏览器解析JS
> 其实只要技巧熟练，所有关于ajax生成的动态内容都是可以通过方法1获取到的。但是对于刚开始的爬虫新手来说，方法1有一定的门槛，这时就可以通过方法2来操作。目前有很多这样的浏览器引擎，如主要用于自动化测试的[Selenuim](http://www.seleniumhq.org/)、没有浏览器界面的[PhantomJS](http://phantomjs.org/)、[HtmlUnit](http://htmlunit.sourceforge.net/) 以及[CasperJS](http://casperjs.org/)，也有很多基于webkit的其他浏览器引擎，也就是说我们可以用这些引擎来渲染js，最后生成DOM树对元素进行操控，但是模拟浏览器环境内存和CPU消耗都非常严重，因此抓取效率将大打折扣。

下面的一段python代码则是用方法2的应用(基于PhantomJS)：
``` python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
def getjsContent(url, xpath_exp):
	service_args = [
		'--load-images=false',
		'--disk-cache=true',
	]
	url = "http://disqus.com/embed/comments/?base=default&\
		version=208e70781fad1709ad376036d91294bc&\
		f=fattoquotidiano&t_i=2592376"
	dr  = webdriver.PhantomJS('/usr/bin/phantomjs',service_args=service_args)
	dr.get(url)
	dr.execute_script("window.scrollTo(0, document.body.scrollHeight);")
	try:
		# print dr.page_source
		element = WebDriverWait(dr, 2).until(
			EC.presence_of_element_located((By.XPATH, xpath_exp))
		print element.text
	finally:
		dr.quit()
```
上述代码中用到了*xpath*，后续章节会讲到。

### 总结:
| method    |    strength| weakness|
| :------: | :--------:| :-------: |
| 方法1  | 无需render、高效、抓取速度快 |  较为复杂、需要仔细分析页面   |
| 方法2     |  傻瓜式解决、无需分析请求|  消耗cpu和内存、渲染js,css、速度慢  |

