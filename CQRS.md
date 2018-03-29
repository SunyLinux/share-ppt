title: DDD: CQRS
speaker: 乌鸦
url: 
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

<!--  -->

[slide]
![](/img/ddd.jpg)


[slide]
<iframe style="position: absolute;
    height: 100%;
    left: -60px;
    top: -40px;
    right: 0px;
    width: 120%;" data-src="http://cqrs.nu/Faq" src="about:blank;"></iframe>

<!--  -->


[slide]

![](/img/domain_model.jpg)

<!--  -->




[slide]

# DDD: CQRS
# Command Query Responsibility Segregation

<!--  -->

[slide]

## 概要
----
* 基本概念介绍

* CQRS\ES 架构介绍
* 示例代码
* Event Sourcing 优缺点
* Event Store 设计

* Flowable 与 Command


<!-- 
DDD领域驱动设计、CQRS架构、事件溯源（Event Sourcing，简称ES）、事件驱动架构（EDA)
 
基本概念、设计初衷、一致性模型、实现方式、适用场景、架构的基本数据流 
 -->

[slide]
## 基本概念 - 聚合

![](/img/aggre.jpg)

<!-- 
首先，我们需要先理解DDD中的聚合、聚合根这两个概念。

聚合，它通过定义对象之间清晰的所属关系和边界来实现领域模型的内聚，并避免了错综复杂的难以维护的对象关系网的形成。聚合定义了一组具有内聚关系的相关对象的集合，我们把聚合看作是一个修改数据的最小原子单元。
聚合根，每个聚合都有一个根对象，根对象管理聚合内的其他子对象（实体、值对象）；聚合之间的交互都是通过聚合根来交互，不能绕过聚合根去直接和聚合下的子实体进行交互。

上面的例子中，Car、Wheel、Position、Tire四个对象构成一个聚合，其中Car是聚合根；Customer也是聚合根，Customer不能直接访问Car下的Tire（子实体），而是只能通过聚合根Car来访问。
 -->


[slide]
## 基本概念 - 聚合
## 聚合根 实体 值对象 关系
----
1. 实体有 id，有生命周期，有状态（值对象描述），通过 id 判等
2. 聚合根是实体，id 全局唯一
3. 值对象核心思想是值，无生命周期，与是否复杂类型无关，通过值判等


<!--  -->


[slide]
## 基本概念 - 聚合
```java
// 聚合根
class Order extends Entity implements AggregateRoot {
    Long id                 // 值对象
    String orderNo          // 值对象
    Address customAddr      // 值对象
    List<OrderItem> items   // 实体集合
}

// 实体
class OrderItem extends Entity {
    Long productId          // 值对象
    String productName      // 值对象
    Double price            // 值对象
    int count               // 值对象
}

// 值对象
class Address implements ValueObject {
    String province         // 值对象
    String city             // 值对象
}
```
<!--  -->



[slide]
## 聚合设计原则
-----
* 聚合作为一种边界，主要用于维护业务完整性，此时应遵循业务规则中定义的不变量（Invariant）
* 作为聚合边界内的非聚合根实体对象，若可能被别的调用者单独调用，则应该作为单独的聚合分离出来
* 在聚合边界内的非聚合根对象，与聚合根之间应该存在直接或间接的引用关系，且可以通过对象的引用方式；若必须采用Id来引用，则说明被引用的对象不属于该聚合
* 若一个对象缺少另一个对象作为其主对象就不可能存在，则该对象一定属于该主对象的聚合边界内
* 若一个实体对象，可能被多个聚合引用，则该实体对象应首先考虑作为单独的聚合

<!-- http://zhangyi.xyz/mongodb-schema-design-using-aggregate/  -->


[slide]
## 聚合设计 - 案例
----
> 为了能够保存分析报表以及用户设置的报表查询条件:


> * 报表分类（ReportCategory）
> * 报表（Report）
> * 报表查询条件（QeuryCondition）

> 一个报表分类会包含多个报表，同一个报表只能属于一个分类。
<br>
> 每个报表提供了多个标准查询条件和多个用户自定义查询条件。

<!-- 
从业务完整性看，Report虽属于ReportCategory，但二者未尝有强的约束关系，即不存在业务上的不变量（Invariant）。例如ReportCategory可以没有Report，成为一个空的分类，我们也可以撇开ReportCategory，单独查询所有的Report。倘若我们将Report放到ReportCategory聚合中，由于Report可能会被单独调用，聚合的边界保护反而成为了障碍，不合理。

于是，我们可以得出第一个结论：ReportCategory和Report应该属于两个不同的聚合。

