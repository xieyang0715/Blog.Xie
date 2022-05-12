---

title: MySQL8.0 创建用户及授权
date: 2022-05-12 17:55:00
categories: 
   - [MySQL] 
toc: true
---



MySQL8.0 创建用户及授权

<!--more-->

### mysql版本

MySql8.0+

### 具体步骤

#### 1.[命令行](https://so.csdn.net/so/search?q=命令行&spm=1001.2101.3001.7020)进入MySql

> 使用 mysql -u#UserName -p#PassWord 命令进入MySql

**#UserName** 代表你的MySql用户名
**#PassWord** 代表你的MySql密码

用户名是root,密码是root

```
mysql -uroot -proot
```

#### 2.进入数据库

如果没有创建数据库则先使用命令,若已存在数据库则跳过此步骤

> create database #databaseName;

**#databaseName** 代表你操作的数据库

要创建的是b2b数据库,切记加上分号;

```mysql
create database b2b;
1
```

> use databaseName;

**#databaseName** 代表你操作的数据库

要操作的是b2b数据库,切记加上分号;

```mysql
use b2b;
```

#### 3.创建用户

> create user ‘#userName’@’#host’ identified by ‘#passWord’;

**#userName** 代表你要创建的此数据库的新用户账号
**#host** 代表访问权限，如下

- %代表通配所有host地址权限(可远程访问)
- localhost为本地权限(不可远程访问)
- 指定特殊Ip访问权限 如10.138.106.102

**#passWord** 代表你要创建的此数据库的新用密码

要创建的用户是testUser，密码是Haier…123,并且可远程访问
⚠️密码强度需要大小写及数字字母，否则会报密码强度不符合

用户名如果重复，会报错ERROR 1396 (HY000): Operation CREATE USER failed for ‘testUser’@’%’

```mysql
create user 'testUser'@'%' identified by 'Haier...123';
```

#### 4.查看用户

进入mysql系统数据库

> use mysql;

查看用户的相关信息

> select host, user, authentication_string, plugin from user;

```
use mysql;  select host, user, authentication_string, plugin from user;
```

若展示的信息中有刚加入的用户testUser，则添加成功。切记查看完要切换回操作的数据库,本狗需要操作的是b2b

```mysql
use b2b; 
```

#### 5.用户授权

> grant #auth on #databaseName.#table to ‘#userName’@’#host’;

**#auth** 代表权限，如下

- all privileges 全部权限
- select 查询权限
- select,insert,update,delete 增删改查权限
- select,[…]增…等权限

**#databaseName** 代表数据库名
**#table** 代表具体表，如下

- *代表全部表
- A,B 代表具体A,B表

**#userName** 代表用户名

**#host** 代表访问权限，如下

- %代表通配所有host地址权限(可远程访问)
- localhost为本地权限(不可远程访问)
- 指定特殊Ip访问权限 如10.138.106.102

赋予b2b数据库area_code表增删改差权限

```mysql
grant select,insert,update,delete on b2b.area_code to 'testUser'@'%';
```

#### 6.刷新

🔥切记一定要刷新授权才可生效

> flush privileges;

#### 7.查看[用户权限](https://so.csdn.net/so/search?q=用户权限&spm=1001.2101.3001.7020)

> show grants for ‘#userName’@’#host’;

**#userName** 代表用户名

**#host** 代表访问权限，如下

- %代表通配所有host地址权限(可远程访问)
- localhost为本地权限(不可远程访问)
- 指定特殊Ip访问权限 如10.138.106.102

要查看的是testUser

```mysql
show grants for 'testUser'@'%';
```

#### 撤销权限

> revoke #auth on #databaseName.#table from ‘#userName’@’#host’;

**#auth** 代表权限，如下

- all privileges 全部权限
- select 查询权限
- select,insert,update,delete 增删改查权限
- select,[…]增…等权限

**#databaseName** 代表数据库名
**#table** 代表具体表，如下

- *代表全部表
- A,B 代表具体A,B表

**#userName** 代表用户名

**#host** 代表访问权限，如下

- %代表通配所有host地址权限(可远程访问)
- localhost为本地权限(不可远程访问)
- 指定特殊Ip访问权限 如10.138.106.102

要撤销testUser用户对b2b数据库中的area_code表的增删改差权限

```mysql
revoke select,insert,update,delete on b2b.area_code from 'testUser'@'%';
```

再查看用户权限

```mysql
show grants for 'testUser'@'%';
```

#### 删除用户

> drop user ‘#userName’@’#host’;

**#userName** 代表用户名

**#host** 代表访问权限，如下

- %代表通配所有host地址权限(可远程访问)
- localhost为本地权限(不可远程访问)
- 指定特殊Ip访问权限 如10.138.106.102

要删除用户是testUser

```mysql
drop user 'testUser'@'%';
```

[MySQL8.0 创建用户及授权]: https://blog.csdn.net/baidu_25986059/article/details/104042858

