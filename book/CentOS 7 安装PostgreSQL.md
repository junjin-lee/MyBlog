# CentOS 7 安装PostgreSQL
## 创建postgres用户
```
[root@VMTest postgresql11]# useradd -g postgres postgres
```
## 下载并安装rpm包
```
wget https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
yum install  https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
yum -y install postgresql11 postgresql11-server postgresql11-libs


```

## 初始化数据库
```
[root@VMTest ~]#  /usr/pgsql-11/bin/postgresql-11-setup initdb
Initializing database ... OK
```
## 设置开机启动，启动数据库
```
[root@VMTest ~]# systemctl enable postgresql-11  #设置开机启动
[root@VMTest ~]# systemctl start postgresql-11   #启动postgresql
 
#查看启动效果
方法一:
[root@VMTest ~]# ps -ef | grep postgre
postgres 35460     1  0 16:19 ?        00:00:00 /usr/pgsql-11/bin/postmaster -D /var/lib/pgsql/11/data/
postgres 35462 35460  0 16:19 ?        00:00:00 postgres: logger   
postgres 35464 35460  0 16:19 ?        00:00:00 postgres: checkpointer   
postgres 35465 35460  0 16:19 ?        00:00:00 postgres: background writer   
postgres 35466 35460  0 16:19 ?        00:00:00 postgres: walwriter   
postgres 35467 35460  0 16:19 ?        00:00:00 postgres: autovacuum launcher   
postgres 35468 35460  0 16:19 ?        00:00:00 postgres: stats collector   
postgres 35469 35460  0 16:19 ?        00:00:00 postgres: logical replication launcher   
root     35474 35326  0 16:19 pts/0    00:00:00 grep --color=auto postgre
 
#方法二：
[root@VMTest ~]# systemctl status postgresql-11
● postgresql-11.service - PostgreSQL 11 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-11.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2018-12-03 16:19:40 CST; 2min 14s ago
     Docs: https://www.postgresql.org/docs/11/static/
  Process: 35454 ExecStartPre=/usr/pgsql-11/bin/postgresql-11-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 35460 (postmaster)
   CGroup: /system.slice/postgresql-11.service
           ├─35460 /usr/pgsql-11/bin/postmaster -D /var/lib/pgsql/11/data/
           ├─35462 postgres: logger   
           ├─35464 postgres: checkpointer   
           ├─35465 postgres: background writer   
           ├─35466 postgres: walwriter   
           ├─35467 postgres: autovacuum launcher   
           ├─35468 postgres: stats collector   
           └─35469 postgres: logical replication launcher   
 
Dec 03 16:19:40 VMTest systemd[1]: Starting PostgreSQL 11 database server...
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.843 CST [35460] LOG:  listening on IPv4 address "127.0.0.1", port 5432
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.843 CST [35460] LOG:  could not bind IPv6 address "::1": Cannot assign...address
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.843 CST [35460] HINT:  Is another postmaster already running on port 5... retry.
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.845 CST [35460] LOG:  listening on Unix socket "/var/run/postgresql/.s...L.5432"
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.848 CST [35460] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.858 CST [35460] LOG:  redirecting log output to logging collector process
Dec 03 16:19:40 VMTest postmaster[35460]: 2018-12-03 16:19:40.858 CST [35460] HINT:  Future log output will appear in directory "log".
Dec 03 16:19:40 VMTest systemd[1]: Started PostgreSQL 11 database server.
Hint: Some lines were ellipsized, use -l to show in full.

```
## 移动数据库到指定目录
### 移动目录
```
[root@VMTest ~]# mv /var/lib/pgsql/11/* /data/pgsql/
[root@VMTest ~]# chown -R postgres:postgres /data/pgsql/
```
###修改配置文件
```

#a.修改指定的数据目录
[root@VMTest ~]# vi /usr/lib/systemd/system/postgresql-11.service 
#修改Environment=PGDATA=/var/lib/pgsql/11/data/为
Environment=PGDATA=/data/pgsql/data/
 
#b.修改数据目录
[root@VMTest ~]# vi /data/pgsql/data/postgresql.conf
#修改data_directory:
 data_directory = '/data/pgsql/data

```

##重新加载配置文件，重启数据库
```
[root@VMTest ~]# systemctl daemon-reload
[root@VMTest ~]# systemctl restart postgresql-11
[root@VMTest ~]# ps -ef | grep postgres  #确认启动成功
```

##修改密码
```
[root@VMTest ~]# su postgres
[postgres@VMTest root]$ psql
could not change directory to "/root": Permission denied
psql (11.1)
Type "help" for help.
 
postgres=# 
 
#------------------------------------------------------
#执行命令
postgres=# ALTER ROLE postgres WITH PASSWORD '123abc';
```

##修改授权
###设置远程连接
```
#修改1， ps：认证方式解释见附录
[postgres@VMTest root]$ vi /data/pgsql/data/pg_hba.conf
# IPv4 local connections: 
#host    all             all             127.0.0.1/32            ident
host    all             all             0.0.0.0/0                 md5
 
#new
host    all             all             0.0.0.0/0               trust
 
#修改2
[root@vmfiend01 ~]# vi /data/pgsql/data/postgresql.conf
#修改listen_addresses 
listen_addresses = '*'
#有需求修改port
#port = 5432 
 
#重启数据库
[root@VMTest ~]# systemctl restart postgresql-11
--------------------- 
```
**参考安装地址https://blog.csdn.net/justlpf/article/details/84769813**
