---
title: Oracle Goldengate数据库复制与容灾实施
date: 2018-09-07 09:25:00
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
  - GoldenGate
  - 备份容灾
---

# OracleGoldengate数据库复制与容灾实施

[Oracle 11G R2 官方文档][1]

## 1.GoldenGate 文件系统-文件系统

### 1.1 安装OGG软件(源端-目标端)

```sql

```

### 1.2 创建OGG用户(源端-目标端)

* 源端

```sql
CREATE TABLESPACE ogg_tbs
	DATAFILE '/u01/oradata/orcl/ogg_tbs.dbf'
		SIZE 100 M
	    AUTOEXTEND ON NEXT 10 M MAXSIZE 500 M
    LOGGING
    EXTENT MANAGEMENT LOCAL
    SEGMENT SPACE MANAGEMENT AUTO;


CREATE USER goldengate IDENTIFIED BY goldengate DEFAULT TABLESPACE ogg_tbs TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON users;

GRANT CONNECT,RESOURCE TO GOLDENGATE;
GRANT CREATE SESSION TO GOLDENGATE;
GRANT ALTER SESSION TO GOLDENGATE;
GRANT ALTER ANY TABLE TO GOLDENGATE;
GRANT ALTER SYSTEM TO GOLDENGATE;
GRANT CREATE TABLE TO GOLDENGATE;
GRANT INSERT ANY TABLE,UPDATE ANY TABLE,DELETE ANY TABLE,LOCK ANY TABLE TO GOLDENGATE;
GRANT SELECT ANY TRANSACTION TO GOLDENGATE;
GRANT SELECT ANY DICTIONARY TO GOLDENGATE;
GRANT FLASHBACK ANY TABLE TO GOLDENGATE;
GRANT UNLIMITED TABLESPACE TO GOLDENGATE;
GRANT EXECUTE on DBMS_FLASHBACK TO GOLDENGATE;
GRANT EXECUTE on DBMS_GOLDENGATE_AUTH TO GOLDENGATE;
GRANT DBA TO GOLDENGATE;
EXEC dbms_goldengate_auth.grant_admin_privilege('GOLDENGATE','*',TRUE)
```

* 目标端

```sql
CREATE TABLESPACE ogg_tbs
		DATAFILE '/u01/oradata/itpux/ogg_tbs.dbf'
		SIZE 100 M
    AUTOEXTEND ON NEXT 10 M MAXSIZE 500 M
    LOGGING
    EXTENT MANAGEMENT LOCAL
    SEGMENT SPACE MANAGEMENT AUTO;

CREATE USER goldengate IDENTIFIED BY goldengate DEFAULT TABLESPACE ogg_tbs TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON users;

GRANT CONNECT,RESOURCE TO GOLDENGATE;
GRANT CREATE SESSION TO GOLDENGATE;
GRANT ALTER SESSION TO GOLDENGATE;
GRANT ALTER ANY TABLE TO GOLDENGATE;
GRANT ALTER SYSTEM TO GOLDENGATE;
GRANT CREATE TABLE TO GOLDENGATE;
GRANT INSERT ANY TABLE,UPDATE ANY TABLE,DELETE ANY TABLE,LOCK ANY TABLE TO GOLDENGATE;
GRANT SELECT ANY TRANSACTION TO GOLDENGATE;
GRANT SELECT ANY DICTIONARY TO GOLDENGATE;
GRANT FLASHBACK ANY TABLE TO GOLDENGATE;
GRANT UNLIMITED TABLESPACE TO GOLDENGATE;
GRANT EXECUTE on DBMS_FLASHBACK TO GOLDENGATE;
GRANT EXECUTE on DBMS_GOLDENGATE_AUTH TO GOLDENGATE;
GRANT DBA TO GOLDENGATE;
EXEC dbms_goldengate_auth.grant_admin_privilege('GOLDENGATE','*',TRUE)
```

### 1.3 创建测试用户(源端-目标端)
* 源端

