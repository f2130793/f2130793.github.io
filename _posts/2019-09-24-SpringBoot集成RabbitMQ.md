---
layout: post
title: SpringBoot集成RabbitMQ简单示例
categories: [java, RabbitMQ]
description: SpringBoot集成RabbitMQ简单示例
keywords: 消息队列, RabbitMQ
---

##### 什么是RabbitMQ
> RabbitMQ是一个被广泛使用的开源消息队列。它是轻量级且易于部署的，它能支持多种消息协议。RabbitMQ可以部署在分布式和联合配置中，以满足高规模、高可用性的需求。

##### RabbitMQ架构
![image](https://macrozheng.github.io/mall-learning/images/arch_screen_52.png)


标志| 中文名|英文名 |描述|
---|--- |---|---
P	|生产者	|Producer|消息的发送者，可以将消息发送到交换机
C   |	消费者	|Consumer|消息的接收者，从队列中获取消息进行消费
X   |	交换机|	Exchange|接收生产者发送的消息，并根据路由键发送给指定队列
Q   |	队列	|Queue	|存储从交换机发来的消息
type   |交换机类型|type|direct表示直接根据路由键（orange/black）发送消息

##### Spring Boot中使用RabbitMQ

###### 添加依赖
```
<!--消息队列相关依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

###### 修改配置
```
 rabbitmq:
    host: localhost # rabbitmq的连接地址
    port: 5672 # rabbitmq的连接端口号
    virtual-host: /moxuanran # rabbitmq的虚拟host
    username: moxuanran # rabbitmq的用户名
    password: moxuanran # rabbitmq的密码
    publisher-confirms: true #如果对异步消息需要回调必须设置为true
```

##### 添加消息队列的枚举配置类QueueEnum
```
@Getter
public enum QueueEnum {
    /**
     * 消息通知队列
     */
    QUEUE_ORDER_CANCEL("yxdplus.order.direct", "yxdplus.order.cancel", "yxdplus.order.cancel"),
    /**
     * 消息通知ttl队列
     */
    QUEUE_TTL_ORDER_CANCEL("yxdplus.order.direct.ttl", "yxdplus.order.cancel.ttl", "yxdplus.order.cancel.ttl");

    /**
     * 交换名称
     */
    private String exchange;
    /**
     * 队列名称
     */
    private String name;
    /**
     * 路由键
     */
    private String routeKey;

    QueueEnum(String exchange, String name, String routeKey) {
        this.exchange = exchange;
        this.name = name;
        this.routeKey = routeKey;
    }
}

```

##### 添加RabbitMQ的配置
```
@Configuration
public class RabbitMqConfig {

    /**
     * 订单消息实际消费队列所绑定的交换机
     */
    @Bean
    DirectExchange orderDirect() {
        return (DirectExchange) ExchangeBuilder
                .directExchange(QueueEnum.QUEUE_ORDER_CANCEL.getExchange())
                .durable(true)
                .build();
    }

    /**
     * 订单延迟队列队列所绑定的交换机
     */
    @Bean
    DirectExchange orderTtlDirect() {
        return (DirectExchange) ExchangeBuilder
                .directExchange(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange())
                .durable(true)
                .build();
    }

    /**
     * 订单实际消费队列
     */
    @Bean
    public Queue orderQueue() {
        return new Queue(QueueEnum.QUEUE_ORDER_CANCEL.getName());
    }

    /**
     * 订单延迟队列（死信队列）
     */
    @Bean
    public Queue orderTtlQueue() {
        return QueueBuilder
                .durable(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getName())
                .withArgument("x-dead-letter-exchange", QueueEnum.QUEUE_ORDER_CANCEL.getExchange())//到期后转发的交换机
                .withArgument("x-dead-letter-routing-key", QueueEnum.QUEUE_ORDER_CANCEL.getRouteKey())//到期后转发的路由键
                .build();
    }

    /**
     * 将订单队列绑定到交换机
     */
    @Bean
    Binding orderBinding(DirectExchange orderDirect,Queue orderQueue){
        return BindingBuilder
                .bind(orderQueue)
                .to(orderDirect)
                .with(QueueEnum.QUEUE_ORDER_CANCEL.getRouteKey());
    }

    /**
     * 将订单延迟队列绑定到交换机
     */
    @Bean
    Binding orderTtlBinding(DirectExchange orderTtlDirect,Queue orderTtlQueue){
        return BindingBuilder
                .bind(orderTtlQueue)
                .to(orderTtlDirect)
                .with(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey());
    }

}

```

##### 业务代码
> 消息发送者，接受者，服务，控制器等常规操作
