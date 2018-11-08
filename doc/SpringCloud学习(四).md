[TOC]

<strong style="color:red">springboot 2.0.1.RELEASE</strong>
<strong style="color:red">springcloud Finchley.SR1</strong>

# 消息和异步
异步主要应用于三种场景：通知，请求/异步响应，消息
消息队列（MQ）是一种应用程序对应用程序的通信方法：主要应用场景是：异步处理，日志处理（kafka)，流量削峰，应用解耦

## RabbitMQ的基本使用

在这里用到了消息队列rabbitmq作为示范：
### RabbitMQ配置
要用到RabbitMQ，我们需要安装它，这里我们选用容器安装的方法。
首先在本机安装docker[安装方法](https://www.jianshu.com/p/9142187552db),确认docker安装成功后运行指令：`docker run -d --name my-rabbit -p 5672:5672 -p 15672:15672 -h my-rabbit rabbitmq:3.7.7-management`

然后我们可以打开浏览器输入`localhost:15672`(用户名密码都是guest)验证一下，可以看到：
![RabbitMQ](/img/cloud_4_2.png)

既然RabbitMQ容器安装成功，紧接着在项目中引入依赖
![引入依赖](/img/cloud_4_7.png)

在配置文件中加入：
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest 
```
大功告成，RabbitMQ和应用连接成功

### 使用RabbitMQ获取消息

#### 方法一 @RabbitListtener(queues = "xxx")
```java
@RabbitListener(queues = "myQueue")
public void process(String message) {
    log.info("MqReceiver: {}", message);
}
```
使用这个注解我们需要自己在RabbitMQ图形界面添加消息队列:
![添加queue](/img/cloud_4_3.png)
这种方法写起来简单，但是手动添加队列的方法太蠢了，不符合实际

#### 方法二 自动创建队列
```java
//2. 自动创建队列
@RabbitListener(queuesToDeclare = @Queue("myQueue"))
public void process(String message) {
    log.info("MqReceiver: {}", message);
}
```

比较方法一，这种方法更方便，应用启动时就可以看到队列添加上去了，满足某些简单的场景

#### 方法三 自动创建， Exchange和Queue绑定
```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue("myQueue"),
        exchange = @Exchange("myExchange")
))
public void process(String message) {
    log.info("MqReceiver: {}", message);
}
```
运行之后进入myQueue可以看到：
![myQueue](/img/cloud_4_4.png)
将exchange和queue绑定起来了。

可能会不解，这两者绑定起来有什么用呢？
在Rabbit MQ中，无论是生产者发送消息还是消费者接受消息，都首先需要声明一个MessageQueue。这就存在一个问题，是生产者声明还是消费者声明呢？要解决这个问题，首先需要明确：
a)消费者是无法订阅或者获取不存在的MessageQueue中信息。
b)消息被Exchange接受以后，如果没有匹配的Queue，则会被丢弃。[^1]
Exchange是接受生产者消息并将消息路由到消息队列的关键组件。

这样说起来可能有点抽象，用下面这个例子来实际应用一下：

```java
/**
    * 数码供应商服务 接收消息
    * @param message
    */
@RabbitListener(bindings = @QueueBinding(
        exchange = @Exchange("myOrder"),
        key = "computer",
        value = @Queue("computerOrder")
))
public void processComputer(String message) {
    log.info("computer MqReceiver: {}", message);
}

/**
    * 水果供应商服务 接收消息
    * @param message
    */
@RabbitListener(bindings = @QueueBinding(
        exchange = @Exchange("myOrder"),
        key = "fruit",
        value = @Queue("fruitOrder")
))
public void processFruit(String message) {
    log.info("fruit MqReceiver: {}", message);
}
```
RabbitMQ中myOrder(Exchange)的Bindings:
![bindings](/img/cloud_4_5.png)

这时候我们可以自己创建消息的发送
```java
@Component
public class MqSenderTest extends OrderApplicationTests {

    @Autowired
    private AmqpTemplate amqpTemplate;

    @Test
    public void send() {
        amqpTemplate.convertAndSend("myQueue", "now " + new Date());
    }

    @Test
    public void sendOrder() {
        amqpTemplate.convertAndSend("myOrder", "computer", "now " + new Date());
    }
}
```
从这里的sendOrder方法可以看到，**myOrder**这个Exchange担任接受消息，并且通过**routingKey**控制消息的分发到对应的消息队列消费的职责。
这样就实现了同一个消息，能够通知到两个不同的服务，并且做出不同的操作。

## SpringCloudStream的使用

操作消息队列的另一种方法是SpringCloudStream
![SpringCloudStream](/img/cloud_4_6.png)

使用SpringCloudStream首先引入依赖：
![stream-rabbitmq](/img/cloud_4_1.png)

这里发现@Input和@Output不能使用同名(在不同服务之间可以使用同名),如果要在同一个服务测试则需要如下操作：

### yml配置
```yaml
spring:
  cloud:
    stream:
      bindings:
        input: # @Input
          destination: myMessage # 在mq中创建的消息队列名称
          group: order # 表示是在哪个组，可以是同名不同组
          content-type: application/json # mq获取消息的格式
        output: # @Output
          destination: myMessage
          group: order
          content-type: application/json
        input2:
          destination: myMessage2
        output2:
          destination: myMessage2
```

### 编写消息的Input和Output

消息的Input
```java
public interface Sink {

    @Input("input")
    SubscribableChannel input();

    @Input("input2")
    SubscribableChannel input2();
}
```

消息的Output
```java
public interface Source {

    @Output("output")
    MessageChannel output();

    @Output("output2")
    MessageChannel output2();
}
```
消息的获取
```java
@Component
@EnableBinding({Sink.class,Source.class})
@Slf4j
public class StreamReceiver {

//    @StreamListener("input")
//    public void process(String message) {
//        log.info("StreamReceiver: {}", message);
//    }

    /**
     * 接受orderDTO对象
     * @param message
     */
    @StreamListener("input")
    @SendTo("output2") //指定output
    public String process(OrderDTO message) {
        log.info("StreamReceiver: {}", message);
        return "received.";
    }

    @StreamListener("input2")
    public void process2(String message) {
        log.info("StreamReceiver: {}", message);
    }
}
```

测试接口
```java
@RestController
public class SendMessageController {

    @Autowired
    private Source source;

    /**
     * 发送orderdto对象
     */
    @GetMapping("/sendMessage")
    public void process() {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setOrderId("123456");
        //source.output().send(MessageBuilder.withPayload("now" + new Date()).build());
        source.output().send(MessageBuilder.withPayload(orderDTO).build());
    }
}
```
通过调用相应的output发送消息到input并获取消息

## 具体服务应用

在MQ选型时，最重要的还是可靠地消息投递
在product和order服务之间使用mq，具体逻辑如下：
![mq](/img/cloud_4_8.png)

服务调用关系
+ 查询商品信息（调用商品服务）
+ 计算总价（生成订单详情）
+ 商品扣库存（调用商品服务）
+ 订单入库（生成订单）

redis结合
+ 库存在redis中
+ redis判断库存是否充足， 减掉redis中库存 
// 读redis
// 减库存并将新值重新设置进redis
// 订单入库异常，手动回滚（try catch）
+ 订单服务创建订单，写入数据库，并发动消息

![](/img/cloud_4_9.png)

**Tips** yml文件中spring的配置的前缀不能多加

[^1]: https://www.cnblogs.com/linkenpark/p/5393666.html




