# 1.Oracle GoldenGate的体系结构
Oracle GoldenGate(OGG)是一种基于日志的结构化数据复制方式，它通过解析源数据库在线日志或归档日志获得数据的增删改变化，再将这些变化应用到目标数据库，实现源数据库与目标数据库同步,双活。

物理结构可分为源端(Source Server)，目标端(Target Server).
逻辑结构可分为数据抽取进程(Extract)，传输进程(Data Pump)，复制进程(Replicat).

OGG各个进程的作用:
进程统一由管理进程(Manager)管理。
抽取进程(Extract)将Redo日志或归档日志作为数据源，当其发生变化时抽取进程会将主键字段和变化字段(如果是既无主键又无唯一索引的表就会抽取全部字段)形成本地的Trail文件(Local Trail)。
传输进程(Data Pump)根据目标端的IP和端口配置将本地Trail文件发送至目标端，生成远程Trail文件(Temote Trail)。
复制进程(Replicat)根据远程Trail文件反向生成SQL语句在目标数据库中执行。


OGG要求源端数据库必须开启归档模式，以保证正常获取归档数据。
针对既无主键又无唯一索引的表，OGG的处理方式为:
一是打开数据最小附加日志开关: alter database add supplemental log data.
二是增加单表级别的表结构字段信息的获取: add trandata.
这两项操作可以保证所用的表都能正确抽取及复制。

# 2.Oracle GoldenGate 12c下载地址
进入Oracle官方网址www.oracle.com，选择Downloads/Middleware/GoldenGate
http://www.oracle.com/technetwork/middleware/goldengate/downloads/index.html


选择下面的下载选项:
Oracle GoldenGate 12.2.0.1.1 for Oracle on Linux x86-64 (454 MB)
下载文件: fbo_ggs_Linux_x64_shiphome.zip

#3.Oracle GoldenGate安装
Oracle GoldenGate安装很简单，但首先需要安装Oracle数据库，安装方法参考Oracle数据库安装文档。

本例安装OGG的环境:

    |             | 源端               |  目标端 |
    | --------    | -----:             | :----: |
    | 操作系统     | Centos7.4         |   Centos7.4    |
    | 数据库       | Oracle11.2.0.4.0  |   mysql    |
    | OGG版本      | Ogg               |   Ogg      |
    | 安装目录     | /opt/ggs          |   /opt/ggs   |
    | tail文件目录 | /opt/ggs/dirdat   |   /opt/ggs/dirdat       |
    | 端口号       | 7809              |   7809      |

需要设置Oracle数据库的全局变量:
```
export ORACLE_BASE=/opt/oracle

export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1

export ORACLE_SID=iih

export GG_HOME=/opt/ggs

export PATH=$JAVA_HOME/bin:$GG_HOME:$ORACLE_HOME/bin:$PATH

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GG_HOME:$ORACLE_HOME/lib:/lib:/usr/lib

```
建立OGG的安装目录:
```
 mkdir /opt/ggs
 chown -R oracle /opt/ggs
 chmod -R 777 /opt/ggs
```
源数据库配置步骤
1） 开启归档，--如未开启，重新开启需要重启实例，在mount状态下
```
SQL>Alter database archivelog
```
2） 开启Force logging
```
SQL>Alter database force logging
```
3） 开启supplemental logging
```
SQL>Alter databaseadd supplemental log data;
```
4） 设置数据库GoldenGate参数
```
SQL> show parameter enable_goldengate_replication；
SQL> alter system set enable_goldengate_replication=true scope=both ; --RAC的所有实例也需要设置
```
5） 创建OGG表空间 
```
SQL>create tablespace ggs datafile '/data/oradata/mcpdts1/ggs01.dbf' size 200m;
   create user ggs identified by ggs default tablespace ggs;
   grant resource, connect, dba to ggs; 
    Oracle 11.2.0.2 and later: 
    exec dbms_goldengate_auth.grant_admin_privilege('ggs');
```

--如要启用DDL功能，OGG用户需要独立的表空间。

#3.ntp
  Oracle ggs 对源端数据的抽取跟时序有依赖，所以在源端为RAC环境的系统，强烈建议在RAC节点间使用NTP进行时钟同步，以减少时序错乱而导致ggs Extract意外停止的风险。
  
