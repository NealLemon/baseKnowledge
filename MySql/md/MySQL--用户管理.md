# MySQL--基础内容(一)

  对MySQL做系统的复习。

## 用户管理

数据库账号

- 用户名@可访问列表

  - 访问列表种类

    - `%`--> 代表可以从所有外部主机访问
    - `192.169.1.%`--> 代表可以从192.168.1网段访问
    - `localhost` --> 代表只能本地访问

- 创建数据库用户

  - CREATE USER 建立用户 具体语法信息可以使用 `\h create user` 查看，部分语法如图 

  p1.png

- 常用权限  使用 `show privileges;`查看当前MYSQL版本的权限列表。

  p2.png

|       |     语句      |         说明         |
| :---: | :-----------: | :------------------: |
| Admin |  Create User  |   建立新用户的权限   |
| Admin | Grant options | 为其他用户授权的权限 |
| Admin |     Super     |   管理服务器的权限   |
|  DDL  |    Create     |     新建表的权限     |
|  DDL  |     Alter     |     修改表的权限     |
|  DDL  |     Drop      |     删除表的权限     |
|  DDL  |     Index     | 建立和删除索引的权限 |
|  DML  |    Select     |    查询表数据权限    |
|  DML  |    Insert     | 向表中插入数据的权限 |
|  DML  |    Update     |  更新表中的数据权限  |
|  DML  |    Delete     |  删除表中的数据权限  |
|  DML  |    Execute    |  执行存储过程的权限  |

   - 	用户授权
      - 	遵循最小权限原则
      - 	使用 Grant 命令对用户授权
           - 	grant 权限语句 on 库名.表名 to user@ip;
                 - 	例子：比如给用户`test01` 赋权限给`test`数据库。(`grant create to test.*  on test01@ip identified by ‘密码’` )
      - 	使用revoke命令收回用户的权限
         - 	revoke 权限 on 库名.表名 from user@ip;
            - 	例子: 比如给用户`test01`在库数据库`test`中取消`CREATE`权限。（`revoke create on test.* from test01@'localhost'`  ）
         - 	刷新权限  `flush privileges`



### 服务器配置

### SQL_MODE

set [session/global/persist] sql_mode = 'XXXX';

#### 主要分为两大类

- 宽松模式：SQL_MODE = 'ANSI';   
- 传统模式:  SQL_MODE = 'TRADITIONAL';



### 系统配置

set session 系统变量名 = 'xxxxx';

set global 系统变量名 = 'xxxxx';

set persist 系统变量名 = 'xxxxx' (MYSQL 8)



#### 常用性能参数

p3.png

p4.png

p5.png



## 日志以及使用场景

#### 常用日志

|         日志名称         |                             作用                             |
| :----------------------: | :----------------------------------------------------------: |
|  错误日志（error_log）   |           记录mysql在启动、运行或停止时出现的问题            |
|  常规日志(general_log)   | 记录所有发现MySQL的请求(连接请求，管理命令，数据库操作请求)  |
| 慢查日志(slow_query_log) |                      记录符合条件的查询                      |
| 二进制日志（binary_log） | 记录全部有效的数据修改日志(记录所有提交的事务) --主从复制，增量备份，数据恢复 |
|   中继日志(relay_log)    |         用于主从复制，临时存储从主库同步的二进制日志         |



##### 错误日志（error_log）

- 分析排除MYSQL运行错误：异常重启，启动失败，主从同步异常等。
- 记录未经授权的访问。

###### 错误日志参数配置

- log_error = 'xxxxxxxxxxxx'   错误日志输出路径
- log_error_verbosity = [1,2,3]  日志输出级别

p6.png

- log_error_services = [服务组件1;服务组件2]

p7.png

###### 查看错误日志路径:`select @@log_error;`

###### 查看错误日志级别:`select @@log_error_verbosity;`

###### 查看日志服务组件:`select @@log_error_services;`



##### 常规日志(general_log)：必要时打开，要及时关闭

- 分析客户端发送到MySQL的实际请求。（比如 从客户端连接开始，改变数据库中的信息，到客户端断开连接的整个过程）

###### 常规日志(general_log)的参数配置

- general_log = [ON|OFF]  常规日志开关闭  默认OFF
- general_log_file = 'xxxxxxxxxxxxxx'  常规日志的存储路径
- log_output = [FILE|TABLE|NONE]  日志输出形式，如果为TABLE 默认保存在 GENERAL_LOG表中。



###### 查看常规日志开启状态:`select @@general_log`

###### 查看常规日志文件路径：`select @@general_log_file`

###### 查看常规日志输出形式：`select @@log_output`



##### 慢查日志(slow_query_log)	:解决性能问题

-  将执行成功并符合条件的查询录到日志中。
- 找到需要优化的SQL。



###### 慢查日志(slow_query_log)	的参数配置

- slow_query_log = [ON|OFF]  慢查日志开关  默认OFF
- slow_query_log_file = 'xxxxxxxxxx' 慢查日志位置
- long_query_time = xxx（以秒为单位，可以记录到微妙 可以6位小数） 当SQL执行时间超过这个值时，就会被记录到慢查日志中，如果想记录所有SQL 则设置为 0
- log_queries_not_using_indexes = [ON | OFF ]  将所有没有使用索引的SQL记录到日志中 默认OFF
- log_slow_admin_statements = [ON | OFF ] 记录操作的管理命令 比如 alter table ,create index 等 默认为OFF



