# 一、RabbitMQ 简介以及使用场景
## 1、基本概念
|概念名|说明|
|:--|:--|
MQ|1、全称为Message Queue, 消息队列（MQ）是一种应用程序对应用程序的通信方法。2、应用程序通过读写出入队列的消息（针对应用程序的数据）来通信，而无需专用连接来链接它们
消息传递|指的是程序之间通过在消息中发送数据进行通信，而不是通过直接调用彼此来通信，直接调用通常是用于诸如远程过程调用的技术。
排队|1、指的是应用程序通过 队列来通信。2、队列的使用除去了接收和发送应用程序同时执行的要求。
Erlang编程语言|最初目的是进行大型电信交换设备的软件开发，是一种适用于大规模并行处理环境的高可靠性编程语言。
AMQP协议|高级消息队列协议。1、AMQP的主要特征是面向消息、队列、路由(包括点对点和发布/订阅)、可靠性、 安全。2、AMQP协议更多用在企业系统内，对数据一致性、稳定性和可靠性要求很高的场景，对性能和吞吐量的要求还在其次。


## 2、实现
RabbitMQ是使用Erlang语言开发的开源消息队列系统，基于AMQP协议(高级消息队列协议)来实现。


## 3、应用场景
### (1) 解耦
（为面向服务的架构（SOA）提供基本的最终一致性实现）
例子：用户下单以后，库存数量需要减少。


### (2) 异步提升效率
异步读取消息队列

### (3) 流量削峰
背景：系统其他时间A系统每秒请求量就100个，系统可以稳定运行。系统每天晚间八点有秒杀活动，每秒并发请求量增至1万条，但是系统最大的处理能力只能每秒处理1000个请求，于是系统崩溃，服务器宕机。
之前架构：大量用户（100万用户）通过浏览器在晚上八点高峰期同时参与秒杀活动。大量的请求涌入我们的系统中，高峰期达到每秒钟5000个请求，大量的请求打到MySQL上，每秒钟预计执行3000条SQL。但是一般的MySQL每秒钟扛住2000个请求就不错了，如果达到3000个请求的话可能MySQL直接就瘫痪了，从而系统无法被使用。但是高峰期过了之后，就成了低峰期，可能也就1万用户访问系统，每秒的请求数量也就50个左右，整个系统几乎没有任何压力。
引入MQ：100万用户在高峰期的时候，每秒请求有5000个请求左右，将这5000请求写入MQ里面，系统A每秒最多只能处理2000请求，因为MySQL每秒只能处理2000个请求。系统A从MQ中慢慢拉取请求，每秒就拉取2000个请求，不要超过自己每秒能处理的请求数量即可。MQ，每秒5000个请求进来，结果只有2000个请求出去，所以在秒杀期间（将近一小时）可能会有几十万或者几百万的请求积压在MQ中。

考虑MQ消息积压问题：我们在此计算一下，每秒在MQ积压3000条消息，1分钟会积压18万，1小时积压1000万条消息，高峰期过后，1个多小时就可以将积压的1000万消息消费掉。

## 4、引入消息队列的优缺点
### (1) 优点
优点就是以上的那些场景应用，就是在特殊场景下有其对应的好处，解耦、异步、削峰。

### (2) 缺点
① 系统的可用性降低
系统引入的外部依赖越多，系统越容易挂掉，本来只是A系统调用BCD三个系统接口就好，ABCD四个系统不报错整个系统会正常运行。
引入了MQ之后，虽然ABCD系统没出错，但MQ挂了以后，整个系统也会崩溃。

② 系统的复杂性提高
引入了MQ之后，需要考虑的问题也变得多了，如何保证消息没有重复消费？如何保证消息不丢失？怎么保证消息传递的顺序？

③ 一致性问题
A系统发送完消息直接返回成功，但是BCD系统之中若有系统写库失败，则会产生数据不一致的问题。