```sql
CREATE  TABLESPACE "KYLE"
    DATAFILE '/u01/app/oracle/oradata/orcl/kyle01.dbf'
        SIZE 100 M
        AUTOEXTEND ON NEXT 10 M MAXSIZE 500 M
    LOGGING
    EXTENT MANAGEMENT LOCAL
    SEGMENT SPACE MANAGEMENT AUTO;

CREATE USER "KYLE"
    PROFILE "DEFAULT" 
    IDENTIFIED BY "kyle" 
    DEFAULT TABLESPACE "KYLE" 
    TEMPORARY TABLESPACE "TEMP" 
    ACCOUNT UNLOCK;

--测试数据
SELECT table_name FROM user_tables;

CREATE SEQUENCE seq_kyle START WITH 1 INCREMENT BY 1;
CREATE TABLE kyle01 (id INT PRIMARY KEY, rand INT , name VARCHAR2(30));
CREATE TABLE kyle02 (id INT PRIMARY KEY, rand INT , name VARCHAR2(30));
CREATE TABLE kyle03 (id INT PRIMARY KEY, rand INT , name VARCHAR2(30));

INSERT INTO kyle01 VALUES(1,1,'Kyle01');
INSERT INTO kyle01 VALUES(2,2,'Kyle02');
INSERT INTO kyle01 VALUES(3,3,'Kyle03');
INSERT INTO kyle01 VALUES(4,4,'Kyle04');

DECLARE
  rnd number(9,2);
BEGIN
   for i in 1..20000 loop
     IF MOD(i,2000)=0 THEN
     	commit;
     ELSE
     	insert into kyle03 values(seq_kyle.nextval,i*dbms_random.value,'Kyle Is Testing');
     END IF;
   END LOOP;
END;
/

BEGIN
   LOOP
    delete from kyle03 where rownum=1;
     commit;
     insert into kyle03 values(seq_kyle.nextval,200000*dbms_random.value,'MACLEAN IS UPDATING');
     commit;
	 insert into kyle03 values(seq_kyle.nextval,300000*dbms_random.value,'MACLEAN IS UPDATING');
	 commit;
	update kyle03 set rand=rand+10 where rownum=1;
	commit;
     dbms_lock.sleep(1);
     END loop;
END;

/

	
```
* 目标端

```sql
CREATE  TABLESPACE "KYLE"
    DATAFILE '/u01/oradata/itpux/kyle01.dbf'
        SIZE 100 M
        AUTOEXTEND ON NEXT 10 M MAXSIZE 500 M
    LOGGING
    EXTENT MANAGEMENT LOCAL
    SEGMENT SPACE MANAGEMENT AUTO;

CREATE USER "KYLE"
    PROFILE "DEFAULT" 
    IDENTIFIED BY "kyle" 
    DEFAULT TABLESPACE "KYLE" 
    TEMPORARY TABLESPACE "TEMP" 
    ACCOUNT UNLOCK;
```
### 1.4 数据导入导出
```sql
create directory bakdir as '/home/oracle';
grant read,write on directory bakdir to system;
grant create any directory to system;

--导出结构
CREATE DIRECTORY BAKDIR AS '/home/oracle';
GRANT READ,WRITE ON DIRECTORY BAKDIR TO SYSTEM;
GRANT CREATE ANY DIRECTORY TO SYSTEM;
--导出某个或多个schema
expdp system/jia  DIRECTORY=bakdir DUMPFILE=expdp_schema_kyle.dmp LOGFILE=expdp_schema_kyle.log SCHEMAS=kyle
impdp system/jia  DIRECTORY=bakdir DUMPFILE=expdp_schema_kyle.dmp LOGFILE=expdp_schema_kyle.log TABLE_EXISTS_ACTION=truncate SCHEMAS=kyle
scp /home/oracle/expdp_schema_kyle.dmp orcl-122:/home/oracle/
```

### 1.5 配置环境变量
```sql
alias sqlplus="rlwrap sqlplus"
alias ggsci="rlwrap ggsci"
alias rman="rlwrap rman"
alias asmcmd="rlwrap asmcmd"

LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib; export LD_LIBRARY_PATH
CLASSPATH=/ggs:$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH
OGG_PATH=/u01/ggs export OGG_PATH
PATH=.:$PATH:$OGG_PATH:$HOME/bin:$ORACLE_BASE/product/11.2.0/db_1/bin:$ORACLE_HOME/bin; export PATH
```

### 1.6 修改系统参数开启归档
```sql
ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION = TRUE SCOPE=BOTH;
alter database add supplemental log data;
alter database force logging;
```

### 1.7 配置OGG(源端)
* 1.创建目录

```sql
create subdirs
```
* 2.配置MGR

```sql
GGSCI>
EDIT PARAMS mgr
PORT 7809
AUTOSTART ER * 
AUTORESTART ER *, RETRIES 3, WAITMINUTES 3, RESETMINUTES 15
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 2
*/
--端口 7809
--自动启动 ER(EXTARCT REPLACT)

```

