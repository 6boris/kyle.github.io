---
title: Oracle Rman 备份3
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


# Oracle Rman 的备份 （高级）

[Oracle 11G R2 官方文档][1] 

## 1.关于RMAN内存缓冲与块跟踪
```sql
## 6.使用RMAN的dbms_backup_restore包
```
* 查看RMAN 备份进程

```sql
select sid,serial#,context,sofar,totalwork,round(sofar/totalwork*100,2) "%_COMPLETE"
     from v$session_longops
    where opname like 'RMAN%'
     AND OPNAME NOT LIKE '%aggregate%'
     and totalwork != 0
     and sofar <>totalwork;
```

## 2.使用RMAN的dbms_backup_restore包
* 1.恢复controlfile

> dbms_backup_restore包是一个非常强大的package，可以在数据库nomount下使用，用于从RMAN备份集中读取各类文件。

```sql
DECLARE
    devtype varchar2(256);
    done boolean;
    BEGIN
    devtype:=sys.dbms_backup_restore.deviceAllocate (type=>'',ident=>'t1');
    sys.dbms_backup_restore.restoreSetDatafile;
    sys.dbms_backup_restore.restoreControlfileTo(cfname=>'/backup/test/control01.ctl');
    sys.dbms_backup_restore.restoreBackupPiece(done=>done,handle=>'/backup/full/itpux19_ctl_db01_3_1_919281645',params=>null);
    sys.dbms_backup_restore.deviceDeallocate;
   END;
/ 
```

* 2.恢复数据文件

```sql
select 'sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>' || file# ||
       ',toname=>' ||chr(39)|| name ||chr(39) || ');',
       'sys.dbms_backup_restore.applySetDatafile(dfnumber=>' || file# ||
       ',toname=>' ||chr(39)|| name ||chr(39) || ');'
  from v$datafile; 
  
 
 DECLARE
    devtype varchar2(256);
    done boolean;
    BEGIN
    devtype:=sys.dbms_backup_restore.deviceAllocate (type=>'',ident=>'t1');
	  sys.dbms_backup_restore.restoreSetDatafile;
	  sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>1,toname=>'/backup/test/system01.dbf');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>2,toname=>'/backup/test/sysaux01.dbf');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>3,toname=>'/backup/test/undotbs01.dbf');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>4,toname=>'/backup/test/users01.dbf');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>5,toname=>'/backup/test/example01.dbf');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>6,toname=>'/backup/test/scdata01.ora');
    sys.dbms_backup_restore.restoreDatafileTo(dfnumber=>7,toname=>'/backup/test/rman.dbf');
    sys.dbms_backup_restore.restoreBackupPiece(done=>done,handle=>'/backup/full/itpux19_full_db01_1_1_919278149', params=>null);
    sys.dbms_backup_restore.deviceDeallocate;
 END;
 /

```

* 3.恢复归档日志文件

```sql
---从备份集中还原归档日志
declare
devtype varchar2(256);
done boolean;
begin
devtype := dbms_backup_restore.DeviceAllocate (type =>'',ident => 't1');
dbms_backup_restore.RestoreSetArchivedLog(destination=>'/backu/test/archive');   --设置恢复对话
dbms_backup_restore.RestoreArchivedLog(thread =>1 ,sequence =>9);   --日志的线程号和日志号
dbms_backup_restore.RestoreBackupPiece(done => done,handle =>'/oracle/full/itpux19_pfile_db01_4_1_919281649', params => null);    --指定使用的备份集
dbms_backup_restore.DeviceDeallocate;
end;


---从备份集中按scn范围还原归档日志
declare
devtype varchar2(256);
done boolean;
begin
devtype := dbms_backup_restore.DeviceAllocate (type =>'',ident => 't1');
dbms_backup_restore.RestoreSetArchivedLog(destination=>'/backu/test/archive');   --设置恢复对话
dbms_backup_restore.RestoreArchivedLogRange(log_change =>1125933 ,high_change =>1126275);  --日志的线程号和日志号
dbms_backup_restore.RestoreBackupPiece(done => done,handle =>'/oracle/full/itpux19_pfile_db01_4_1_919281649', params => null);    --指定使用的备份集
dbms_backup_restore.DeviceDeallocate;
end;

```


## 3.使用RMAN的BlockRecovery恢复坏块
## 4. Oracle RMAN Recovery Advisor案列
## 5.RMAN 备份压缩案列
### 5.1修改参数开启/关闭备份压缩
* 1.开启备份压缩

```sql
--使用COMPRESSED启用RMAN备份压缩
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;
```

* 2.关闭备份压缩