#4.数据库归档：
  数据库应处于归档模式：
    archive log list
  归档目录：
    RAC各个节点的归档目录在GoldenGate运行实例上是否读。可以通过NFS方式设置。使在goldengate运行节点上可以读到所有节点的归档日志。或将归档目标指向ASM 空间中。如果需要双向复制或反向回切，GoldenGate目标端也同样设置。
  打开force logging：
    alter database force logging;

#5.数据库参数设置：
  源主机ORACLE数据库为11.2.0.4或以后的版本，需要设置以下参数：
  源端：
    alter system set enable_goldengate_replication=true scope=both;
  打开源端的补充日志(DBA 执行):
    alter database add supplemental log data;
    select supplemental_log_data_min from v$database;
  注：打开补充日志最好在夜里业务很少的时候进行。如果是RAC 需要在每个节点上都执行。
  完成后建议执行一次归档操作：
    alter system archive log current;
#6.网络设置：
Oracle GoldenGate 只需要复制两端的IP 地址之间能够建立TCP 连接，一个Goldengate 复制链路需要10 个TCP 动态端口，具体端口建议使用7839~7849

#7. 操作系统用户：
使用oracle安装、管理GoldenGate。

#8.创建文件系统：
为 GoldenGate 创建文件系统，要求：
1、Oracle GoldenGate 安装空间应当全部位于共享阵列；
2、每个数据库对应一个Oracle GoldenGate 安装，如果是一个数据库上有多个Oracle 实例同样安装一套GoldenGate；
3、在阵列上为每个Oracle GoldenGate 安装划分单独空间，所需空间大小可以参照源端数据库每天产生的归档日志量。例如源数据库每天产生150G 归档，则为该数据库对应的Oracle GoldenGate分配150G 空间；建议为Oracle GoldenGate 分配的空间不小于150G 左右。

#9.GoldenGate源端的安装：
安装 GoldenGate for Oracle x.x 必须遵循以下步骤：
在 RAC 两个节点的任意节点以oracle 身份登录；
运行如下命令完成GoldenGate 的安装：
解压：
[oracle@lddbd app]$ unzip 123012_fbo_ggs_Linux_x64_shiphome.zip 
图形化安装：
./runInstaller
静默安装：（修改好oggcore.rsp文件，只需修改：INSTALL_OPTION=ORA11g；SOFTWARE_LOCATION=/u01/app/ogg；START_MANAGER=false）
./runInstaller -ignoreSysPrereqs -silent -responseFile /tmp/fbo_ggs_Linux_x64_shiphome/Disk1/oggcore.rsp
此时，GoldenGate 源端产品安装部分已经完成。

#10. 创建 GoldenGate 运行时目录：
在安装 GoldenGate 的节点上以oracle 用户身份登录；
在源端执行：
```
cd /opt/ggs
 ./ggsci
 create subdirs
 ```
 
 (2).添加需要同步的表
语法结构:
ADD SCHEMATRANDATA schema [ALLCOLS | NOSCHEDULINGCOLS]
ADD TRANDATA [container.]schema.table [, COLS (columns)] [, NOKEY] [, ALLCOLS | NOSCHEDULINGCOLS]
例子:
$ cd /opt/ggs
$ ggsci
GGSCI> dblogin userid ggs
GGSCI> add trandata ggs.*


(3).编辑源端系统配置参数.
管理器配置参数:
```
GGSCI (iihdb) 1> edit param mgr
PORT 7809
DYNAMICPORTLIST 7810-7909
--AUTOSTART ER *
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```
```
edit params ./GLOBALS
ggschema ggs
```
日志提取配置参数:
```
GGSCI (iihdb) 1> edit params extastt
EXTRACT extastt
USERID ggs, PASSWORD ggs
EXTTRAIL /opt/ggs/dirdat/lt
TABLE ggs.test_ogg;
```


数据传输配置参数:
```
GGSCI (iihdb) 1> edit params pumpastt
extract pumpastt
USERID ggs, PASSWORD ggs
RMTHOST 154.8.157.145, MGRPORT 7809
RMTTRAIL /opt/ggs/dirdat/rt
table ggs.test_ogg;
```

(4).添加源端系统进程.
(4.1).添加提取主进程(Adding the Primary Extract):
DBLOGIN USERIDALIAS alias
ADD EXTRACT group name
{, TRANLOG | , INTEGRATED TRANLOG}
{, BEGIN {NOW | yyyy-mm-dd[ hh:mi:[ss[.cccccc]]]} | SCN value}
[, THREADS n]


