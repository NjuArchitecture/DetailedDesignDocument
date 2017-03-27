#OnlineGoodsService模块
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


























