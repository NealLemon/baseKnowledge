# MySQL

  对MySQL做系统的复习。

## 用户管理

### 数据库账号

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
         - 	刷新权限  `flush privileges`。