# 二、SpringBoot中使用RabbitMQ
## 第一步：引入
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## 第二步：配置application.yml
```yaml
spring:
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: test
    password: 123456
#    virtual-host: /
```
## 第三步：新建RabbitConfig

|概念名|说明|
|:--|:--|
Broker|它提供一种传输服务,它的角色就是维护一条从生产者到消费者的路线，保证数据能按照指定的方式进行传输,
Exchange|消息交换机,它指定消息按什么规则,路由到哪个队列。
Queue|消息的载体,每个消息都会被投到一个或多个队列。
Binding|绑定，它的作用就是把exchange和queue按照路由规则绑定起来.
Routing Key|路由关键字,exchange根据这个关键字进行消息投递。
vhost|虚拟主机,一个broker里可以有多个vhost，用作不同用户的权限分离。
Producer|消息生产者,就是投递消息的程序.
Consumer|消息消费者,就是接受消息的程序.
Channel|消息通道,在客户端的每个连接里,可建立多个channel.




```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


/**
 * 类功能描述：<br>
 * Broker:它提供一种传输服务,它的角色就是维护一条从生产者到消费者的路线，保证数据能按照指定的方式进行传输,
 * Exchange：消息交换机,它指定消息按什么规则,路由到哪个队列。
 * Queue:消息的载体,每个消息都会被投到一个或多个队列。
 * Binding:绑定，它的作用就是把exchange和queue按照路由规则绑定起来.
 * Routing Key:路由关键字,exchange根据这个关键字进行消息投递。
 * vhost:虚拟主机,一个broker里可以有多个vhost，用作不同用户的权限分离。
 * Producer:消息生产者,就是投递消息的程序.
 * Consumer:消息消费者,就是接受消息的程序.
 * Channel:消息通道,在客户端的每个连接里,可建立多个channel.
 * <ul>
 * <li>类功能描述1<br>
 * <li>类功能描述2<br>
 * <li>类功能描述3<br>
 * </ul>
 * 修改记录：<br>
 * <ul>
 * <li>修改记录描述1<br>
 * <li>修改记录描述2<br>
 * <li>修改记录描述3<br>
 * </ul>
 */
@Configuration
public class RabbitConfig {
    //扇形交换器类型
    public static final String FANOUT_TYPE = "FANOUT";
    //扇形交换器测试队列
    public static final String FANOUT_QUEUE_NAME = "test_fanout_queue";
    public static final String FANOUT_QUEUE_NAME1 = "test_fanout_queue1";
    //扇形交换器
    public static final String TEST_FANOUT_EXCHANGE = "testFanoutExchange";

    //直接交换器类型
    public static final String DIRECT_TYPE = "DIRECT";
    //直接交换器测试队列
    public static final String DIRECT_QUEUE_NAME = "test_direct_queue";
    //直接交换器
    public static final String TEST_DIRECT_EXCHANGE = "testDirectExchange";
    //直接交换器ROUTINGKEY
    public static final String DIRECT_ROUTINGKEY = "test";

    //主题交换器类型
    public static final String TOPIC_TYPE = "TOPIC";
    //主题交换器队列
    public static final String TOPIC_QUEUE_NAME = "test_topic_queue";
    public static final String TOPIC_QUEUE_NAME1 = "test_topic_queue1";
    //主题交换器
    public static final String TEST_TOPIC_EXCHANGE = "testTopicExchange";
    //主题交换器ROUTINGKEY
    public static final String TOPIC_ROUTINGKEY = "test.#";

    //创建扇形交换器测试队列
    @Bean
    public Queue createFanoutQueue() {
        return new Queue(FANOUT_QUEUE_NAME);
    }

    //创建扇形交换器测试队列1
    @Bean
    public Queue createFanoutQueue1() {
        return new Queue(FANOUT_QUEUE_NAME1);
    }

    //创建直接交换器测试队列
    @Bean
    public Queue createDirectQueue() {
        return new Queue(DIRECT_QUEUE_NAME);
    }

    //创建主题交换器测试队列
    @Bean
    public Queue createTopicQueue() {
        return new Queue(TOPIC_QUEUE_NAME);
    }

    //创建主题交换器测试队列1
    @Bean
    public Queue createTopicQueue1() {
        return new Queue(TOPIC_QUEUE_NAME1);
    }

    //创建扇形交换器
    @Bean
    public FanoutExchange defFanoutExchange() {
        return new FanoutExchange(TEST_FANOUT_EXCHANGE);
    }

    //扇形交换器和扇形队列绑定
    @Bean
    Binding bindingFanout() {
        return BindingBuilder.bind(createFanoutQueue()).
                to(defFanoutExchange());
    }

    //扇形交换器和扇形队列绑定
    @Bean
    Binding bindingFanout1() {
        return BindingBuilder.bind(createFanoutQueue1()).
                to(defFanoutExchange());
    }

    //创建直接交换器
    @Bean
    DirectExchange directExchange() {
        return new DirectExchange(TEST_DIRECT_EXCHANGE);
    }

    //直接交换器和直接队列绑定
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(createDirectQueue()).
                to(directExchange()).
                with(DIRECT_ROUTINGKEY);
    }

    //创建主题交换器
    @Bean
    TopicExchange defTopicExchange() {
        return new TopicExchange(TEST_TOPIC_EXCHANGE);
    }

    //主题交换器和主题队列绑定
    @Bean
    Binding bindingTopic() {
        return BindingBuilder.bind(createTopicQueue()).
                to(defTopicExchange()).
                with(TOPIC_ROUTINGKEY);
    }

    //主题交换器和主题队列绑定
    @Bean
    Binding bindingTopic1() {
        return BindingBuilder.bind(createTopicQueue1()).
                to(defTopicExchange()).
                with(TOPIC_ROUTINGKEY);
    }
}
```

