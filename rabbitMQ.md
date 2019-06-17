---------------------
作者：niaobirdfly
来源：CSDN
原文:<https://blog.csdn.net/hellozpc/article/details/81436980>
### 1. 什么是MQ
- 消息队列（Message Queue,简称MQ），本质是个队列，FIFO先入先出，只不过队列中存放的内容
是message而已</br>
主要用途：不同进程/线程之间通信

为什么会产生消息队列：
  -  不同进程之间传递消息时，两个进程之间耦合程度过高，改动一个进程，引发必须修改另一个
  进程，为了隔离这两个进程，在两进程之间抽离出一层（一个模板），所有两进程之间传递的消息，
  都必须通过消息队列来传递，单独修改某一个进程，不会影响另一个。
  - 不同进程之间传递消息时，为了实现标准化了，将消息的格式规范化了，并且，某一个进程接受
  的消息太多，一下子无法处理完，并且也有先后顺序，必须对收到的消息进行排队，因此诞生了事
  实上的消息队列；

### 2. RabbitMQ
#### 2.1 RabbitMQ的简洁
  - MQ 为Message Queue，消息队列是应用层和应用程序之间的通信方法。
  - RabbitMQ 是一个开源的，在AMQP基础上完整的，可复用的企业消息系统。
  - 支持主流的操作系统，Linux，Windows，MacOX等。
  - 多种开发语言支持，Java，Python,Ruby,.NET,PHP,C/C++,nodo.js等