基于第四条原则，我们可以提出问题：当QueryCondition缺少Report对象后，还有存在意义吗？答案一目了然，没有Report，就没有QueryCondition。皮之不存毛将焉附！第二个结论自然得来：Report与QueryCondition应属于同一个聚合。于是，模型呼之欲出：

 -->

[slide]
![](\img\aggre_eg.jpg)


<!--  -->



[slide]
## 聚合设计原则
-----
* 聚合是用来封装真正的不变性，而不是简单的将对象组合在一起
* 聚合应尽量设计的小
* 聚合之间的关联通过id，而不是对象引用
* 聚合内强一致性，聚合之间最终一致性

<!-- http://www.cnblogs.com/netfocus/p/3307971.html -->



[slide]
## 基本概念 - Eventual Consistency

![](/img/eventually_consistent.jpg)

<!-- 
上面表达了一个关于聚合的一致性设计原则：聚合内的数据修改，是ACID强一致性的；跨聚合的数据修改，是最终一致性的。
遵守这个原则，可以让我们最大化的降低并发冲突，从而最大化的提高整个系统的吞吐。

跨聚合事务：saga；
 -->


[slide]
## 基本概念 - In Memory

![](/img/in_mem.jpg)

<!-- 
In-Memory的意思是指整个系统中的所有的聚合根对象都活在内存, 而不是像我们平时那样，用到的时候才从DB获取对象，然后再做修改，再保存回去。

在In-Memory的架构下，当要修改某个聚合根的状态时，它已经在内存，我们可以直接拿到该对象的引用，且框架会尽量保证聚合根对象的状态就是最新的。聚合根是在内存中的最小计算单元，每个聚合内部都封装了业务规则，并保证数据的强一致性。

上图我是挪用了之前比较或的LMAX架构中的一个图，表达的思想就是in-memory架构。其中Business Logic Processor就是中央业务逻辑处理器，内部承载了大量在机器内存中活着的聚合根对象。
 -->



[slide]
## 基本概念 - Event Sourcing

* 不保存对象的最新状态, 而是保存对象产生的所有事件
* 通过事件溯源(Event Sourcing, ES)得到对象最新的状态

![](/img/evt_src.jpg)

<!-- 
接下来，我们再来看一下什么是事件溯源。

一个对象从创建开始到消亡会经历很多事件，以前我们是在每次对象参与完一个业务动作后把对象的最新状态持久化保存到数据库中，也就是说我们的数据库中的数据是反映了对象的当前最新的状态。而事件溯源则相反，不是保存对象的最新状态，而是保存这个对象所经历的每个事件，所有的由对象产生的事件会按照时间先后顺序有序的存放在数据库中。可以看出，事件溯源的这种做法是更符合事实观的，因为它完整的描述了对象的整个生命周期过程中所经历的所有事件。

那么，事件到底如何影响一个领域对象的状态的呢？很简单，当我们在触发某个领域对象的某个行为时，该领域对象会先产生一个事件，然后该对象自己响应该事件并更新其自己的状态，同时我们还会持久化在该对象上所发生的每一个事件；这样当我们要重新得到该对象的最新状态时，只要先创建一个空的对象，然后将和该对象相关的所有事件按照事件发生先后顺序从先到后再全部应用一遍即可还原得到该对象的最新状态，这个过程就是所谓的事件溯源。

另一方面，因为是用事件来表示对象的状态，而事件是只会增加不会修改。这就能让数据库里的表示对象的数据非常稳定，不可能存在DELETE或UPDATE等操作。因为一个事件就是表示一个事实，事实是不能被磨灭或修改的。这种特性可以让领域模型非常稳定，在数据库级别不会产生并发更新同一条数据的问题。

e.g. Binlog
 -->



[slide]
## 基本概念 - Event Sourcing VS CRUD
* CRUD: DB 的记录可变，可以增删改
* ES: 没有更新、删除，只有 Append Event，不可变
![](/img/es_vs_crud.jpg)

<!-- 
通过上面这个图，大家应该可以更直观的理解事件溯源和传统CRUD思想的区别。
 -->





[slide]
## 基本概念 - Actor Model
![](/img/actor.jpg)

<!-- 
Actor模型，这个概念大家应该都了解。Actor模型的核心思想是，对象直接不会直接调用来通信，而是通过发消息来通信。每个Actor都有一个Mailbox，它收到的所有的消息都会先放入Mailbox中，然后Actor内部单线程处理Mailbox中的消息。从而保证对同一个Actor的任何消息的处理，都是线性的，无并发冲突。从全局上来看，就是整个系统中，有很多的Actor，每个Actor都在处理自己Mailbox中的消息，Actor之间通过发消息来通信。

