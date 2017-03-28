# Register 模块

## 一、概述

### 承担的需求（主要秘密）

服务器在运行中会动态地增加,或出现故障而宕机,因此难以手工维护.注册中心解决的是服务器在运行时互相发现的问题.

### 可能会修改的实现（次要秘密）

* 注册中心的监听者
* 客户端可能是不同的平台.

### 涉及的相关质量属性

* R2 服务可靠性

### 设计概述

同一个服务可能有多个服务提供者,同时可能有新的服务器注册进来,或者服务器宕机而被剔除,服务器也可能会主动解除注册.  

同时,服务调用者需要向注册中心查询服务列表,以便调用服务.  

因此,Register分为Server和Client两部分.Server是注册中心服务器,负责服务器列表的维护,Client负责向服务器注册/注销或汇报健康状态,同时也能像服务器查询服务.

另外简述一下系统所使用的心跳机制.client按固定频率发送心跳,假设5s一次.服务器端对服务器表定期进行扫描,假设扫描周期为11s一次.那么每个client应该在一个周期内发送两次心跳.若是扫描到一周期内发送不到2次,则将client冻结起来,若是下一个周期没有心跳,则删除客户端,并通知所有consumer.

之所以冻结而不直接删除,是因为通知consumer的开销较大,频繁的增删可能影响性能.

### 模块对外接口

**ReigisterServer**  
public void register\(ServerInfo\)  
public void cancel\(ServerInfo\)  
public List&lt;ServerInfo&gt; getService\(serviceName\)
public void heartBeat(ServerInfo)

**RegisterClient**

public void register\(\)
public void cancel\(\)
public List&lt;ServerInfo&gt; getService\(serviceName\)
public void registerListener\(RegisterOberver\)

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

![](/assets/健康检测顺序图.png)
---

## 四、设计模式应用

### 策略模式

负载均衡模块有两处使用了策略模式.分别是负载均衡的策略和服务列表更新的策略.

**负载均衡策略**

负载均衡有多种算法,为了方便算法的替换,采用LoadBalanceStrategy封装具体算法.

**服务更新策略**

服务列表可以来源于注册中心或手动配置.使用注册中心这意味着不需要自己去检测其状态,只需要监听即可.手动配置时需要定期ping服务器已确定其状态.这里采用不同的UpdateStrategy封装.

### 适配器模式

对于面向注册中心的服务更新,需要监听Register发来的变化.这里让RegisterUpdateStrategy适配RegisterObserver接口,以监听变化并完成更新.

### Builder

负载均衡的Config类是复杂的,并且其数据可能来源于配置文件,硬编码或者网络,因此使用Builder来处理其构造过程