```sql
--取消COMPRESSED取消RMAN备份压缩
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO BACKUPSET;
```
### 5.2直接使用命令开启/取消备份压缩
```sql
--对整个数据库压缩备份
BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
--对指定数据文件压缩备份
BACKUP AS COMPRESSED BACKUPSET DATAFILE 1,3,4;
```

## 6.RMAN 增量备份与恢复
## 7.RMAN 备份的加密
## 8. RMAN 克隆数据库测试
## 9.生产环境 RMAN 异机恢复场景
## 10.FlashBack

* 要求

```sql
1.版本大于等于10.2
2.数据必须开启归档
3.数据必须配置Flash recovery
4.
5.没有特殊要求就正常闪回
```
* 基本步骤

```sql
1.关闭数据库
2.startup mount[exclusive模式]
3.闪回  时间/SCN/还原点
4.open resetlogs/**;
4.1 打开有2种方法
	1.open resetlog 方法，整个数据库都都恢复到丢表之前的状态（假如丢表之后对其他表进行了某些操作也将丢失）
	2.open read only 只读打开，导标，完全恢复，在导入丢失的表。
	也可以先把备份放到其他的数据库恢复后把丢失的表导入的目标数据库
```

* 具体操作

```sql

--SQLPLUS
flashback database <db_name> to scn <scn>   --scn 
flashback database <db_name> to timestamp <timestamp> --time
flashback database to restore point <point_name> --restore point

--RMAN
flashback database to scn=<scn> --scn
flashback database to time="to_date('2018-01-30 15:00:41','YYYY-MM-DD HH24:MI:SS')"; --time
flashback database to sequence=88 thread=1; --sequence
```

### 10.1.基于SCN的闪回
```sql

SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    1349018

SQL> select * from v$flashback_database_log;

OLDEST_FLASHBACK_SCN OLDEST_FLASHBACK_TI RETENTION_TARGET FLASHBACK_SIZE
-------------------- ------------------- ---------------- --------------
ESTIMATED_FLASHBACK_SIZE
------------------------
             1345507 2018-01-30 13:18:09             1440      104857600
               327008256
14:13:37 SQL> select * from test;

        ID NAME
---------- --------------------
         1 Anonycurse01
         2 Anonycurse02
         3 Anonycurse03
         4 Anonycurse04
         5 Anonycurse05
         6 Anonycurse06
         7 Anonycurse07

7 rows selected.

14:13:47 SQL> delete from test where id=7;

1 row deleted.

14:14:06 SQL> commit;

Commit complete.

SQL> startup mount;
ORACLE instance started.

Total System Global Area  835104768 bytes
Fixed Size                  2257840 bytes
Variable Size             281021520 bytes
Database Buffers          549453824 bytes
Redo Buffers                2371584 bytes
Database mounted.
SQL> flashback database orcl to scn   1349018;

```

### 10.2.基于时间的闪回
* 步骤

```sql
1.数据库启动到MOUNT，恢复到制定时间点
2.把数据的打开导只读模式
3.把表导出
4.重启数据库到MOUNT后恢复数据库到最新
4.正常打开后导入表
```
* 操作

```sql
SQL> set time on
15:00:12 SQL> select * from test;

        ID NAME
---------- --------------------
         1 Anonycurse01
         2 Anonycurse02
         3 Anonycurse03
         4 Anonycurse04
         5 Anonycurse05
         6 Anonycurse06
         7 Anonycurse07

7 rows selected.

15:00:17 SQL> create table test01 as select * from test;

Table created.

15:00:41 SQL> select * from test01;

        ID NAME
---------- --------------------
         1 Anonycurse01
         2 Anonycurse02
         3 Anonycurse03
         4 Anonycurse04
         5 Anonycurse05
         6 Anonycurse06
         7 Anonycurse07

7 rows selected.

15:00:48 SQL> commit;

Commit complete.

15:00:57 SQL> drop table test01;

Table dropped.

15:01:10 SQL> drop table test;

Table dropped.

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.

Total System Global Area  835104768 bytes
Fixed Size                  2257840 bytes
Variable Size             281021520 bytes
Database Buffers          549453824 bytes
Redo Buffers                2371584 bytes
Database mounted.

RMAN> flashback database to time="to_date('2018-01-30 15:00:41','YYYY-MM-DD HH24:MI:SS')";

Starting flashback at 2018-01-30 15:04:25
using channel ORA_DISK_1


starting media recovery
media recovery complete, elapsed time: 00:00:07

Finished flashback at 2018-01-30 15:04:32

SQL> alter database open read only;

Database altered.

[oracle@orcl:/u01]$exp anonycurse/jia tables=test01 file=/u01/test01.dmp log=/u01/test01.log 

Export: Release 11.2.0.4.0 - Production on Tue Jan 30 15:22:07 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

  

Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
Export done in ZHS16GBK character set and UTF8 NCHAR character set

About to export specified tables via Conventional Path ...
. . exporting table                         TEST01          7 rows exported
Export terminated successfully without warnings.


SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.

Total System Global Area  835104768 bytes
Fixed Size                  2257840 bytes
Variable Size             281021520 bytes
Database Buffers          549453824 bytes
Redo Buffers                2371584 bytes
Database mounted.
SQL> recovery database;
SP2-0734: unknown command beginning "recovery d..." - rest of line ignored.
SQL> recover database;
Media recovery complete.

```

