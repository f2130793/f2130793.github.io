---
layout: post
title: Seata解决SpringCloud分布式事务问题
categories: [SpringCloud]
description: Seata解决SpringCloud分布式事务问题
keywords: seata,SpringCloud
---

Seata是Alibaba开源的一款分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务,本文将详细介绍seata的安装配置以及一个简单的demo

### 为什么需要分布式事务？
在单体应用中，数据的一致性可以由本地事务来保证，但在微服务应用中，服务内部的数据一致性由本地事务来保证，但全局的数据一致性问题没法保证。

### 分布式事务解决方案
#### 二阶段提交协议
基于XA协议的，采取强一致性，遵从ACID.

##### XA
XA是由X/Open组织提出的分布式事务的架构（或者叫协议）。XA架构主要定义了（全局）事务管理器（Transaction Manager）和（局部）资源管理器（Resource Manager）之间的接口。
XA接口是双向的系统接口，在事务管理器（Transaction Manager）以及一个或多个资源管理器（Resource Manager）之间形成通信桥梁。
也就是说，在基于XA的一个事务中，我们可以针对多个资源进行事务管理，例如一个系统访问多个数据库，或即访问数据库、又访问像消息中间件这样的资源。
这样我们就能够实现在多个数据库和消息中间件直接实现全部提交、或全部取消的事务。XA规范不是java的规范，而是一种通用的规范,XA协议的JAVA实现是JTA。

