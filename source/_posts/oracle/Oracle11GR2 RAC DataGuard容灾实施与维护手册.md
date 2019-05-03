---
title: Oracle11GR2 RAC DataGuard容灾实施与维护手册
date: 2018-10-07 10:25:00
author: Kyle Liu
<!-- img: /source/images/xxx.jpg -->
top: false
<!-- cover: false -->
<!-- coverImg: /images/1.jpg -->
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: true
summary: 这是你自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要
categories: Oracle
tags:
  - Oracle
  - RAC
  - DataGuard
  - 备份容灾
---


# Oracle11gR2RAC DataGuard容灾实施与维护

[Oracle 11G R2 官方文档][1] 


## 1.DataGuard的概述


## 2.应用规划

|   参数  |  主库   |  备库   |
:---:|:---:|:---:
|系统架构     |  RAC 双机+ASM   |  单实例+ASM   |
|  数据库版本   |  11.2.0.4   |   11.2.0.4  |
|   内存  |   16G  |   8G  |
|  CPU   |  4   |  4   |
|   GoldenGate  |  11.2.0.3   |   11.2.0.3  |
|     |     |     |
|     |     |     |
|     |     |     |

## 3.案列一（文件系统到文件系统）
### 3.1 系统架构
|   参数  |  主库   |  备库   |
:---:|:---:|:---:
|系统架构     |  单实例文件系统   |  单实例文件系统    |
|  数据库版本   |  11.2.0.4   |   11.2.0.4  |
|   内存  |   16G  |   8G  |
|  CPU   |  4   |  4   |
|   GoldenGate  |  11.2.0.3   |   11.2.0.3  |
|     |     |     |

### 3.2  安装步骤1（准备）
* 1.安装相关的软件包
* 2.准备OGG安装的环境变量
* 3.上传并安装OGG软件
* 4.准备测试数据

#### 3.2.1 安装相关的软件包

#### 3.2.1 准备OGG安装的环境变量

#### 3.2.1 上传并安装OGG软件

#### 3.2.1 准备测试数据
```sql

create tablespace itpux01 datafile '/u01/app/oracle/oradata/orcl/itpux01.dbf' size 100m autoextend off;
create user itpux01 identified by itpux01 default tablespace itpux01 temporary tablespace temp;
grant dba to itpux01;   


create tablespace itpux02 datafile '/u01/app/oracle/oradata/orcl/itpux02.dbf' size 100m autoextend off;
create user itpux02 identified by itpux02 default tablespace itpux02 temporary tablespace temp;
grant dba to itpux02;   


create tablespace itpux03 datafile '/u01/app/oracle/oradata/orcl/itpux03.dbf' size 100m autoextend off;
create user itpux03 identified by itpux03 default tablespace itpux03 temporary tablespace temp;
grant dba to itpux03;  


【源库导出、导入数据结构】
create directory itpuxbak as '/home/oracle';
grant read,write on directory itpuxbak to system; 
grant create any directory to system;

expdp system/jia directory=itpuxbak dumpfile=expdp_full_db01.dmp logfile=expdp_full_db01.log content=metadata_only

--expdp system/jia  directory=itpuxbak dumpfile=expdp_itpux010203.dmp logfile=expdp_itpux010203.log schemas=itpux01,itpux02,itpux03


impdp system/oracle  directory=itpuxbak dumpfile=expdp_itpux010203.dmp logfile=expdp_itpux010203.log full=y

--impdp system/oracle  directory=itpuxbak dumpfile=expdp_itpux010203.dmp expdp_itpux010203.dmp schemas=itpux01,itpux02,itpux03

```


### 3.3 安装步骤2（配置环境）
* 1.源端打开归档，目标端一般不需要
* 2.源端数据库打开补充日志
* 3.源端是数据库开启FORCE_LOGGING
* 4.关闭回收站功能
* 5.源和目标的网络通信正常
* 6.创建专用的GoldenGate用户来同步数据
* 7.修改数据库参数（源和目标端）

#### 3.3.1 源端打开归档，目标端一般不需要
> &emsp;&emsp; 因为有OGG所以只要源端开启归档，目标端不用开启归档,下面将介绍文件系统单实例怎么开启关闭归档.

* 开启归档 