### 10.4 Restore point in Oracle（Normal）
* 基本使用

```sql
--1.创建还原点
create restore point test_normal;
--2.查看还原点
select name,time,storage_size from v$restore_point;
--3.删除还原点
drop restore point test_normal;

```

* 步骤

```sql
1.创建还原点
2.执行某操作后需要还原
3.执行还原
4.OPEN RESETLOGS打开数据库
5.删除还原点
```

* 操作

```sql
SQL> create restore point test_normal;

Restore point created.

SQL> select name,time,storage_size from v$restore_point;

NAME
--------------------------------------------------------------------------------
TIME
---------------------------------------------------------------------------
STORAGE_SIZE
------------
TEST_NORMAL
30-JAN-18 03.46.27.000000000 PM
           0
		   

SQL> select table_name from user_tables;

TABLE_NAME
------------------------------
TEST01
TEST02

SQL> drop table test01 purge;

Table dropped.

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount;
ORACLE instance started.

Total System Global Area  835104768 bytes
Fixed Size                  2257840 bytes
Variable Size             281021520 bytes
Database Buffers          549453824 bytes
Redo Buffers                2371584 bytes
Database mounted.
SQL> flashback database to restore point test_normal;

Flashback complete.

SQL> alter database open  resetlogs;

Database altered.


SQL> select table_name from user_tables;

TABLE_NAME
------------------------------
TEST01
TEST02

```

### 10.5 Restore point in Oracle（Guaranteed）
* 基本使用

```sql
--1.创建还原点
create restore point test_guaranteed Guarantee flashback database;
--2.
--3.

```
* 步骤

```sql
和Normal基本一致
```

* 操作

```sql
flashback database to restore point test_guaranteed;

```

## 10.3 日常监控
```sql
--1.监控flashback
SELECT * FROM V$FLASH_RECOVERY_AREA_USAGE;
SELECT * FROM V$RECOVERY_FILE_DEST;

--2.清理空间
rm datafile.dbf

RMAN>
crosscheck backup;
crosscheck archivelog all;
delete expired backup;
delete expired archivelog all;
delete force obsolete;

--3.闪回点清理
SELECT * FROM v$resore_point;
drop restore point <point_name>;

--4.相关视图
SELECT * FROM DICTIONARY WHERE table_name like '%FLASH%';

v$database  
	SELECT FLASHBACK_ON FROM v$database;
v$flashback_database_log
	SELECT * FROM v$flashback_database_log;
v$flashback_database_stat
    SELECT * FROM v$flashback_database_stat;
V$FLASH_RECOVERY_AREA_USAGE
    SELECT * FROM V$FLASH_RECOVERY_AREA_USAGE;
V$RECOVERY_FILE_DEST
    SELECT * FROM V$RECOVERY_FILE_DEST;

```

## 11.回收站管理
```sql
SELECT * FROM dba_recyclebin;
drop table test purge; 
--清空回收站
purge dba_recyclebin;	--All User
purge recyclebin;	--Current User
drop table test purge;	--Skip

purge tablespace tbs_name;
purge tablespace tbs_name user user_name;
purge index BIN$Y/nEtmKPVY3gUwEAAH92GA==$0;

--回收站还原表
--FlashBack
flash table "BIN$Y/nEtmKPVY3gUwEAAH92GA==$0" to before drop;
flashback table test to before drop;
flashback table test before drop rename to test-new;

--Change Name
SELECT table_name,table_owner,indedx_name from dba_indexes WHERE table_owner='anonycurse';
SELECT table_name,constraint_name from dba_constraints WHERE table_name='test01';

alter table test01 rename constraint "BIN$Y5intw3VMY3gUwEAAH+aYw==$0"  rename to test_id_pk;
alter index "BIN$Y5intw3VMY3gUwEAAH+aYw==$0" rename to test_pk;

```