#OnlineGoodsService模块
## 一、概述 ##

### 承担的需求（主要秘密） ###
  该模块主要承担的任务是从Item Pipleline中获取数据，并规范化数据，将规范化后的商品数据增量更新到数据库中。
  
### 可能会修改的实现(次要秘密) ###
* 可能会新增不同的数据更新方式和策略。
* 可能会修改数据库表结构的设计。

### 设计概述 ###



### 角色 ###
在整个系统中扮演商品数据信息获取的角色，并将提取的商品信息传递给数据处理模块。


### 对外接口 ###
```java
JSONArray getData();
```
## 二、类图 ##

![](/assets/dtj/OnlineGoodsService类图1.png)



## 三、类描述 ##

###UpdatorController类
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
 
 

# 四、重要协作 #
##顺序图
![](/assets/dtj/OnlineGoodsService顺序图.png)
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


