* 3.配置检查点

```sql
--GLOBALS储存了运行的一些信息
GGSCI> EDIT PARAMS ./GLOBALS
CHECKPOINTTABLE goldengate.checkpoint
GGSCI> dblogin userid goldengate,password goldengate
GGSCI> ADD CHECKPOINTTABLE goldengate.checkpoint
```

* 4.添加补充日志

```sql
--ADD TRANDATA itpux.*
ADD SCHEMATRANDATA kyle
--配置ddl 的时候，一定要用ADD SCHEMATRANDATA
--如果不用ddl,可以用ADD TRANDATA
info SCHEMATRANDATA kyle
```

* 5.配置extract进程

```sql
--建立EXTRACT目录
mkdir -p ./dirdat/rkyle
mkdir -p ./dirrpt/rkyle
mkdir -p ./dirdat/ekyle
mkdir -p ./dirrpt/ekyle

GGSCI>
EDIT PARAMS ekyle

setenv(NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
setenv(ORACLE_SID="itpux")
EXTRACT ekyle
DDL INCLUDE ALL
DDLOPTIONS ADDTRANDATA,REPORT
USERID goldengate, PASSWORD goldengate
EXTTRAIL ./dirdat/ekyle/ex
TRANLOGOPTIONS excludeuser goldengate
TRANLOGOPTIONS convertucs2clobs
WARNLONGTRANS 12h,CHECKINTERVAL 30m
DISCARDFILE ./dirrpt/ekyle/ekyle.dsc, APPEND, MEGABYTES 200
TABLE kyle.*;

--设置语言
--设置SID
--设置EXTRACT名称不能操作过8个字符
--抽取所有的DDL操作
--
--定义OGG使用用户
--设置抽取目录
--排除用于管理抽取的OGG用户
--某某参数，可以传输大字段
--设置告警阈值
--配置文件存放位置
--抽取的表

--添加一个抽取进程
--THREADS 2表示源端有2个RAC节点
ADD EXTRACT ekyle, TRANLOG, BEGIN NOW
ADD EXTRACT ekyle, TRANLOG, BEGIN NOW, THREADS 2
--添加一个队列文件
GGSCI>
ADD EXTTRAIL ./dirdat/ekyle/ex, EXTRACT ekyle, MEGABYTES 200

--启动停止测试
GGSCI>
START ekyle
STOP ekyle
VIEW REPORT ekyle
```

* 6.配置PUMP进程

```sql
cd /u01/ggs
mkdir -p ./dirdat/ekyle
mkdir -p ./dirrpt/ekyle

GGSCI>
EDIT PARAMS pkyle

setenv(NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
setenv(ORACLE_SID="orcl")
EXTRACT pkyle
USERID goldengate,PASSWORD goldengate
PASSTHRU						
RMTHOST 172.17.0.122,MGRPORT 7809
RMTTRAIL ./dirdat/rkyle/re
DISCARDFILE ./dirrpt/rkyle/rkyle.dsc, APPEND, MEGABYTES 200
TABLE ekyle.*;

--
--
--
-- 禁止extract与数据库交互，适合于PUMP传输进程
-- 远端的IP,端口
-- 指定写入到远程目标端的哪个队列
-- 指定报错输出文件
-- 指定传输表


--增加pump 进程(指定本地trail 文件)
GGSCI>
ADD EXTRACT pkyle,EXTTRAILSOURCE ./dirdat/ekyle/ex

--增加rmttail 文件
GGSCI>
ADD RMTTRAIL ./dirdat/rkyle/re, EXTRACT pkyle, MEGABYTES 200
--检查:
GGSCI>
INFO  pkyle
```

### 1.8 配置OGG(目标端)
* 1.配置mgr

```sql
GGSCI> EDIT PARAMS MGR
PORT 7809
AUTOSTART ER * 
AUTORESTART ER *, RETRIES 3, WAITMINUTES 3, RESETMINUTES 15
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 2
*/
GGSCI>
START MGR
STOP MGR
```

* 2.配置检查点

