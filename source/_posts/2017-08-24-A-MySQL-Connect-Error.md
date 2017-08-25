title: MySQL Error2003(111) 问题解决
date: 2017-08-24 09:35:05
tags: [MySQL]
categories: [Blog]
---
## 问题描述
MySQL服务器运行在内网的一台独立机器上，在其他机器上进行MySQLdb连接的时候出现：

```
_mysql_exceptions.OperationalError: (2003, "Can't connect to MySQL server on '10.108.xxx.xxx' (111)")
```

连接方式为Python3 的 MySQLdb包

MySQL版本：```5.7.19-0ubuntu0.16.04.1```

## 问题排查
首先考虑到MySQL服务器没有进行网关的配置，怀疑是网关拦截了连接。

在网关关闭后，发现还是抛出这个错误，开始Google该问题代码。

经确认，是MySQL服务默认监听地址有问题。在默认配置下，MySQL会监听127.0.0.1本地IP，如果外部主机尝试进行连接则不被处理

## 问题解决
修改MySQL服务器的配置文件，注释掉监听IP配置

配置文件路径：``` /etc/mysql/mysql.conf.d/mysqld.cnf ```

``` [mysqld] ```组中，有一条配置``` bind-address = 127.0.0.1 ```

将该条注释掉，然后重启MySQL服务，即可正常连接