## 第四步、新建生产者(producer)
```java
import com.heima.rabbitmq.config.RabbitConfig;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;


@Component
public class MsgProducer {

    //使用RabbitTemplate进行操作
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 向扇形交换器发送数据
     *
     * @param routingKey
     * @param massage
     */
    public void send2FanoutTestQueue(String routingKey, String massage) {
        rabbitTemplate.convertAndSend(RabbitConfig.TEST_FANOUT_EXCHANGE,
                routingKey, massage);
    }

    /**
     * 向直接交换器发送数据
     *
     * @param routingKey
     * @param massage
     */
    public void send2DirectTestQueue(String routingKey, String massage) {
        rabbitTemplate.convertAndSend(RabbitConfig.TEST_DIRECT_EXCHANGE,
                routingKey, massage);
    }

    /**
     * 向主题交换器发送数据
     *
     * @param routingKey
     * @param massage
     */
    public void send2TopicTestAQueue(String routingKey, String massage) {
        rabbitTemplate.convertAndSend(RabbitConfig.TEST_TOPIC_EXCHANGE,
                routingKey, massage);
    }   
}
```

## 第五步、新建消费者(consumer)
```java
import com.heima.rabbitmq.config.RabbitConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;

@Component
public class MsgConsumer {

    private static final Logger logger = LoggerFactory.getLogger(MsgConsumer.class);

    /**
     * 监听扇形测试队列数据
     * @param massage
     */
    @RabbitListener(
            bindings =
                    {
                            @QueueBinding(value = @Queue(value = RabbitConfig.FANOUT_QUEUE_NAME, durable = "true"),
                                    exchange = @Exchange(value = RabbitConfig.TEST_FANOUT_EXCHANGE, type = "fanout"))
                    })
    @RabbitHandler
    public void processFanoutMsg(Message massage) {
        String msg = new String(massage.getBody(), StandardCharsets.UTF_8);
        logger.info("接收到FANOUT消息 : " + msg);
    }

    /**
     * 监听扇形测试队列1数据
     * @param massage
     */
    @RabbitListener(
            bindings =
                    {
                            @QueueBinding(value = @Queue(value = RabbitConfig.FANOUT_QUEUE_NAME1, durable = "true"),
                                    exchange = @Exchange(value = RabbitConfig.TEST_FANOUT_EXCHANGE, type = "fanout"))
                    })
    @RabbitHandler
    public void processFanout1Msg(Message massage) {
        String msg = new String(massage.getBody(), StandardCharsets.UTF_8);
        logger.info("接收到FANOUT1消息 : " + msg);
    }

    /**
     * 监听直接测试队列数据
     * @param massage
     */
    @RabbitListener(
            bindings =
                    {
                            @QueueBinding(value = @Queue(value = RabbitConfig.DIRECT_QUEUE_NAME, durable = "true"),
                                    exchange = @Exchange(value = RabbitConfig.TEST_DIRECT_EXCHANGE),
                                    key = RabbitConfig.DIRECT_ROUTINGKEY)
                    })
    @RabbitHandler
    public void processDirectMsg(Message massage) {
        String msg = new String(massage.getBody(), StandardCharsets.UTF_8);
        logger.info("接收到DIRECT消息 : " + msg);
    }

    /**
     * 监听主题测试队列数据
     * @param massage
     */
    @RabbitListener(
            bindings =
                    {
                            @QueueBinding(value = @Queue(value = RabbitConfig.TOPIC_QUEUE_NAME, durable = "true"),
                                    exchange = @Exchange(value = RabbitConfig.TEST_TOPIC_EXCHANGE, type = "topic"),
                                    key = RabbitConfig.TOPIC_ROUTINGKEY)
                    })
    @RabbitHandler
    public void processTopicMsg(Message massage) {
        String msg = new String(massage.getBody(), StandardCharsets.UTF_8);
        logger.info("接收到TOPIC消息 : " + msg);
    }

    /**
     * 监听主题测试1队列数据
     * @param massage
     */
    @RabbitListener(
            bindings =
                    {
                            @QueueBinding(value = @Queue(value = RabbitConfig.TOPIC_QUEUE_NAME1, durable = "true"),
                                    exchange = @Exchange(value = RabbitConfig.TEST_TOPIC_EXCHANGE, type = "topic"),
                                    key = RabbitConfig.TOPIC_ROUTINGKEY)
                    })
    @RabbitHandler
    public void processTopic1Msg(Message massage) {
        String msg = new String(massage.getBody(), StandardCharsets.UTF_8);
        logger.info("接收到TOPIC1消息 : " + msg);
    }

}
```

