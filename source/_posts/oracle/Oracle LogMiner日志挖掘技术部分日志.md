---
title: Oracle LogMiner日志挖掘技术部分日志
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
  - LogMiner
  - 日志分析
---

# Oracle LogMiner日志挖掘技术部分日志

[Oracle 11G R2 官方文档][1] 

## 1.Logminer相关概念


## 2.Logminer使用详解

### 2.1 安装logminer
```sql
$ORACLE_HOME/rdbms/admin/dbmslm.sql : DBMS_LOGMNR
$ORACLE_HOME/rdbms/admin/dbmslmd.sql :DBMS_LOGMNR_D
$ORACLE_HOME/rdbms/admin/dbmslms.sql
--过程
dbms_logmnr_d.build('dict.ora','/u01') --创建一个数据字典文件
dbms_logmnr.add_logfile
dbms_logmnr.start_logmnr
dbms_logmnr.end_logmnr
--视图
select * from v$logmnr_dictionary;
select * from v$logmnr_logs;
select * from v$logmnr_contents;
```
### 2.2  使用源数据库的数据字典（Online catalog)来分析DML操作
```sql
--01.开启补充日志
select SUPPLEMENTAL_LOG_DATA_MIN from v$database;
alter database add supplemental log data;
--02.建立日志分析列表
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.new)
--继续添加
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.addfile)
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.addfile)
//execute dbms_logmnr.add_logfile('日志文件'，dbms_logmnr.addfile)
--移除
execute dbms_logmnr.remove_logfile(logfilename=>'日志文件')
--03.启动分析
execute dbms_logmnr.start_logmnr(Options => dbms_logmnr.dict_from_online_catalog)
//execute dbms_logmnr.start_logmnr(Options => dbms_logmnr.dict_from_online_catalog,startscn=>123,endScn => 124);
//exec dbms_logmnr.start_logmnr(Options => dbms_logmnr.dict_from_online_catalog,
starttime => to_date('2016-08-15 00:00:00','YYYY-MM-DD HH24:MI:SS'),
endtime => to_date('2016-08-15 01:00:00','YYYY-MM-DD HH24:MI:SS');
--04.查看日志分析结果
select username,scn,timestamp,sql_redo,sql_undo from v$logmnr_contents;
--05.结束分析
dbms_logmnr.end_logmnr;
```
### 2.3 使用LogMiner字典到字典文件来分析DDL操作
```sql
--01.提取logminer字典
--设置一个字典文件路径：
show parameter utl_file_dir --需要重启DB
alter system set utl_file_dir='/oracle' scope=spfile;
--创建一个数据字典文件
exec dbms_logmnr_d.build('dict.ora','/oracle');
--02.建立日志分析列表
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.new)
--继续添加
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.addfile)
execute dbms_logmnr.add_logfile(logfilename=>'日志文件'，options=>dbms_logmnr.addfile)
//execute dbms_logmnr.add_logfile('日志文件'，dbms_logmnr.addfile)
--移除
execute dbms_logmnr.remove_logfile(logfilename=>'日志文件')
--03.启动分析
exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora');---无条件分析
//exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora',startscn=>123,endScn => 124); --有条件分析
//exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora',
starttime => to_date('2016-08-15 00:00:00','YYYY-MM-DD HH24:MI:SS'),
endtime => to_date('2016-08-15 01:00:00','YYYY-MM-DD HH24:MI:SS');
--有条件分析：
scn: startscn,endScn
time: starttime,endtime
--04.查看日志分析结果
select username,scn,timestamp,sql_redo,sql_undo from v$logmnr_contents;
--05.结束分析
dbms_logmnr.end_logmnr;
```
### 2.4使用LogMiner进行日志分析
```sql
exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora');---无条件分析
//exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora',startscn=>123,endScn => 124); --有条件分析
//exec dbms_logmnr.start_logmnr(DictFileName => '/oracle/dict.ora',
starttime => to_date('2016-08-15 00:00:00','YYYY-MM-DD HH24:MI:SS'),
endtime => to_date('2016-08-15 01:00:00','YYYY-MM-DD HH24:MI:SS');
--有条件分析：
scn: startscn,endScn
time: starttime,endtime
```
### 2.5 查看logminer分析结果
```sql
select username,scn,timestamp,sql_redo,sql_undo from v$logmnr_contents;
SQL> desc v$logmnr_contents;
名称 类型
----------------------------------------- ----------------------------
TIMESTAMP DATE //SQL执行时间
COMMIT_TIMESTAMP DATE //事务提交时间
SEG_OWNER VARCHAR2(32) //被修改对象创建者
SEG_NAME VARCHAR2(256) //被修改对象的名字，如表名
SEG_TYPE NUMBER //被修改对象类型
SEG_TYPE_NAME VARCHAR2(32) //被修改对象类型名
TABLE_SPACE VARCHAR2(32) //被修改对象所属表空间
ROW_ID VARCHAR2(19) //被修改行的ROWID，如果
SESSION# NUMBER //执行修改的SESSION号
SERIAL# NUMBER //执行修改的SESSION序号
USERNAME VARCHAR2(30) //执行事务的用户名
SESSION_INFO VARCHAR2(4000) //执行修改的SESSION信息,例如：login_username= client_info= OS_username=SYSTEM Machine_name=ZFMISERVER OS_termi
nal=ZFMISERVER OS_process_id=1812 OS_program name=ORACLE.EXE
TX_NAME VARCHAR2(256) //执行的事务名，当该事务被命名时
ROLLBACK NUMBER //回滚标记
OPERATION VARCHAR2(32) //操作类型
INSERT
UPDATE
DELETE
DDL
START
COMMIT
ROLLBACK
LOB_WRITE
LOB_TRIM
LOB_ERASE
SELECT_FOR_UPDATE
SEL_LOB_LOCATOR
MISSING_SCN
INTERNAL
UNSUPPORTED
OPERATION_CODE NUMBER //操作类型代码
	0 = INTERNAL
	1 = INSERT
	2 = DELETE
	3 = UPDATE
	5 = DDL
	6 = START
	7 = COMMIT
	9 = SELECT_LOB_LOCATOR
	10 = LOB_WRITE
	11 = LOB_TRIM
	25 = SELECT_FOR_UPDATE
	28 = LOB_ERASE
	34 = MISSING_SCN
	36 = ROLLBACK
	255 = UNSUPPORTED
SQL_REDO VARCHAR2(4000) //重做日志SQL
SQL_UNDO VARCHAR2(4000) //相反操作SQL
SEQUENCE# NUMBER //重做日志的序号
```


