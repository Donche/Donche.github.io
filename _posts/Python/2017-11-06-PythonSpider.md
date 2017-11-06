---
layout: post
title: 手写豆瓣爬虫爬到了八万多个专辑信息
category: 爬虫
catalog: true
tags: 
    - 2017
    - Python
    - 爬虫
---

*这个项目参考了一位大佬的博客[[1]](#1)。这位大佬也是机械转出来的，在我下定决心搞代码之前很仔细地看了大佬的学习轨迹，为我提供了不少学习思路。然而大佬貌似最近一年都没有更新博客了，~~果然是已经找到工作了~~。此外还仔细看了《精通Python 网络爬虫》[[2]](#2)好几遍。本来打算用Scrapy做的，结果手写了一个，试了一下就爬到了五万多个专辑，而且也没有因为爬太快而被豆瓣403掉（下次我会努力加上多线程争取被403），所以就直接用手写的爬虫了*    

这个小项目主要是为了爬到豆瓣音乐上的专辑信息，主要包括专辑名称、艺术家名称、评分、评分人数、专辑流派（摇滚、布鲁斯、流行等等...）、专辑类型（专辑、EP、单曲等等...）和发行时间。其中一些信息在很多专辑中是缺失的，在爬取过程中这些信息都用None 来代替。爬虫的起点由我最近超级喜欢的《Guruguru Brain Wash》合集来完成，新的url从豆瓣专辑页面的推荐区域获取（喜欢听"Guruguru Brain Wash"的人也喜欢的唱片  · · · · · ·）。这次爬虫一共爬到了八万多个专辑信息。因为由数据库存储所有信息，所以可以随时继续爬，理论上可以爬到豆瓣大部分高分专辑（因为我发现豆瓣推荐的大部分都是高分专辑，很少是7分以下）。所以在爬虫继续工作的时候，写一些~~段子~~内容总结一下爬虫的实现方法。

# 1. 爬虫框架
关于一般的爬虫框架，引用慕课网的图片[[3]](#3)，如下：
![img](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/SpiderStructure.jpg)
其中，价值数据即为所要获取的内容，下载器负责下载网页，解析器负责将下载到的内容解析，URL管理器负责管理链接（已经爬取的链接、等待爬取的链接和爬取失败的链接等...），调度端负责整个爬虫的调度。     
具体爬虫工作的流程如下：
1. 调度器查询是否有未爬取的链接
2. 如果无则完成爬虫应用，如果有则获取一个链接
3. 下载器根据获取的链接下载网页，调度器将下载到的网页传递给解析器
4. 解析器解析下载的数据，获得新的url和有价值数据 
5. 调度器将获得的url传递给url管理器
6. 调度器将获得的有价值数据传递给数据库
7. 跳转至1     

# 2. 调度器
调度器主要负责协调各个部分的工作，确定了整个爬虫的工作顺序。这部分也没什么可以解释的...直接上代码吧：
```python
class Spider(object):
    def __init__(self):
        #初始化
        self.mongo = mongoDBThing.MongoDBThing()
        self.downloader = Downloader.Downloader()
        self.parser = Parser.Parser()

    def craw(self, root_url):
        #爬虫计数
        count = 1
        self.mongo.add_new_url(root_url)
        while self.mongo.has_new_url():
            try:
                new_url = self.mongo.get_new_url()
                print('craw %d : %s' % (count, new_url))
                html_cont = self.downloader.download(new_url)
                if html_cont == 404:
                    self.mongo.add_404_url(new_url)
                else:
                    print('parsing')
                    new_urls, new_data = self.parser.parse(new_url, html_cont)
                    print('adding new urls')
                    self.mongo.add_new_urls(new_urls)
                    print('collecting data')
                    self.mongo.collect_data(new_data)
                if count == 50000:
                    break
                count += 1
            except:
                print('craw failed')
        self.mongo.output()

if __name__ == "__main__":
    obj_spider = Spider()
    url_start = 'https://music.douban.com/subject/26590388/'
    obj_spider.craw(url_start)

```


# 3. 数据库
为什么要在爬虫中用数据库而不直接存储在内存里或者写入文件？    
当然是因为方便了。现爬现用、断点续爬，不用担心电脑死机代码出bug，爬到一个就是一个，这样的爬虫鲁棒性要好很多。     
于是我用mongoDB 来存储所有url 和爬到的内容。关于mongoDB 有很多教程可以参考，一个小时就可以上手，非常方便。在爬虫工作之前，需要记得打开mongoDB 的服务。第一次跑的时候还需要在shell 里创建相应的database 和collection，然后才可以在python中调用。     
数据库主要负责存储获得到的所有有价值数据。此外还数据库有关的是url管理器。url管理器需要完成的主要有以下几个任务：
* 添加新链接
* 查询是否有待爬取的链接
* 获取新链接
因此数据库需要存储的有三类链接，分别为已爬取的、待爬取的和爬取失败的。每次调度器获取新链接之后，都需要把这个链接由待爬取转移到以爬取。Python代码如下：

```python
class MongoDBThing(object):
    def __init__(self):
        client = pymongo.MongoClient('localhost', 27017)
        db = client.MyMusic
        self.newUrlsCol = db.newUrls
        self.oldUrlsCol = db.oldUrls 
        self.notFoundUrls = db.notFoundUrls
        self.music = db.music
        
    def add_new_url(self, url):
        if url is None:
            return
        if self.newUrlsCol.find({'url':url}).count() == self.oldUrlsCol.find({'url':url}).count() == 0:
            self.newUrlsCol.insert({'url': url})

    def add_new_urls(self, urls):
        if urls is None or len(urls) == 0:
            return
        for url in urls:
            self.add_new_url(url)
            
    def has_new_url(self):
        return self.newUrlsCol.find().count() != 0
    
    def get_new_url(self):
        urlDoc = self.newUrlsCol.find_one()
        self.newUrlsCol.remove(urlDoc)
        self.oldUrlsCol.insert(urlDoc)
        return urlDoc['url']
    
    def add_404_url(self, url):
        self.notFoundUrls.insert({'url':url})
         
    def collect_data(self, data):
        if data is None: 
            return
        self.music.insert(data)

```

# 4. 解析器
解析器是爬虫实现过程中最复杂的一部分。因为需要根据爬取的具体网页来设计，每次网页有所变动的时候都需要重新设计解析器。    
在这里我主要使用beautifulsoup 和re 库来解析网页。beautifulsoup 是一个很强大的python 包，内置的find() 可以通过id 或者class快速定位到网页的任意区域，同时还支持正则表达式搜索。    
解析器主要包括三个部分：网页解析、获取新链接、获取有价值数据。其中网页解析到的数据直接传递给后两个部分做处理。第一个部分的Python实现如下：
```python
    def parse(self, page_url, html_cont):
        if page_url is None or html_cont is None:
            print('page_url is none')
            return
        soup = BeautifulSoup(html_cont, 'html.parser')
        print('getting new urls')
        new_urls = self._get_new_urls(soup)
        print('getting new data')
        new_data = self._get_new_data(page_url, soup)
        return new_urls, new_data
```

## 4.1 获取新链接
在网页上按F12打开chrome的开发工具，然后找到豆瓣推荐那一部分，发现整个推荐区域在一个class ="content clearfix"的部分中，如下图：
![img](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/doubanF12.jpg)
其中每个推荐专辑的链接都在超链接标签内，且格式均为"https://music.douban.com/subject/"，于是我们可以将这个区域内部所有满足这个条件的超链接均作为新链接来使用。代码如下：
```python
def _get_new_urls(self, soup):
    new_urls = set()
    recommend = soup.find('div', class_='content clearfix')
    links = recommend.find_all('a', href=re.compile(r"https://music\.douban\.com/subject/\d+/$")) 
    for link in links:
        new_url = link['href']
        new_urls.add(new_url)
    return new_urls
```

## 4.2 获取有价值数据
这些信息的获取方法与上面一样。例如专辑名称，找到的截图如下：
![img](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/doubanAlbumName.jpg)
wrapper内的第一个h1 即为专辑名称。评分以及评分人数同理。其他的可以简单一点，在确定所有信息都在"info"内之后，直接用正则表达式在其中搜索即可。代码如下：
```python
def _get_new_data(self, page_url, soup):
    res_data = {}
    res_data['url'] = page_url
    print('parse url success')
    try:
        res_data['AlbumName'] = soup.find('div', id='wrapper').h1.text.strip()
        res_data['score'] = soup.find('strong', class_='ll rating_num').string
        res_data['sco_num'] = soup.find('div',class_='rating_sum').a.span.text
        info = soup.find('div', id='info')
        if info.find(text=re.compile('表演者')):
            res_data['performer'] = info.find(text=re.compile('表演者')).next_element.text.strip()
        else:
            res_data['performer'] = 'None'
        if info.find(text=re.compile('流派')):
            res_data['genre'] = info.find(text=re.compile('流派')).next_element.strip()
        else:
            res_data['genre'] = 'None'
        if info.find(text=re.compile('专辑类型')):
            res_data['type'] = info.find(text=re.compile('专辑类型')).next_element.strip()
        else:
            res_data['type'] = 'None'
        if info.find(text=re.compile('发行时间')):
            res_data['time'] = info.find(text=re.compile('发行时间')).next_element.strip()
        else:
            res_data['time'] = 'None'
    except:
        print('parse data failed')
    finally:
        return res_data
```
为什么要加那么多if else呢？因为后面这些信息一些专辑是没有的。没有的话，beautifulsoup自然找不到对应的内容。如果对一个None 调用next_element 的话，会直接抛出异常。然而给每个信息都加上try except感觉很麻烦，于是就写成了if else。这样做，即使其中一个信息缺失，还可以继续解析下一个信息，保证不会漏掉。
# 5. 下载器
下载器主要负责把网页内容下载下来，所以是相对简单的。但是很多网站有反爬虫机制，检测到短时间内大量访问时会403拒绝访问，这就造成了一些麻烦。具体关于爬虫与反爬虫与反反爬虫与反反反爬虫与反反反...是一个很深刻的问题，具体做法网上有很多教程，[比如这个](http://www.shenjian.io/blog/?p=275)。因为我的爬虫比较慢（并没有用多线程，平均下来大概一秒一个），而且没有设置cookie，所以爬了几万个网页豆瓣也没有管我（以后还是加上多线程再试试吧）。但是即使豆瓣没有给我特殊待遇，我依然贴心地加上了403的处理步骤：每拒绝一次等半分钟，依次叠加。代码如下：
```python
def _download(self, url):
    if url is None:
        print('url None')
        return None
    try:  
        print('Requesting')
        headers = {'user-agent':'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.79 Safari/537.36',
        'Accept-Language':' zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
        'Connection':'keep-alive',
        'referer':'baidu.com'}
        opener = urllib.request.build_opener()
        headall = []
        for key,value in headers.items():
            item = (key,value)
            headall.append(item)
        opener.addheaders = headall
        urllib.request.install_opener(opener)
        print('Opening url')
        response = urllib.request.urlopen(url, timeout = 10)
        print('checking attributes')
    except urllib.error.HTTPError as e:
        print('error: ' + str(e))
        if e.code == 403:
            return 403
        elif e.code == 404:
            return 404
        else:
            return
    if response.getcode() != 200:
        print('get_new_url failed')
        return None
    return response.read()

def download(self, url):
    count = 1
    sleeptime = 30
    while True:
        res = self._download(url)
        if res == 404:
            return 404
        elif res == 403:
            if count > 30:
                return
            print('waiting 403, waiting time:',sleeptime * count)
            sleep(sleeptime * count)
            count += 1
        else:
            return res
```


*参考资料*
1. <span id="1"></span> http://blog.csdn.net/xuelabizp/article/details/51068441
2. <span id="2"></span> 《精通Python 网络爬虫》 韦玮著
3. <span id="3"></span> http://www.imooc.com/article/15028