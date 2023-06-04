---
title: Spring Cloud Stream RocketMQ入门
date: 2023-05-28 17:39:55
tags: SpringCloud
categories: Spring
description: 以 RocketMQ 举例，学习 Spring Cloud Stream。
---

# Spring Cloud Stream

Spring Cloud Stream 提出 publish/subscribe、consumer-group、partition 等概念，内部还有 binder、binding 两个核心概念。

## Binder

用于集成消息中间件，创建各种消息总线对应的 Binding。换句话讲，各个消息总线都有自己的 Binder。

```java
public interface Binder<T, 
			C extends ConsumerProperties, // 消费者配置 
			P extends ProducerProperties> { // 生产者配置
  // 创建消费者 binding
	Binding<T> bindConsumer(String name, String group, T inboundBindTarget,
                          C consumerProperties);
	// 创建生产者 binding
	Binding<T> bindProducer(String name, T outboundBindTarget, P producerProperties);

}
```

## Binding

Binding 包括 input binding 和 output binding。

Binding 是用于隔离业务代码和具体的消息中间的时间的一层抽象。

![binding](http://image.hanelalo.cn/image/202305281650482.png)

# Spring Cloud Stream RocketMQ

## 生产者

Spring Cloud Stream 本身有一些配置，同时还允许各个消息中间件的 binder、binding 增加自定义扩展配置。

现在看看一个简单的生产者配置：

```yaml
server:
  port: 18800
spring:
  application:
    name: rocket-producer
  cloud:
    stream:
      bindings:
        demo-output:
          destination: TEST-TOPIC-01
          content-type: application/json
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
        bindings:
          demo-output:
            producer:
              group: test-group
              sync: true
```

* demo-output 是消息通道的名称，destination 则是 topic 的名称。

* `spring.cloud.stream.rocketmq` 系列的配置，则是 rocketmq 的自定义配置。
  * `spring.cloud.stream.rocketmq.binder.name-server` 配置的是 rocketmq name server 的地址。`spring.cloud.stream.rocketmq.bindings.demo-output.producer` 则是使用 demo-input 这个通道的生产者的专属配置，`group` 配置项是生产者组配置，`sync` 配置则是决定发送同步或是异步消息。

然后再看看代码中要如何发送消息。

```java
public interface OrderEventPublisher {

    String CHANNEL="demo-output";

    @Output(CHANNEL)
    MessageChannel demoOuput();

}
```

这里是使用 `@Output` 注解使用 demo-output 这个通道，返回的是一个 MessageChannel，后面就可以通过 MessageChannel 来发送消息。需要注意的是，`@Output` 注解的 value 填写的是在配置文件中声明的通道名称，而不是最终的 topic 的名称。

然后，还需要将这个接口注册成 Spring Bean，直接使用 `@EnableBinding` 注解即可，会自动为 OrderEventPublisher 接口创建代理类。

```java
@SpringBootApplication
@EnableBinding(OrderEventPublisher.class)
public class RocketMqProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(RocketMqProducerApplication.class, args);
    }

}
```

到这里，已经具备了发送消息的能力，再看看要如何使用 MessageChannel 发送消息。

```java
@RestController
@RequestMapping("order")
@Slf4j
public class OrderController {

    @Autowired
    private OrderEventPublisher orderEventPublisher;

    @GetMapping("finish")
    public void finish(){

        OrderEvent event = OrderEvent.builder()
                .orderId(UUID.randomUUID().toString().replace("-", ""))
                .quantity(100L)
                .totalAmount(10000L)
                .type("realtime")
                .build();
        MessageBuilder<OrderEvent> builder = MessageBuilder.withPayload(event);
        boolean send = orderEventPublisher.demoOuput().send(builder.build());
        log.info("发送结果：{}", send);
    }
}
```

这里定义了一个 Http 接口，调用逻辑就是发送一个 OrderEvent，当构建好 OrderEvent，只需要通过 MessageBuilder 构建一个 Message，然后就能通过 MessageChannel 的 send() 方法进行发送。

## 消费者

消费者配置：

```yaml
server:
  port: ${random.int(10000,19999)}
spring:
  application:
    name: rocket-consumer
  cloud:
    stream:
      bindings:
        demo-input:
          destination: TEST-TOPIC-01
          content-type: application/json
          group: CONSUMER-GROUP-02
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
        bindings:
          demo-input:
            consumer:
              enable: true
              broadcasting: false
```

* 为了方便模拟多个消费者的集群消费，这里将 `server.port` 设置成随机的端口。
* `spring.cloud.stream.bindings` 是 Spring Cloud Stream 的配置。
  * demo-input 是定义的通道名称。
  * destination 是 topic 名称。
  * group 是消费者组的名称。
* `spring.cloud.stream.rockqtmq` 是 rocketmq 的自定义配置。
  * `binder.name-server` 指定 rocketmq name server 的地址。
  * `bindings.demo-input.consumer` 指定 demo-input 这个消息通道的消费者配置。 
    * enable 表示是否开启消费。
    * broadcasting 表示是否广播消费。

> 集群消费：一条消息，在一个消费者组中，只能有一个消费者实例进行消费。
>
> 广播消费：一条消息，在一个消费者组中，所有的消费者实例都会消费。

和生产者类似，消费者也需要定义一个接口来使用。

```java
public interface OrderEventSink {

    String CHANNEL="demo-input";

    @Input(CHANNEL)
    SubscribableChannel demoInput();
}
```

同样，`@Input` 注解的 value 也要和配置文件中定义的通道名称一致，而不是和 topic 名称一致。

然后，还需要增加一个消费的监听器：

```java
@Component
@Slf4j
public class OrderEventConsumer {

    @StreamListener(OrderEventSink.CHANNEL)
    public void onMessage(@Payload OrderEvent event) throws Exception {
        log.info("接收到消息{}", event);
    }
}
```

监听器通过 `@StreamListener` 定义，value 填写要监听的消息通道的名称即可，方法参数使用 `@Payload` 注解表示直接取消息体。

onMessage() 方法就是消费的逻辑了，如果消费有问题，抛出了异常，则消费失败，在集群消费的模式下，RoacketMQ 会进行重试。

最后，OrderEventSink 接口也需要通过 `@EnableBinding` 注解创建代理 bean。

```java
@SpringBootApplication
@EnableBinding(OrderEventSink.class)
public class RocketMqConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(RocketMqConsumerApplication.class, args);
    }

}
```

那么现在将生产者、消费者都启动起来，访问消费者的 `/order/finish` 接口，消费者端将收到消息：

```
2023-05-28 17:38:25.841  INFO 14881 --- [UMER-GROUP-02_1] c.h.rocket.consumer.OrderEventConsumer   : 接收到消息OrderEvent(orderId=af1d50f08d5c46f58cd528baf7959dbb, totalAmount=10000, quantity=100, type=realtime)
```

到这里已经知道比较常规的使用。

## 延时消息

RocketMQ 支持了 18 个延迟级别，发送延时消息时，并不能自定义延迟的时间，只能指定延迟的级别，每个延迟级别有固定的延时时长。

延时消息主要是生产方发送时，指定延迟级别，RocketMQ 负责当延迟时间到达时再开放给消费者消费，换句话说，收到的是延时消息还是普通的消息，消费者是无感知的。

通过 Spring Cloud Stream RocketMQ 发送延迟消息时，因为 Spring Cloud Stream 的 API 是没有直接支持延迟消息的，所以在集成时，Spring Cloud Stream RocketMQ 选则通过 message 的 header 来指定延迟级别。

```java
    @GetMapping("finish/delay")
    public void finishDelay(){
        OrderEvent event = OrderEvent.builder()
                .orderId(UUID.randomUUID().toString().replace("-", ""))
                .quantity(100L)
                .totalAmount(10000L)
                .type("delay")
                .build();
        MessageBuilder<OrderEvent> builder = MessageBuilder.withPayload(event);
        builder.setHeader(MessageConst.PROPERTY_DELAY_TIME_LEVEL, 3);
        boolean send = orderEventPublisher.demoOuput().send(builder.build());
        log.info("发送结果：{}", send);
    }
```

这里是通过 `MessageConst.PROPERTY_DELAY_TIME_LEVEL` 消息头指定延迟级别，延迟级别 3 对应的延时时长是 10 秒。

不同级别对应的延迟级别：

| 延迟级别 | 延时时长 | 延迟级别 | 延时时长 | 延迟级别 | 延时时长 |
| -------- | -------- | -------- | -------- | -------- | -------- |
| 1        | 1s       | 7        | 3m       | 13       | 9m       |
| 2        | 5s       | 8        | 4m       | 14       | 10m      |
| 3        | 10s      | 9        | 5m       | 15       | 20m      |
| 4        | 30s      | 10       | 6m       | 16       | 30m      |
| 5        | 1m       | 11       | 7m       | 17       | 1h       |
| 6        | 2m       | 12       | 8m       | 18       | 2h       |

## 消息重试

在开始讲消息重试之前，再回顾一下两种消费模式：

* 集群消费：同一条消息，一个消费者组中只能有一个消费者实例进行消费。
* 广播消费：同一条消息，一个消费者组中所有消费者实例都会进行消费。

因为这两种消费模式的机制不同，所以消费重试的机制，也只会在集群消费的模式下才会有。

消息重试，是通过延时消息实现，首次重试的延迟级别是 3，也就是 10 秒，要是再失败，就按延迟级别 4 进行重试，如果延迟级别 18 之后依然没能消费成功，就进入死信队列。

消费重试，则是在消费者端进行配置。

```yaml
server:
  port: ${random.int(10000,19999)}
spring:
  application:
    name: rocket-consumer
  cloud:
    stream:
      bindings:
        demo-input:
          destination: TEST-TOPIC-01
          content-type: application/json
          group: CONSUMER-GROUP-02
          consumer:
            max-attempts: 1
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
        bindings:
          demo-input:
            consumer:
              enable: true
              broadcasting: false
              push:
              	delay-level-when-next-consume: -1

```

* 增加了一个 consumer.max-attemps 配置，设置成 1，这是 Spring Cloud Stream 基于 Spring Retry 实现的本地重试，这里设置成  1。
* 在 RocketMQ 自定义配置中增加了 `delay-level-when-next-consume` 配置，这是 RocketMQ 提供的服务端发起的重试机制。如果是 -1，则失败后不重试，直接进入死信队列；如果是 0，则是由 broker 控制重试，即前面提到的从延时级别 3 开始重试；如果大于 0，则是由客户端自行控制重试。

对于消费异常，Spring Cloud Stream 又提供了针对消费者组的异常处理，以及全局的消费异常处理。

```java
    @ServiceActivator(inputChannel = "TEST-TOPIC-01.CONSUMER-GROUP-02.errors")
    public void handleError(ErrorMessage errorMessage) {
        log.error("[handleError][payload：{}]", ExceptionUtils.getRootCauseMessage(errorMessage.getPayload()));
        log.error("[handleError][originalMessage：{}]", errorMessage.getOriginalMessage());
        log.error("[handleError][headers：{}]", errorMessage.getHeaders());
    }

    @StreamListener(IntegrationContextUtils.ERROR_CHANNEL_BEAN_NAME)
    public void globalErrorHandler(ErrorMessage errorMessage){
        log.error("[globalHandleError][payload：{}]", ExceptionUtils.getRootCauseMessage(errorMessage.getPayload()));
        log.error("[globalHandleError][originalMessage：{}]", errorMessage.getOriginalMessage());
        log.error("[globalHandleError][headers：{}]", errorMessage.getHeaders());
    }
```

其中 `@ServiceActivator` 用于接收指定消费者组的消费异常。

`@StreamListener(IntegrationContextUtils.ERROR_CHANNEL_BEAN_NAME)` 用于监听全局的异常，这其实和普通的消费一样，只不过全局错误有一个专属的通道而已。

## 顺序消息

顺序消息，RocketMQ 支持分区顺序消息、全局顺序消息两种。

分区顺序消息，对于指定的 topic 中的消息，通过 sharding key 进行分区，同一个分区内按严格 FIFO 的进行发布、消费。sharding key 是区别于普通消息 key 的字段。RocketMQ 通过将同一个 sharding 分区的消息发送的同一个队列来实现分区顺序消息。

全局顺序消息，指的是 topic 级别有序，topic 内的所有消息严格遵循 FIFO 的顺序进行发布、消费，这种保证了严格顺序的机制，就需要付出性能的代价，所以也只适用于性能要求不高，但是要求严格 FIFO 的场景。RocketMQ 是在分区顺序消息的基础上进行实现全局顺序消息，可以理解为全局顺序消息其实是通过只有一个消息分区的 topic 实现的，本质上还是分区顺序消息，只不过因为只有 1 个分区，就变成了全局顺序消息。

要是实现分区顺序消息很简单，只需要生产者将同一类消息发到同一个队列即可。通过增加一个 `spring.cloud.stream.bindings.<name>.producer.partition-key-expression` 配置来保证同一类消息发送到同一队列。

```yaml
spring:
  application:
    name: rocket-producer
  cloud:
    stream:
      bindings:
        demo-output:
          destination: TEST-TOPIC-01
          content-type: application/json
          producer:
            partition-key-expression: headers['partition']
```

`headers[partition]` 意思是取消息头中的 `partition` 字段作为分片的 sharding key。

如果要使用消息体中的某个字段作为 sharding key，也可以通过 `payload['id']` 的方式来指定，这里是指定 payload 中的 id 字段作为 sharding key。

需要注意的是，虽然在实现顺序消息时，生产方将同一种消息放到了同一个分区里面，但如果消费方是多个线程消费，那么就存在消费顺序被打乱的风险。

因此，一方面，对于广播消费这种模式，其实是天生就不支持顺序消息的，另一方面，如果消费方本身就是异步的消费，就算消息按顺序到达消费方，其实依然还是有乱序的风险。

所以在消费者的配置上，通过  `spring.cloud.stream.rocketmq.bindings.<name>.consumer.push.orderly` 控制是否顺序消费，默认是 false，表示并发多线程消费，设置为 true 就是顺序消费。可以看看 `RocketMQInboundChannelAdapter#doInit` 方法。

> Spring Cloud Stream RocketMQ 默认是 Push 模式的消费者，所以配置路径里面还使用了 `push` 的字样。

## 事务消息

事务消息并不是 Spring Cloud Stream 原生支持的，而是 RocketMQ 自己的特殊功能。

> 下面是从 [RoacketMQ 官网：事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage) 扒下来的图：
>
> ![事务消息](http://image.hanelalo.cn/image/202306040904379.png)
>
> 1. 生产者将消息发送至Apache RocketMQ服务端。
>
> 2. Apache RocketMQ服务端将消息持久化成功之后，向生产者返回Ack确认消息已经发送成功，此时消息被标记为"暂不能投递"，这种状态下的消息即为半事务消息。
>
> 3. 生产者开始执行本地事务逻辑。
>
> 4. 生产者根据本地事务执行结果向服务端提交二次确认结果（Commit或是Rollback），服务端收到确认结果后处理逻辑如下：
>
>    - 二次确认结果为Commit：服务端将半事务消息标记为可投递，并投递给消费者。
>
>    - 二次确认结果为Rollback：服务端将回滚事务，不会将半事务消息投递给消费者。
>
> 5. 在断网或者是生产者应用重启的特殊情况下，若服务端未收到发送者提交的二次确认结果，或服务端收到的二次确认结果为Unknown未知状态，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。 **说明** 服务端回查的间隔时间和最大回查次数，请参见[参数限制](https://rocketmq.apache.org/zh/docs/introduction/03limits)。
>
> 6. 生产者收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
> 7. 生产者根据检查到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤4对半事务消息进行处理。

从上面的事务消息的发送的步骤来看，这其实和消费者没多大关系，消费端根本感知不到这条消息是不是事务消息，主要还是在生产者进行处理。

在生产者一方，要发送事务消息，相比之前的配置文件，还需要增加两个配置：

```yaml
server:
  port: 18800
spring:
  application:
    name: rocket-producer
  cloud:
    stream:
      bindings:
        demo-output:
          destination: TEST-TOPIC-03
          content-type: application/json
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
        bindings:
          demo-output:
            producer:
              group: test-group
              sync: true
              producerType: Trans # 事务消息
              transactionListener: orderTransactionalListener # 事务消息监听器
```

* `spring.cloud.stream.rocketmq.bindings.<name>.producer.producerType`，将生产者类型设置为事务消息，默认是 Normal，普通消息。
* `spring.cloud.stream.rocketmq.bindings.<name>.producer.transactionListener`，事务消息监听器，用于执行本地事务和事务回查。

```java
@Component("orderTransactionalListener")
@Slf4j
public class OrderTransactionalListener implements TransactionListener {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        log.info("execute local transaction:{}", JSON.toJSONString(msg));
        return LocalTransactionState.UNKNOW;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        log.info("消息回查:{}", JSON.toJSONString(msg));
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

> 这里会首先调用 `#executeLocalTransaction` 方法执行本地事务，但是返回的是 UNKNOW，所以隔一段时间会调用 `#checkLocalTransaction` 方法检查本地事务执行状态。

事务消息监听器，需要实现 RocketMQ 提供的 `TransactionListener` 接口。

* `#executeLocalTransaction` 方法用于执行本地事务，返回是的 LocalTransactionState 枚举类型。

* `#checkLocalTransaction` 方法用于处理事务消息回查，返回的也是 LocalTransactionState 枚举类型。

LocalTransactionState 枚举有 3 个值：

* *COMMIT_MESSAGE*，表示本地事务已提交。
* *ROLLBACK_MESSAGE*，表示本地事务已回滚。
* *UNKNOW*，表示状态还未知。

# 参考文档

* [事务消息](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage)
* [顺序消息](https://rocketmq.apache.org/zh/docs/featureBehavior/03fifomessage)
* [Ordered Messages](https://www.alibabacloud.com/help/en/message-queue-for-apache-rocketmq/latest/ordered-messages-1-0#section-utp-a6a-u2p)
