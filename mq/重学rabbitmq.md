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

### 持久化机制

#### 内存控制

> 当内存使用超过**阈值**,或者磁盘剩余空间低于设置的值时,broker会阻塞所有生产者的连接,并且**停止接收生产者**发送的消息,从而**避免服务崩溃**,客户端和服务端的心跳检测也会失效.

```bash
rabbitmqctl set_vm_memory_high_watemark <fraction>
# 可以设置内存阈值,默认为0.4,表示当rabbitmq使用了超过40%的内存时,会产生警告并且阻塞所有生产者连接。
#命令行重启后失效
/etc/rabbitmq/rabbitmq.conf
vm_memory_high_watemark.relative=0.4
vm_memory_high_watemark.absolute=1GB
#第一条是百分比,第二条是绝对值
```

##### 内存换页

> 在broker节点触及内存并阻塞生产者之前,会尝试将队列中的消息,换页到磁盘中.以释放内存空间,持久化和非持久化数据都会换页,持久化数据本身磁盘中就已经存在了,内存中会直接清楚。
>
> 默认情况下:内存达到内存阈值的50%就会触发内存换页,默认情况下阈值是0.4,所以0.4*0.5=0.2,达到当前内存的20%就会换页!

```bash
vm_memory_high_watemark.relative=0.4#阈值
vm_memory_high_watemark.paging_radio=0.75 #当使用了阈值内存的75%触发内存换页
#就是40%*75%=%30
#当paging_radio>1的时候相当于禁用换页
#也就是当使用了30%内存的时候启动内存换页
```

#### 磁盘控制

> 当磁盘剩余空间低于阈值的时候,broker也会阻塞生产者发送消息,避免非持久化的消息持续换页耗尽磁盘空间
>
> ,默认情况下,磁盘阈值=50MB,
>
> 一般会将磁盘**剩余阈值会设置成和机器内存大小相同**
>
> RabbitMQ定期**检查可用磁盘空间量**。**检查磁盘空间的频率与上次检查时的空间大小有关**（以确保在空间耗尽时磁盘报警及时消失）。通常情况下，磁盘空间每10秒检查一次，但随着达到极限，频率会增加。当接近极限时，RabbitMQ将每秒检查10次。这可能会对系统负载有一些影响。
>
> 在群集中运行RabbitMQ时，磁盘警报是集群范围内的; 如果**一个节点超出限制**，那么所**有节点都将阻止传入的消息**。

