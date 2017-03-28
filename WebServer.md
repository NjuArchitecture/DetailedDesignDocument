# WebServer 模块

## 一、概述

### 承担的需求（主要秘密）

WebServer主要负责接收用户请求并返回网页.它需要调用其他的WebService来完成工作.

### 可能会修改的实现（次要秘密）

* 页面的变化
* 请求路由的变化

### 涉及的相关质量属性

* R2 服务可靠性
* R10 高峰吞吐量

### 设计概述

WebServer是生成网页的服务器.它需要调用其他的API,整合数据并生成网页,是系统其他模块的连接部分.

WebServer的设计主要关注三点.
* 要让外界访问和具体的API服务器解耦,即有自己的请求路由,而不是直接使用API的路由.
* 要应对页面的变化,尽量减少页面改变时带来的修改.
* 自身要保持简单,尽量避免业务逻辑.这是因为WebServer是单点的,维护的代价较大.

### 模块对外接口

public Response request\(Request\)  


## 二、类的设计

### 2.1 类图

![](/assets/sbin/webServerClass.png)

### 2.2 类描述

#### Dispatcher类

##### 类职责

请求的分派器,负责将请求分派给各个RequestHandler.

##### 类方法

* public void dispatch\(Request request\): 
  * 职责：将请求分配给特定的RequestHandler
  * 前置条件: RequestHandler已经正确配置.
  * 后置条件: 处理请求

#### RequestHandler类

##### 类职责

实际处理请求的类,包含一系列filter和实际的controller.

##### 类方法

* public void handle\(Request request\): 
  * 职责：处理一个请求
  * 前置条件: 无
  * 后置条件：请求被处理

---

#### RequestFilter类

##### 类职责

请求的Filter,为请求处理添加切面逻辑

##### 类方法

* public void doFilter\(Request request,Response response\):
* 职责：完成请求处理的切面逻辑
* 前置条件: 无
* 后置条件：无

---


#### Controller类

##### 类职责

负责实际的请求处理,生成页面的逻辑.

##### 类方法

* public void handleRquest(Request request,Response response):
* 职责：处理请求并生成页面
* 前置条件: 请求已通过filter
* 后置条件：返回页面


---

#### SearchService类

##### 类职责

对搜索API的封装,负责调用搜索服务

##### 类方法

* public SearchResult search(String key):
* 职责：调用搜索API,包装返回结果
* 前置条件: key不为空
* 后置条件：请求正确发送,返回结果正确

---


#### CommentService类

##### 类职责

对评论API的封装,负责调用评论服务

##### 类方法

* public List getComment\(goodsId\)
  * 职责：获得某个商品的评论信息
  * 前置条件：List不为空
  * 后置条件：请求被正确发送,返回结果正确

---

#### LoadBalancer类

##### 类职责

来自LoadBalancer模块,负责对请求进行负载均衡.

##### 类方法

* public Response execute\(Request\)
  * 职责：对请求进行负载均衡,选取一个服务器发起调用
  * 前置条件：Request不为空,格式正确
  * 后置条件：无

---

#### ViewResolver类

##### 类职责

选择视图,根据Model中的信息解析视图,生成页面

##### 类方法

* public Page render\(PageName,Model\)
  * 职责：用视图名查找视图,解析并返回页面
  * 前置条件：PageName存在
  * 后置条件：无

---

#### Pages类

##### 类职责

来自CommentPages和SearchResultPages的模块,能根据Model的数据生成页面

##### 类方法

* public Page render\(Model\)
  * 职责：根据Model中的数据生成页面
  * 前置条件：Model数据完整
  * 后置条件：无

---



## 三、重要协作

### 顺序图

![](/assets/sbin/WebServer顺序图.png)
---

## 四、设计模式应用

### 过滤器模式

为了方便的向controller添加切面逻辑,WebServer中采用了过滤器模式.可以通过Filter来向controller处理的前后添加功能.


