# LoadBalance 模块

## 词汇表

| 英文 | 中文 | 备注 |
| :--- | :--- | :--- |
| Tokenlization | 序列化 |  |
|  | Normalization | 标准化 |
| Front Service | 前端服务 | 前端是相对而言的,这里是指调用搜索服务的一方 |
| Data Provider | 数据提供者 | 如爬虫 |
| Query | 查询语言 | 内部定义的查询语言类 |

## 一、概述

### 承担的需求（主要秘密）

同一个服务在多台机器上部署,负载均衡用来解决选取哪一个服务器的问题.需要尽量挑选空闲服务器,避开忙碌或宕机的服务器.

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

* public String serverId; //server的ID
* public  

---

## 三、重要协作

### 顺序图

#### 存储模块顺序图

---

### 状态图

#### 存储模块状态图

---

#### 搜索模块状态图

## 四、设计模式应用

### 策略模式

> 策略模式作为一种软件设计模式，指对象有某个行为，但是在不同的场景中，该行为有不同的实现算法。-- wikipedia

在Search Engine模块中，由于外界因素的不确定性，搜索的具体实现细节可能会不断地变化，为了符合开闭原则，应当避免在修改搜索策略时破坏已有代码的封装，为此，SearchEngine模块中大量使用了策略模式，将易于变化的部分独立出来，声明一个策略接口，含有算法的接口，并将实现存放于具体的策略实现子类，实现了将算法的具体实现推迟到了运行时的绑定，使得修改策略仅需要实现一个新的策略实现子类即可，并调用setStrategy\(\)方法修改策略即可，从而保证了高内聚和低耦合

