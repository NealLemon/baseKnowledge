# MySQL

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







