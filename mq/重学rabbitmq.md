![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/9825bc315c6034a8b7e1a56f6737965008237682.jpeg)

![image-20200606093111940](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200606093111940.png)

# 重学MQ

## 什么时候使用MQ

> 在互联网架构中，MQ是一种非常常见的上下游“**逻辑解耦+物理解耦**”的消息通信服务。使用了MQ之后，消息发送上游只需要依赖MQ，**逻辑上和物理上都不用依赖**其他服务。
>
> **但是：**
>
> 1. 系统更复杂了
> 2. 消息传递路径更长,延时增加
> 3. 消息的可靠性和重复性互为矛盾,**消息不丢失不重复难以同时保证**
> 4. **上游消息无法知道下游执行结果**,这点很致命。
>
> **例子：**
>
> 页面登陆调用了passport服务，passport服务的**返回结果直接影响了登陆结果**，此时登陆页面和passport服务之间**不可使用MQ**，**必须是调用关系。**
>
> 调用方**依赖结果的业务场景**，请使用**rpc**

## RabbitMQ五种队列

- 简单队列
 ![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805224818627)

> P：消息的生产者
> C：消息的消费者
> 红色：队列
>
> 生产者将消息发送到队列，消费者从队列中获取消息。
>
> 1个生产者对应一个消费者

- 工作模式



![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805224950612)

> 1对多,实际和简单模式一样
>
> 默认情况下rabbitmq会对所有的消费者使用**轮询算法**依次发送消息

- 订阅模式

![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805225208186)

![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805225218886)

> 1. 一个生产者,多个消费者
> 2. 每个消费者都有自己的队列
> 3. 生产者将消息发送到**交换器**,而不是发送到队列中
> 4. 每个队列都要**绑定交换器**
> 5. 生产者发送消息到交换器,交换器分发到队列,实现一个消息被多个消费者消费
> 6. **注意**:交换器没有消息存储能力,若交换器没有绑定队列,那么生产到交换器的数据将被丢弃
>
> 交换器负责将生产者数据发送到队列中.队列内所有消费者默认负载均衡。交换器会将数据一次性传递到所有队列中
>
> 会同一时间发送到所有的队列中

- 路由模式

![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805225313904)

> 使用**direct**模式,会对routeKey做全值匹配,根据绑定的routeKey,将交换器收到的数据发送到对应的队列中
>
> 一个queue可以绑定多个routeKey

- 主题模式(通配符模式)

![这里写图片描述](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20180805225515258)

> 通过routeKey #和*通配符发送到exchange中bind对应routeKey的queue中

## MQ的type

### direct:

> 消息中的**路由键**（routing key）如果和 **Binding 中的 binding key** 一致，
> 交换器就将消息发到对应的队列中。路由键与队列名完全匹配，如果一个队列绑定到交换机要求路由键为“dog”，则只转发 routing key 标记为“dog”的消息，不会转发“dog.puppy”，也不会转发“dog.guard”等等。它是完全匹配、单播的模式。

### fanout:

> 类似于组播模式
>
> 每个发到 fanout 类型交换器的消息都会**分到所有绑定的队列**上去。fanout交换器**不处理路由键**，只是简单的将队列绑定到交换器上，每个发送到交换器的消息都会被**转发到与该交换器绑定的所有队列**上。很像子网广播，每台子网内的主机都**获得了一份复制的消息**。fanout类型转发消息是最快的。

### topic:

> 通配符模式
>
> topic 交换器通过模式匹配分配消息的路由键属性，将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。它将路由键和绑定键的字符串切分成单词，这些**单词之间用点隔开**。它同样也会识别两个通配符：符号“#”和符号"*"。**#**匹配**0**个或多个单词**，****匹配一个单词。

## 持久化

### 交换器持久化

![image-20200607150420456](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200607150420456.png)

> exchange:定义交换器的名称
>
> type:定义交换器的类型
>
> durable:定义交换器是否持久化(交换器以及交换器中的bind关系)

### 队列持久化

![image-20200606094609268](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200606094609268.png)

> queue：queue的名称
>
> **durable**:持久化队列,**并不是持久化队列中的message**
>
> exclusive：排他队列，如果一个队列被声明为排他队列，该队列仅对首次申明它的连接可见，并在连接断开时自动删除。这里需要注意三点：1. 排他队列是基于连接可见的，同一连接的不同信道是可以同时访问同一连接创建的排他队列；2.“首次”，如果一个连接已经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同；3.即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除的，这种队列适用于一个客户端发送读取消息的应用场景。
>
> autoDelete：自动删除，如果该队列没有任何订阅的消费者的话，该队列会被自动删除。这种队列适用于临时队列。