```sql
-- 1.确定当前没有开启归档
SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     19
Current log sequence           21

--2.创建归档目录

--3.重启数据库到MOUNT
SHUTDOWN IMMEDIATE
STARTUP MOUNT
--4.修改归档路径并开启归档
ALTER SYSTEM SET  db_recovery_file_dest='/u01/backup' scope=spfile;
ALTER DATABASE ARCHIVELOG;
--5.重启数据库
SHUTDOWN IMMEDIATE
STARTUP
```

* 关闭归档

```sql
--1.确定当前为归档
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     19
Next log sequence to archive   21
Current log sequence           21

--2.重启数据库到MOUNT
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.

Total System Global Area 2137886720 bytes
Fixed Size                  2254952 bytes
Variable Size             570427288 bytes
Database Buffers         1560281088 bytes
Redo Buffers                4923392 bytes
Database mounted.

--3.关闭归档并打开数据库
SQL>  alter database noarchivelog;

Database altered.
SQL> alter database open;

Database altered.

SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     19
Current log sequence           21
```

#### 3.3.2 源端数据库打开补充日志
```sql
--1.查看是否打开最小日志记录
SELECT SUPPLEMENTAL_LOG_DATA_MIN FROM v$database;

--2.开启最小日志记录
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

--3.切换归档
ALTER SYSTEM SWITCH LOGFILE;

```
#### 3.3.3 源端是数据库开启FORCE_LOGGING
```sql
--1.查询是否开启强制日志模式
force_logging from v$database;

--2.打开
alter database force logging;
alter system switch logfile;
```

#### 3.3.4 关闭回收站功能
> &emsp;&emsp;官方没有要求关闭回收站，但是我们一般都会关闭。关闭回收站必须重启生效。
```sql
--1.查看回收是否关闭
show parameter recyclebin

--2.关闭回收站
alter system set recyclebin=off scope=spfile;

--3.重启数据库
```
#### 3.3.5 源和目标的网络通信正常
```sql

```

#### 3.3.6 创建专用的GoldenGate用户来同步数据

#### 3.3.7 修改数据库参数（源和目标端）
```sql
alter system set enable_goldengate_replication=true scope=both;
show paramteter enable_goldengate_replication;
```

### 3.4 安装步骤（配置OGG）
```sql
1.创建文件目录
create subdirs

01.配置配置管理进程mgr
ggsci> edit params mgr
port 7809
autostart er *
autorestart er *,waitminutes 3,retries 15
purgeoldextracts ./dirdat/*,usecheckpoints,minkeepdays 7*/

02
alter system set enable_goldengate_replication=true scope=both;


```



## 4.案列二（ASM到文件系统）

## 5.案列三（RAC ASM 到 单机ASN）
### 3.1 系统架构
|   参数  |  主库   |  备库   |
:---:|:---:|:---:
|系统架构     |  RAC   |  单实例ASM    |
|  数据库版本   |  11.2.0.4   |   11.2.0.4  |
|   内存  |   4G  |   4G  |
|  CPU   |  4   |  4   |
|   GoldenGate  |  12.1.0.3   |   12.1.0.3  |
|     |     |     |

### 3.2  安装步骤（准备）
* 1.安装相关的软件包
* 2.准备OGG安装的环境变量
* 3.上传并安装OGG软件
* 4.准备测试数据

#### 3.2.1 安装相关的软件包

#### 3.2.1 准备OGG安装的环境变量

#### 3.2.1 上传并安装OGG软件

#### 3.2.1 准备测试数据


### 3.3 安装步骤（配置环境）
* 1.源端打开归档，目标端一般不需要
* 2.源端数据库打开补充日志
* 3.源端是数据库开启FORCE_LOGGING
* 4.关闭回收站功能
* 5.源和目标的网络通信正常
* 6.创建专用的GoldenGate用户来同步数据

#### 3.3.1 源端打开归档，目标端一般不需要
> &emsp;&emsp; 因为有OGG所以只要源端开启归档，目标端不用开启归档,下面将介绍文件系统单实例怎么开启关闭归档.


#### 3.3.2 源端数据库打开补充日志
#### 3.3.3 源端是数据库开启FORCE_LOGGING
#### 3.3.4 关闭回收站功能
#### 3.3.5 源和目标的网络通信正常
#### 3.3.6 创建专用的GoldenGate用户来同步数据




  [1]: https://docs.oracle.com/cd/E11882_01/index.htm