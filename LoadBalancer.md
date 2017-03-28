# LoadBalance 模块



## 一、概述

### 承担的需求（主要秘密）

同一个服务在多台机器上部署,负载均衡用来解决选取哪一个服务器的问题.

### 可能会修改的实现（次要秘密）

* 负载均衡算法
* 服务列表的维护
* 服务的健康监测
* 不同的注册中心

### 涉及的相关质量属性

* R2 服务可靠性
* R10 高峰吞吐量

### 设计概述

一个服务可能调用其他多个服务,而每个服务有多台实际的服务器.所以我们采用LoadBalancerClient,下简称Client,作为各个服务的门面,发起者可以直接调用Client.每一种服务对应一个LoadBalancer对象,提供对服务列表的初始化/选择/增加/删除等操作.

对于不同的负载均衡算法,封装在LoadBalanceStrategy中.对于不同的服务列表维护方法,比如读配置中的静态服务列表,或注册中心查询到的列表,通过UpdateStrategy封装.

### 角色

### 模块对外接口

1. Object **execute**\(Request request\);

## 二、类的设计

### 2.1 类图

![](/assets/sbin/LoadBalanceClass.png)

### 2.2 类描述

#### LoadBalancerClient类

##### 类职责

本类是LoadBalancer模块的门面,提供execute方法来发起服务调用

##### 类方法

* public Object execute\(Request request\): 
  * 职责：发起一次请求
  * 前置条件：配置正确,服务初始化成功
  * 后置条件：完成调用返回结果

---

#### Config类

##### 类职责

负载均衡模块的配置,包括服务列表是否静态,是否启用ping检测服务健康,负载均衡算法,配置中心的信息等.

##### 类属性

* bool isStaticServer  //server列表是否是静态的

* bool ping   //是否使用ping命令检测server状态

* loadBalanceRuleClass    //负载均衡的策略类

* String registerServerType    //注册中心的类型

* String registerServerHost   //注册中心的地址

---

#### ConfigBuilder类

##### 类职责

因为Config类构造过程比较复杂,而且将来的配置项可能增多.因此使用专门的Builder来构造Config

##### 类方法

* ConfigBuilder staticServerList\(bool\)

* ConfigBuilder ping\(bool\)

* ConfigBuilder loadBalanceStrategy\(Obj\)

* ConfigBuilder registerServer\(String type,String host\)

* Config build\(\)

---

#### LoadBalancer类

##### 类职责

负责一个具体服务的负载均衡.包括挑选服务器,维护服务器列表等.

##### 类方法

* public Server selectServer\(Requesst\)
  * 职责：选择一个被调用的Server
  * 前置条件：配置正确,服务正常启动
  * 后置条件：无
* public Object execute\(Server\)
  * 职责：对指定的Server执行调用
  * 前置条件: Server的信息正确
  * 后置条件：无
* public void initServer\(\)
  * 职责：初始化Server的列表
  * 前置条件：关于列表的配置正确
  * 后置条件：server列表被初始化
* public void updateServer\(\)
  * 职责：更新Server的列表
  * 前置条件：列表已被初始化
  * 后置条件：列表被更新

---

#### LoadBalanceStrategy类

##### 类职责

负载均衡算法的策略接口,从Server列表中选出一个Server.

##### 类方法

* public Server choose\(ServerList,Requesst\)
  * 职责：从list中选一个server
  * 前置条件：List不为空
  * 后置条件：无

---

#### RandomStrategy类

##### 类职责

负载均衡算法的策略接口的一种实现,采用随机法选取服务器,

##### 类方法

* public Server choose\(ServerList,Requesst\)
  * 职责：用随机法从List中选取一个Server
  * 前置条件：配置正确,服务正常启动
  * 后置条件：无

---

#### UpdateStrategy类

##### 类职责

服务器列表的更新策略,负责服务器列表的增删.

##### 类方法

* public void update\(ServerList\)
  * 职责：对服务器列表进行增删
  * 前置条件：服务器列表需要发生变动
  * 后置条件：服务器列表变动

---

#### RegisterUpdateStrategy类

##### 类职责

针对注册中心实现的更新策略,接收注册中心的通知,更新服务列表

##### 类方法

* public void update\(ServerList\)

  * 职责：对服务器列表进行增删
  * 前置条件：服务器列表需要发生变动
  * 后置条件：服务器列表变动

* public void registerChange\(ChangeEvent\)

  * 职责：适配注册中心提供的接口,接收注册变动的通知
  * 前置条件：注册中心的服务列表发生变动
  * 后置条件：服务列表变动

---

#### Server类

##### 类职责

VO类,记录Server的各类属性

##### 类属性

* public String serverId; //服务ID
* public String serviceName; //服务名称
* public String host;  //服务地址
* public int port;    //服务端口

---

## 三、重要协作

### 顺序图

![](/assets/sbin/负载均衡服务调用顺序图.png)

## 四、设计模式应用

### 策略模式

![](/assets/sbin/负载均衡策略.png)


负载均衡模块有两处使用了策略模式.分别是负载均衡的策略和服务列表更新的策略.

**负载均衡策略**

负载均衡有多种算法,为了方便算法的替换,采用LoadBalanceStrategy封装具体算法.

**服务更新策略**

服务列表可以来源于注册中心或手动配置.使用注册中心这意味着不需要自己去检测其状态,只需要监听即可.手动配置时需要定期ping服务器已确定其状态.这里采用不同的UpdateStrategy封装.


### 适配器模式

![](/assets/sbin/负载均衡Adpter.png)

对于面向注册中心的服务更新,需要监听Register发来的变化.这里让RegisterUpdateStrategy适配RegisterObserver接口,以监听变化并完成更新.

### Builder

![](/assets/sbin/负载均衡Builder.png)

负载均衡的Config类是复杂的,并且其数据可能来源于配置文件,硬编码或者网络,因此使用Builder来处理其构造过程.