#### 2.1.1 AMQP
AMQP是消息队列的一个协议
![AMQP](https://github.com/liuyashuang/rabbitMQ/blob/master/img/AMQP.png)
#### 2.1.2 RabbitMQ 中的用户角色
  1. 超级管理员（administrator）</br>
  可登陆管理控制台，可查看所有的信息，并且可以对用户，策略进行操作</br>
  2. 监控者（monitoring）</br>
  可登陆管理控制台，同时可以查看rabbitmq节点的相关信息(进程数，内存使用等)</br>
  3. 策略制定者（policymaker）</br>
  可登陆管理控制台，同时可以对policy进行管理，但无法查看节点的相关信息</br>
  4. 普通登陆者（management）</br>
  仅可登陆管理控制台，无法看到节点信息，也无法对策略进行管理
  5. 其他</br>
  无法登陆管理控制台，通常就是普通的生产者和消费者

#### 2.2 RabbitMQ的五种队列
#### 2.2.1 简单队列
**2.2.1.1 图示**</br>
![简单队列](https://github.com/liuyashuang/rabbitMQ/blob/master/img/simplelist.png)</br>
P:消息的生产者</br>
C:消息的消费者</br>
红色：队列</br>
生产者将消息发送到队列，消费者从队列中获取消息</br>

**2.2.1.2 代码案例**</br>
##### *获取MQ的连接*
```
package com.zpc.rabbitmq.util;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;

public class ConnectionUtil {

    public static Connection getConnection() throws Exception {
        //定义连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        //设置服务地址
        factory.setHost("localhost");
        //端口
        factory.setPort(5672);
        //设置账号信息，用户名、密码、vhost
        factory.setVirtualHost("testhost");
        factory.setUsername("admin");
        factory.setPassword("admin");
        // 通过工程获取连接
        Connection connection = factory.newConnection();
        return connection;
    }
}
```
##### *生产者发送消息到队列*
```
package com.zpc.rabbitmq.simple;

import com.zpc.rabbitmq.util.ConnectionUtil;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

public class Send {

    private final static String QUEUE_NAME = "q_test_01";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();

        // 声明（创建）队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 消息内容
        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        //关闭通道和连接
        channel.close();
        connection.close();
    }
}
```
##### *消费者从队列中获取消息*
```
package com.zpc.rabbitmq.simple;

import com.zpc.rabbitmq.util.ConnectionUtil;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;

public class Recv {

    private final static String QUEUE_NAME = "q_test_01";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        // 从连接中创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);

        // 监听队列
        channel.basicConsume(QUEUE_NAME, true, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [x] Received '" + message + "'");
        }
    }
}
```

#### 2.2.2 Work模式
**2.2.2.1 图示**</br>
![work](https://github.com/liuyashuang/rabbitMQ/blob/master/img/word.png)</br>
一个生产者、两个消费者、一个消息只能被一个消费者获取</br>
**2.2.2.2 代码案例**</br>
##### *消费者1*
```
package com.zpc.rabbitmq.work;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;
import com.zpc.rabbitmq.util.ConnectionUtil;

public class Recv {

    private final static String QUEUE_NAME = "test_queue_work";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 同一时刻服务器只会发一条消息给消费者
        //channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，false表示手动返回完成状态，true表示自动
        channel.basicConsume(QUEUE_NAME, true, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [y] Received '" + message + "'");
            //休眠
            Thread.sleep(10);
            // 返回确认状态，注释掉表示使用自动确认模式
            //channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```
##### *消费者2*
```
package com.zpc.rabbitmq.work;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;
import com.zpc.rabbitmq.util.ConnectionUtil;

public class Recv2 {

    private final static String QUEUE_NAME = "test_queue_work";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 同一时刻服务器只会发一条消息给消费者
        //channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，false表示手动返回完成状态，true表示自动
        channel.basicConsume(QUEUE_NAME, true, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [x] Received '" + message + "'");
            // 休眠1秒
            Thread.sleep(1000);
            //下面这行注释掉表示使用自动确认模式
            //channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```
##### *生产者(向队列中发送了100条信息)*
```
package com.zpc.rabbitmq.work;

import com.zpc.rabbitmq.util.ConnectionUtil;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

public class Send {

    private final static String QUEUE_NAME = "test_queue_work";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        for (int i = 0; i < 100; i++) {
            // 消息内容
            String message = "" + i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");

            Thread.sleep(i * 10);
        }

        channel.close();
        connection.close();
    }
}
```
##### *测试结果*</br>
1. 消费者1和消费者2获取到的消息内容是不同的，同一个消息只能被一个消费者获取。</br>
2. 消费者1和消费者2获取到的消息的数量是相同的，一个是消费奇数号消息，一个是偶数。
  - 其实，这样是不合理的，因为消费者1线程停顿时间短。应该是消费者1要比消费者2获取的消息多才对</br>
    RabbitMQ 默认将消息顺序发送给下一个消费者，这样，每个消费者会得到相同数量的信息，即轮训发布消息
  - 怎么样才能做到按照每个消费者的能力分配消息呢，联合使用Qos和Acknowledge就可以做到。</br>
    basicQos 方法设置了当前信道最大预获取（prefetch）消息数量为1。消息从队列异步推送给</br>
    消费者，消费者的 ack 也是异步发送给队列，从队列的视角去看，总是会有一批消息已推送但尚</br>
    未获得 ack 确认，Qos 的 prefetchCount 参数就是用来限制这批未确认消息数量的。设为1时,</br>
    队列只有在收到消费者发回的上一条消息 ack 确认后，才会向该消费者发送下一条消息。</br>
    prefetchCount 的默认值为0，即没有限制，队列会将所有消息尽快发给消费者

  - 2个概念
  - 轮询分发 ：使用任务队列的优点之一就是可以轻易的并行工作。如果我们积压了好多工作，我</br>
    们可以通过增加工作者（消费者）来解决这一问题，使得系统的伸缩性更加容易。在默认情况下，</br>
    RabbitMQ将逐个发送消息到在序列中的下一个消费者(而不考虑每个任务的时长等等，且是提前</br>
    一次性分配，并非一个一个分配)。平均每个消费者获得相同数量的消息。这种方式分发消息机制</br>
    称为Round-Robin（轮询）
  - 公平分发：虽然上面的分配法方式也还行，但是有个问题就是：比如：现在有2个消费者，所有</br>
    的奇数的消息都是繁忙的，而偶数则是轻松的。按照轮询的方式，奇数的任务交给了第一个消费</br>
    者，所以一直在忙个不停。偶数的任务交给另一个消费者，则立即完成任务，然后闲得不行。</br>
    而RabbitMQ则是不了解这些的。这是因为当消息进入队列，RabbitMQ就会分派消息。它不看消</br>
    费者为应答的数目，只是盲目的将消息发给轮询指定的消费者</br>

为了解决这个问题，我们使用basicQos(prefetchCount=1)方法，来限制RabbitMQ只发不超过1条的</br>
消息给同一个消费者。当消息处理完毕后，有了反馈，才会进行第二次发送。使用公平分发，必须关闭</br>
自动应答，改为手动应答

#### 2.2.3 Work模式的“能者多劳”
![work1](https://github.com/liuyashuang/rabbitMQ/blob/master/img/work1.png)</br>

#### 2.2.4 消息的确认模式
消费者从队列中获取消息，服务端如何知道消息已经被消费呢？</br>
模式1：自动确认</br>
只有消息从队列中获取，无论消费者获取到消息后是否成功消费，都认为是消息已经成功消费.</br>
模式2：手动确认</br>
消费者从队列中获取消息后，服务器会将改消息标记为不可用状态，等待消费者的反馈，如果一直没有</br>
反馈，那么该消息将一直处于不可用状态。</br>
**手动模式**</br>
![手动模式](https://github.com/liuyashuang/rabbitMQ/blob/master/img/shoudong.png)</br>
**自动模式**</br>
![自动模式](https://github.com/liuyashuang/rabbitMQ/blob/master/img/zidong.png)</br>

#### 2.2.5 订阅模式
**2.2.5.1 图示**</br>
![订阅模式](https://github.com/liuyashuang/rabbitMQ/blob/master/img/dingyue.png)</br>
一个生产者，多个消费者</br>
每一个消费者都有自己的一个队列</br>
生产者没有将消息直接发送到队列，而是发送到了交换机</br>
每个队列都要绑定到交换机
生产者发送的消息，经过交换机，到达队列，实现一个消息被多个消费者获取的目的</br>
注意：一个消费者队列可以有多个消费者实例，只有其中一个消费者实例会消费</br>

##### *消息的生产者(看作是后台系统)*</br>
向交换机中发送消息</br>
```
package com.zpc.rabbitmq.subscribe;

import com.zpc.rabbitmq.util.ConnectionUtil;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

public class Send {

    private final static String EXCHANGE_NAME = "test_exchange_fanout";

    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明exchange
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        // 消息内容
        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}
```
注意：消息发送到没有队列绑定的交换机时，消息将丢失，因为，交换机没有存储消息</br>
的能力，消息只能存在队列中。</br>


##### *消费者1(看作是前台系统)*
```
package com.zpc.rabbitmq.subscribe;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;

import com.zpc.rabbitmq.util.ConnectionUtil;

public class Recv {

    private final static String QUEUE_NAME = "test_queue_work1";

    private final static String EXCHANGE_NAME = "test_exchange_fanout";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 同一时刻服务器只会发一条消息给消费者
        channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，手动返回完成
        channel.basicConsume(QUEUE_NAME, false, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [Recv] Received '" + message + "'");
            Thread.sleep(10);

            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```
##### *消费者2(看作是搜索系统)*
```
package com.zpc.rabbitmq.subscribe;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;

import com.zpc.rabbitmq.util.ConnectionUtil;

public class Recv2 {

    private final static String QUEUE_NAME = "test_queue_work2";

    private final static String EXCHANGE_NAME = "test_exchange_fanout";

    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 同一时刻服务器只会发一条消息给消费者
        channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，手动返回完成
        channel.basicConsume(QUEUE_NAME, false, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [Recv2] Received '" + message + "'");
            Thread.sleep(10);

            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```
##### *测试结果*</br>
同一个消息被多个消费者获取，一个消费者队列可以有多个消费者实例，只有其中一个消费者实例</br>
会消费到消息。

#### 2.2.6 路由模式
![路由模式](https://github.com/liuyashuang/rabbitMQ/blob/master/img/luyou.png)</br>
![路由1](https://github.com/liuyashuang/rabbitMQ/blob/master/img/luyou1.png)</br>
![路由2](https://github.com/liuyashuang/rabbitMQ/blob/master/img/luyou2.png)</br>
![路由3](https://github.com/liuyashuang/rabbitMQ/blob/master/img/luyou3.png)</br>