```bash
# 临时设置
rabbitmqctl set_disk_free_limit <disk_limit># 固定大小,KB,MB,GB
rabbitmqctl set_disk_free_limit mem_relative <fraction> # 一般1.0-2.0之间,相对值,相对于内存的大小
# 配置文件
disk_free_limit.relative=1.0 #剩余在1倍的内存大小时报警
#disk_free_limit.absolute=50mb #或者设置固定大小
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

二.work工作模式(资源的竞争)

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







1、使用RabbitMQ有什么好处？
1.解耦，系统A在代码中直接调用系统B和系统C的代码，如果将来D系统接入，系统A还需要修改代码，过于麻烦！

2.异步，将消息写入消息队列，非必要的业务逻辑以异步的方式运行，加快响应速度

3.削峰，并发量大的时候，所有的请求直接怼到数据库，造成数据库连接异常

 

2、RabbitMQ 中的 broker 是指什么？cluster 又是指什么？
broker 是指一个或多个 erlang node 的逻辑分组，且 node 上运行着 RabbitMQ 应用程序。cluster 是在 broker 的基础之上，增加了 node 之间共享元数据的约束。

 

3、RabbitMQ 概念里的 channel、exchange 和 queue 是逻辑概念，还是对应着进程实体？分别起什么作用？
queue 具有自己的 erlang 进程；exchange 内部实现为保存 binding 关系的查找表；channel 是实际进行路由工作的实体，即负责按照 routing_key 将 message 投递给 queue 。由 AMQP 协议描述可知，channel 是真实 TCP 连接之上的虚拟连接，所有 AMQP 命令都是通过 channel 发送的，且每一个 channel 有唯一的 ID。一个 channel 只能被单独一个操作系统线程使用，故投递到特定 channel 上的 message 是有顺序的。但一个操作系统线程上允许使用多个 channel 。


4、vhost 是什么？起什么作用？
vhost 可以理解为虚拟 broker ，即 mini-RabbitMQ  server。其内部均含有独立的 queue、exchange 和 binding 等，但最最重要的是，其拥有独立的权限系统，可以做到 vhost 范围的用户控制。当然，从 RabbitMQ 的全局角度，vhost 可以作为不同权限隔离的手段（一个典型的例子就是不同的应用可以跑在不同的 vhost 中）。


5、消息基于什么传输？
由于TCP连接的创建和销毁开销较大，且并发数受系统资源限制，会造成性能瓶颈。RabbitMQ使用信道的方式来传输数据。信道是建立在真实的TCP连接内的虚拟连接，且每条TCP连接上的信道数量没有限制。


6、消息如何分发？
若该队列至少有一个消费者订阅，消息将以循环（round-robin）的方式发送给消费者。每条消息只会分发给一个订阅的消费者（前提是消费者能够正常处理消息并进行确认）。


7、消息怎么路由？
从概念上来说，消息路由必须有三部分：交换器、路由、绑定。生产者把消息发布到交换器上；绑定决定了消息如何从路由器路由到特定的队列；消息最终到达队列，并被消费者接收。

消息发布到交换器时，消息将拥有一个路由键（routing key），在消息创建时设定。
通过队列路由键，可以把队列绑定到交换器上。
消息到达交换器后，RabbitMQ会将消息的路由键与队列的路由键进行匹配（针对不同的交换器有不同的路由规则）。如果能够匹配到队列，则消息会投递到相应队列中；如果不能匹配到任何队列，消息将进入 “黑洞”。
常用的交换器主要分为一下三种：

direct：如果路由键完全匹配，消息就被投递到相应的队列
fanout：如果交换器收到消息，将会广播到所有绑定的队列上
topic：可以使来自不同源头的消息能够到达同一个队列。 使用topic交换器时，可以使用通配符，比如：“*” 匹配特定位置的任意文本， “.” 把路由键分为了几部分，“#” 匹配所有规则等。特别注意：发往topic交换器的消息不能随意的设置选择键（routing_key），必须是由"."隔开的一系列的标识符组成。

8、什么是元数据？元数据分为哪些类型？包括哪些内容？与 cluster 相关的元数据有哪些？元数据是如何保存的？元数据在 cluster 中是如何分布的？
在非 cluster 模式下，元数据主要分为 Queue 元数据（queue 名字和属性等）、Exchange 元数据（exchange 名字、类型和属性等）、Binding 元数据（存放路由关系的查找表）、Vhost 元数据（vhost 范围内针对前三者的名字空间约束和安全属性设置）。在 cluster 模式下，还包括 cluster 中 node 位置信息和 node 关系信息。元数据按照 erlang node 的类型确定是仅保存于 RAM 中，还是同时保存在 RAM 和 disk 上。元数据在 cluster 中是全 node 分布的。
下图所示为 queue 的元数据在单 node 和 cluster 两种模式下的分布图。

 

 

9、在单 node 系统和多 node 构成的 cluster 系统中声明 queue、exchange ，以及进行 binding 会有什么不同？
答：当你在单 node 上声明 queue 时，只要该 node 上相关元数据进行了变更，你就会得到 Queue.Declare-ok 回应；而在 cluster 上声明 queue ，则要求 cluster 上的全部 node 都要进行元数据成功更新，才会得到 Queue.Declare-ok 回应。另外，若 node 类型为 RAM node 则变更的数据仅保存在内存中，若类型为 disk node 则还要变更保存在磁盘上的数据。

死信队列&死信交换器：DLX 全称（Dead-Letter-Exchange）,称之为死信交换器，当消息变成一个死信之后，如果这个消息所在的队列存在x-dead-letter-exchange参数，那么它会被发送到x-dead-letter-exchange对应值的交换器上，这个交换器就称之为死信交换器，与这个死信交换器绑定的队列就是死信队列。


10、如何确保消息正确地发送至RabbitMQ？
RabbitMQ使用发送方确认模式，确保消息正确地发送到RabbitMQ。发送方确认模式：将信道设置成confirm模式（发送方确认模式），则所有在信道上发布的消息都会被指派一个唯一的ID。一旦消息被投递到目的队列后，或者消息被写入磁盘后（可持久化的消息），信道会发送一个确认给生产者（包含消息唯一ID）。如果RabbitMQ发生内部错误从而导致消息丢失，会发送一条nack（not acknowledged，未确认）消息。发送方确认模式是异步的，生产者应用程序在等待确认的同时，可以继续发送消息。当确认消息到达生产者应用程序，生产者应用程序的回调方法就会被触发来处理确认消息。

 

11、如何确保消息接收方消费了消息？
接收方消息确认机制：消费者接收每一条消息后都必须进行确认（消息接收和消息确认是两个不同操作）。只有消费者确认了消息，RabbitMQ才能安全地把消息从队列中删除。这里并没有用到超时机制，RabbitMQ仅通过Consumer的连接中断来确认是否需要重新发送消息。也就是说，只要连接不中断，RabbitMQ给了Consumer足够长的时间来处理消息。

下面罗列几种特殊情况：

如果消费者接收到消息，在确认之前断开了连接或取消订阅，RabbitMQ会认为消息没有被分发，然后重新分发给下一个订阅的消费者。（可能存在消息重复消费的隐患，需要根据bizId去重）
如果消费者接收到消息却没有确认消息，连接也未断开，则RabbitMQ认为该消费者繁忙，将不会给该消费者分发更多的消息。
12、如何避免消息重复投递或重复消费？
在消息生产时，MQ内部针对每条生产者发送的消息生成一个inner-msg-id，作为去重和幂等的依据（消息投递失败并重传），避免重复的消息进入队列；在消息消费时，要求消息体中必须要有一个bizId（对于同一业务全局唯一，如支付ID、订单ID、帖子ID等）作为去重和幂等的依据，避免同一条消息被重复消费。

这个问题针对业务场景来答分以下几点：

1.比如，你拿到这个消息做数据库的insert操作。那就容易了，给这个消息做一个唯一主键，那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。

2.再比如，你拿到这个消息做redis的set的操作，那就容易了，不用解决，因为你无论set几次结果都是一样的，set操作本来就算幂等操作。

3.如果上面两种情况还不行，上大招。准备一个第三方介质,来做消费记录。以redis为例，给消息分配一个全局id，只要消费过该消息，将<id,message>以K-V形式写入redis。那消费者开始消费前，先去redis中查询有没消费记录即可。


13、如何解决丢数据的问题?
1.生产者丢数据

生产者的消息没有投递到MQ中怎么办？从生产者弄丢数据这个角度来看，RabbitMQ提供transaction和confirm模式来确保生产者不丢消息。

transaction机制就是说，发送消息前，开启事物(channel.txSelect())，然后发送消息，如果发送过程中出现什么异常，事物就会回滚(channel.txRollback())，如果发送成功则提交事物(channel.txCommit())。

然而缺点就是吞吐量下降了。因此，按照博主的经验，生产上用confirm模式的居多。一旦channel进入confirm模式，所有在该信道上面发布的消息都将会被指派一个唯一的ID(从1开始)，一旦消息被投递到所有匹配的队列之后，rabbitMQ就会发送一个Ack给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了.如果rabiitMQ没能处理该消息，则会发送一个Nack消息给你，你可以进行重试操作。

2.消息队列丢数据

处理消息队列丢数据的情况，一般是开启持久化磁盘的配置。这个持久化配置可以和confirm机制配合使用，你可以在消息持久化磁盘后，再给生产者发送一个Ack信号。这样，如果消息持久化磁盘之前，rabbitMQ阵亡了，那么生产者收不到Ack信号，生产者会自动重发。

那么如何持久化呢，这里顺便说一下吧，其实也很容易，就下面两步

①、将queue的持久化标识durable设置为true,则代表是一个持久的队列

②、发送消息的时候将deliveryMode=2

这样设置以后，rabbitMQ就算挂了，重启后也能恢复数据。在消息还没有持久化到硬盘时，可能服务已经死掉，这种情况可以通过引入mirrored-queue即镜像队列，但也不能保证消息百分百不丢失（整个集群都挂掉）

3.消费者丢数据

启用手动确认模式可以解决这个问题

①自动确认模式，消费者挂掉，待ack的消息回归到队列中。消费者抛出异常，消息会不断的被重发，直到处理成功。不会丢失消息，即便服务挂掉，没有处理完成的消息会重回队列，但是异常会让消息不断重试。

②手动确认模式，如果消费者来不及处理就死掉时，没有响应ack时会重复发送一条信息给其他消费者；如果监听程序处理异常了，且未对异常进行捕获，会一直重复接收消息，然后一直抛异常；如果对异常进行了捕获，但是没有在finally里ack，也会一直重复发送消息(重试机制)。

③不确认模式，acknowledge="none" 不使用确认机制，只要消息发送完成会立即在队列移除，无论客户端异常还是断开，只要发送完就移除，不会重发。

 

14、死信队列和延迟队列的使用
死信消息：

消息被拒绝（Basic.Reject或Basic.Nack）并且设置 requeue 参数的值为 false
消息过期了
队列达到最大的长度
过期消息：

    在 rabbitmq 中存在2种方可设置消息的过期时间，第一种通过对队列进行设置，这种设置后，该队列中所有的消息都存在相同的过期时间，第二种通过对消息本身进行设置，那么每条消息的过期时间都不一样。如果同时使用这2种方法，那么以过期时间小的那个数值为准。当消息达到过期时间还没有被消费，那么那个消息就成为了一个 死信 消息。
    
    队列设置：在队列申明的时候使用 x-message-ttl 参数，单位为 毫秒
    
    单个消息设置：是设置消息属性的 expiration 参数的值，单位为 毫秒

延时队列：在rabbitmq中不存在延时队列，但是我们可以通过设置消息的过期时间和死信队列来模拟出延时队列。消费者监听死信交换器绑定的队列，而不要监听消息发送的队列。

有了以上的基础知识，我们完成以下需求：

需求：用户在系统中创建一个订单，如果超过时间用户没有进行支付，那么自动取消订单。

分析：

        1、上面这个情况，我们就适合使用延时队列来实现，那么延时队列如何创建
    
        2、延时队列可以由 过期消息+死信队列 来时间
    
        3、过期消息通过队列中设置 x-message-ttl 参数实现
    
        4、死信队列通过在队列申明时，给队列设置 x-dead-letter-exchange 参数，然后另外申明一个队列绑定x-dead-letter-exchange对应的交换器。

ConnectionFactory factory = new ConnectionFactory(); 
factory.setHost("127.0.0.1"); 
factory.setPort(AMQP.PROTOCOL.PORT); 
factory.setUsername("guest"); 
factory.setPassword("guest"); 
Connection connection = factory.newConnection(); 
Channel channel = connection.createChannel();

// 声明一个接收被删除的消息的交换机和队列 
String EXCHANGE_DEAD_NAME = "exchange.dead"; 
String QUEUE_DEAD_NAME = "queue_dead"; 
channel.exchangeDeclare(EXCHANGE_DEAD_NAME, BuiltinExchangeType.DIRECT); 
channel.queueDeclare(QUEUE_DEAD_NAME, false, false, false, null); 
channel.queueBind(QUEUE_DEAD_NAME, EXCHANGE_DEAD_NAME, "routingkey.dead"); 

String EXCHANGE_NAME = "exchange.fanout"; 
String QUEUE_NAME = "queue_name"; 
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT); 
Map<String, Object> arguments = new HashMap<String, Object>(); 
// 统一设置队列中的所有消息的过期时间 
arguments.put("x-message-ttl", 30000); 
// 设置超过多少毫秒没有消费者来访问队列，就删除队列的时间 
arguments.put("x-expires", 20000); 
// 设置队列的最新的N条消息，如果超过N条，前面的消息将从队列中移除掉 
arguments.put("x-max-length", 4); 
// 设置队列的内容的最大空间，超过该阈值就删除之前的消息
arguments.put("x-max-length-bytes", 1024); 
// 将删除的消息推送到指定的交换机，一般x-dead-letter-exchange和x-dead-letter-routing-key需要同时设置
arguments.put("x-dead-letter-exchange", "exchange.dead"); 
// 将删除的消息推送到指定的交换机对应的路由键 
arguments.put("x-dead-letter-routing-key", "routingkey.dead"); 
// 设置消息的优先级，优先级大的优先被消费 
arguments.put("x-max-priority", 10); 
channel.queueDeclare(QUEUE_NAME, false, false, false, arguments); 
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ""); 
String message = "Hello RabbitMQ: "; 

for(int i = 1; i <= 5; i++) { 
	// expiration: 设置单条消息的过期时间 
	AMQP.BasicProperties.Builder properties = new AMQP.BasicProperties().builder()
			.priority(i).expiration( i * 1000 + ""); 
	channel.basicPublish(EXCHANGE_NAME, "", properties.build(), (message + i).getBytes("UTF-8")); 
} 
channel.close(); 
connection.close();

15、使用了消息队列会有什么缺点?
1.系统可用性降低:你想啊，本来其他系统只要运行好好的，那你的系统就是正常的。现在你非要加个消息队列进去，那消息队列挂了，你的系统不是呵呵了。因此，系统可用性降低

2.系统复杂性增加:要多考虑很多方面的问题，比如一致性问题、如何保证消息不被重复消费，如何保证保证消息可靠传输。因此，需要考虑的东西更多，系统复杂性增大。
————————————————
版权声明：本文为CSDN博主「jeffrey_ding」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/jerryDzan/article/details/89183625