Akka框架就是实现Actor模型的并行开发框架，并且Akka框架融入了聚合、In-Memory、Event Sourcing这些概念。Actor非常适合作为DDD聚合根。Actor的状态修改是由事件驱动的，事件被持久化起来，然后通过Event Sourcing的技术，还原特定Actor的最新状态到内存。
 -->
 


[slide]
## 基本概念 - Event-driven Architecture (EDA)
![](/img/eda.jpg)

<!-- 
上图表达的是事件驱动的架构的思想。Node表示节点，每个节点负责处理逻辑；Event表示消息，节点之间通过消息进行通信。消息通过分布式消息队列如RocketMQ，Equeue进行通信。

事件驱动架构的核心思想是：

不同于SOA架构，EDA架构是pub-sub模式；Node1处理完逻辑后产生消息，Node2订阅消息并进行处理，Node1不知道Node2的存在；
最终一致性原则，Node1，Node2之间的数据一致性通过MQ最终保证一致；
如何保证最终一致性（消息链不会断开）：
1）MQ保证消息不丢；
2）任何一个Node要保证自己完全处理完后才发送ACK给MQ；
3）每个Node做到对任何消息处理的幂等性；

整个架构具有所有分布式MQ所带来的优点：如异步解耦、削峰、降低整个系统的整体部署成本；
 -->
 



[slide]
## 基本概念 - 分布式消息队列
![](/img/mq.jpg)

<!-- 
上图是一个面向Topic的分布式MQ的逻辑架构图，采用这种架构的MQ有：Kafka，RocketMQ，EQueue

Producer发送消息到某个Topic的某个Queue；
消息都存储在Broker上；
Consumer从Broker拉取消息进行消费，并支持消费者负载均衡；



分布式消息队列：

分布式队列编程优化篇
https://tech.meituan.com/distributed_queue_based_programming-optimization.html

分布式队列编程：模型、实战
https://tech.meituan.com/distributed_queue_based_programming.html

高性能队列——Disruptor
https://tech.meituan.com/disruptor.html

好了，上面是基本概念的介绍。接下来我们来看一下CQRS/ES架构。
 -->
 

<!--  -->

[slide]
## CQRS Intro
http://codebetter.com/gregyoung/2010/02/16/cqrs-task-based-uis-event-sourcing-agh/

> Many people have been getting confused over what CQRS is. They look at CQRS as being an architecture; it is not. CQRS is a very simple pattern that enables many opportunities for architecture that may otherwise not exist. CQRS is not eventual consistency, it is not eventing, it is not messaging, it is not having separated models for reading and writing, nor is it using event sourcing. 

balabala...

> Going through all of these we can see that CQRS itself is actually a fairly trivial pattern. What is interesting around CQRS is not CQRS itself but the architectural properties in the integration of the two services. In other words the interesting stuff is not really the CQRS pattern itself but in the architectural decisions that can be made around it. Don’t get me wrong there are a lot of interesting decisions that can be made around a system that has had CQRS applied … just don’t confuse all of those architectural decisions with CQRS itself.

<!--  -->

[slide]
## CQRS Intro
https://martinfowler.com/bliki/CQRS.html

> CQRS stands for Command Query Responsibility Segregation. It's a pattern that I first heard described by Greg Young. At its heart is the notion that you can use a different model to update information than the model you use to read information. For some situations, this separation can be valuable, but beware that for most systems CQRS adds risky complexity.




[slide]
## CQRS\ES - 架构简介

![](/img/cqrs_arch.jpg)

<!-- 
上图是CQRS架构的典型架构图。
 -->


[slide]

![](/img/ali_b2b_cqrs.jpg)

<!--

直观感受下现实中的 CQRS 架构

阿里B2B CRM 业务系统架构
吴子棋分享的文章<<领域建模实战案例解析>>，发现跟之前看到的一篇风格类似，看作者：张建飞，另一作者 Frank，谷歌之，看到作者的 Linkedin，Frank zhang 阿里巴巴，应该就是 14年7月至今，在做 CRM，加了微信，聊了两句，确认了信息；

使用 CQRS 但是没有使用 EventSource

-->


[slide]

https://github.com/Chinchilla-Software-Com/CQRS

![](\img\chinchilla_cqrs.jpg)