```sql
GGSCI> EDIT PARAMS ./GLOBALS
CHECKPOINTTABLE goldengate.checkpoint
GGSCI> dblogin userid goldengate,password goldengate
GGSCI> ADD CHECKPOINTTABLE goldengate.checkpoint
```
* 3.配置replicat进程
```sql
cd /u01/ggs
mkdir -p ./dirdat/rkyle
mkdir -p ./dirrpt/rkyle

ggsci> edit params rkyle

setenv(NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
setenv(ORACLE_SID="itpux")
REPLICAT rkyle
USERID goldengate,PASSWORD goldengate
handlecollisions
assumetargetdefs
DISCARDFILE ./dirrpt/rkyle/rkyle.dsc, APPEND, MEGABYTES 200
MAP kyle.*, target kyle.*;

--添加replicat 进程
GGSCI>
ADD REPLICAT rkyle EXTTRAIL ./dirdat/rkyle/re, CHECKPOINTTABLE goldengate.checkpoint
--启动相关进程(目标)
GGSCI>
START rkyle
VIEW REPORT rkyle
INFO ALL
--启动相关进程(源端)
GGSCI>
START rkyle
START pkyle
VIEW REPORT rkyle
VIEW REPORT pkyle
INFO ALL
```

### 1.9 disable 目标库所有的trigger、cascading delete、check、job
```sql
SET PAGESIZE 2000
SET LINESIZE 100
--Foreign key Constraints/Cascading Deletes
SELECT    'alter table '
       || owner
       || '.'
       || table_name
       || ' DISABLE CONSTRAINT '
       || constraint_name
       || ';'
  FROM dba_constraints
 WHERE     constraint_type = 'R'
       AND delete_rule = 'CASCADE'
       AND owner IN ('KYLE',
                     'KYLE01');


--查找生成并disable 约束(Check)：
SELECT    'alter table '
       || owner
       || '.'
       || table_name
       || ' DISABLE CONSTRAINT '
       || constraint_name
       || ';'
  FROM dba_constraints
 WHERE     constraint_type = 'C'
       AND owner IN ('KYLE',
                     'KYLE01');

--查找生成并disable trigger：
SELECT 'alter trigger ' || owner || '.' || object_name || ' disable;'
  FROM dba_objects
 WHERE     object_type = 'TRIGGER'
       AND owner IN ('KYLE',
                     'KYLE01');


--查找并disable job
SELECT job,
       next_date,
       next_sec,
       failures,
       broken
  FROM dba_jobs
 WHERE SCHEMA_USER IN ('KYLE',
                       'KYLE01');

BEGIN
    sys.DBMS_JOB.broken (job => 21, broken => TRUE);
    COMMIT;
END;


--生成结果如下，然后在目标数据库中执行：
alter table ... DISABLE CONSTRAINT SYS_C0017353;
alter table ... DISABLE CONSTRAINT SYS_C0017355;…
…
alter trigger ... disable;
alter trigger ... disable;
alter trigger ... disable;
--确认外键已经被禁用
SELECT OWNER,
       CONSTRAINT_NAME,
       CONSTRAINT_TYPE,
       STATUS
  FROM dba_CONSTRAINTS
 WHERE     CONSTRAINT_TYPE = 'R'
       AND status = 'ENABLED'
       AND owner IN ('KYLE',
                     'KYLE01');
--确认目标端目标表的主键可用:
SELECT T1.STATUS,
       T1.VALIDATED,
       T2.status,
       T1.constraint_name,
       T1.owner
  FROM dba_constraints T1, dba_objects T2
 WHERE     T2.OBJECT_NAME = T1.constraint_name
       AND T1.OWNER IN ('KYLE',
                        'KYLE01');

--验证job 确实被禁用
SELECT job,
       LOG_USER,
       PRIV_USER,
       SCHEMA_USER,
       broken
  FROM dba_jobs
 WHERE schema_user IN ('KYLE',
                       'KYLE01');

SELECT OWNER, JOB_NAME, STATE
  FROM DBA_SCHEDULER_JOBS
 WHERE OWNER IN ('KYLE',
                 'KYLE01');

--确认trigger 已经全部关闭
SELECT DISTINCT status
  FROM dba_triggers
 WHERE owner IN ('KYLE',
                 'KYLE01');
```

## 2,.添加DDL功能与加密

### 2.1 添加DDL功能
* 1.关闭所有OGG进程

```sql
STOP ekyle
STOP pkyle
STOP rkyle
```
* 2.用户的默认表空间不能自动SYSTEM表空间

```sql
set PAGRSIZE 200
set LINESIZE 200
col USERNAME format A20
col DEFAULT_TABLESPACE format A20 

SELECT username, default_tablespace FROM dba_users;
```

* 3.关闭回收站并清空回收站