(4.2).添加Trail文件(Add the Local Trail):
ADD EXTTRAIL pathname, EXTRACT group name


(4.3).添加传输进程(Add the Data Pump Extract Group):
ADD EXTRACT group name, EXTTRAILSOURCE trail name


(4.4).添加远程Trail文件(Add the Remote Trail):
ADD RMTTRAIL pathname, EXTRACT group name

例子:
```
$ ggsci
add extract extastt, tranlog, begin now
add exttrail /opt/ggs/dirdat/lt, extract extastt
add extract pumpastt, exttrailsource /u01/app/ogg/12.2.0/dirdat/lt
add rmttrail /opt/ggs/dirdat/rt, extract pumpastt
start extract extastt
start extract pumpastt
```

源端系统打开防火墙端口:1521,7809.


5.Oracle GoldenGate目标端系统配置
(1).修改Oracle系统参数，允许执行OGG复制.
$ sqlplus / as sysdba;
SQL> alter system set enable_goldengate_replication=true scope=both;


(2).编辑目标端系统配置参数.
管理器配置参数:
GGSCI (iihdb) 1> edit param mgr
```
PORT 7809
DYNAMICPORTLIST 7810-7909
--AUTOSTART ER *
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
```
GLOBALS配置参数:
GGSCI (iihdb) 1> edit params ./GLOBALS
 ```
CHECKPOINTTABLE ggs.checkpoint
```
二：目标端mysql ogg安装：

1.把相应的table和database创建好，注意字段类型长度等，举例：
oracle-->mysql
number-->float
number(7)-->decimal(7,0)
number(7,2)-->decimal(7,2)
varchar2(5)-->varchar(5)
date-->datetime
--------------------- 


下载：http://www.oracle.com/technetwork/cn/middleware/goldengate/downloads/index.html

解压即可使用：
[root@node02 ~]# unzip /opt/ggs/123015_ggs_Linux_x64_MySQL_64bit
[root@node02 ~]# mv ggs_Linux_x64_MySQL_64bit.tar /opt/ggs
[root@node02 ~]# cd /opt/ggs
[root@node02 ~]# tar -xf ggs_Linux_x64_MySQL_64bit.tar

[root@node02 ~]# ./ggsci

Oracle GoldenGate Command Interpreter for MySQL
Version 12.3.0.1.2 OGGCORE_12.3.0.1.0_PLATFORMS_171208.0005
Linux, x64, 64bit (optimized), MySQL Enterprise on Dec  8 2017 11:42:23
Operating system character set identified as UTF-8.

Copyright (C) 1995, 2017, Oracle and/or its affiliates. All rights reserved.

GGSCI (node02) 1> create subdirs  
GGSCI (node02) 1> edit params ./GLOBALS
```
ggschema ggs
CHECKPOINTTABLE ggs.checkpoint
```
 GGSCI (node02) 1>edit params mgr
```
PORT 7809
DYNAMICPORTLIST 7810-7909
--AUTOSTART ER *
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
ACCESSRULE, PROG *, IPADDR 62.234.*.*, ALLOW
```

数据复制配置参数:
GGSCI (node02) 1> edit params repastt
REPLICAT repastt
dboptions HOST 127.0.0.1,connectionport 3306

targetdb ggs,userid ggs,PASSWORD ggs123
ASSUMETARGETDEFS
MAP ggs.test_ogg, TARGET ggs.test_ogg;


GGSCI (node02 DBLOGIN as ggs) 51> dblogin SOURCEDB ggs,userid ggs
add checkpointtable ggs.checkpoint -- 如果已存在需删除 delete checkpointtable ogg.checkpoint
add replicat repastt, exttrail /opt/ggs/dirdat/rt, checkpointtable ggs.checkpoint
start replicat repastt

6.测试同步
在源端数据库的表中插入数据.
检查源端ogg的dirdat目录中是否有生成提取文件.
检查目标端ogg的dirdat目录中是否有接收到文件.
检查目标端数据库中是否已同步数据.


7.重新安装OGG.
先停止OGG的进程
$ ggsci
stop extastt
stop pumpastt
stop mgr


删除ogg目录所有文件，修改下inventory.xml，去掉OGG的HOME配置即可.
$ cd /u01/app
$ rm -r ogg
$ cd /u01/app/oraInventory/ContentsXML
$ vi inventory.xml


8.查看错误方法及常见错误信息 
如OGG的进程有中断，可以通过命令查看错误信息:
$ ggsci
view report repastt