<!-- 在CQRS中，查询方面，直接通过方法查询数据库，然后通过DTO将数据返回。在操作(Command)方面，是通过发送Command实现，由CommandBus处理特定的Command，然后由Command将特定的Event发布到EventBus上，然后EventBus使用特定的Handler来处理事件，执行一些诸如，修改，删除，更新等操作。这里，所有与Command相关的操作都通过Event实现。这样我们可以通过记录Event来记录系统的运行历史记录，并且能够方便的回滚到某一历史状态。Event Sourcing就是用来进行存储和管理事件的。 -->


[slide]
## CQRS\ES - 架构简介
----
* 什么是 CQRS架构
* 采用 CQRS 架构的一个前提
* CQRS 架构的实现方式
* CQRS 架构的适用场景



[slide]
## CQRS\ES - 什么是 CQRS架构
----
* Command Query Responsibility Segregation，即命令查询职责分离
* CQRS本身只是一个读写分离的架构思想，表示在架构层面，将一个系统分为写入（命令）和查询两部分
* 一个命令表示一种意图，表示命令系统做什么修改，命令的执行结果通常不需要返回；一个查询表示向系统查询数据并返回



[slide]
## CQRS\ES - Event
----
命令操作领域中的聚合根，然后聚合根的状态发生变化后产生的事件


<!-- 
Command 一般用动词，Event 一般命名为动词的过去式，代表既定已经发生的事情，比如 createCustomer 与 CustomerCreated； 
-->



[slide]
## CQRS\ES - 采用 CQRS 架构的一个前提
----
由于CQRS架构的一致性模型为最终一致性，所以，你的系统要接受查询到的数据可能不是最新的，而是有几个毫秒的延迟。

<!-- 
之所以会有这个前提，是因为CQRS架构考虑到，作为一个多用户同时访问的互联网应用，当在高并发修改数据的情况下，比如秒杀、12306购票等场景，用户UI上看到的数据总是旧的。比如你秒杀时提交订单前看到库存还大于0，但是当你提交订单时，系统提示你宝贝卖完了。这个就说明，在这种高并发修改同一资源的情况下，任何人看到的数据总是Stale的，即旧的。
 -->




[slide]
## CQRS\ES - 实现方式
----
> CQRS作为一种架构思想，可以有多种实现方式
1. 最常见的CQRS架构是数据库的读写分离；
2. 系统底层存储不分离，但是上层逻辑代码分离；
3. 系统底层存储分离，C端采用Event Sourcing的技术，在EventStore中存储事件；Q端存储对象的最新状态，用于提供查询支持；


<!-- 
阿里 CRM 的架构属于第二种
 -->



[slide]
## CQRS\ES - 标准CQRS架构的适用场景
----
1. 当我们的应用的写模型和读模型差别比较大时；
2. 当我们希望对系统的查询性能和写入性能分开进行优化时，尤其是读/写比非常高的系统，CQ分离是必须的；
3. 当我们希望我们的系统同时满足高并发的写、高并发的读的时候；因为CQRS架构可以做到C端最大化的写，Q端非常方便的提供可扩展的读模型；


<!--  -->


[slide]
## CQRS\ES - CQRS架构的数据流
----
**C端的命令的执行流程**

### 客户端如（MVC Controller）发送命令通知系统做修改：

1. 发送命令到分布式MQ；
2. 然后命令的订阅者处理命令；
3. 订阅者内部根据不同的命令调用不同的Command Handler进行处理；
4. Command Handler内部根据命令所指定的聚合根ID从In-Memory内存中直接获取聚合根对象的引用，然后操作聚合根对象；
5. 聚合根对象状态发生变化并产生事件；
6. 框架负责自动持久化事件到Event Storage（简称EventStore）；
7. 框架负责将事件发布到Event MQ；
8. Event订阅者订阅事件，然后调用对应的Event Handler进行处理，如更新Data Storage（保存了聚合根的最新状态，通常叫读库，ReadDB）；


[slide]
## CQRS\ES - CQRS架构的数据流
----
**Q端的查询的执行流程**

### 客户端如（MVC Controller）发出查询请求系统返回数据：

1. 调用轻薄的Query Service，传如Query DTO；
2. Query Service从读库进行查询并返回结果；
4. 读库可以有很多种，依据我们的业务场景来选择：比如关系型DB、分布式缓存等NoSQL、搜索引擎，etc.




<!--  
什么是CQRS架构？


CQRS本身只是一个读写分离的架构思想，全称是：Command Query Responsibility Segregation，即命令查询职责分离，表示在架构层面，将一个系统分为写入（命令）和查询两部分。一个命令表示一种意图，表示命令系统做什么修改，命令的执行结果通常不需要返回；一个查询表示向系统查询数据并返回。

CQRS架构中，另外一个重要的概念就是事件，事件表示命令操作领域中的聚合根，然后聚合根的状态发生变化后产生的事件。

