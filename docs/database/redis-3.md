---
title: 【Redis】Redis集群环境的设计与搭建、主从复制、哨兵模式
date: 2021-12-28 22:56:12
tags:
- Database
- Redis
---

## 十二、主从复制

### 基本介绍

#### 互联网的“三高”特性

高并发、高性能、**高可用**

其中，什么是高可用？假设有一处的服务器，一年之中的宕机时间为866467秒。

![image-20210531151105863](https://img-blog.csdnimg.cn/img_convert/197e2c59c438ede485bd9989d8e33e9d.png)

**判断Redis是否高可用**

单机redis的风险与问题

- 问题1：机器故障
  - 现象：硬盘故障、系统崩溃
  - 本质：数据丢失，很可能对业务造成灾难性打击
  - 结论：基本上会放弃使用redis
- 问题2：容量瓶颈
  - 现象：内存不足，从16G升级到64G，从64G升级到128G，无限升级内存
  - 本质：穷，硬件条件跟不上
  - 结论：放弃使用redis
- 结论： 为了避免单点Redis服务器故障，准备多台服务器，互相连通。将数据复制多个副本保存在不同的服务器上，连接在一起，并保证数据是同步的。即使有其中一台服务器宕机，其他服务器依然可以继续 提供服务，实现Redis的高可用，同时实现**数据冗余备份**。

#### 多台服务器连接方案

- 提供数据方：master
  - 主服务器，主节点，主库（主客户端）
- 接收数据方：slave
  - 从服务器，从节点，从库（从客户端）
- 需要解决的问题： 数据同步
- 核心工作： **master的数据复制到slave中**

![image-20210531151942161](https://img-blog.csdnimg.cn/img_convert/1920a90c9788fed6355bbe5ce91c8d94.png)

其中，为了避免数据产生大大小小的问题，我们需要对多台服务器的连接方案进行一些规定限制

- 主从复制即将master中的数据即时、有效的复制到slave中
- 特征：一个master可以拥有多个slave，一个slave只对应一个master
- 职责：
  - master:
    - 写数据
    - 执行写操作时，将出现变化的数据自动同步到slave
    - 读数据（可忽略）
  - slave:
    - 读数据
    - 写数据（禁止）

#### 高可用集群

![image-20210531152453098](https://img-blog.csdnimg.cn/img_convert/291889f858f8fc2bad4c784d0f1ad6bf.png)

#### 主从复制的作用

- 读写分离：master写、slave读，提高服务器的读写负载能力
- 负载均衡：基于主从结构，配合读写分离，由slave分担master负载，并根据需求的变化，改变slave的数量，通过多个从节点分担数据读取负载，大大提高Redis服务器并发量与数据吞吐量
- 故障恢复：当master出现问题时，由slave提供服务，实现快速的故障恢复
- 数据冗余：实现数据热备份，是持久化之外的一种数据冗余方式
- 高可用基石：基于主从复制，构建哨兵模式与集群，实现Redis的高可用方案

### 工作流程

总述：大致上分为三个步骤

- 建立连接阶段（即准备阶段）
- 数据同步阶段
- 命令传播阶段

![image-20210531153020881](https://img-blog.csdnimg.cn/img_convert/4c43d58dc1998052e4674d976d5b10b0.png)

#### 建立连接

- 步骤1：设置master的地址和端口，保存master信息
- 步骤2：建立socket连接
- 步骤3：发送ping命令（定时器任务）
- 步骤4：身份验证
- 步骤5：发送slave端口信息 至此，主从连接成功！
- 状态：
  - slave： 保存master的地址与端口
  - master： 保存slave的端口
- 总体： 之间创建了连接的socket

![image-20210531153652475](https://img-blog.csdnimg.cn/img_convert/3b18785535cca2c5164cda6a4025fb3a.png)

#### 阶段一、主从连接（slave连接master）

方式一：客户端发送命令

```
slaveof <masterip> <masterport>
```

方式二：启动服务器参数

```
redis-server -slaveof <masterip> <masterport>
```

方式三：服务器配置

```
slaveof <masterip> <masterport>
```

slave系统信息

```
master_link_down_since_seconds
masterhost
masterport 
```

master系统信息

```
slave_listening_port(多个)
```

现在的启动方式一般不直接在从服务器上敲连接主服务器的命令，而是直接在从服务器的配置文件中添加对应的连接配置信息redis-6380.conf

> 关闭守护进程、关闭日志文件的输出

```
slaveof 127.0.0.1 6379
```

![image-20210531155505870](https://img-blog.csdnimg.cn/img_convert/99208f9242d7ced9e3c18cbbe139352c.png)

连接成功之后，可以在服务器的启动页面看到对应的连接master的相关信息

![image-20210531155735351](https://img-blog.csdnimg.cn/img_convert/8092bc718a2df0879e7f15b5af1bd964.png)

这样九简单完成了主从服务器的搭建，根据业务的建议逻辑，我们只在master中写入数据，而在slave中读取数据，这样可以有效地避免数据的同步的一些问题。

![image-20210531160122937](https://img-blog.csdnimg.cn/img_convert/6d59800526b7ead85f0a21a1785beb13.png)

![image-20210531160133593](https://img-blog.csdnimg.cn/img_convert/a8b3d65e866a8b6f044237c60da793f0.png)

这样配置，就可以完成上述的操作了

在服务器端输入`info`指令，可以看到关于连接的一些信息

**其他配置**

客户端发送命令

```
slaveof no one
```

说明： slave断开连接后，不会删除已有数据，只是不再接受master发送的数据

因为主服务器上可能存在多个连接，所以断开连接只能是在从服务器这一侧断开，也就是在从服务器的客户端发送断开指令，由主服务器接受指令之后，执行断开操作

**授权访问**

master客户端发送命令设置密码

```
config get requirepass  requirepass
```

master配置文件设置密码

```
config set requirepass
```

slave客户端发送命令设置密码

```
auth <password>
```

slave配置文件设置密码

```
masterauth <password>
```

slave启动服务器设置密码

```
redis-server –a <password>
```

#### 阶段二、数据同步阶段（master和slave数据同步）

**数据同步的过程目的分析**

- 在slave初次连接master后，复制master中的所有数据到slave
- 将slave的数据库状态更新成master当前的数据库状态

![image-20210531161531762](https://img-blog.csdnimg.cn/img_convert/7d27ee2e0e873e5d5317c223bc3a11aa.png)

数据同步整个过程为上图的1-8条，分为全量复制和部分复制（增量复制）

全量复制，主要复制的是master中从开始到连接状态完成之间的数据部分；这一部分主要使用的是RDB的形式恢复数据。

增量复制，主要复制的是全量复制过程中master可能产生的新数据，这些数据一般数据量不是非常大，主要利用AOF形式进行数据备份和恢复，这样更符合现实情况。这一阶段产生的新数据放在复制缓冲区内。

- 步骤1：请求同步数据
- 步骤2：创建RDB同步数据
- 步骤3：恢复RDB同步数据
- 步骤4：请求部分同步数据
- 步骤5：恢复部分同步数据
- 至此，数据同步工作完成！
- 状态：
  - slave： 具有master端全部数据，包含RDB过程接收的数据
  - master： 保存slave当前数据同步的位置
- 总体： 之间完成了数据克隆

**注意事项（master说明）**

1. 如果master数据量巨大，数据同步阶段应避开流量高峰期，避免造成master阻塞，影响业务正常执行

2. 复制缓冲区大小设定不合理，会导致数据溢出。如进行全量复制周期太长，进行部分复制时发现数据已经存在丢失的情况，必须进行第二次全量复制，致使slave陷入死循环状态。

   ```
   repl-backlog-size 1mb
   ```

3. master单机内存占用主机内存的比例不应过大，建议使用50%-70%的内存，留下30%-50%的内存用于执 行bgsave命令和创建复制缓冲区

这里涉及到的问题主要是复制缓冲区的内存大小，设置不合理要么导致全量复制阶段产生的新数据部分丢失（后产生的数据溢出就会挤掉最先产生的数据），要么导致还速度下降（复制缓冲区内存过大，导致分配给其他部分的内存太小，影响IO性能）

![image-20210531162933595](https://img-blog.csdnimg.cn/img_convert/65a94f35719900c82edfa7efee302888.png)

**注意事项（slave后期）**

1. 为避免slave进行全量复制、部分复制时服务器响应阻塞或数据不同步，建议关闭此期间的对外服务、

   ```
   slave-serve-stale-data yes|no
   ```

2. 数据同步阶段，master发送给slave信息可以理解master是slave的一个客户端，主动向slave发送命令

3. 多个slave同时对master请求数据同步，master发送的RDB文件增多，会对带宽造成巨大冲击，如果 master带宽不足，因此数据同步需要根据业务需求，适量错峰

4. slave过多时，建议调整拓扑结构，由一主多从结构变为树状结构，中间的节点既是master，也是 slave。注意使用树状结构时，由于层级深度，导致深度越高的slave与最顶层master间数据同步延迟 较大，*数据一致性变差*，应谨慎选择

这里的问题主要是：根据设计原理，关闭slave的对外写数据服务；slave数量过多时候如何处理slave的数据复制问题。

#### 阶段三：命令传播阶段

当master数据库状态被修改后，导致主从服务器数据库状态不一致，此时需要让主从数据同步到一致的 状态，同步的动作称为命令传播

master将接收到的数据变更命令发送给slave，slave接收命令后执行命令

主从复制过程大体可以分为3个阶段

1. 建立连接阶段（即准备阶段）
2. 数据同步阶段
3. 命令传播阶段

**命令传播阶段的部分复制**

命令传播阶段出现了断网现象

- 网络闪断闪连		忽略
- 短时间网络中断	部分复制
- 长时间网络中断	全量复制

部分复制的三个核心要素

- 服务器的运行 id（run id）

  ![image-20210531163605924](https://img-blog.csdnimg.cn/img_convert/7a578d4798b24da753478fd1ae19d122.png)

- 主服务器的复制积压缓冲区

- 主从服务器的复制偏移量

**服务器运行ID（runid）**

- 概念：服务器运行ID是每一台服务器每次运行的身份识别码，一台服务器多次运行可以生成多个运行id
- 组成：运行id由40位字符组成，是一个随机的十六进制字符；例如：fdc9ff13b9bbaab28db42b3d50f852bb5e3fcdce
- 作用：运行id被用于在服务器间进行传输，识别身份 如果想两次操作均对同一台服务器进行，必须每次操作携带对应的运行id，用于对方识别
- 实现方式：运行id在每台服务器启动时自动生成的，master在首次连接slave时，会将自己的运行ID发送给slave，slave保存此ID，通过info Server命令，可以查看节点的runid

**复制缓冲区**

- 概念：复制缓冲区，又名复制积压缓冲区，是一个先进先出（FIFO）的队列，用于存储服务器执行过的命令，每次传播命令，master都会将传播的命令记录下来，并存储在复制缓冲区
  - 复制缓冲区默认数据存储空间大小是1M，由于存储空间大小是固定的，当入队元素的数量大于队列长度时，最先入队的元素会被弹出，而新元素会被放入队列
- 由来：每台服务器启动时，如果开启有AOF或被连接成为master节点，即创建复制缓冲区
- 作用：用于保存master收到的所有指令（仅影响数据变更的指令，例如set，select）
- 数据来源：当master接收到主客户端的指令时，除了将指令执行，会将该指令存储到缓冲区中

内部工作原理：

![image-20210531164646455](https://img-blog.csdnimg.cn/img_convert/51eb156981ab136d7fa9df43f954389f.png)

**主从服务器复制偏移量（offset）**

- 概念：一个数字，描述复制缓冲区中的指令字节位置
- 分类：
  - master复制偏移量：记录发送给所有slave的指令字节对应的位置（多个）
  - slave复制偏移量：记录slave接收master发送过来的指令字节对应的位置（一个）
- 数据来源：
  - master端：发送一次记录一次
  - slave端：接收一次记录一次
- 作用：同步信息，比对master与slave的差异，当slave断线后，恢复数据使用

#### 数据同步+命令传播阶段工作流程

![image-20210531165411947](https://img-blog.csdnimg.cn/img_convert/54f6cad86aab6e25fe622bc2fc794f4e.png)

#### 心跳机制

- 进入命令传播阶段候，master与slave间需要进行信息交换，使用心跳机制进行维护，实现双方连接保持在线
- master心跳：
  - 指令：PING
  - 周期：由repl-ping-slave-period决定，默认10秒
  - 作用：判断slave是否在线
  - 查询：INFO replication 获取slave最后一次连接时间间隔，lag项维持在0或1视为正常
- slave心跳任务
  - 指令：REPLCONF ACK {offset}
  - 周期：1秒
  - 作用1：汇报slave自己的复制偏移量，获取最新的数据变更指令
  - 作用2：判断master是否在线

**注意事项**

- 当slave多数掉线，或延迟过高时，master为保障数据稳定性，将拒绝所有信息同步操作

  - ```
    min-slaves-to-write 2
    min-slaves-max-lag 8
    ```

  - slave数量少于2个，或者所有slave的延迟都大于等于10秒时，强制关闭master写功能，停止数据同步

- slave数量由slave发送REPLCONF ACK命令做确认

- slave延迟由slave发送REPLCONF ACK命令做确认

**主从复制工作流程**

![image-20210531170127664](https://img-blog.csdnimg.cn/img_convert/b590f9dfbf30a2d6e12908e11d6e759c.png)

### 常见问题

**频繁的全量复制（1）**

伴随着系统的运行，master的数据量会越来越大，一旦master重启，runid将发生变化，会导致全部slave的全量复制操作

- 内部优化调整方案：
  - 1. master内部创建master_replid变量，使用runid相同的策略生成，长度41位，并发送给所有slave
  2. 在master关闭时执行命令 shutdown save，进行RDB持久化，将runid与offset保存到RDB文件中
    - repl-id repl-offset
    - 通过redis-check-rdb命令可以查看该信息
  3. master重启后加载RDB文件，恢复数据重启后，将RDB文件中保存的repl-id与repl-offset加载到内存中
    - master_repl_id = repl master_repl_offset = repl-offset
    - 通过info命令可以查看该信息
- 作用： 本机保存上次runid，重启后恢复该值，使所有slave认为还是之前的master

**频繁的全量复制（2）**

问题现象：网络环境不佳，出现网络中断，slave不提供服务

问题原因：复制缓冲区过小，断网后slave的offset越界，触发全量复制

最终结果：slave反复进行全量复制

解决方案：修改复制缓冲区大小

```
repl-backlog-size
```

建议设置如下：

 	1. 测算从master到slave的重连平均时长second
 	2. 获取master平均每秒产生写命令数据总量write_size_per_second
 	3. 最优复制缓冲区空间 = 2 * second * write_size_per_second

**频繁的网络中断（1）**

问题现象：master的CPU占用过高 或 slave频繁断开连接

问题原因：

- slave每1秒发送REPLCONF ACK命令到master
- 当slave接到了慢查询时（keys * ，hgetall等），会大量占用CPU性能
- master每1秒调用复制定时函数replicationCron()，比对slave发现长时间没有进行响应

最终结果：master各种资源（输出缓冲区、带宽、连接等）被严重占用

解决方案：通过设置合理的超时时间，确认是否释放slave 该参数定义了超时时间的阈值（默认60秒），超过该值，释放slave

**频繁的网络中断（2）**

问题现象：slave与master连接断开

问题原因：

- master发送ping指令频度较低
- master设定超时时间较短
- ping指令在网络中存在丢包

解决方案：

提高ping指令发送的频度

```
repl-ping-slave-period
```

超时时间repl-time的时间至少是ping指令频度的5到10倍，否则slave很容易判定超时

**数据不一致**

问题现象：多个slave获取相同数据不同步

问题原因：网络信息不同步，数据发送有延迟

解决方案：

- 优化主从间的网络环境，通常放置在同一个机房部署，如使用阿里云等云服务器时要注意此现象

- 监控主从节点延迟（通过offset）判断，如果slave延迟过大，暂时屏蔽程序对该slave的数据访问

  - ```
    slave-serve-stale-data yes|no
    ```

  - 开启后仅响应info、slaveof等少数命令（慎用，除非对数据一致性要求很高）

## 十三、哨兵模式

### 哨兵简介

master宕机该如何解决？

- 关闭期间的数据服务谁来承接？
- 找一个主？怎么找法？
- 修改配置后，原始的主恢复了怎么办？

出现的情况

- 关闭master和所有slave
- 找一个slave作为master
- 修改其他slave的配置，连接新的主
- 启动新的master与slave
- `全量复制*N+部分复制*N`

**定义**

哨兵(sentinel) 是一个分布式系统，用于对主从结构中的每台服务器进行**监控**，当出现故障时通过投票机制**选择**新的master并将所有slave连接到新的master。

![image-20210531173012265](https://img-blog.csdnimg.cn/img_convert/98ac28107d893422445d9b8cce3231ef.png)

**作用**

- 监控
  - 不断的检查master和slave是否正常运行。 master存活检测、master与slave运行情况检测
- 通知（提醒）
  - 当被监控的服务器出现问题时，向其他（哨兵间，客户端）发送通知。
- 自动故障转移
  - 断开master与slave连接，选取一个slave作为master，将其他slave连接到新的master，并告知客户端新的服务器地址
- 注意
  - 哨兵也是一台redis服务器，只是不提供数据服务通常哨兵配置数量为**单数**（防止竞选中打平）

### 启用哨兵模式

**配置哨兵**

- 配置一拖二的主从结构

- 配置三个哨兵（配置相同，端口不同）

  - 参看sentinel.conf
  - ![image-20210531173701117](https://img-blog.csdnimg.cn/img_convert/be1140bc6ecbda9a298cdbd8452e7c21.png)

- 启动哨兵

  - ```
    redis-sentinel sentinel-端口号.conf
    ```

| 配置项                                                       | 范例                                              | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------ |
| sentinel auth-pass  <服务器名称>                             | sentinel auth-pass mymaster itcast                | 连接服务器口令                                               |
| sentinel down-after-milliseconds <自定义服 务名称><主机地址><端口><主从服务器总量> | sentinel monitor mymaster  192.168.194.131 6381 1 | 设置哨兵监听的主服务器信息，最后的参数决定了最终参与选举的服务器 数量（-1） |
| sentinel down-after-milliseconds  <服务名称><毫秒数（整数）> | sentinel down-after-milliseconds mymaster 3000    | 指定哨兵在监控Redis服务时，判定服务器挂掉的时间周期，默认30秒 （30000），也是主从切换的启动条件之一 |
| sentinel parallel-syncs  <服务名称><服务器数（整数）>        | sentinel parallel-syncs  mymaster 1               | 指定同时进行主从的slave数量，数值越大，要求网络资源越高，要求约 小，同步时间约长 |
| sentinel failover-timeout  <服务名称><毫秒数（整数）>        | sentinel failover-timeout  mymaster 9000          | 指定出现故障后，故障切换的最大超时时间，超过该值，认定切换失败， 默认3分钟 |
| sentinel notification-script  <服务名称><脚本路径>           |                                                   | 服务器无法正常联通时，设定的执行脚本，通常调试使用           |

### 哨兵工作原理

**主从切换**

哨兵在进行主从切换过程中经历三个阶段

- 监控
- 通知
- 故障转移

#### 阶段一：监控阶段

用于同步各个节点的状态信息

- 获取各个sentinel的状态（是否在线）
- 获取master的状态
  - master属性
    - runid
    - role：master
  - 各个slave的详细信息
- 获取所有slave的状态（根据master中的slave信息）
  - slave属性
    - runid
    - role：slave
    - master_host、master_port
    - offset
    - ……

![image-20210531175409360](https://img-blog.csdnimg.cn/img_convert/7010cb2c9caaab26c442d0ee68152dad.png)

创建哨兵之后，哨兵向master和slave发送相关的指令获取到对应的信息。只要其中有一个哨兵获取到新的信息，就会使用专门的通信信道在所有sentinel中传播，相当于以广播的方式传播到所有sentinel

![image-20210531180019874](https://img-blog.csdnimg.cn/img_convert/8b5db18cf60076e3bf2585aa2a7530a7.png)

#### 阶段二：通知阶段

维护一个长期的信息对等的阶段

![image-20210531180138581](https://img-blog.csdnimg.cn/img_convert/68d36ae46d202d6e05617bf5f9ffd089.png)

#### 阶段三：故障转移阶段

![image-20210531180247374](https://img-blog.csdnimg.cn/img_convert/b22ce0f217198188e4b4c4515772fba6.png)

超过半数的sentinel判断出服务器宕机，则标记为这个服务器客观下线。主观下线则是部分（不到半数）认为服务器宕机。

这个时候sentinel们就要派出一个代表来处理故障，派出的sentinel该怎么选择？

![image-20210531180825865](https://img-blog.csdnimg.cn/img_convert/57a337e5130b6916765b0c998f1c92d5.png)

通过投票机制，选出代表去处理故障问题。为了防止票数相同等问题，我们在一开始就设置sentinel的数量为单数；如果经过一轮投票没有选出来，则开始第二轮投票，直到选出代表的sentinel。

- 服务器列表中挑选备选master
  - 在线的
  - 响应慢的
  - 与原master断开时间久的
  - 优先原则
    - 优先级
    - offset
    - runid
- 发送指令（ sentinel ）
  - 向新的master发送slaveof no one
  - 向其他slave发送slaveof新masterIP端口

**总述**

- 监控
  - 同步信息
- 通知
  - 保持联通
- 故障转移
  - 发现问题
  - 竞选负责人
  - 优选新master
  - 新master上任，其他slave切换master，原master作为slave故障回复后连接

## 十四、集群

### 集群简介

**业务发展遇到的瓶颈问题**

- redis提供的服务OPS可以达到10万/秒，当前业务OPS已经达到10万/秒
- 内存单机容量达到256G，当前业务需求内存容量1T
- 使用集群的方式可以快速解决上述问题

**集群架构**

集群就是使用网络将若干台计算机联通起来，并提供统一的管理方式，使其对外呈现单机的服务效果

**集群作用**

- 分散单台服务器的访问压力，实现**负载均衡**
- 分散单台服务器的存储压力，实现**可扩展性**
- 降低单台服务器宕机带来的业务灾难

![image-20210531182839372](https://img-blog.csdnimg.cn/img_convert/e24f5ecfa8e4e3c52dab12aff196b696.png)

### Redis集群结构设计

**数据存储设计**

- 通过算法设计，计算出key应该保存的位置
- 将所有的存储空间计划切割成16384份，每台主机保存一部分
  - 每份代表的是一个存储空间，不是一个key的保存空间
-    将key按照计算出的结果放到对应的存储空间

![image-20210531190410315](https://img-blog.csdnimg.cn/img_convert/96def6b93c923b49537b1eb6b83ed95a.png)

其中用到了循环冗余校验（CRC16），用这个计算出数据应该存放的位置，之后再将数据存放到目的位置。

**增强可扩展性**

![image-20210531190615849](https://img-blog.csdnimg.cn/img_convert/1847656cdcadd07bb635e32f35f0a357.png)

**集群内部通讯设计**

- 各个数据库相互通信，保存各个库中槽的编号数据
- 一次命中，直接返回
- 一次未命中，告知具体位置

![image-20210531190758428](https://img-blog.csdnimg.cn/img_convert/ddb7b10b46866d6013792da062bdb852.png)

### cluster集群结构搭建

**搭建方式**

- 原生安装（单条命令）
  - 配置服务器（3主3从）
  - 建立通信（Meet）
  - 分槽（Slot）
  - 搭建主从（master-slave）
- 工具安装（批处理）

**Cluster配置**

添加节点

```
cluster-enabled yes|no
```

cluster配置文件名，该文件属于自动生成，仅用于快速查找文件并查询文件内容

```
cluster-config-file <filename>
```

节点服务响应超时时间，用于判定该节点是否下线或切换为从节点

```
cluster-node-timeout <milliseconds>
```

master连接的slave最小数量

```
cluster-migration-barrier <count>
```

**Cluster节点操作命令**

查看集群节点信息

```
cluster nodes
```

进入一个从节点 redis，切换其主节点

```
cluster replicate <master-id>
```

发现一个新节点，新增主节点

```
cluster meet ip:port
```

忽略一个没有solt的节点

```
cluster forget <id>
```

手动故障转移

```
cluster failover
```

**redis-trib命令**

添加节点

```
redis-trib.rb add-node
```

删除节点

```
redis-trib.rb del-node
```

重新分片

```
redis-trib.rb reshard
```

**实践搭建**

一、在原先的redis-6379.conf文件中添加对应的配置

![image-20210531191543452](https://img-blog.csdnimg.cn/img_convert/ed6f9b70d3b651565b904dea4de19178.png)

二、将原先的6379的配置文件分别复制五个，生成新的文件

![image-20210531191633549](https://img-blog.csdnimg.cn/img_convert/938256be018680434748f19e3d8bd497.png)

三、数据上的存储会对应哪个端口，对应存放在对应的槽；取数据的时候要针对cluster做特别的适配操作。

在客户端连接的时候使用指令

```
redis-cli -c
```

表示针对cluster进行操作，这样就可以在原数据存放的位置之外的端口客户端访问到。（重定向）

## 十五、企业级解决方案

### 缓存预热

**业务场景**

服务器启动后迅速“宕机”

1. 请求数量较高
2. 主从之间数据吞吐量较大，数据同步操作频度较高

**解决方案**

前置准备工作：

 	1. 日常例行统计数据访问记录，统计访问频度较高的热点数据
 	2. 利用LRU数据删除策略，构建数据留存队列。例如：storm与kafka配合

准备工作：

 	1. 将统计结果中的数据分类，根据级别，redis优先加载级别较高的热点数据
 	2. 利用分布式多服务器同时进行数据读取，提速数据加载过程
 	3. 热点数据主从同时预热

实施：

 	1. 使用脚本程序固定触发数据预热过程
 	2. 如果条件允许，使用了CDN（内容分发网络），效果更好

**总结**

缓存预热就是系统启动前，提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题，用户直接查询事先被预热的缓存数据。

### 缓存雪崩

**业务场景**

数据服务器崩溃（1）

1. 系统平稳运行过程中，忽然数据库连接量激增
2. 应用服务器无法及时处理请求
3. 大量408，500错误页面出现
4. 客户反复刷新页面获取数据
5. 数据库崩溃
6. 应用服务器崩溃
7. 重启应用服务器无效
8. Redis服务器崩溃
9. Redis集群崩溃
10. 重启数据库后再次被瞬间流量放倒

**问题排查**

1. 在一个较短的时间内，缓存中较多的key集中过期
2. 此周期内请求访问过期的数据，redis未命中，redis向数据库获取数据
3. 数据库同时接收到大量的请求无法及时处理
4. Redis大量请求被积压，开始出现超时现象
5. 数据库流量激增，数据库崩溃
6. 重启后仍然面对缓存中无数据可用
7. Redis服务器资源被严重占用，Redis服务器崩溃
8. Redis集群呈现崩塌，集群瓦解
9. 应用服务器无法及时得到数据响应请求，来自客户端的请求数量越来越多，应用服务器崩溃
10. 应用服务器，redis，数据库全部重启，效果不理想

**问题分析**

短时间范围内，大量key集中过期

**解决方案一**

1. 更多的页面静态化处理
2. 构建多级缓存架构 Nginx缓存+redis缓存+ehcache缓存
3. 检测Mysql严重耗时业务进行优化 对数据库的瓶颈排查：例如超时查询、耗时较高事务等
4. 灾难预警机制 监控redis服务器性能指标
  - CPU占用、CPU使用率
  - 内存容量
  - 查询平均响应时间
  - 线程数
5. 限流、降级 短时间范围内牺牲一些客户体验，限制一部分请求访问，降低应用服务器压力，待业务低速运转后再逐步放开访问

**解决方案二**

1. LRU与LFU切换
2. 数据有效期策略调整
  - 根据业务数据有效期进行分类错峰，A类90分钟，B类80分钟，C类70分钟
  - 过期时间使用固定时间+随机值的形式，稀释集中到期的key的数量
3. 超热数据使用永久key
4. 定期维护（自动+人工） 对即将过期数据做访问量分析，确认是否延时，配合访问量统计，做热点数据的延时
5. 加锁 （慎用！）

**总结**

缓存雪崩就是瞬间过期数据量太大，导致对数据库服务器造成压力。如能够有效避免过期时间集中，可以有效解决雪崩现象的出现（约40%），配合其他策略一起使用，并监控服务器的运行数据，根据运行记录做快速调整。

### 缓存击穿

**数据库服务器崩溃（2）**

 	1. 系统平稳运行过程中
 	2. 数据库连接量瞬间激增
 	3. Redis服务器无大量key过期
 	4. Redis内存平稳，无波动
 	5. Redis服务器CPU正常
 	6. 数据库崩溃

**问题排查**

1. Redis中某个key过期，该key访问量巨大
2. 多个数据请求从服务器直接压到Redis后，均未命中
3. Redis在短时间内发起了大量对数据库中同一数据的访问

**问题分析** ：单个key高热数据、key过期

**解决方案**

1. 预先设定 以电商为例，每个商家根据店铺等级，指定若干款主打商品，在购物节期间，加大此类信息key的过期时长 注意：购物节不仅仅指当天，以及后续若干天，访问峰值呈现逐渐降低的趋势
2. 现场调整 监控访问量，对自然流量激增的数据延长过期时间或设置为永久性key
3. 后台刷新数据 启动定时任务，高峰期来临之前，刷新数据有效期，确保不丢失
4. 二级缓存 设置不同的失效时间，保障不会被同时淘汰就行
5. 加锁 分布式锁，防止被击穿，但是要注意也是性能瓶颈，慎重！

**总结**

缓存击穿就是单个高热数据过期的瞬间，数据访问量较大，未命中redis后，发起了大量对同一数据的数据库访问，导致对数据库服 务器造成压力。应对策略应该在业务数据分析与预防方面进行，配合运行监控测试与即时调整策略，毕竟单个key的过期监控难度较高，配合雪崩处理策略即可

### 缓存穿透

**数据库服务器崩溃（3）**

1. 系统平稳运行过程中
2. 应用服务器流量随时间增量较大
3. Redis服务器命中率随时间逐步降低
4. Redis内存平稳，内存无压力
5. Redis服务器CPU占用激增
6. 数据库服务器压力激增
7. 数据库崩溃

**问题排查**

1. Redis中大面积出现未命中
2. 出现非正常URL访问

**问题分析**

- 获取的数据在数据库中也不存在，数据库查询未得到对应数据
- Redis获取到null数据未进行持久化，直接返回
- 下次此类数据到达重复上述过程
- 出现黑客攻击服务器

**解决方案**

1. 缓存null

  - 对查询结果为null的数据进行缓存（长期使用，定期清理），设定短时限，例如30-60秒，最高5分钟

2. 白名单策略

  - 提前预热各种分类数据id对应的bitmaps，id作为bitmaps的offset，相当于设置了数据白名单。当加载正常数据时，放行，加载异常数据时直接拦截（效率偏低）
  - 使用布隆过滤器（有关布隆过滤器的命中问题对当前状况可以忽略）

3. 实施监控

  - 实时监控redis命中率（业务正常范围时，通常会有一个波动值）与null数据的占比

  - 非活动时段波动：通常检测3-5倍，超过5倍纳入重点排查对象
  - 活动时段波动：通常检测10-50倍，超过50倍纳入重点排查对象
  - 根据倍数不同，启动不同的排查流程。然后使用黑名单进行防控（运营）

4. key加密

  - 问题出现后，临时启动防灾业务key，对key进行业务层传输加密服务，设定校验程序，过来的key校验 例如每天随机分配60个加密串，挑选2到3个，混淆到页面数据id中，发现访问key不满足规则，驳回数据访问

**总结**

缓存击穿访问了不存在的数据，跳过了合法数据的redis数据缓存阶段，每次访问数据库，导致对数据库服务器造成压力。通常此类数据的出现量是一个较低的值，当出现此类情况以毒攻毒，并及时报警。应对策略应该在临时预案防范方面多做文章。 无论是黑名单还是白名单，都是对整体系统的压力，警报解除后尽快移除。

### 性能指标监控

**监控指标**

- 性能指标：Performance
- 内存指标：Memory
- 基本活动指标：Basic activity
- 持久性指标：Persistence
- 错误指标：Error

**监控方式**

- 工具
  - Cloud Insight Redis
  - Prometheus
  - Redis-stat
  - Redis-faina
  - RedisLive
  - zabbix
- 命令
  - benchmark
  - redis cli
  - monitor
  - showlog

**benchmark**

命令

```
redis-benchmark [-h ] [-p ] [-c ] [-n  [-k ]
```

范例1

```
redis-benchmark
```

说明：50个连接，10000次请求对应的性能

范例2

```
redis-benchmark -c 100 -n 5000
```

说明：100个连接，5000次请求对应的性能

**monitor**

命令

```
monitor
```

打印服务器调试信息

**showlong**

命令

```
showlong [operator]
```

- get ：获取慢查询日志
- len ：获取慢查询日志条目数
- reset ：重置慢查询日志

相关配置

```
slowlog-log-slower-than 1000  #设置慢查询的时间下线，单位：微妙
slowlog-max-len 100 #设置慢查询命令对应的日志显示长度，单位：命令数
```

