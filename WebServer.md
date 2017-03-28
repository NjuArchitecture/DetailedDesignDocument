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
* 一是要让外界访问和具体的API服务器解耦,即有自己的请求路由,而不是直接使用API的路由.
* 二是要应对页面的变化,尽量减少页面改变时带来的修改.
* 三是自身要保持简单,尽量避免业务逻辑.这是因为WebServer是单点的,维护的代价较大.

### 模块对外接口

public Response request\(Request\)  


## 二、类的设计

### 2.1 类图

![](/assets/sbin/RegisterServerClass.png)

### 2.2 类描述

#### Register类

##### 类职责

本类是Register server部分的门面,暴露了Register server相关的方法.

##### 类方法

* public void register\(ServerInfo serverInfo\): 
  * 职责：注册一个服务器
  * 前置条件: serverInfo信息正确
  * 后置条件：注册该服务

---

* public void cancel\(ServerInfo serverInfo\): 
  * 职责：注销一个服务器
  * 前置条件: serverInfo信息正确
  * 后置条件：注销该服务

---

* public List&lt;ServerInfo&gt; getService\(String sericeName,ConsumerInfo consumerInfo): 
  * 职责：根据名称来查询服务
  * 前置条件: serviceName不为空,consumerInfo正确
  * 后置条件：无

---


* public void heartbeat\(ServerInfo serverInfo\): 
  * 职责：接收某一服务器的心跳
  * 前置条件: serverInfo信息正确
  * 后置条件：接受该服务器的心跳.



#### HealthMap类

##### 类职责

服务器健康状态的Map,记录心跳数和被冻结的服务器.

##### 类方法

* public void put\(int serviceId,ServerHealth serverHealth\): 
  * 职责：新增一个记录项
  * 前置条件: serviceId还不存在,ServerHealth不为空
  * 后置条件：新增该记录

---

* public void heartbeat\(int serviceId,ServerHealth serverHealth\): 
  * 职责：新增一次心跳计数
  * 前置条件: serviceId已经存在
  * 后置条件：增加该服务器的心跳计数

---

* public void froze\(int serviceId\): 
  * 职责：冻结一个服务器
  * 前置条件: serviceId已经存在
  * 后置条件：冻结该服务器

---

* public void remove\(int serviceId\): 
  * 职责：移除一个服务器记录
  * 前置条件: serviceId存在,并且已被冻结
  * 后置条件：移除该服务

---

#### ServiceMap类

##### 类职责

服务名与服务器的对应表,一个服务名对应多个服务器

##### 类方法

* public void put\(String serviceName,ServerInfo serverInfo\):
* 职责：对该服务名增加一个服务器
* 前置条件: serverInfo配置正确
* 后置条件：增加该服务器

---

* public void remove\(String serviceName,int serviceId\):
* 职责：移除该服务名下的一个服务器
* 前置条件: 服务名和服务器都存在,并且是对应关系
* 后置条件：移除服务器,并通知对应的调用者

---

* public void find\(String serviceName,ConsumerInfo consumerInfo\):
* 职责：根据服务名查找服务
* 前置条件: serviceName存在,ConsumerInfo正确
* 后置条件：无

---


#### HealthChecker类

##### 类职责

对HealthMap进行定期检查,清零心跳数,冻结服务或者移除服务

##### 类方法

* public Server check\(\)
  * 职责：检测各服务器的健康状态
  * 前置条件：无
  * 后置条件：对于心跳数正常的服务器,清空心跳.不足的服务器冻结,冻结的服务器移除.

---

#### Notifier类

##### 类职责

服务器的新增和移除需要通知他人,如服务的消费者.Notify是通知的策略接口,抽象不同的通知实现.

通知可能是向客户端通信,可能是通过配置的url接口等,因此Notify可能有多种实现.

##### 类方法

* public Server notify\(List comsumer\)
  * 职责：向列表中的comsumer进行通知
  * 前置条件：List不为空
  * 后置条件：发送通知

---


#### ClientNotifer类

##### 类职责

通知策略的实现类.完成向客户端通信的逻辑

##### 类方法

* public Server notify\(List comsumer\)
  * 职责：向列表中的comsumer进行通知
  * 前置条件：List不为空
  * 后置条件：发送通知

---


## 三、重要协作

### 顺序图

**注册及心跳顺序图**
![](/assets/sbin/Register顺序图.png)

**健康监测顺序图**

![](/assets/sbin/健康检测顺序图.png)
---

## 四、设计模式应用

### 策略模式

为了方便拓展不同的通知方式,使用了策略模式对notifier进行抽象

![](/assets/sbin/RegisterNotifier.png)

