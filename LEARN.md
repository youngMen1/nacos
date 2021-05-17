# Nacos

引用官方的介绍，他主要提供以下几个功能点：

* 动态配置服务

就是通过一个系统，管理系统中的配置项，在配置项需要更新的时候，可以通过管理系统去操作更新， 更新完了之后，会主动推送到订阅了这个配置的客户端 具体的使用场景，例如，在系统中，会有数据库的链接串，账号密码等配置信息，
常规的做法是写在配置文件里面，如果需要修改更新，需要重新打包编译， 如果你是分布式集群，那成本太大了，通常我们都会将它抽取出来，存放到db， 或者一个管理系统，Nacos，就是这个管理系统，Nacos还提供主动通知的能力，
DB是没有的，在自己的系统代码里面，可以监听某个配置，如果在管理系统上修改了配置项， 客户端的监听函数，会立刻执行，在里面，你可以拿到最新的配置，执行自己的业务逻辑

* 服务发现及管理

这个主要是针对分布式的微服务集群系统，某A集群提供服务出去，其他应用集群， 需要消费到A集群的服务，需要一个系统去管理A集群的ip列表，其他应用集群， 去这个系统才能获取到A集群的ip列表，进行调用，
同时该系统需要能够自动将A集群中无法工作的ip进行去除掉， 这样才能保证调用方调用成功，Nacos就是提供这种能力的一个系统

* 动态DNS服务

这个理解起来也简单，我们平常在代码里面 ，访问一个http的api， 通常是带一个域名的，请求的时候，一般会先去DNS域名服务器上面寻找该域名对应的ip， 再发起http请求，Nacos可以充当这个DNS域名服务器的角色的，优点是什么呢？
Nacos提供了比DNS域名服务器更方便的管理能力，新增一个域名， 只需要在Nacos的控制台上面配置一下，同时它还提供了包括权重， 健康检查，属性，路由等DNS服务器不具备的能力，比DNS的产品功能， 稳定性，灵活性，超出太多了

## 说明

Nacos的工程，包含了Nacos Server和Nacos Client2个工程的代码

Nacos Server:由配置和服务发现2个大功能，分别由config和naming承载

## 源码说明

### Nacos 源码分析 —— 如何做心跳续约

http心跳请求

NamingHttpClientProxy ->BeatReactor-> BeatTask

### Nacos 源码分析 —— 如何做健康检查

服务发现如何做心跳检查

#### 方式一 tcp

创建连接，500ms后看连接是否创建成功: TimeOutTask

TcpSuperSenseProcessor->call()->TimeOutTask

#### 方式二 http

发一个http请求，看返回码是不是200

HttpHealthCheckProcessor->process()->HttpHealthCheckCallback

#### 方式三 mysql

看sql请求是不是抛异常

MysqlHealthCheckProcessor->process()

#### Nacos 源码分析 —— Raft 如何心跳保持
1.raft的一个基本逻辑是leader隔一段时间给所有的follower发心跳。如果follower长时间没收到心跳，
就认为leader已经挂了，就发起投票选举新的leader。
    * HeartBeat 就是leader的心跳定时任务
    * MasterElection 就是follower长时间没收到心跳就选举的定时任务
2.HeartBeat的sendBeat就是具体发送心跳信息了
    * follower收到心跳请求的时候(/beat())
3.receivedBeat 方法会执行 resetLeaderDue();
4.MasterElection里面follower就是根据这个变量判断是否要重新选leader的。

RaftCore->init()->/beat->receivedBeat()->MasterElection

### Nacos 源码分析 —— Raft 如何选举
1.发起请求：
2.其他节点收到选举请求
3.如果对方的term比自己小，voteFor为自己，然后返回结果。意思是我自己更适合做leader，这一票我投给自己。
4.如果对方的term比自己大，设置voteFor为对方，然后返回结果，意思是就按你说的做，这一票就投给你了。
5.把所有的节点投票信息放到TreeBag，这个可以看成是个按value排序的有序map。排第一的就是得票最多的节点
6.假如一个节点选举自己成功，他会认为自己是leader，就会定时发送心跳给其他的节点，这个时候其他节点的leader还是旧的，收到心跳会报错的。
7.所以其他节点都经历一次选举：
8.因为已经选举成功过，所以local.voteFor都有值，为上一次选举成功的节点，所以其他节点选举的结果都会统一了。
9.但是这里有个关键逻辑就是term的比较，这个是决定了所有的逻辑的。
10.假如节点2开始选举，它的term是最高的，选举自己是可以成功的。
11.假如节点2和节点3同时选举呢，节点2得到自己和节点4的票，节点3得到自己和节点5的票。这个时候两边都不能成功。所以等待下一轮，因为下一次开始的时间是随机的，所以同时的概率很小。谁先，谁就是新的leader了。
12.假如所有的节点的term相同，其实是选举不出leader的，因为都只有自己一票。这个是怎么解决的呢？
13.收到投票请求的时候，如果对方的term比自己的大，为什么要放弃这一轮的发起选举
14.为什么每次发布新的内容，term都会加100呢

RaftCore->init()->/beat->receivedBeat()->MasterElection->/vote