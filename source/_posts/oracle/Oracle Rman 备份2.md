---
title: Oracle Rman 备份2
date: 2018-10-04 10:25:00
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
  - RMAN
---


# Oracle Rman 的备份 （案列）

[Oracle 11G R2 官方文档][1]

## 1.如何设计一个TB级数据库的RMAN备份策略
### 1.1 TB以下
* 用ORACLE即可实现（保存在磁盘上，5TB），也可以选择第三方备份软件 集中管理：

>1）RMAN全备：每周，每天；开归档模式
2）逻辑全导出：每周
3）归档备份（第三方管理）

### 1.2 TB以下
* 用ORACLE+结合第三方软件（集中管理，带库，虚拟带库，大的备份存储，10TB）：

> 1）RMAN全备：每周，开归档模式
2）归档备份：每天
3）逻辑导出：一个月只导一次结构就行了，或每天导一些重要的表。

### 1.3 2TB以上（集中管理，虚拟带库，第三方备份介质，几十TB，压缩功能，重复删除）：
>1）RMAN全备：每周，开归档模式；每个月一次加每周增量。
2）归档备份：每天
3）逻辑导出：一个月只导一次结构就行了，或每天导一些重要的表。

### 1.4 备份案列
* 环境

> ORACLE RMAN+存储备份（1TB）
数据库1TB，每天的归档量50GB

* 备份策略

>每周5,20:00全备，保留2份。5TB
归档保留10天。 1TB
恢复，全备+归档。
逻辑全导出：每周 2TB

* 备份脚本 rman_full_orcl.sh

```sql
rman target / catalog rman/rman@rman msglog '/u01/backup/logs/rman_full_itpux.log' << EOF
run {
	CONFIGURE RETENTION POLICY TO REDUNDANCY 2;
	allocate channel d1 type disk;
	allocate channel d2 type disk;

	setlimit channel d1 kbytes 102428750 maxopenfiles 32 readrate 200;
	setlimit channel d2 kbytes 102428750 maxopenfiles 32 readrate 200;
	sql 'alter system archive log current';
	backup
		incremental level 0
		skip inaccessible
		tag itpux_level0
		filesperset 3
		format '/u01//backup/datafile/orcl_rman_full_%s_%p_%t'
		(database);
	release channel d1;
	release channel d2;

	ALLOCATE CHANNEL d3 TYPE disk;
		BACKUP
			# recommended format
			FORMAT '/u01//backup/orcl_rman_cntrl_%s_%p_%t'
			CURRENT CONTROLFILE;
	RELEASE CHANNEL d3;

	ALLOCATE CHANNEL d4 TYPE DISK;
		copy current controlfile to '/u01//backup/datafile/control_itpux.bak';
	RELEASE CHANNEL d4;
}

ALLOCATE CHANNEL FOR MAINTENANCE DEVICE TYPE DISK;
run
{
	report obsolete;
	DELETE noprompt EXPIRED BACKUP;
	DELETE noprompt EXPIRED COPY;
	delete noprompt obsolete device type disk;
}
exit
EOF
```

* 删除归档 rman_delarch_orcl.sh

```sql
rman target / catalog rman/rman@rman msglog '/u01/backup/logs/rman_delarch_itpux.log' <<EOF
run{
	crosscheck archivelog all;
	delete noprompt archivelog until time "sysdate-7";
	delete noprompt expired archivelog all;
}
exit

```
* 设置crontab

```sql
00 5 * * * su - oracle -c /u01/backup/scripts/rman_delarch_orcl.sh
00 18 * * 5 su - oracle -c /u01/backup/scripts/rman_full_orcl.sh
```
## 2.在非归档模式的RMAN备份案例
> &emsp;&emsp;备份分为一致性备份和不完全性备份，也就是我们所说的归档模式与非归档模式的备份.创建非归档备份可以是在非归档模式下创建，并且数据库必须处于mount状态下，而且恢复的时候值能恢复到最后一次备份的状态。也就说从备份到发生故障的这段时间都将丢失

* 1.1. 检查归档状态

```sql
archive log list;
```

* 1.2 将数据库启动到mount状态

```sql
shutdown immediate
startup mount;
```

* 1.3 执行备份

```sql
rman target /
backup database;
```

* 2.1 非归档模式备份

```sql
shutdown immediate;
startup mount;

rman target /
backup database;
```

* 3.1 归档模式

```sql
alter database open
```

## 3.在归档模式的RMAN备份与恢复案例-丢失所有文件
>  &emsp;&emsp;如果要创建正式库的备份，一般不建议用非归档模式备份，也不建议用很简单的命令来完成。而是更多的采用脚本实现归档模式备份，这样将可通过backup+archive log+redo有效的将数据恢复到最近一次改变的状态，可以达到数据的丢失最小化。

* 创建归档模式备份
> &emsp;&emsp; 创建归档模式备份数据库必须处于归档(archivelog)模式，因为归档模式备份的数据库文件和控制文件的SCN号可能会不一致。
并且可以在数据库打开并不影响业务的情况下完成数据的备份工作；那么这样的备份将是归档模式的备份，那么如果要恢复可以通过backup+archive log+redo来恢复到最近一次日志切换时候的数据，而不是最后一次备份时候的数据。

* 寻找dbid

```sql
--1.可以在查看数据库警告日志
vim /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
--2.在备份的控制文件中查找
	
--3.用BBED在备份的文件中查找

--4.在备份日志中查找 

--5.配置控制文件强制启动数据库
--5.1 添加一个参数文件

```



## 4.RMAN恢复案例-丢失单个数据文件如何恢复
## 5.RMAN恢复案例-丢失整个数据表空间如何恢复
## 6.RMAN恢复案例-丢失SYSTEM表空间如何恢复
## 7.RMAN恢复案例-丢失控制文件如何恢复
## 8.RMAN恢复案例-丢失参数文件如何恢复
## 9.RMAN恢复案例-丢失重做日志文件如何恢复
## 10.RMAN恢复案例-存储损坏数据丢失如何恢复
## 11.RMAN恢复案例-不完全恢复
## 12.RMAN基于时间点(time)的不完全恢复
## 13.RMAN基于系统改变号(scn)的不完全恢复
## 14.RMAN基于部分数据文件与部分归档丢失的cancel不完全恢复
## 15.RMAN基于当前重做日志丢失的cancel不完全恢复
## 16.RMAN基于备份控制文件不完全恢复


  [1]: https://docs.oracle.com/cd/E11882_01/index.htm