##### 二阶段提交协议流程图
![](https://f2130793.github.io/images/2020-07-22-1.jpg)

##### 过程
```
1.请求阶段（commit-request phase，或称表决阶段，voting phase）
在请求阶段，协调者将通知事务参与者准备提交或取消事务，然后进入表决过程。
在表决过程中，参与者将告知协调者自己的决策：同意（事务参与者本地作业执行成功）或取消（本地作业执行故障）。

2.提交阶段（commit phase）
在该阶段，协调者将基于第一个阶段的投票结果进行决策：提交或取消。
当且仅当所有的参与者同意提交事务协调者才通知所有的参与者提交事务，否则协调者将通知所有的参与者取消事务。
参与者在接收到协调者发来的消息后将执行响应的操作。
```

##### 缺点
- 单点故障： 事务的发起、提交和取消，均由协调者管理。
- 同步阻塞： 本地事务全部阻塞等待协调者发起事务指令
- 数据不一致： 真正提交事务时，部分网络故障导致数据不一致

#### 三阶段提交协议
##### 流程图
![](https://f2130793.github.io/images/2020-07-22-2.jpg)
##### 过程
```
1.CanCommit阶段
3PC的CanCommit阶段其实和2PC的准备阶段很像。
协调者向参与者发送commit请求，参与者如果可以提交就返回Yes响应，否则返回No响应。

2.PreCommit阶段
Coordinator根据Cohort的反应情况来决定是否可以继续事务的PreCommit操作。
根据响应情况，有以下两种可能。
A.假如Coordinator从所有的Cohort获得的反馈都是Yes响应，那么就会进行事务的预执行：
发送预提交请求。Coordinator向Cohort发送PreCommit请求，并进入Prepared阶段。
事务预提交。Cohort(一群大兵)接收到PreCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中。
响应反馈。如果Cohort成功的执行了事务操作，则返回ACK响应，同时开始等待最终指令。

B.假如有任何一个Cohort向Coordinator发送了No响应，或者等待超时之后，Coordinator都没有接到Cohort的响应，那么就中断事务：
发送中断请求。Coordinator向所有Cohort发送abort请求。
中断事务。Cohort收到来自Coordinator的abort请求之后（或超时之后，仍未收到Cohort的请求），执行事务的中断。

3.DoCommit阶段

该阶段进行真正的事务提交，也可以分为以下两种情况:

执行提交

A.发送提交请求。Coordinator接收到Cohort发送的ACK响应，那么他将从预提交状态进入到提交状态。并向所有Cohort发送doCommit请求。
B.事务提交。Cohort接收到doCommit请求之后，执行正式的事务提交。并在完成事务提交之后释放所有事务资源。
C.响应反馈。事务提交完之后，向Coordinator发送ACK响应。
D.完成事务。Coordinator接收到所有Cohort的ACK响应之后，完成事务。

中断事务
协调者没有接收到参与者发送的ACK响应，那么就执行中断事务。

A.发送中断请求
协调者向所有参与者发送abort请求
B.事务回滚
参与者接收到abort请求之后，利用其在阶段二记录的undo信息来执行事务的回滚操作，并在完成回滚之后释放所有的事务资源。
C.反馈结果
参与者完成事务回滚之后，向协调者发送ACK消息
D.中断事务
协调者接收到参与者反馈的ACK消息之后，执行事务的中断。
```

##### 2PC和3PC的区别
```
3PC增加了询问，增大了成功概率
```

#### TCC(Try、Confirm、Cancel)
二阶段补偿型方案，实现一个事务，需要定义三个API：预先占有资源，确认提交实际操作资源，取消占有=回滚。
```
2PC：是资源层面的分布式事务，一直会持有资源的锁。
	如果跨十几个库，一下锁这么多数据库，会导致，极度浪费资源。降低了吞吐量。
TCC：在业务层面的分布式事务，最终一致性，不会一直持有锁。将锁的粒度变小，每操作完一个库，就释放了锁。	

性能是相对的：如果每天只有一个请求，用2PC 比 TCC 要性能高。因为tcc多了多次接口调用。而此时的2PC 不怕占用资源，反正就一个调用。高并发场景下TCC 优势要大。
```

#### 消息对了实现柔性事务
![](https://f2130793.github.io/images/2020-07-22-3.jpg)

#### seata框架
##### 官网
<http://seata.io/zh-cn/docs/overview/what-is-seata.html>

##### seata如何定义一个分布式事务？
我们可以把一个分布式事务理解成一个包含了若干分支事务的全局事务，全局事务的职责是协调其下管辖的分支事务达成一致，要么一起成功提交，要么一起失败回滚。此外，通常分支事务本身就是一个满足ACID的本地事务。这是我们对分布式事务结构的基本认识，与 XA 是一致的。
![](https://f2130793.github.io/images/2020-07-22-4.jpg)

##### 协调分布式事务的三个组件
- Transaction Coordinator (TC)： 事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚；
- Transaction Manager (TM)： 控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议；
- Resource Manager (RM)： 控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。
![](https://f2130793.github.io/images/2020-07-22-5.jpg)

##### 典型分布式事务过程
- TM 向 TC 申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的 XID；
- XID 在微服务调用链路的上下文中传播；
- RM 向 TC 注册分支事务，将其纳入 XID 对应全局事务的管辖；
- TM 向 TC 发起针对 XID 的全局提交或回滚决议；
- TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求。
![](https://f2130793.github.io/images/2020-07-22-6.jpg)

##### seata-server的安装与配置
- 从官网下载seata-server，这里下载的1.3.0版本，下载地址：<https://github.com/seata/seata/releases> 
- 采用nacos作为注册中心，配置中心，采用redis存储事务日志
- 解压seata-server安装包到指定目录，修改file.conf文件
```
## transaction log store, only used in seata-server
service {
  #vgroup->rgroup
  vgroup_mapping.my_test_tx_group = "default" #修改事务组名称为：my_test_tx_group，和客户端自定义的名称对应
  #only support single node
  default.grouplist = "127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
store {
  ## store mode: file、db、redis
  mode = "redis"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "mysql"
    password = "mysql"
    minConn = 5
    maxConn = 30
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    queryLimit = 100
  }

}
```
- 修改register.conf，指明注册中心和配置中心均为nacos，修改nacos连接信息
```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "DEFAULT_GROUP"
    namespace = "public"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = 0
    password = ""
    cluster = "default"
    timeout = 0
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "public"
    group = "DEFAULT_GROUP"
    username = "nacos"
    password = "nacos"
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    appId = "seata-server"
    apolloMeta = "http://192.168.1.204:8801"
    namespace = "application"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}
```

##### 将配置写入Nacos
有很多配置文件已经不放在解压包下面了，需要从Github上找。
![](https://f2130793.github.io/images/2020-07-22-7.jpg)
进入Github上config-center目录，将config.txt文件拷贝到我们下载压缩包顶级目录下,然后进入nacos目录，将nacos-config.sh文件复制到conf目录下
```
执行 ./sh nacos-config.sh localhost
```
1.3.0版本有85个配置文件。
![](https://f2130793.github.io/images/2020-07-22-8.jpg)

##### 启动seata-server
```
./sh seata-server.sh
```

#### 数据库准备
- seata-order：存储订单的数据库；
- seata-storage：存储库存的数据库；
- seata-account：存储账户信息的数据库。

##### 初始化表
###### order表
```
CREATE TABLE `order` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) DEFAULT NULL COMMENT '用户id',
  `product_id` bigint(11) DEFAULT NULL COMMENT '产品id',
  `count` int(11) DEFAULT NULL COMMENT '数量',
  `money` decimal(11,0) DEFAULT NULL COMMENT '金额',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

ALTER TABLE `order` ADD COLUMN `status` int(1) DEFAULT NULL COMMENT '订单状态：0：创建中；1：已完结' AFTER `money` ;
```

###### storage表
```
CREATE TABLE `storage` (
                         `id` bigint(11) NOT NULL AUTO_INCREMENT,
                         `product_id` bigint(11) DEFAULT NULL COMMENT '产品id',
                         `total` int(11) DEFAULT NULL COMMENT '总库存',
                         `used` int(11) DEFAULT NULL COMMENT '已用库存',
                         `residue` int(11) DEFAULT NULL COMMENT '剩余库存',
                         PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

INSERT INTO `seat-storage`.`storage` (`id`, `product_id`, `total`, `used`, `residue`) VALUES ('1', '1', '100', '0', '100');
```

###### account表
```
CREATE TABLE `account` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `user_id` bigint(11) DEFAULT NULL COMMENT '用户id',
  `total` decimal(10,0) DEFAULT NULL COMMENT '总额度',
  `used` decimal(10,0) DEFAULT NULL COMMENT '已用余额',
  `residue` decimal(10,0) DEFAULT '0' COMMENT '剩余可用额度',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

INSERT INTO `seat-account`.`account` (`id`, `user_id`, `total`, `used`, `residue`) VALUES ('1', '1', '1000', '0', '1000');
```

###### 在每个数据库下创建日志回滚表
```
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

#### 客户端配置
##### 修改application.yml文件
```
spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: my_test_tx_group #自定义事务组名称需要与seata-server中的对应
```

##### 添加file.conf文件
```
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  # the client batch send request enable
  enableClientBatchSendRequest = true
  #thread factory for netty
  threadFactory {
    bossThreadPrefix = "NettyBoss"
    workerThreadPrefix = "NettyServerNIOWorker"
    serverExecutorThread-prefix = "NettyServerBizHandler"
    shareBossWorker = false
    clientSelectorThreadPrefix = "NettyClientSelector"
    clientSelectorThreadSize = 1
    clientWorkerThreadPrefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    bossThreadSize = 1
    #auto default pin or 8
    workerThreadSize = "default"
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}
service {
  #transaction service group mapping
  vgroupMapping.my_test_tx_group = "default"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #degrade, current not support
  enableDegrade = false
  #disable seata
  disableGlobalTransaction = false
}

client {
  rm {
    asyncCommitBufferLimit = 10000
    lock {
      retryInterval = 10
      retryTimes = 30
      retryPolicyBranchRollbackOnConflict = true
    }
    reportRetryCount = 5
    tableMetaCheckEnable = false
    reportSuccessEnable = false
    sagaBranchRegisterEnable = false
  }
  tm {
    commitRetryCount = 5
    rollbackRetryCount = 5
    degradeCheck = false
    degradeCheckPeriod = 2000
    degradeCheckAllowTimes = 10
  }
  undo {
    dataValidation = true
    onlyCareUpdateColumns = true
    logSerialization = "jackson"
    logTable = "undo_log"
  }
  log {
    exceptionRate = 100
  }
}
```

##### 添加register.conf文件
```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "DEFAULT_GROUP"
    namespace = "public"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = 0
    password = ""
    cluster = "default"
    timeout = 0
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "public"
    group = "DEFAULT_GROUP"
    username = "nacos"
    password = "nacos"
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    appId = "seata-server"
    apolloMeta = "http://192.168.1.204:8801"
    namespace = "application"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}
```

##### 启动类取消数据源自动创建
```
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
@EnableDiscoveryClient
@MapperScan("com.moxuanran.learning.mapper")
public class SeataOrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(SeataOrderApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

##### 创建配置使用seata对数据源代理
```
@Configuration
public class DataSourceConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

    @Primary
    @Bean("dataSource")
    public DataSourceProxy dataSource(DataSource druidDataSource){
        return new DataSourceProxy(druidDataSource);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSourceProxy dataSourceProxy)throws Exception{
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSourceProxy);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath*:/mapper/*.xml"));
        sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
        return sqlSessionFactoryBean.getObject();
    }
}

```

##### 编写对应的CURD操作，通过创建订单并扣账户余额并扣库存
```
@Autowired
    private OrderDao orderDao;

    @Autowired
    private RestTemplate restTemplate;

    @GlobalTransactional(name = "test-create-order",rollbackFor = Exception.class)
    public void create(Order order) {
        System.out.println("--------->下单开始");
        orderDao.create(order);

        System.out.println("--------->扣库存开始");
        //扣库存
        restTemplate.getForObject(
                "http://seata-storage-service/storage/decrease?productId="
                        + order.getProductId() + "&count=" + order.getCount()
                , String.class);
        System.out.println("--------->扣库存结束");
        //扣余额
        System.out.println("--------->扣余额开始");
        restTemplate.getForObject("http://seata-account-service/account/decrease?userId=" + order.getUserId() +
                "&money=" + order.getMoney(), String.class);
        System.out.println("--------->扣余额结束");

        System.out.println("--------->下单结束");
    }
```

##### 测试
```
调用http://localhost:8602/order/create?userId=1&productId=1&count=10&money=100
```

##### account服务或者storage服务制造异常观察数据库变化
undo_log中会有对应日志
![](https://f2130793.github.io/images/2020-07-22-9.jpg)

#### 源码地址
```
https://github.com/f2130793/SpringCloudLearning
```