采用CQRS架构的一个前提
由于CQRS架构的一致性模型为最终一致性，所以，你的系统要接受查询到的数据可能不是最新的，而是有几个毫秒的延迟。之所以会有这个前提，是因为CQRS架构考虑到，作为一个多用户同时访问的互联网应用，当在高并发修改数据的情况下，比如秒杀、12306购票等场景，用户UI上看到的数据总是旧的。比如你秒杀时提交订单前看到库存还大于0，但是当你提交订单时，系统提示你宝贝卖完了。这个就说明，在这种高并发修改同一资源的情况下，任何人看到的数据总是Stale的，即旧的。

CQRS作为一种架构思想，可以有多种实现方式
1. 最常见的CQRS架构是数据库的读写分离；
2. 系统底层存储不分离，但是上层逻辑代码分离；
3. 系统底层存储分离，C端采用Event Sourcing的技术，在EventStore中存储事件；Q端存储对象的最新状态，用于提供查询支持；


CQRS架构的适用场景
1. 当我们的应用的写模型和读模型差别比较大时；
2. 当我们希望实践DDD时；因为CQRS架构可以让我们实现领域模型不受任何ORM框架带来的对象和数据库的阻抗失衡的影响；
3. 当我们希望对系统的查询性能和写入性能分开进行优化时，尤其是读/写比非常高的系统，CQ分离是必须的；
4. 当我们希望我们的系统同时满足高并发的写、高并发的读的时候；因为CQRS架构可以做到C端最大化的写，Q端非常方便的提供可扩展的读模型；
这里我主要分享的CQRS架构是上面第3种实现方式，也就是上图所画的架构。在我心目中，只有第三种才是真正意义上的CQRS架构。

下面简单描述一下上面的CQRS架构的数据流
C端的命令的执行流程

客户端如（MVC Controller）发送命令通知系统做修改：

发送命令到分布式MQ；
然后命令的订阅者处理命令；
订阅者内部根据不同的命令调用不同的Command Handler进行处理；
Command Handler内部根据命令所指定的聚合根ID从In-Memory内存中直接获取聚合根对象的引用，然后操作聚合根对象；
聚合根对象状态发生变化并产生事件；
框架负责自动持久化事件到Event Storage（简称EventStore）；
框架负责将事件发布到Event MQ；
Event订阅者订阅事件，然后调用对应的Event Handler进行处理，如更新Data Storage（保存了聚合根的最新状态，通常叫读库，ReadDB）；
Q端的查询的执行流程

客户端如（MVC Controller）发出查询请求系统返回数据：

调用轻薄的Query Service，传如Query DTO；
Query Service从读库进行查询并返回结果；
读库可以有很多种，依据我们的业务场景来选择：比如关系型DB、分布式缓存等NoSQL、搜索引擎，etc.
-->
 



[slide]
## CQRS\ES - CQRS架构的数据流
![](\img\cqrs_data_stream.jpg)

<!-- 

https://github.com/Chinchilla-Software-Com/CQRS


- Step 1 - A.P.I. - Call made 
- Step 2 - A.P.I. - Request authorised
- Step 3 - A.P.I. - Create service request
- Step 4 - A.P.I. - Send service request to service layer
- Step 5 - Service Layer - Create new command

- Step 6 - Service Layer - Publish command
Send the command to the command bus.

- Step 7 - Domain - Create command handler
The command is received from the command bus.
The corresponding command handler (only one can exist) is instantiated to handle the command.
The command is passed into the command handler for processing.

- Step 8 - Domain - Validate command
The command is validated to make sure it adheres to basic requirements such as required fields are provided, strings match certain criteria, such as being an email address. Further details are available.

- Step 9 - Domain - Create aggregate
An instance of an aggregate is created and instantiated.

- Step 10 - Domain - Rehydrate aggregate 组装领域数据
If the command is intended to change the state of an instance of an existing aggregate, the aggregate is rehydrated.

- Step 11 - Domain - Apply command
The command is applied. Business logic and rules are evaluated, state is not changed. Further details are available.
- Step 11.1 - Domain - Create new event
Depending on the work flow taken based on your business logic and the commands values, one or more events are created.

- Step 12 - Domain - Publish event
Send the event to the event bus.
- Step 12.1 - Domain - Apply event to aggregate
The event is applied to the aggregate, here the state if the aggregate is finally updated.

- Step 13 - Domain - Create event handler(s)
The event is received from the event bus.
The corresponding event handler(s) are instantiated to handle the event.
The event is passed into each event handler for processing.
一个事件多个事件处理器
事件分为内部事件与外部事件, 外部事件
外部事件通常会通过 mq 等设施 会被外部系统订阅

