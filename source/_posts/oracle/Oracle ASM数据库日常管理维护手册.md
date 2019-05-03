---
title: Oracle ASM数据库日常管理维护手册
date: 2018-10-02 14:25:00
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
  - SQL
  - DML
---


# Oracle ASM数据库日常管理维护

[Oracle 11G R2 官方文档][1]

## 1.单机从ASM到文件系统
### 1.1  前期准备
* 01.收集信息

```sql
--1.查看数据文件
SELECT name FROM v$datafile;
SELECT file_name FROM dba_data_files;
--2.查看日志文件
SELECT MEMBER FROM v$logfile;
--3.查看临时文件
SELECT file_name FROM dba_temp_files;
--4.查看控制文件
SELECT * FROM v$controlfile;
```
* 02.备份数据库

```sql
1.RMAN 备份
backup tag racdb_full format '/u01/backup/datafile/racdb_full_%s_%p_%t' (database);
backup tag racdb_ctrl format '/u01/backup/datafile/racdb_ctrl_%s_%p_%t' (current controlfile);
backup tag racdb_pfile format '/u01//backup/datafile/racdb_spfile_%s_%p_%t' (spfile);

--2.SQLPLUS
alter database backup controlfile to '/u01/backup/datafile/control.ctl';
 alter database backup controlfile to trace as  '/u01/backup/datafile/control.trc';
```

* 03.准备目录等环境

```sql

```

* 04.开始迁移

```sql
--1.控制文件
 alter system set control_files='/oracle/oradata/racdb/control01.ctl' scope=spfile;
--2.参数文件
alter system set db_create_file_dest='/oracle/oradata/racdb/' scope=spfile;
--3.归档日志
show parameter recover；
alter system set db_recovery_file_dest='/oracle/recovery' scope=spfile;
--4.将参数文件导出(需要手动删除ASM信息)
create pfile='/oracle/racdbpfile.ora' from spfile;

--5.关机后用新的PFILE启动数据库到NOMOUNT
shutdown immediate
startup pfile='/oracle/racdbpfile.ora' nomount;
--6.根据现在的PFILE生成SPFILE
create spfile from pfile='/oracle/racdbpfile.ora';
--7.关闭数据库后用SPFILE启动数据库
shutdown immediate
startup nomount
show parameter spfile;
--8.RMAN恢复控制文件
restore controlfile from '+DG_DATA/RACDB/CONTROLFILE/Current.256.966890297';
--10.挂载数据库
alter database mount;
--11.RMAN将数据文件从ASM到文件系统
backup as copy database format '/oracle/oradata/racdb/%U';
switch database to copy;
--12.恢复数据库(不恢复)
 recover database using backup controlfile until cancel;
--13.打开数据库
alter database open resetlogs;
--14.处理临时文件（删掉再添加）
select file_name from dba_temp_files;
alter database tempfile '+DG_DATA/racdb/tempfile/temp.263.966890331' drop including datafiles;
alter tablespace temp add tempfile '/oracle/oradata/racdb/temp01.dbf' size 100m autoextend off;
--注意：如果临时文件删不掉可以重启数据库再删

--15.处理日志
select member from v$log;
alter database drop logfile group 3;
alter database add logfile group 2 ('/oracle/oradata/racdb/redo01.log') size 100m;
 alter system checkpoint;
--注意:这个步骤再文件系统和ASM中相互转换
--16.添加控制文件（重启后生效）
alter system set control_files='/oracle/oradata/racdb/control01.ctl','/oracle/recovery/racdb/control02.ctl'scope=spfile;
--17.修改数据文件名字
mv data_D-RACDB_I-964507061_TS-SYSTEM_FNO-1_0ksq4vvu	SYSTEM01.dbf
mv data_D-RACDB_I-964507061_TS-SYSAUX_FNO-2_0lsq5012	SYSAUX01.dbf
mv data_D-RACDB_I-964507061_TS-UNDOTBS1_FNO-3_0jsq4vur	UNDOTBS01.dbf
mv data_D-RACDB_I-964507061_TS-USERS_FNO-4_0msq501r    USERS01.dbf

alter database rename file '/oracle/oradata/racdb/data_D-RACDB_I-964507061_TS-SYSTEM_FNO-1_0ksq4vvu' 
to '/oracle/oradata/racdb/SYSTEM01.dbf';
alter database rename file '/oracle/oradata/racdb/data_D-RACDB_I-964507061_TS-SYSAUX_FNO-2_0lsq5012' 
to '/oracle/oradata/racdb/SYSAUX01.dbf';
alter database rename file '/oracle/oradata/racdb/data_D-RACDB_I-964507061_TS-UNDOTBS1_FNO-3_0jsq4vur' 
to '/oracle/oradata/racdb/UNDOTBS01.dbf';
alter database rename file '/oracle/oradata/racdb/data_D-RACDB_I-964507061_TS-USERS_FNO-4_0msq501r' 
to '/oracle/oradata/racdb/USERS01.dbf';
```
* 转换前文件目录

