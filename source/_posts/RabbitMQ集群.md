---
title: RabbitMQ集群
date: 2022-01-09 23:00:52
categories: RabbitMQ
---

# **使用集群的原因** 

最开始我们介绍了如何安装及运行 RabbitMQ 服务，不过这些是单机版的，无法满足目前真实应用的要求。如果 RabbitMQ 服务器遇到内存崩溃、机器掉电或者主板故障等情况，该怎么办？单台 RabbitMQ服务器可以满足每秒 1000 条消息的吞吐量，那么如果应用需要 RabbitMQ 服务满足每秒 10 万条消息的吞吐量呢？购买昂贵的服务器来增强单机 RabbitMQ 务的性能显得捉襟见肘，搭建一个 RabbitMQ 集群才是解决实际问题的关键.

 

# **搭建步骤** 

```java
1.修改 3 台机器的主机名称
	vim /etc/hostname
2.配置各个节点的 hosts 文件，让各个节点都能互相识别对方
	vim /etc/hosts
		10.211.55.74 node1
		10.211.55.75 node2
		10.211.55.76 node3
3.以确保各个节点的 cookie 文件使用的是同一个值
	在 node1 上执行远程操作命令
		scp /var/lib/rabbitmq/.erlang.cookie root@node2:/var/lib/rabbitmq/.erlang.cookie
		scp /var/lib/rabbitmq/.erlang.cookie root@node3:/var/lib/rabbitmq/.erlang.cookie
4.启动 RabbitMQ 服务,顺带启动 Erlang 虚拟机和 RbbitMQ 应用服务(在三台节点上分别执行以
下命令)
	rabbitmq-server -detached
5.在节点 2 执行
	rabbitmqctl stop_app
	(rabbitmqctl stop 会将 Erlang 虚拟机关闭，rabbitmqctl stop_app 只关闭 RabbitMQ 服务)
	rabbitmqctl reset
	rabbitmqctl join_cluster rabbit@node1
	rabbitmqctl start_app(只启动应用服务)
6.在节点 3 执行
	rabbitmqctl stop_app
	rabbitmqctl reset
	rabbitmqctl join_cluster rabbit@node2
	rabbitmqctl start_app
7.集群状态
	rabbitmqctl cluster_status
8.需要重新设置用户
	创建账号
		rabbitmqctl add_user admin 123456
	设置用户角色
		rabbitmqctl set_user_tags admin administrator
	设置用户权限
		rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
9.解除集群节点(node2 和 node3 机器分别执行)
	rabbitmqctl stop_app
	rabbitmqctl reset
	rabbitmqctl start_app
	rabbitmqctl cluster_status
	rabbitmqctl forget_cluster_node rabbit@node2(node1 机器上执行)
```

# **镜像队列**

## **使用镜像的原因** 

如果 RabbitMQ 集群中只有一个 Broker 节点，那么该节点的失效将导致整体服务的临时性不可用，并且也可能会导致消息的丢失。可以将所有消息都设置为持久化，并且对应队列的durable属性也设置为true，但是这样仍然无法避免由于缓存导致的问题：因为消息在发送之后和被写入磁盘井执行刷盘动作之间存在一个短暂却会产生问题的时间窗。通过 publisherconfirm 机制能够确保客户端知道哪些消息己经存入磁盘，尽管如此，一般不希望遇到因单点故障导致的服务不可用。

引入镜像队列(Mirror Queue)的机制，可以将队列镜像到集群中的其他 Broker 节点之上，如果集群中的一个节点失效了，队列能自动地切换到镜像中的另一个节点上以保证服务的可用性。

## **搭建步骤** 

**1.启动三台集群节点**

**2.随便找一个节点添加 policy**

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110214407.png)

**3.在 node1 上创建一个队列发送一条消息，队列存在镜像队列**

 

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110214428.png)

4.停掉 node1 之后发现 node2 成为镜像队列

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110214519.png)

5.就算整个集群只剩下一台机器了 依然能消费队列里面的消息说明队列里面的消息被镜像队列传递到相应机器里面了

# **Nginx+Keepalive** **实现高可用负载均衡**

## **整体架构图** 

​			

## **Nginx实现负载均衡** 

Nginx提供高可用性、负载均衡及基于 TCPHTTP 应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案，包括 Twitter,Reddit,StackOverflow,GitHub 在内的多家知名互联网公司在使用。HAProxy 实现了一种事件驱动、单一进程模型，此模型支持非常大的井发连接数。
扩展 nginx,lvs,haproxy 之间的区别: http://www.ha97.com/5646.html