###### 查询SQL执行时间阈值:`show variables like 'long_query_time'`;



##### 二进制日志（binary_log）

- 记录所有对数据库中数据的修改 (insert update delete等)。
- 由于记录的所有数据库的修改操作可以基于时间点的备份和恢复。
- 主从复制

###### 二进制日志（binary_log）常用配置

- log-bin [=base_name]    是否启用二进制日志,base_name是存储的文件目录以及前缀名， 静态配置，只能在配置文件中修改 。

  p8.png

- binlog_format = [ROW | STATEMENT | MIXED]  日志记录方式

- binlog_row_image = [FULL | MINIMAL | NOBLOB ] 针对 ROW方式记录的记录方式

- binlog_row_query_log_events = [ON | OFF ] 开启针对ROW日志记录方式时，记录当前执行SQL。

- log_slave_updates = [ON | OFF ]   正常情况下 slave服务器不会记录从主服务器上同步的日志，开启后会记录从主同步到slave服务器时的日志。

- sync_binlog = [1 | 0 ] 1 是每写一次二进制日志，就会像磁盘刷新， 0 是 不会主动刷新至磁盘。

- expire_logs_days = days    设置日志过期时间，自动清理过期日期

- PURGE BINARY LOGS TO '二进制日志文件名'  清理 设置的 二进制日志文件名 之前的所有日志

- PURGE BINARY LOGS BEFORE '2008-04-22 12:12:12';  清理日期之前的二进制日志。 

###### 查询二进制日志的行记录方式的指令: mysqlbinlog --no-defaults -vv --base64-output=DECODE-ROWS 日志文件名



## 存储引擎

#### 引擎种类

p10.png

#### 锁种类

- 共享锁（Share Lock）：共享锁又称读锁，是读取操作创建的锁。其他用户可以并发读取数据，但任何事务都不能对数据进行修改（获取数据上的排他锁），直到已释放所有共享锁。

- 排他锁（Exclusive Lock）：排他锁又称写锁、独占锁，如果事务`T`对数据`A`加上排他锁后，则其他事务不能再对`A`加任何类型的封锁。获准排他锁的事务既能读数据，又能修改数据。



##### MyISAM引擎

- 非事务型存储引擎
- 以堆表方式存储
- 使用表级锁 ： 比如查询的时候就加共享锁，修改数据的时候就加排他锁。读写操作之间会有相互阻塞的情况。
- 支持Btree索引，空间索引，全文索引.
- 表的数据和索引是分开存储的。

###### 部分命令

1. 修复表 : `repair table tb`
2. 压缩表: `myisampack -b -f tb`

###### 使用场景

- 读操作远远大于写操作的场景
- 不需要事务



##### CSV存储引擎

- 非事务型存储引擎
- 数据以CSV格式存储
- 所有列不能为NULL
- 不支持索引

###### 使用场景

- 作为数据交换的中间表使用，可以把EXCEL等表数据文件转换成CSV然后赋值到MYSQL 数据存放的目录下。



##### Archive引擎

- 非事务型存储引擎
- 表数据使用zlib压缩
- 只支持insert和 select
- 只允许在自增ID列上建立索引

###### 使用场景

- 日志和数据采集类的数据应用。
- 数据归档



##### Memory引擎

- 非事务型存储引擎
- 数据存储在内存中
- 所有字段长度固定
- 支持Btree索引和Hash索引

###### 使用场景（类似于redis）

- 缓存字典映射表

- 缓存周期性分析数据

  

##### Innodb引擎

- 事务型存储引擎，支持ACID
- 数据按主键聚集存储。（主键是逻辑存储）
- 支持行级锁以及MVCC(多版本并发控制)
- 支持Btree和自适应Hash索引
- 支持全文索引和空间索引

###### 使用场景

- 支持绝大部分的OLTP场景

##### NDB引擎

- 事务型存储引擎，支持ACID
- 数据在使用前要从磁盘读取到内存
- 支持行级锁
- 支持集群
- 支持Ttree索引

###### 使用场景

- 需要数据完全同步的高可用场景

  

## Innodb相关问题

##### 什么情况下，无法在线修改表？

答：增加索引，主键，自增列，修改列类型以及表的字符集等。

p11.png

##### 在线DDL存在的问题?

- 有些DDL语句不支持在线修改
- 长时间的DDL操作，影响主从复制
- 无法对DDL操作进行资源限制，会导致磁盘或内存暴增。

##### 如何安全的在线DDL？

使用Percona 工具 相关命令 `pt-online-schema-change [OPTIONS] DSN`

样例:

p12.png

##### Innodb如何实现事务

[数据库事务](https://baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1/9744607)(Database Transaction) ，是指作为单个逻辑工作单元执行的一系列[操作](https://baike.baidu.com/item/%E6%93%8D%E4%BD%9C/33052)，要么完全地执行，要么完全地不执行。

共享锁：如果事务T对数据A加上共享锁后，则其他事务只能对A再加共享锁，不能加排它锁。获准共享锁的事务只能读数据，不能修改数据。

排他锁：如果事务T对数据A加上排他锁后，则其他事务不能再对A加任任何类型的封锁。获准排他锁的事务既能读数据，又能修改数据。



p13.png



举例：

p14.png



MVCC流程

p15.png