```sql
SELECT name FROM v$datafile;
SELECT file_name FROM dba_data_files;

+DG_DATA/racdb/datafile/system.260.966890307
+DG_DATA/racdb/datafile/sysaux.261.966890319
+DG_DATA/racdb/datafile/undotbs1.262.966890327
+DG_DATA/racdb/datafile/users.264.966890339

SELECT MEMBER FROM v$logfile;

+DG_DATA/racdb/onlinelog/group_1.257.966890301
+DG_BACKUP/racdb/onlinelog/group_1.257.966890301
+DG_DATA/racdb/onlinelog/group_2.258.966890303
+DG_BACKUP/racdb/onlinelog/group_2.258.966890303
+DG_DATA/racdb/onlinelog/group_3.259.966890305
+DG_BACKUP/racdb/onlinelog/group_3.259.966890305

SELECT file_name FROM dba_temp_files;

+DG_DATA/racdb/tempfile/temp.263.966890331

SELECT * FROM v$controlfile;

        +DG_DATA/racdb/controlfile/cur NO       16384           2924
        rent.256.966890297

        +DG_BACKUP/racdb/controlfile/c YES      16384           2924
        urrent.256.966890299
```
* 转换后文件目录

![数据文件][2]

![日志文件][3]

![临时文件][4]

![控制文件][5]

## 2.单机从ASM到文件系统
* 步骤

```sql
1.利用PFILE和SPFILE将一个包含控制文件文字的SPFILE存到ASM
2.RMAN将备份的控制文件恢复到指定目录
3.恢复数据文件
4.处理日志文件
```
* 操作

