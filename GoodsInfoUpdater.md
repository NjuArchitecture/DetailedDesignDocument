#GoodsInfoUpdater模块
该模块在初始化商品数据和更新商品数据时都需要使用。可以将该模块分为CrawlerLib和OnlineGoodsService两个部分，分别负责爬取商品数据和处理存储数据。
#CrawlerLib
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

![](/assets/dtj/CrawlerLib类图.png)



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
 
 * -process_links:String----该spider中同名的函数将会被调用。 从extract_links()中获取到链接列表时将会调用该函数。该方法主要用来过滤  
 
 * -process_request:String----该spider中同名的函数将会被调用。 该规则提取到每个request时都会调用该函数。该函数必须返回一个request或者None。 (用来过滤request)  
 
###Scheduler类  
调度器从引擎接受request并将它们入队，以便之后引擎请求他们时提供给引擎。
* +enqueue(request): void----接受request并将它们入队

* +dequeue(): void----将request出队返回给Engine  

###ItemPipeline类  
Item Pipeline负责处理被spider提取出来的item。当Item在Spider中被收集之后，它将会被传递到Item Pipeline，一些组件会按照一定的顺序执行对Item的处理。
* +process_item(item, spider):Item----清理HTML数据,验证爬取的数据(检查item包含某些字段),查重(并丢弃)

* +open_spider(spider): boolean----当spider被开启时，这个方法被调用
* +close_spider(spider):boolean----当spider被关闭时，这个方法被调用  

###DownloaderMiddleware类  
 下载器中间件是在引擎及下载器之间的specific hook，处理Downloader传递给引擎的response。  
 
* +process_request(request, spider):Response----当每个request通过下载中间件时，该方法被调用。如果其返回 None ，Scrapy将继续处理该request

* +process_response(request, response, spider):Response----返回的response会被在链中的其他中间件的 process_response() 方法处理。  

###Downloader类
下载器负责获取页面数据并提供给引擎，而后提供给spider。调用python的urllib。  
* +urlopen(url[,data[,proxies]]): file----打开一个url的方法，返回一个文件对象  

* +urlcleanup():void----清除产生的缓存

# 四、重要协作 #
##顺序图
![](/assets/dtj/crawlerLib顺序图.png)
##协作描述

初始化时，引擎打开一个网站，从spider获取到URL后由Scheduler进行调度，接着下载器下载下一个页面，返回一个Response，由Spider来处理Response并将Item返回给引擎，引擎再交给ItemPipeline处理。重复以上步骤直到调度器没有更多的Request。
# 五、使用的设计模式 
##中介者模式
![](/assets/dtj/中介者模式.png)

###使用场景
数据流需要在系统的所有组件中流动，会产生混乱的相互依赖关系，所以使用Engine来控制交互。
###要达到的效果
可以使各对象不需要显式地互相引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。它使数据流的控制都集中在了Engine里。  

##策略模式
![](/assets/dtj/策略模式.png)

###使用场景
对于不同的网站，爬取网页的规则不同，而这些规则又可能会改变，之后还会增加新的网站，因此可以定义一些类来封装不同的抓取网页的规则和提取数据的规则，将这些变化封装起来，使得这些规则可以独立于使用它的客户而变化。
###要达到的效果
消除了一些条件语句，使Spider不必再进行网站的条件语句判断。当需要增加新的网站时，只需要新增一个Spider的子类，封装好该网站的爬取规则和数据提取规则，而不必再修改Engine类和其他类。  


#OnlineGoodsService
## 一、概述 ##

### 承担的需求（主要秘密） ###
  该模块主要承担的任务是从Item Pipleline中获取数据，并规范化数据，将规范化后的商品数据增量更新到数据库中。
  
### 可能会修改的实现(次要秘密) ###
* 可能会新增不同的数据更新方式和策略。
* 可能会修改数据库表结构的设计。

### 设计概述 ###
因为进行的是异步请求，而且数据库的操作开销会比较大，所以可以先将从Item Pipeline中传来的数据放入buffer中，等缓冲区满了再写入数据库。


### 角色 ###
处理商品信息，负责将数据更新到数据库中。


### 对外接口 ###
```java
void saveData();
```
## 二、类图 ##

![](/assets/dtj/OnlineGoodsService类图1.png)



## 三、类描述 ##

###UpdatorController类
爬虫数据更新模块的集中控制类，其中包括分析处理过后的规范化数据和一个更新器。
* startUpdate():void----开始数据的更新 


###Updator类
定义RealUpdator和BufferProxy的公用接口，这样就在任何使用RealUpdator的地方都可以使用Proxy。
 
* +saveData():List----数据更新方法  

###RealUpdator类
是爬虫数据更新模块的更新器，负责对数据进行更新。当缓存满时被调用。

* +saveData(List buffer):void----数据更新方法，将数据存储到数据库  

###BufferProxy类
 保存一个RealUpdator的引用，接受Item Pipeline传来的数据.
 * +saveData():void----若缓冲区未满则将数据存入缓冲区，若缓冲区已满则将缓冲区的数据写入数据库。  
 
# 四、重要协作 #
##顺序图
![](/assets/dtj/OnlineGoodsService顺序图.png)
##协作描述
在数据更新时，更新控制类将待更新数据发送给 BufferProxy 类，如果此时缓冲区未满则将数据存入缓冲区，若缓冲区已满则调用RealUpdator的saveData()方法将缓冲区的数据写入数据库。 

# 五、使用的设计模式 
##代理模式
![](/assets/dtj/OnlineGoodsService类图.png)

###使用场景
因为连接数据库的开销比较大，所以将对数据库的访问进行限制，在需要的时候实例化真正的实体类再进行访问。
###要达到的效果
引入了一定的间接性，可以进行最优化，根据需求写入数据库。




















