## 3.logminer日志挖掘案例1-分析生产系统表数据丢失的原因
```sql
--3.1 安装logminer
@$ORACLE_HOME/rdbms/admin/dbmslm.sql
@$ORACLE_HOME/rdbms/admin/dbmslmd.sql
@$ORACLE_HOME/rdbms/admin/dbmslms.sql
--3.2、使用LogMiner字典到字典文件来分析DDL操作
--01.提取logminer字典
--设置一个字典文件路径：
show parameter utl_file_dir --需要重启DB
alter system set utl_file_dir='/u01' scope=spfile;
--创建一个数据字典文件
exec dbms_logmnr_d.build('dict.ora','/u01');
--02.建立日志分析列表
execute dbms_logmnr.add_logfile(logfilename=>'/u01/fast_recovery_area/ORCL/archivelog/2018_01_30/o1_mf_1_42_f7043xy4_.arc',options=>dbms_logmnr.new);
execute dbms_logmnr.add_logfile(logfilename=>'/u01/fast_recovery_area/ORCL/archivelog/2018_01_30/o1_mf_1_41_f7043xtq_.arc',options=>dbms_logmnr.addfile);
--03.启动分析
exec dbms_logmnr.start_logmnr(DictFileName => '/u01/dict.ora');
--04.查看日志分析结果
select username,scn,timestamp,sql_redo,sql_undo from v$logmnr_contents;
create table test01.logmnr_temp nologging as select * from v$logmnr_contents;
select * from itpux01.logmnr_temp where seg_owner='test01' and seg_name='TEST01';
select * from itpux01.logmnr_temp where seg_owner='test01' and seg_name='TEST01';
--05.结束分析
exec dbms_logmnr.end_logmnr;

```

## 4.logminer日志挖掘案例2-恢复DML误操作导致的表数据丢失


## 5.logminer日志挖掘案例3-RMAN表空间基于时间点的自动恢复


## 6.logminer使用总结