```sql
SQL>  alter system set control_files='+DG_DATA' scope=spfile;

System altered.

SQL> alter system set db_create_file_dest='+DG_DATA/' scope=spfile;

System altered.

SQL> alter system set db_recovery_file_dest='+DG_BACKUP' scope=spfile;

System altered.

SQL> alter system set control_files='+DG_DATA/racdb/control01.ctl' scope=spfile;

System altered.

SQL> create pfile '/oracle/racdbpfile.ora' from spfile;
create pfile '/oracle/racdbpfile.ora' from spfile
             *
ERROR at line 1:
ORA-00923: FROM keyword not found where expected


SQL> create pfile='/oracle/racdbpfile.ora' from spfile;

File created.

SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> 
SQL> startup pfile='/oracle/racdbpfile.ora' nomount;
ORACLE instance started.

Total System Global Area 1043886080 bytes
Fixed Size                  2259840 bytes
Variable Size             335545472 bytes
Database Buffers          700448768 bytes
Redo Buffers                5632000 bytes
SQL> create spfile from pfile='/oracle/racdbpfile.ora';

File created.

SQL> create spfile='+DG_DATA/racdb/spfileracdb.ora' from pfile='/oralce/racdbpfile.ora'
  2  ;
create spfile='+DG_DATA/racdb/spfileracdb.ora' from pfile='/oralce/racdbpfile.ora'
*
ERROR at line 1:
ORA-01078: failure in processing system parameters
LRM-00109: could not open parameter file '/oralce/racdbpfile.ora'


SQL> create spfile='+DG_DATA/racdb/spfileracdb.ora' from pfile='/oralce/racdbpfile.ora'
  2  ;
create spfile='+DG_DATA/racdb/spfileracdb.ora' from pfile='/oralce/racdbpfile.ora'
*
ERROR at line 1:
ORA-01078: failure in processing system parameters
LRM-00109: could not open parameter file '/oralce/racdbpfile.ora'


SQL> create spfile='+DG_DATA/racdb/spfileracdb.ora' from pfile='/oracle/racdbpfile.ora';

File created.

SQL> shutdown immediate;
ORA-01507: database not mounted


ORACLE instance shut down.
SQL> startup nomount;
ORACLE instance started.

Total System Global Area 1043886080 bytes
Fixed Size                  2259840 bytes
Variable Size             335545472 bytes
Database Buffers          700448768 bytes
Redo Buffers                5632000 bytes
SQL> show parameter spfile
SQL> show parameter control

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
control_file_record_keep_time        integer                7
control_files                        string                 +DG_DATA/racdb/control01.ctl
control_management_pack_access       string                 DIAGNOSTIC+TUNING


[oracle@rac1:/]$rman target /

Recovery Manager: Release 11.2.0.4.0 - Production on Thu Feb 1 15:37:25 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

connected to target database: RACDB (not mounted)

RMAN> restore controlfile from '/u01/backup/datafile/racdb_ctrl_32_1_966957454';

Starting restore at 2018-02-01 15:37:57
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=10 device type=DISK

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
output file name=+DG_DATA/racdb/control01.ctl
Finished restore at 2018-02-01 15:38:01

SQL> alter database mount;

Database altered.


RMAN> list backup;

released channel: ORA_DISK_1

List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
24      Full    1.05G      DISK        00:00:15     2018-02-01 15:17:28
        BP Key: 24   Status: AVAILABLE  Compressed: NO  Tag: RACDB_FULL
        Piece Name: /u01/backup/datafile/racdb_full_30_1_966957433
  List of Datafiles in backup set 24
  File LV Type Ckp SCN    Ckp Time            Name
  ---- -- ---- ---------- ------------------- ----
  1       Full 874708     2018-02-01 15:17:14 /oracle/oradata/racdb/SYSTEM01.dbf
  2       Full 874708     2018-02-01 15:17:14 /oracle/oradata/racdb/SYSAUX01.dbf
  3       Full 874708     2018-02-01 15:17:14 /oracle/oradata/racdb/UNDOTBS01.dbf
  4       Full 874708     2018-02-01 15:17:14 /oracle/oradata/racdb/USERS01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time    
------- ---- -- ---------- ----------- ------------ -------------------
25      Full    46.05M     DISK        00:00:01     2018-02-01 15:17:30
        BP Key: 25   Status: AVAILABLE  Compressed: NO  Tag: TAG20180201T151729
        Piece Name: +DG_BACKUP/racdb/autobackup/2018_02_01/s_966957449.256.966957449
  SPFILE Included: Modification time: 2018-02-01 14:40:33
  SPFILE db_unique_name: RACDB
  Control File Included: Ckp SCN: 874719       Ckp time: 2018-02-01 15:17:29
  
  List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    750      SYSTEM               ***     /oracle/oradata/racdb/SYSTEM01.dbf
2    600      SYSAUX               ***     /oracle/oradata/racdb/SYSAUX01.dbf
3    885      UNDOTBS1             ***     /oracle/oradata/racdb/UNDOTBS01.dbf
4    5        USERS                ***     /oracle/oradata/racdb/USERS01.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    100      TEMP                 100         /oracle/oradata/racdb/temp01.dbf

RMAN> run{
2> set newname for datafile 1 to "+DG_DATA";
3> set newname for datafile 2 to "+DG_DATA";
4> set newname for datafile 3 to "+DG_DATA";
5> set newname for datafile 4 to "+DG_DATA";
6> set newname for tempfile 1 to "+DG_DATA";
7> restore database;
8> switch datafile all;
9> recover database;
10> }

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

Starting restore at 2018-02-01 15:44:20
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to +DG_DATA
channel ORA_DISK_1: restoring datafile 00002 to +DG_DATA
channel ORA_DISK_1: restoring datafile 00003 to +DG_DATA
channel ORA_DISK_1: restoring datafile 00004 to +DG_DATA
channel ORA_DISK_1: reading from backup piece /u01/backup/datafile/racdb_full_30_1_966957433
channel ORA_DISK_1: piece handle=/u01/backup/datafile/racdb_full_30_1_966957433 tag=RACDB_FULL
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:45
Finished restore at 2018-02-01 15:45:05

datafile 1 switched to datafile copy
input datafile copy RECID=15 STAMP=966959106 file name=+DG_DATA/racdb/datafile/system.261.966959061
datafile 2 switched to datafile copy
input datafile copy RECID=16 STAMP=966959106 file name=+DG_DATA/racdb/datafile/sysaux.260.966959061
datafile 3 switched to datafile copy
input datafile copy RECID=17 STAMP=966959106 file name=+DG_DATA/racdb/datafile/undotbs1.262.966959061
datafile 4 switched to datafile copy
input datafile copy RECID=18 STAMP=966959106 file name=+DG_DATA/racdb/datafile/users.265.966959061

Starting recover at 2018-02-01 15:45:08
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 33 is already on disk as file /oracle/oradata/racdb/redo03.log
archived log file name=/oracle/oradata/racdb/redo03.log thread=1 sequence=33
media recovery complete, elapsed time: 00:00:01
Finished recover at 2018-02-01 15:45:11


--这里不知道为什么控制文件也帮我恢复了
ASMCMD [+DG_DATA/racdb] > ls
CONTROLFILE/
DATAFILE/
PARAMETERFILE/
control01.ctl
spfileracdb.ora
ASMCMD [+DG_DATA/racdb] > cd datafile
ASMCMD [+DG_DATA/racdb/datafile] > ls
SYSAUX.260.966959061
SYSTEM.261.966959061
UNDOTBS1.262.966959061
USERS.265.966959061
ASMCMD [+DG_DATA/racdb/datafile] >  

QL> alter database add logfile group 3 ('+DG_DATA') size 100m;

Database altered.

SQL> alter database add logfile group 4 ('+DG_DATA') size 100m;

Database altered.

SQL> alter system switch logfile;

System altered.

SQL> select member from v$log;
select member from v$log
       *
ERROR at line 1:
ORA-00904: "MEMBER": invalid identifier


SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS         FIRST_CHANGE# FIRST_TIME   NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1          1  104857600        512          1 YES ACTIVE        875061 2018-02-01 15:50:41       876161 2018-02-01 16:02:43
         2          1          2  104857600        512          1 NO  CURRENT       876161 2018-02-01 16:02:43   2.8147E+14
         3          1          0  104857600        512          1 YES UNUSED     0                                 0
         4          1          0  104857600        512          1 YES UNUSED     0                                 0

SQL> alter system switch logfile;

System altered.

SQL> alter system switch logfile;

System altered.

SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS         FIRST_CHANGE# FIRST_TIME   NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1          1  104857600        512          1 YES ACTIVE        875061 2018-02-01 15:50:41       876161 2018-02-01 16:02:43
         2          1          2  104857600        512          1 YES ACTIVE        876161 2018-02-01 16:02:43       876177 2018-02-01 16:03:03
         3          1          3  104857600        512          1 YES ACTIVE        876177 2018-02-01 16:03:03       876180 2018-02-01 16:03:04
         4          1          4  104857600        512          1 NO  CURRENT       876180 2018-02-01 16:03:04   2.8147E+14

SQL> alter system checkpoint;

System altered.

SQL> alter system checkpoint;

System altered.

SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS         FIRST_CHANGE# FIRST_TIME   NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1          1  104857600        512          1 YES INACTIVE      875061 2018-02-01 15:50:41       876161 2018-02-01 16:02:43
         2          1          2  104857600        512          1 YES INACTIVE      876161 2018-02-01 16:02:43       876177 2018-02-01 16:03:03
         3          1          3  104857600        512          1 YES INACTIVE      876177 2018-02-01 16:03:03       876180 2018-02-01 16:03:04
         4          1          4  104857600        512          1 NO  CURRENT       876180 2018-02-01 16:03:04   2.8147E+14

SQL> alter system drop logfile group 1;
alter system drop logfile group 1
             *
ERROR at line 1:
ORA-02065: illegal option for ALTER SYSTEM


SQL> alter database drop logfile group 1;

Database altered.

SQL> alter database drop logfile group 2;

Database altered.

SQL> 
SQL> 
SQL> 
SQL> 
SQL> 
SQL> 
SQL> 
SQL> alter database add logfile group 2 ('+DG_DATA') size 100m;

Database altered.

SQL> alter database add logfile group 1('+DG_DATA') size 100m;

Database altered.

SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS         FIRST_CHANGE# FIRST_TIME   NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1          0  104857600        512          1 YES UNUSED     0                                 0
         2          1          0  104857600        512          1 YES UNUSED     0                                 0
         3          1          3  104857600        512          1 YES INACTIVE      876177 2018-02-01 16:03:03       876180 2018-02-01 16:03:04
         4          1          4  104857600        512          1 NO  CURRENT       876180 2018-02-01 16:03:04   2.8147E+14

SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS         FIRST_CHANGE# FIRST_TIME   NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1          0  104857600        512          1 YES UNUSED     0                                 0
         2          1          0  104857600        512          1 YES UNUSED     0                                 0
         3          1          3  104857600        512          1 YES INACTIVE      876177 2018-02-01 16:03:03       876180 2018-02-01 16:03:04
         4          1          4  104857600        512          1 NO  CURRENT       876180 2018-02-01 16:03:04   2.8147E+14

SQL> alter system switch logfile;

System altered.


SQL> /

System altered.

SQL> alter system checkpoint
  2  ;

System altered.

```