- Step 14 - Domain - Apply event
Such as updating a projection.

- Step 14.1 - Domain Edge - Republish event to an external system
Such as publishing to front-end client via web-sockets.


 -->



[slide]
## hydrate
----- 
> Hydrating an object is taking an object that exists in memory, that doesn't yet contain any domain data ("real" data), and then populating it with domain data (such as from a database, from the network, or from a file system).

--- 
From Erick Robertson's comments on this answer:

> deserialization == instantiation + hydration

---- 
<!-- 

hydrate

https://stackoverflow.com/questions/6991135/what-does-it-mean-to-hydrate-an-object


If you don't need to worry about blistering performance, and you aren't debugging performance optimizations that are in the internals of a data access API, then you probably don't need to deal with hydration explicitly. You would typically use deserialization instead so you can write less code. Some data access APIs don't give you this option, and in those cases you'd also have to explicitly call the hydration step yourself. 

-->





<!-- 


-->

[slide]

## CQRS\ES - CQRS架构进阶
----
![](\img\cqrs_arch2.jpg)



<!-- 
前面的CQRS架构图我介绍了CQRS架构的基本概念、设计初衷、一致性模型、实现方式、适用场景、架构的基本数据流这些方面。但这不是CQRS架构的全部，我们还可以挖掘出更多有用的特性出来。比如假设我们为这个架构引入以下一些特性，就可以达到更多意想不到的好处：

1. 遵守一个原则：一个命令只允许修改一个聚合根；
2. 命令或事件在分布式MQ的路由根据聚合根ID来路由，也就是同一个聚合根的命令和事件都在一个队列里；
3. 引入Command Mailbox，Event Mailbox这两个概念，将聚合根需要处理的命令和产生的事件都队列化，去并发；做到架构上最大的并行，将并发降低到最低；
4. 引入Group Commit技术，做到整个C端的架构层面支持批量提交聚合根产生的事件，从而极大的提高C端的整体吞吐量；比如可以实现对同一个聚合根的每秒修改TPS达到5W？这个在传统的架构下是很难做到的。而在这个架构下，框架就可以提供支持。
5. 通过引入Saga（不了解的同学可以网上搜一下什么是CQRS Saga）的概念，做到基于事件驱动的最终一致性，大家可以回想一下前面介绍的Node通过Event连接的架构；整个系统的所有节点的交互通过消息来驱动；

通过引入上面这些架构设计原则，我们可以让CQRS架构的C端更强大，性能更高；当然，复杂性也大大增加。所以，要完成这样一套架构，没有成熟框架的支撑，是几乎不可能的，ENode框架就是在为做这样的一个框架而努力。
 -->




[slide]
## CQRS\ES - 架构非功能特性分析
----
* 数据一致性模型
* 并发、并行、性能
* 消息幂等
* 并发冲突处理
* 扩展性
* 可用性
* 瓶颈、伸缩性

<!-- 
我们可以从上面几个非功能性特性去考察这个架构。大部分大家应该都可以体会到，关于消息的幂等处理这块，CQRS\ES这个架构可以做的非常彻底。

平时传统我们的消息驱动的架构，或者是RPC调用的SOA风格的应用，消息处理者或者服务被调用方，必须自己做到数据修改的幂等性。而幂等性的实现思路也很多，比如用kv来判重，用DB的唯一索引，等等。

而CQRS\ES架构，由于使用了Event Sourcing的技术，所以可以直接在EventStore中自动做到聚合根并发修改的冲突的检测、以及同一个命令的重复处理的检测。并能通知框架自动做并发处理或做重新发布该命令所产生的事件；

大家可能会疑问，为何已经将命令通过聚合根ID进行路由了，且同一台机器内页已经通过Actor Mailbox技术解决并发问题了，还是有并发冲突的可能呢？原因是当我们的服务器在出现扩容或缩容时，会出现由于集群中服务器变动导致的同一个聚合根的不同命令可能会在不同的机器上同时被处理，从而导致并发冲突。

最后，关于这个架构的瓶颈，相信大家已经可以发现，是在EventStore。所以，这就要求我们设计一个超高性能的EventStore数据库。具体见后面的介绍吧。
 -->




[slide]
## CQRS\ES - CQ数据一致性问题
----
* 关键问题：必须确保C端的事件存储顺序与Q端事件响应的顺序相同
* 例子：
    * C端的事件保存顺序：0 + 1 * 2 - 1 => 1
    * Q端的事件响应顺序：0 + 1 - 1 * 2 => 0

