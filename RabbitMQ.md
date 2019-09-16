# RabbitMQ



## 如何保证可靠性投递？



### 方式1

![1567746284284](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567746284284.png)



step1:先将业务消息入库,MSGDB记录消息状态（发送中）

step2:发送消息到Broker

step3:Broker收到的结果返回给生产端，Confirm Listener异步监听Broker消息

step4:若生产端收到Broker发送的成功消息，则更新MSGDB状态（已接收）



step3中可能存在网络问题导致Broker确认消息无法送达，导致 MSGDB中的状态一直是0（发送中）



step5:用分布式任务保证同一时间点有一个任务去抓取MSGDB等于0的状态（为了保证精细程度去除重复发送的情况，需要设置一个合理的超时抓取时间）

step6:超时重新发送消息，直到生产端接收到成功消息

step7:设置重试次数，防止一直重试抓取



如果多次失败则更新状态为2，需要人工去处理问题



> 优点：消息投递基本不会出错
>
> 缺点：MSGDB数据库操作频繁，不适合高并发场景





### 方式2

![1567747554383](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567747554383.png)

step1:上游系统将业务消息数据入库，等待入库完成发送消息到Broker（互联网大厂不加事务，事务会严重影响效率）

step2:上游系统再发送一条延迟消息投递检查（5分钟或者2分钟之后发发送），此条消息发送至Callback service队列中

step3:下游消费端监听Broker，接收和处理

step4:下游系统确认之后生成确认消息发送给Broker

step5:Callback service回调服务监听下游确认消息，若成功则将消息确认状态持久化到MSGDB中

step6:Callback service接收上游系统发的延迟投递检查消息，去MSGDB中查询消息状态



若当Callback service接收到step2中发送的投递检查消息，并且在MSGDB中查找不到确认状态，则发送一条重发指令给上游系统，上游系统再去业务数据库中查找相应消息重发



> 优点：流程少做一次DB存储，效率大大提高
>
> 缺点：消息不能百分百保证可靠，但是大厂在高并发场景下使用最多的解决方案





## 幂等性

 ![](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567748739041.png)

![1567748859885](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567748859885.png)

![1567750367034](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567750367034.png)





## 消费端限流

![1567753799553](C:\Users\Muse47\AppData\Roaming\Typora\typora-user-images\1567753799553.png)

如果要做限流，一定不能设置自动签收，必须手工签收