## nginx.conf配置文件：

```

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

#tcp连接使用stream配置
stream {
    upstream rabbitmq{
        server 192.168.32.3:5672 max_fails=2 fail_timeout=3s weight=2; ##最大失败2次 超时3秒   weight 权重大小 越大越优先  轮询 
        server 192.168.32.4:5672 max_fails=2 fail_timeout=3s weight=2;
        server 192.168.32.5:5672 max_fails=2 fail_timeout=3s weight=2;
    }
    server {
        listen 5678;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass rabbitmq;
 
    }  
}
```



## **Keepalived 实现双机(主备)热备**

试想如果前面配置的 HAProxy 主机突然宕机或者网卡失效，那么虽然 RbbitMQ 集群没有任何故障但是对于外界的客户端来说所有的连接都会被断开结果将是灾难性的为了确保负载均衡服务的可靠性同样显得十分重要，这里就要引入 Keepalived 它能够通过自身健康检查、资源接管功能做高可用(双机热备)，实现故障转移

## **搭建步骤**

```java
1.下载 keepalived
	yum -y install keepalived
2.节点 node1 配置文件
	vim /etc/keepalived/keepalived.conf
	把资料里面的 keepalived.conf 修改之后替换
3.节点 node2 配置文件
	需要修改 global_defs 的 router_id,如:nodeB
	其次要修改 vrrp_instance_VI 中 state 为"BACKUP"；
	最后要将 priority 设置为小于 100 的值
4.添加 haproxy_chk.sh
(为了防止 HAProxy 服务挂掉之后 Keepalived 还在正常工作而没有切换到 Backup 上，所以这里需要编写一个脚本来检测 HAProxy 务的状态,当 HAProxy 服务挂掉之后该脚本会自动重启HAProxy 的服务，如果不成功则关闭 Keepalived 服务，这样便可以切换到 Backup 继续工作)
	vim /etc/keepalived/haproxy_chk.sh(可以直接上传文件)
	修改权限 chmod 777 /etc/keepalived/haproxy_chk.sh
5.启动 keepalive 命令(node1 和 node2 启动)
	systemctl start keepalived
6.观察 Keepalived 的日志
	tail -f /var/log/messages -n 200
7.观察最新添加的 vip
	ip add show
8.node1 模拟 keepalived 关闭状态
	systemctl stop keepalived 
9.使用 vip 地址来访问 rabbitmq 集群
```

# **Federation Exchange**

## **使用它的原因** 

(broker 北京)，(broker 深圳)彼此之间相距甚远，网络延迟是一个不得不面对的问题。有一个在北京的业务(Client 北京) 需要连接(broker 北京)，向其中的交换器 exchangeA 发送消息，此时的网络延迟很小，(Client 北京)可以迅速将消息发送至 exchangeA 中，就算在开启了 publisherconfirm 机制或者事务机制的情况下，也可以迅速收到确认信息。此时又有个在深圳的业务(Client 深圳)需要向 exchangeA 发送消息，那么(Client 深圳) (broker 北京)之间有很大的网络延迟，(Client 深圳) 将发送消息至 exchangeA 会经历一定的延迟，尤其是在开启了 publisherconfirm 机制或者事务机制的情况下，(Client 深圳) 会等待很长的延迟时间来接收(broker 北京)的确认信息，进而必然造成这条发送线程的性能降低，甚至造成一定程度上的阻塞。
将业务(Client 深圳)部署到北京的机房可以解决这个问题，但是如果(Client 深圳)调用的另些服务都部署在深圳，那么又会引发新的时延问题，总不见得将所有业务全部部署在一个机房，那么容灾又何以实现？这里使用 Federation 插件就可以很好地解决这个问题

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110214845.png)

## **搭建步骤** 

**1.需要保证每台节点单独运行**

**2.在每台机器上开启 federation 相关插件**

```
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management
```

**3.原理图(先运行 consumer 在 node2 创建 fed_exchange)**

**4.添加 policy**

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110214925.png)

5.成功的前提

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110215101.png)

生产者：