<!-- 
上面这个图演示了，当C端产生的事件，在Q端的处理顺序如果不一致时，导致Q端的结果和C端不一致了。所以，事件的处理顺序必须和产生的顺序一致，这点必须保证，但可以由框架来保证，开发者无需关注。需要强调的是，这个顺序处理事件不需要交给分布式消息中间件来保证，而是应该交给Consumer来自己保重。当Consumer收到一个版本为N+2的时间，而当前Q端的版本为N，则N+2的消息需要先hold一下，不要立即处理。然后等待N+1的事件过来，N+1的事件过来并处理后，再处理N+2的事件。如果N+1的事件一直不过来，则需要永远等待。总之，这里的顺序必须保证。如果这个顺序交给分布式消息中间件去保证，那性能上会非常差，而要让分布式消息中间件实现绝对意义上的顺序消费，又要实现高可用，高性能，难度很大。我个人不太赞成，除非是Consumer自己无法处理消息顺序的场景才迫不得已让分布式消息中间件来保证，比如mysql binlog的同步。
-->


[slide]
## CQRS\ES - 并发
![](\img\evt_concurrency.jpg)

<!-- 
上图演示了假设一个命令修改两个或多个聚合根时，会导致阻塞大大增加，从而整个系统的吞吐会降低。而好处是，我们可以得到聚合根之间的数据的强一致性。
 -->


[slide]
## CQRS\ES - 并行
![](\img\evt_parallel.jpg)

<!-- 
上图演示了，当一个命令只修改一个聚合根时，先通过一级路由，将聚合根路由到分布式MQ的同一个队列里，然后同一个队列总是被一台固定的机器消费，从而保证同一个聚合根的命令总是在一台机器上处理。
 -->




[slide]
## CQRS\ES - 并行
![](\img\evt_parallel2.jpg)

<!-- 
上图掩演示了，当命令进入一台机器后，再通过Command Mailbox的二次路由，同样是根据聚合根ID，从而保证单个机器内，同一个聚合根的命令的处理是顺序线性的，从而避免了并发冲突。
 -->


[slide]
## CQRS\ES - 并发处理、幂等处理
----
* AggregateRootId + Version 唯一
* AggregateRootId + CommandId 唯一


<!-- 
EventStore处理并发和命令幂等的根本设计就是上图的两个唯一索引。
1. 聚合根ID + 事件版本号唯一；
2. 聚合根ID + 命令ID唯一；

当万一出现了并发冲突，则框架需要取出重新加载该聚合根的最新状态，然后重试当前命令；当出现了命令的重复处理，则框架需要把该命令之前产生的事件再重新取出来，发布到分布式消息中间件。因为有可能之前虽然这个事件被持久化了，但理论山有可能这个事件没有成功发布到分布式消息中间件（因为那个时候断电了，够倒霉的，呵呵）。所以，事件的消费者可能会再次收到这个事件，并处理。但这么做都是为了保证整个业务流的最终一致性。想想之前的EDA的架构图的说明吧。
-->



[slide]
## CQRS\ES - Event Sourcing 的优点
----
* 记录了数据完整变化的过程，最详细的日志
* 可以将系统还原到任何一个时间点
* Domain Event 非常有业务价值，BI分析师能预测业务未来发展情况
* 可以有效解决线上的数据问题，线下重演一遍，就能知道哪里出问题
* 将 Command、Event 串联起来，可以分析聚合根的整个变化过程，有助于排查问题
* 自动并发冲突检测、命令的幂等处理

[slide]
## CQRS\ES - Event Sourcing 的缺点
----
* 事件数量巨大，如何存储
* 如果单个聚合根事件过多，则重演会造成性能问题
* 领域模型重构被制约，事件的修改必须兼容以前的结构
* 数据库订正不在有效
* 架构实践门槛高，没有成熟的框架支撑基本无望
* 需要具备 DDD 领域建模能力
* 事件驱动状态修改，思维转变难



[slide]
## CQRS 代码实例 - Command、 Event

```java
@Command
@Getter @Setter
@AllArgsConstructor
class CreateNoteCommand {
    private String noteId;
    private String title;
}

@Getter @Setter
@AllArgsConstructor
class NoteCreatedEvent extends Event {
    private String noteId;
    private String title;
}
```

<!-- 
下面我们来看看CQRS架构下，开发者需要写的代码有哪些？

首先是需要定义Command和Event。其中Command相当于DDD经典四层架构中的应用层的一个方法的参数。

Command表示命令系统做什么，表达一种意图，在架构上设计为一个DTO即可。Event表示一个事件，表示领域内发生了什么状态变化，用过去式命名事件。事件是只读的。
 -->


