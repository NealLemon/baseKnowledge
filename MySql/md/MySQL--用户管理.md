# MySQL--基础内容

  对MySQL做系统的复习。

用户管理

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