```java
public class Producer {
    public static  final  String QUEUE_NAME="node1_queue";
    public static  final  String EXCHANGE_NAME="fed_exchange";
    public static  final  String ROUTE_KEY="routekey";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("192.168.32.3");
        connectionFactory.setUsername("admin");
        connectionFactory.setPassword("123456");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME,true,false,false,null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTE_KEY);
        String message="hello world";
        System.out.println("发送把一条消息:"+message);
        channel.basicPublish(EXCHANGE_NAME,ROUTE_KEY,null,message.getBytes());
    }
}

```

消费者:

```java
public class Consumer {
    public static  final  String QUEUE_NAME="node2_queue";
    public static  final  String EXCHANGE_NAME="fed_exchange";
    public static  final  String ROUTE_KEY="routekey";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("192.168.32.4");
        connectionFactory.setUsername("admin");
        connectionFactory.setPassword("123456");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);
        channel.queueDeclare(QUEUE_NAME,true,false,false,null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTE_KEY);
        channel.basicConsume(QUEUE_NAME,true,(consumerTag, message) -> {
            String msg=new String(message.getBody());
            System.out.println("从队列:"+QUEUE_NAME+"中消费一条消息:"+msg);
        },consumerTag -> {
            return;
        });
    }
}

```

# **Federation Queue**

## **使用它的原因** 

联邦队列可以在多个 Broker 节点(或者集群)之间为单个队列提供均衡负载的功能。一个联邦队列可以连接一个或者多个上游队列(upstream queue)，并从这些上游队列中获取消息以满足本地消费者消费消息的需求

## **搭建步骤** 

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110215220.png)

**2.添加 upstream(同上)**  已经配置了node1为上游

**3.添加 policy**

生产者：

```java
public class Producer {

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("192.168.32.3");
        connectionFactory.setUsername("admin");
        connectionFactory.setPassword("123456");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        for (int i = 0; i < 10000; i++) {
            String msg = "hello world" + i;
            channel.basicPublish("","fed.queue",null,msg.getBytes());
        }
        System.out.println("消息发送成功");
    }
}

```

消费者node1:

```java
public class Consumer {
    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("192.168.32.3");
        connectionFactory.setUsername("admin");
        connectionFactory.setPassword("123456");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        System.out.println("消费者node1启动");
        channel.basicConsume(Consumer2.FED_QUEUE_NAME,true,(consumerTag, message) -> {
            String msg="接收到消息:"+new String(message.getBody());
            System.out.println(msg);
        },consumerTag -> {return;});
    }
}

```

消费者node2：  

添加一个一个交换机和队列，并且绑定他们之间的路由关系

```java
public class Consumer2 {
    public static  final String FED_QUEUE_NAME="fed.queue";
    public static  final String ROUTEKEY="route_key";
    public static  final String EXCHANGE_A="exchange_a";
    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("192.168.32.4");
        connectionFactory.setUsername("admin");
        connectionFactory.setPassword("123456");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_A, BuiltinExchangeType.DIRECT);
        channel.queueDeclare(FED_QUEUE_NAME,true,false,false,null);
        channel.queueBind(FED_QUEUE_NAME,EXCHANGE_A,ROUTEKEY);
        System.out.println("消费者node2启动");
        channel.basicConsume(FED_QUEUE_NAME,true,(consumerTag, message) -> {
            String msg="接收到消息:"+new String(message.getBody());
            System.out.println(msg);
        },consumerTag -> {return;});
    }
}

```

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110215940.png)

这里生产者先生产消息到队列中，再启动消费者node2，就可以实现联邦队列，消费者可以从他的上游中相应队列中获取消息。

# **Shovel**

## **使用它的原因** 

Federation 具备的数据转发功能类似，Shovel 够可靠、持续地从一个 Broker 中的队列(作为源端，即source)拉取数据并转发至另一个 Broker 中的交换器(作为目的端，即 destination)。作为源端的队列和作为目的端的交换器可以同时位于同一个 Broker，也可以位于不同的 Broker 上。Shovel 可以翻译为"铲子"，是一种比较形象的比喻，这个"铲子"可以将消息从一方"铲子"另一方。Shovel 行为就像优秀的客户端应用程序能够负责连接源和目的地、负责消息的读写及负责连接失败问题的处理。

## **搭建步骤** 

**1.开启插件(需要的机器都开启)**

rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management

**2.原理图(在源头发送的消息直接回进入到目的地队列)**

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110222502.png)

**3.添加 shovel 源和目的地** 

![](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/20220110222644.png)