```sql
--11G可以启动回收站，10G必须关闭回收站
ALTER SYSTEM SET recyclebin = off SCOPE = SPFILE;
PURGE DBA_RECYCLEBIN;
```

* 4.指明支持DDL的对象放在那个SCHEMA下载

```sql
ggsci> edit params ./GLOBALS
GGSCHEMA goldengate
```

* 5.添加相关权限

```sql
GRANT EXECUTE ON UTL_FILE TO goldengate;
GRANT RESTRICTED SESSION TO goldengate;
GRANT CREATE TABLE, CREATE SEQUENCE TO goldengate;
GRANT GGS_GGSUSER_ROLE TO goldengate;
```

* 6.执行脚本（源目标）

```sql
cd /ggs
sqlplus "/as sysdba"
@marker_setup.sql
@ddl_setup.sql
@role_setup.sql
GRANT GGS_GGSUSER_ROLE TO goldengate;
@ddl_enable.sql
@$ORACLE_HOME/rdbms/admin/dbmspool.sql
@ddl_pin.sql goldengate
```

* 7.修改参数，增加DDL复制参数

```sql
ggsci >edit params eitpux01
--DDL INCLUDE OBJNAME "ITPUX01.*"
DDL INCLUDE ALL
DDLOPTIONS ADDTRANDATA,REPORT

ggsci >edit params ritpux01
--DDL INCLUDE OBJNAME "ITPUX01.*"
DDL INCLUDE ALL
DDLERROR default ignore retryop
```
* 8.启动进程

```sql
START ekyle
START pkyle
START ryle
```
* DDL其他功能

```sql
--开启
ddl_enable.sql
--关闭
ddl_disable.sql

--清空DDL trace ggs_ddl_trace.log
ddl_cleartrace.sql

--ddl 参数：
optype alert
objtype "table"
ojbname "user.tab*"
include mapped object "*";
exclude mapped object "itpux.itux*"
DDL 环境重配（删除就是1-5，12，去掉参数中的DDL）:
1.停止所有的OGG ex/pump/rp 进程
2.@ddl_disable.sql
3.@ddl_remove.sql
4.@marker_remove.sql
5.@marker_setup.sql
6.@ddl_setup.sql
7.@role_setup.sql
8.GRANT GGS_GGSUSER_ROLE TO goldengate;
9.@ddl_enable.sql
10.@$ORACLE_HOME/rdbms/admin/dbmspool.sql
11.@ddl_pin.sql goldengate
12.运行所有的OGG ex/pump/rp 进程
```

### 2.2 加密
* 1.生成加密的KEY 

```sql
[oracle@orcl:/u01/ggs]$keygen 256 1
0x1673D6197E05DA1009B1451420A73134BDF868246D6AC878FBFC69193231C355
```

* 2.将KEY存到本地指定文件

```sql
[oracle@orcl:/u01/ggs]$cat ENCKEYS 
KYLE_KEY 0xE9E07156ACC2411F97E78D117B55C444E935E27A034CBD389CF2F432F8B5F417
```

* 3.设置登录用的的KEY

```sql

GGSCI (orcl) 109> ENCRYPT PASSWORD goldengate AES256 ENCRYPTKEY KYLE_KEY
Encrypted password:  AADAAAAAAAAAAAKABIXBCASDGBHCGFNGMHJIFFMDXFGBRCLHTGJGVIECPCGEUAJEXDSEFBUCGATGKGUGBGCCVFKBBENDYBSAPDZBACQCQILIRHYH
Algorithm used:  AES256
```

* 添加KEY到相应文件

```sql
extract eitpux01
DDL INCLUDE ALL
DDLOPTIONS ADDTRANDATA,REPORT
userid goldengate,password AADAAAAAAAAAAAKACAHHWIRDTBQDNBUJYFUCXCGEVBCDHBCEGFOIQFSEGFKCBGLDPGUGAIIHZDJATJSJYBPBSJSJUFSBJCUBYCXBFBPJPDSBVGBJ,AES256,ENCRYPTKEY KYLE_KEY
exttrail ./dirdat/eitpux01/ex
tranlogoptions excludeuser goldengate
tranlogoptions convertucs2clobs
warnlongtrans 12h,checkinterval 30m
discardfile ./dirrpt/eitpux01/eitpux01.dsc,append,megabytes 200
TABLE itpux.*;
--TABLE itpux01.*;
--TABLE itpux02.*;
--TABLE itpux03.*;
```

  [1]: https://docs.oracle.com/cd/E11882_01/index.htm