## 第六步、测试
### 1、编写controller测试
```java
@RestController
@RequestMapping("/rabbitmq")
public class RabbitmqController {

    @Autowired
    private MsgProducer msgProducer;

    /**
     * 发送测试数据
     *
     * @param type       交换器类型
     * @param routingKey
     * @param message
     * @return
     */
    @RequestMapping("/send")
    public String send(String type, String routingKey, String message) {

        if (RabbitConfig.DIRECT_TYPE.equals(type)) {
            //发送直接交换器
            msgProducer.send2DirectTestQueue(routingKey, message);
        } else if (RabbitConfig.TOPIC_TYPE.equals(type)) {
            //发送主题交换器
            msgProducer.send2TopicTestAQueue(routingKey, message);
        } else if (RabbitConfig.FANOUT_TYPE.equals(type)) {
            //发送扇形交换器
            msgProducer.send2FanoutTestQueue(routingKey, message);
        }
        return "OK";
    }
}
```
### 2、直接交换器测试


### 3、扇形交换器测试

### 4、主题交换器测试



# 三、RabbitMQ原理详解



引文：
[RabbitMQ 简介以及使用场景](https://baijiahao.baidu.com/s?id=1685490895134593473&wfr=spider&for=pc)
[SpringBoot中使用RabbitMQ](https://blog.csdn.net/qq_21040559/article/details/109072693)
[RabbitMQ原理详解](https://blog.csdn.net/weixin_43498985/article/details/119026198)




