#CrawlerLib模块
## 一、概述 ##

### 承担的需求（主要秘密） ###
  该模块承担的主要任务是从电商网站上爬取商品信息，选择爬取策略。
  
  该模块需要实现自动爬取指定网页的功能，然后分析爬取内容，将需要的商品信息交给数据处理模块处理、存储。
  
### 可能会修改的实现(次要秘密) ###
* 该模块需要兼容不同的电商网站，对不同的网站采取不同的网页爬取方式。

* 该模块需要实现爬取策略可变更的功能，以后会增加新的电商网站和新的商品种类，所以需要增加新的网页爬取方式，此时应该面向扩展开放，面向修改关闭。

* 定期进行爬取的控制（调用）方式可能会改变。

* 和数据处理模块的接口可能会改变，如果数据库的组织形式发生改变，则返回的数据格式也需要变化。

### 设计的相关质量属性 ###
* R3 电商网站兼容性

### 设计概述 ###
其需要考虑的核心问题为：爬虫数据的爬取无非就是数据流的转换和处理。数据流控制过程如下：  

1.引擎打开一个网站，找到处理该网站的Spider并向该spider请求第一个要爬取的URL(s)。  

2.引擎从Spider中获取到第一个要爬取的URL并在调度器中以Request调度。  

3.引擎向调度器请求下一个要爬取的URL。  

4.调度器返回下一个要爬取的URL给引擎，引擎将URL通过下载中间件(请求(request)方向)转发给下载器。  

5.一旦页面下载完毕，下载器生成一个该页面的Response，并将其通过下载中间件(返回(response)方向)发送给引擎。  

6.引擎从下载器中接收到Response并通过Spider中间件(输入方向)发送给Spider处理。  

7.Spider处理Response并返回爬取到的Item及(跟进的)新的Request给引擎。  

8.引擎将(Spider返回的)爬取到的Item给Item Pipeline，将(Spider返回的)Request给调度器。  

9.(从第二步)重复直到调度器中没有更多的request，引擎关闭该网站。  

另外需要兼容不同的电商网站后变得容易扩展。设计中不同的网站有不同的Spider，所以将爬到的数据转换成统一的Item数据格式即可满足兼容不同的电商网站。


### 角色 ###
在整个系统中扮演商品数据信息获取的角色，并将提取的商品信息传递给数据处理模块。


### 对外接口 ###
```java
JSONArray getData();
```
数据格式定义为：
```json
"url":网页连接
"source":来源
"name":商品名称
"price":价格
"picture_url":图片连接
"shop":店铺
```

## 二、类图 ##

![](/assets/CrawlerLib类图.png)



## 三、类描述 ##

###Engine类
负责控制数据流在系统中所有组件中流动，并在相应动作发生时触发事件。    

* +openDomain()：void----打开一个网站  

* +getFirstUrl():void----在Scheduler调度第一个URL  

* +getNextUrl(): void----请求下一个要爬取的URL  

* -redirectRespons():void----将Scheduler返回的URL通转发给Downloader

* -passItem2Pipeline():void----将Downloader中返回的Response交给Spider处理

###Spider类
Spider类定义了如何爬取某个(或某些)网站。包括了爬取的动作(例如:是否跟进链接)以及如何从网页的内容中提取结构化数据(爬取item)。每个spider负责处理一个特定(或一些)网站。  
 
* +startRequests():Request----当spider启动爬取并且未制定URL时，该方法被调用  
 
* +parse(response):List----parse 负责处理response并返回处理的数据以及(/或)跟进的URL  

* -log(message[, level, component]):void----记录(log)message  

* +closed(reason):boolean----当spider关闭时，该函数被调用。    

###Spider4Web1/2类
是适用于具体网站 Website1 / Website2的爬虫类。  

* +setRule():void----设置Rule对象集合。  

* -parse_start_url(response):Request----当start_url的请求返回时，该方法被调用。 该方法分析最初的返回值并必须返回一个 Item 对象    

###Rule类
 每个 Rule 对爬取网站的动作定义了特定表现。  
 * extract_links():List----它接收一个 Response 对象,从网页中抽取最终将会被链接的对象
 
 * callback该spider中同名的函数将会被调用。 从extract_links中每次获取到链接时将会调用该函数
 
 *  -allow:reg----必须要匹配这个正则表达式(或正则表达式列表)的URL才会被提取
  
 * -deny: reg----与这个正则表达式(或正则表达式列表)的(绝对)不匹配的URL必须被排除在外(即不提取)   
 
 *  -tags: String----提取链接时要考虑的标记或标记列表  
 
 *  -cb_kwargs:Dictionary----包含传递给回调函数的参数(keyword argument)的字典  
 
 *  -follow:boolean----指定了根据该规则从response提取的链接是否需要跟进  
 
 * -process_links:String----该spider中同名的函数将会被调用。 从extract_links()中获取到链接列表时将会调用该函数。该方法主要用来过滤。
 
# 四、重要协作 #
顺序图与协作描述
# 五、使用的设计模式 #





