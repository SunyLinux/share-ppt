title: DDD: Command Pattern In Flowable
speaker: 乌鸦
url: 
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

[slide]
# Command Pattern In Flowable

<!-- 具体分析 看下 flowable 代码  -->

[slide]
## UML 回顾 
----
https://www.jianshu.com/p/1df742b0de81

![](\img\uml.jpg)

<!--  -->

[slide]
## Command Pattern

> 在面向对象程序设计的范畴中，命令模式（英语：Command pattern）是一种设计模式，它尝试以对象来代表实际行动。命令对象可以把行动(action) 及其参数封装起来，于是这些行动可以被：

----

> 1. 重复多次
> 2. 取消（如果该对象有实现的话）
> 3. 取消后又再重做


<!-- Command 实质上是一个可序列化的高阶函数 -->

<!-- http://craft6.cn/detail/activiti_command.do -->


[slide]
## Command Pattern
![](\img\cmd.jpg)

- **Client**：业务调用者
- **Service**：命令调用者，也称为Receiver
- **Executor**：命令执行者，也称为Invoker
- **Command**：命令接口，一般只有一个执行方法
- **ConcereteCommand**：具体的命令实现。由Service调用和实例化，然后传给Executor来执行


<!-- 整个模式的关键点在于：

对于Client而言，是屏蔽Command细节的，即它是不知道系统里面的业务实际上是通过Command来完成的。

Service每个对外方法和普通的没什么区别，可以都是常规的参数，只是在里面构造各个Command的实现类实例，传入相关的参数，并调用Executor执行者执行。 -->


[slide]
## Command Pattern Ext

![](\img\cmd_ext.jpg)

> ExecutorInterceptor：命令执行者的拦截器(责任链)
> CommandContext：命令执行者上下文信息

<!-- 图中黄色的两部分是扩展方式：

ExecutorInterceptor：命令执行者的拦截器。可以在执行之前和之后执行， 可以用于做日志、校验、重试、入栈（以便回退）等等。通过责任链的方式在Executor中维护ExecutorInterceptor，并可以动态增减不同的拦截器实现类。

CommandContext：命令执行者上下文信息。除了构造Command时传入的参数外，命令执行还需要一些公共的上下文信息，可以通过这个类统一维护，每个ConcereteCommand都可以调用。 -->


[slide]
## Command Pattern of Flowable

![](\img\cmd_flow.jpg)

<!-- Flowable 整个引擎完全 Command 来做的 -->

<!--

Command：命令接口，定义执行方法
  1）执行方法支持传入CommandContext（命令上下文）
  2）通过每个Command的实现类的构造函数传入该命令需要特定的参数

CommandContext：维护各个命令所需的通用上下文信息，包括流程定义、流程实例等

CommandExecutor：命令执行者

CommandInterceptor：实例化CommandExecutor时，会设置拦截器的列表，并通过重新组织形成【责任链模式】

TaskServiceImpl：对外服务接口的实现类，通过调用具体的Command实现（比如图中的DeleteTaskCmd）来完成相应的业务实现。 -->


[slide]
## Command Pattern of Flowable
----
![](\img\flow_eg1.jpg)

<!--  -->


[slide]
## Command Pattern of Flowable
----
![](\img\flow_eg2.jpg)

<!--  Flowable 有8大服务，每个服务都含有一个 CommandExecutor 的引用 -->

[slide]
## Command Pattern of Flowable
----
![](\img\flow_eg3.jpg)


<!-- 
事务脚本的项目，日积月累，service 实现的类会逐渐的膨胀，而 基于CQRS 的架构下，业务逻辑会聚合在 Command 内部，日积月累，Command 的数量会非常多；
 -->



 [slide]
# End