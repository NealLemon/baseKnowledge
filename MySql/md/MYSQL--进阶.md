# MYSQL--进阶

## MYSQL主从复制

#### 类别

- 基于日志点的复制
  - 支持MMM和MHA架构
- 基于GTID方式的复制
  - GTID= source_id:transaction_id
  - Slave增量同步Master的数据依赖于其未同步的事务ID
  - 支持MHA架构
- 在5.7版本之上，建议使用GTID方式。

#### 方式

- 异步复制
  - 异步复制.png
  - 文字解释
    - 在主数据库数据库修改提交后记录到二进制日志中，通知从服务器进行复制操作。

- 半同步复制
  - 半异步复制.png
  - 文字解释
    - 在主服务器数据库修改，并记录到二进制日志中，从服务器将主服务器修改的日志同步到中继日志中，然后向主服务器发送确认信息，然后主服务器提交修改。
    - 主服务器有一个过期时间 ，超过该时间后，主服务器不会等待从服务器发回的确认信息，直接提交。

### 基于日志点的复制

#### 流程图

主从复制的原理.png

#### 流程解释

- 主数据库服务器要把数据的修改记录到二进制日志中（binary_log），需要注意的是 查看是否开启了二进制日志。
- 在从服务器上启动配置一个IO_THREAD,该线程在master打开一个普通连接，主要工作是binlog dump process。
- 如果读取的进度已经跟上了master，就进入睡眠状态并等待master产生新的事件。
- 根据Logfile:pos(同步的日志偏移量) 将主服务器的中新的修改日志传回到从服务器。
- 从服务器接受日志，写入到中继日志中。
- 从服务器通过SQL_THREAD 读取中继日志中的事件，并对从服务器上对这些事件写入到自己的数据库中。



### 配置主从复制步骤

#### 主服务器

- 异步复制
  - 必须开启binlog 日志，开启gtid(可选)
  - 建立同步所用的数据库账号
  - 使用master_data参数备份数据库
    - 我们知道在同步之前，从服务器需要给主服务器传送一个日志偏移量或gtid值，这个值的来源就是现将备份的master_data备份的数据库复制到从服务器时的偏移量值。
  - 将备份文件传输到从服务器上，恢复数据库，为主从复制做准备。

- 半同步复制
  - 安装插件: `rpl_semi_sync_master soname 'semisync_master.so'`
  - 设置半同步超时时间: `set  persist rpl_semi_sync_master_timeout = 500`
  - 启动半同步: `set persist rpl_semi_sync_master_enable = on`


#### 从服务器

- 异步复制
  - 开启binlog日志（可选），开启gtid（可选）。
  - 恢复主服务器上的备份数据库（最后是同一版本）
  - 配置复制链路参数（`change master`命令）
  - 启动复制 `start slave`
- 半同步复制
  - 安装插件：`rpl_semi_sync_slave soname 'semisync_slave.so'`
  - 开启半同步: `set persist rpl_semi_sync_slave_enable = on`
  - 重启线程：`stop slave io_thread;`    `start slave io_thread user = 'xxx' password='xxxx';`



#### 几个基本命令

- 查看是否开启二进制日志：`show variables like 'log_bin%';`
- 查看是否启用了gtid： `show variables like 'gtid_mode';` 如果启用gtid，必须配置全局变量 xx.cnf中的几个配置

gtid全局配置.png

- 数据库备份:`mysqldump`导出命令 

- 赋复制权限:`grant replication slave on \*.\*  to user@ip`

- 数据库导入:`mysql -uroot -p < xx.sql`

- 从服务器配置命令 ：`change master`

  changemaster.png

- 启动复制:`start slave user='xxx' password='xxx';`

  

## 主从复制架构

目前MYSQL主流支持的架构方式有

- MMM架构
- MHA架构
- MGR架构

### 作用

- 对主从复制的集中的MASTER进行健康监控

- 当MASTER宕机后，把VIP迁移到新MASTER

  - VIP：独立于数据库物理IP之外的IP。
  - 前端应用通过VIP绑定数据库

- 重新配置集群中的其他slave对新的master同步




### MMM架构（master master mysql）

#### 优缺点

- 优点
  - 提供读写VIP配置，达到高可用
  - 工具包相对完善。
  - 故障转移后，可对MYSQL继续监控。
- 缺点
  - 故障转移时，无法保重数据的完整性。
  - 不支持GTID

#### 原理图解

MMM图解.png

宕机后.png

##### 文字解释

1. 首先 主服务器和主备服务器是相互复制，同时主的服务器下有从服务器。MMM架构会为主服务器分配一个写VIP，会为多个从服务器分配多个读VIP。
2. 当主服务器宕机时，MMM机制会将从服务器迁移至主备服务器上，并且将写VIP迁移至主备服务器。
3. 从服务器宕机时，会将该服务器上的VIP迁移至其他的从服务器上。
4. 最后需要一台监控服务器，安装 MMM的监控组件。

#### 架构资源

mmm架构资源.png



#### 适用场景

- 基于日志点的主从复制方式
- 主主复制的架构
- ##### 考虑读高可用的场景



### MHA架构

#### 优缺点

- 优点
  - 支持GTID和基于日志的复制方式
  - 保证数据的完整性
- 缺点
  - 需要自行开发写VIP迁移脚本
  - 只监控MASTER而没有对Slave实现高可用的读。

#### 原理图解

MHA1.png

MHA2.png

##### 文字解释

1. 我们可以看到 主从服务器只需要安装MHA的 mha_node组件，而我们需要一个另外的监控服务器，我们需要安装mha_manage和mha_node组件
2. 如果主服务器宕机，则从 从服务器中选择一台最接近主服务器的机器升级为主服务器，同时将写VIP迁移至这台服务器上。

#### 架构架构资源

MHA架构资源.png



#### 使用场景

- 使用GTID复制方式
- 使用一主多从的主从复制方式。
- 更少的数据丢失。

### MGR架构

#### 架构图解

MGR架构.png

MGR已经不存在从服务器的角色了，可以支持客户端通过多个主服务器，同时修改数据。

#### 复制原理

mgr原理.png

##### 文字解释

由若干个节点共同组成一个复制组，一个事务的提交，必须经过组内大多数节点（N / 2 + 1）决议并通过，才能得以提交。如上图所示，由3个节点组成一个复制组，Consensus层为一致性协议层，在事务提交过程中，发生组间通讯，由2个节点决议(certify)通过这个事务，事务才能够最终得以提交并响应。

#### 两种模式

##### 单主模式

mgr单主模式.png

只有一个主服务器可以读写操作，另外的只可以读。具体哪个节点是primary节点 是读写服务器，这个需要MGR自己去判定，如果出现宕机，则会选择另外一台作为primary节点。

PS：需要把 `group_replication_single_primary_mode = ON` 开启。

##### 多主模式

mgr多主模式.png

与单主模式相反，可以由多个主服务器进行读写操作，但是由于事务的顺序不一致，会导致死锁或者数据冲突。如果产生冲突，MGR会选择其中一个节点的数据进行回滚。目前暂时不推荐。

##### 

## 其他问题

#### 如何解决读负载过大问题？

读写分离，使用从服务器进行读操作，主服务器尽量只进行写操作。

#### 如何解决写负载过大问题？

分库分表，最好使用数据库中间件来实现（MyCat,ProxySql,Maxscale）