[slide]
## CQRS 代码实例 - Command Handler

```java
class NoteCommandHandler implements CommandHandler<CreatedNoteCommand>,
                                    CommandHandler<ChangeNoteTitleCommand>
{
    public void handle(CommandContext context, CreateNodeCommand command) {
        context.add(new Note(command.id, command.title));
    }

    public void handle(CommandContext context, ChangeNoteTitleCommand command) {
        return (Note)(context.get(command.id)).changeTitle(command.title);
    }
}
```

<!-- 
Command Handler是无状态的，用于处理一个或多个命令，不同的命令有不同的Handle方法。一个Command Handler做的典型的事情就两个：

1. 根据命令的信息创建一个聚合根；
2. 根据命令的信息修改一个聚合根；

框架可以做到开发人员无需关注底层的技术问题，比如如何存储聚合根产生的事件，如何发布事件到MQ；彻底做到技术架构和业务逻辑分离。这点在传统架构下是很难做到的。
 -->


[slide]
## CQRS 代码实例 - Domain Aggregate

```java
class Node extends Entity implements AggregateRoot<Long> {
    @Getter @Setter
    private String title;
    
    public Node(Long id, String title) {
        this.id = id;
        this.title = title;
        dispatchEvt(new NoteCreatedEvent(id, title));
    }

    public void changeTilte(String title) {
        dispatchEvt(new NoteTitleChangedEvent(id, title));
    }

    public void handle(NoteCreatedEvent event) { }

    public void handle(NoteTitleChangedEvent event) { }
}
```

<!-- 
Note表示一个DDD聚合根，这里最核心的概念是：Note内部的状态的修改都是通过事件来驱动的，也就是Note要做任何修改前，总是应该先产生事件，然后框架根据事件调用到对应的Handle方法，然后我们在Handle方法中修改Note的内部状态。

为何要独立拆分出Handle方法呢？因为是在Event Souring事件溯源还原聚合根状态时，框架需要调用这些Handle方法。根据Event Sourcing的思想，会根据Note聚合根的ID获取该聚合根的所有的事件，然后按照事件发生的顺序，分别调用每个事件的Handle方法，就可以还原出聚合根的最新状态了。
 -->


[slide]
## CQRS 代码实例 - Event Handler

```java
    public class NoteEventHandler implements EventHandler<NoteCreatedEvent>, 
                                             EventHandler<NoteTitleChangedEvent> 
    {
        public void handle(HandlingContext context, NoteCreatedEvent event) { }
        
        public void handle(HandlingContext context, NoteTitleChangedEvent event) { }
    }
```

<!-- 
最后一个需要开发者写的代码就是Event Handler，根据CQRS架构的定义，Event Handler负责根据C端产生的事件来更新读库。上面的例子只是记录日志，实际我们需要在Handle方法中更新读库，如数据库，分布式缓存等。
 -->


[slide]
## CQRS\ES - Event Store 设计
![](\img\evt_store.jpg)


[slide]

## CQRS\ES - SAGE

> What is a saga?
An independent component that reacts to domain events in a cross-aggregate, eventually consistent manner. Time can also be a trigger. Sagas are sometimes purely reactive, and sometimes represent workflows.

-----

> From an implementation perspective, a saga is a state machine that is driven forward by incoming events (which may come from many aggregates). Some states will have side effects, such as sending commands, talking to external web services, or sending emails.

<!-- http://cqrs.nu/Faq -->


[slide]
## 秒杀 - Saga设计
![](\img\saga.jpg)

<!-- 
上图描述了一个DDD CQRS架构的典型的Saga的设计，对应前面的秒杀场景的订单处理流程。

上图中，Order、Conference、Payment为三个聚合根，分别表示订单、库存、支付；Order Process Manager是无状态的，表示一个流程管理器，CQRS架构中一般叫Saga。流程管理器的设计理念是：订阅事件，根据不同的事件，发送不同的命令。也就是说，流程管理器的职责是对流程进行建模，负责封装流程控制逻辑，而聚合根负责业务逻辑。整个订单处理的流程大概为业务层面的2PC。即下单时，要先预扣库存；然后，买家付款后要真正扣库存。

上图中，棕色的线条表示命令，蓝色的线条表示事件。

Saga是CQRS架构中处理复杂业务流程的典型做法，通过事件驱动的方式去替代传统的分布式事务。牺牲强一致性的方式来提高系统的吞吐。实际上，在高并发的情况下，有时我们不得不选择最终一致性，因为分布式事务的成本太高。
 -->















<!--  -->


[slide]
# End