```bash
## channel.queueDeclare(QUEUE_NAME, false, false, false, null); java中第二个参数是定义队列是否持久化
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
exchange_queue2	0
work-queue	100
hello	1 # hello队列中有一条消息没有读取
exchange_queue1	0
##############################
[root@cjsx ~]# docker restart testRabbitmq #重启服务
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
## 可以看到所有队列的数据都消失了
## channel.queueDeclare(QUEUE_NAME, true, false, false, null); java中第二个参数是定义队列是否持久化
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello	1
#这个时候我们定义的队列是开启持久化的,重启服务试试
#重启后队列还在,但是消息却没有了
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello	0

```

### 消息持久化

> ```java
> void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;
> ```
>
> exchange表示exchange的名称 **定义交换器的名称**
> routingKey表示routingKey的名称  **路由键的名称**
> body代表发送的消息体 **二进制数组消息体**
> 有关mandatory和immediate的详细解释可以参考：[RabbitMQ之mandatory和immediate](http://blog.csdn.net/u013256816/article/details/54914525) 
> 这里关键的是BasicProperties props这个参数了，这里看下BasicProperties的定义：**消息体的属性,可以定义持久化**, **类似JMS中的消息头**
>
> ```java
> public BasicProperties(
>             String contentType,//消息类型如：text/plain
>             String contentEncoding,//编码
>             Map<String,Object> headers,
>             //这里的deliveryMode=1代表不持久化，deliveryMode=2代表持久化。
>             Integer deliveryMode,//1:nonpersistent 2:persistent
>             Integer priority,//优先级
>             String correlationId,
>             String replyTo,//反馈队列
>             String expiration,//expiration到期时间
>             String messageId,
>             Date timestamp,
>             String type,
>             String userId,
>             String appId,
>             String clusterId)
>     //常用的格式,jar包中已经帮我们定义好了
>     public static final BasicProperties PERSISTENT_TEXT_PLAIN =//带持久化的字符串
>     new BasicProperties("text/plain",
>                         null,
>                         null,
>                         2,
>                         0, null, null, null,
>                         null, null, null, null,
>                         null, null);
> //也可以用
> AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
> builder.deliveryMode(2);
> AMQP.BasicProperties properties = builder.build();
> channel.basicPublish("exchange.persistent", "persistent",properties, "persistent_test_message".getBytes());
> ```

#### 测试

```bash
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
un_durable_queue	1
durable_queue	1
hello	0
#定义了2个队列 un_durable_queue durable_queue
channel.queueDeclare(QUEUE_NAME, true, false, false, null);#持久化队列
channel.queueDeclare(UN_QUEUE_NAME, false, false, false, null);#非持久化队列
 channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes(StandardCharsets.UTF_8));#message属性定义了持久化
        channel.basicPublish("", UN_QUEUE_NAME, null, message.getBytes(StandardCharsets.UTF_8));# message没有持久化
        
###重启服务后查看
root@0f96768efc56:/opt/rabbitmq/sbin# ./rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
durable_queue	1 #持久化的队列还在,并且队列中还有没有签收的消息
hello	0

```



解耦、异步、削峰是什么? .

消息队列有什么缺点:

Kafka、ActiveMQ、RabbitMQ、 RocketMQ 有什么优缺点?

MQ有哪些常见问题?如何解决这些问题?

什么是RabbitMQ?

rabbitmq的使用场景-

(1)服务间异步通信

(2)顺序消费

(3)定时任务

(4)请求削峰-

RabbitMQ基本概念

RabbitMQ的工作模式

一.simple模式(即最简单的收发模式)

二.work工作模式(资源的竞争)-

三.publish/subscribe发布订阅(共享资源)

四.routing路由模式

五.topic主题模式(路由模式的一种)

如何保证RabbitMQ消息的顺序性?

消息如何分发?

消息怎么路由?

消息基于什么传输?

如何保证消息不被重复消费?或者说，如何保证消息消费时的幕等性?

如何确保消息正确地发送至RabbitMQ?如何确保消 息接收方消费了消息?

下面罗列几种特殊情况

如何保证RabbitMQ消息的可靠传输?

为什么不应该对所有的message 都使用持久化机制?

如何保证高可用的? RabbitMQ的集群

如何解决消息队列的延时以及过期失效问题?消息队列满了以后该怎么处理?有几百万消息持续积压几小时，怎么办?

设计MQ思路？