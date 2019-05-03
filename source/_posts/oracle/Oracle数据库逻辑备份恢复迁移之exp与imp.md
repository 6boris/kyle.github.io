---
title: Oracle数据库逻辑备份恢复迁移之exp与imp
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
  - Exp
  - Imp
  - 备份容灾
---


# Oracle数据库逻辑备份恢复迁移之exp与imp

[Oracle 11G R2 官方文档][1] 

## 1.export与import逻辑备份恢复概述
### 1.1 备份概述
* 备份可分为两类，物理备份和逻辑备份
>&emsp;&emsp;物理备份：该方法实现数据库的完整恢复，但需要极大的外部存储设备，例如磁带库，具体包括冷备份和热备份。冷备份和热备份(热备份要求数据库运行在归档模式下)都是物理备份，它涉及到组成数据库的文件，但不考虑逻辑内容。
>&emsp;&emsp;逻辑备份： 使用软件技术从数据库中导出数据并写入一个输出文件，该文件的格式一般与原数据库的文件格式不同，只是 原数据库中数据内容的一个映像。因此，逻辑备份文件只能用来对数据库进行逻辑恢复，即数据导入，而不能按数据库原来的存储特征进行物理恢复。逻辑备份一般 用于增量备份，即备份那些在上次备份以后改变的数据。
>&emsp;&emsp;Oracle 的导出导入是一个很常用的迁移工具。 在Oracle 10g以后，Oracle 推出了数据泵(expdp/impdp).它可以通过使用并行，从而在效率上要比exp/imp 要高。但是这个exp/imp很多时候还需要使用。
>&emsp;&emsp;导入/导出是ORACLE幸存的最古老的两个命令行工具，其实我从来不认为Exp/Imp是一种好的备份方式，正确的说法是Exp/Imp只能是一个好的转储工具，特别是在小型数据库的转储，表空间的迁移，表的抽取，检测逻辑和物理冲突等中有不小的功劳。当然，我们也可以把它作为小型数据库的物理备份后的一个逻辑辅助备份，也是不错的建议。对于越来越大的数据库，特别是TB级数据库和越来越多数据仓库的出现，EXP/IMP越来越力不从心了，这个时候，数据库的备份都转向了RMAN和第三方工具。下面说明一下EXP/IMP的使用。

## 2.export与import参数详解
### 2.1 export参数详解
```shell
[oracle@orcl:/]$exp -help

Export: Release 11.2.0.4.0 - Production on Tue Jan 30 18:30:39 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.



You can let Export prompt you for parameters by entering the EXP
command followed by your username/password:

     Example: EXP SCOTT/TIGER

Or, you can control how Export runs by entering the EXP command followed
by various arguments. To specify parameters, you use keywords:

     Format:  EXP KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
     Example: EXP SCOTT/TIGER GRANTS=Y TABLES=(EMP,DEPT,MGR)
               or TABLES=(T1:P1,T1:P2), if T1 is partitioned table

USERID must be the first parameter on the command line.

Keyword    Description (Default)      Keyword      Description (Default)
--------------------------------------------------------------------------
USERID     username/password          FULL         export entire file (N)
BUFFER     size of data buffer        OWNER        list of owner usernames
FILE       output files (EXPDAT.DMP)  TABLES       list of table names
COMPRESS   import into one extent (Y) RECORDLENGTH length of IO record
GRANTS     export grants (Y)          INCTYPE      incremental export type
INDEXES    export indexes (Y)         RECORD       track incr. export (Y)
DIRECT     direct path (N)            TRIGGERS     export triggers (Y)
LOG        log file of screen output  STATISTICS   analyze objects (ESTIMATE)
ROWS       export data rows (Y)       PARFILE      parameter filename
CONSISTENT cross-table consistency(N) CONSTRAINTS  export constraints (Y)

OBJECT_CONSISTENT    transaction set to read only during object export (N)
FEEDBACK             display progress every x rows (0)
FILESIZE             maximum size of each dump file
FLASHBACK_SCN        SCN used to set session snapshot back to
FLASHBACK_TIME       time used to get the SCN closest to the specified time
QUERY                select clause used to export a subset of a table
RESUMABLE            suspend when a space related error is encountered(N)
RESUMABLE_NAME       text string used to identify resumable statement
RESUMABLE_TIMEOUT    wait time for RESUMABLE 
TTS_FULL_CHECK       perform full or partial dependency check for TTS
VOLSIZE              number of bytes to write to each tape volume
TABLESPACES          list of tablespaces to export
TRANSPORT_TABLESPACE export transportable tablespace metadata (N)
TEMPLATE             template name which invokes iAS mode export

Export terminated successfully without warnings.
```
* EXP的所有参数（括号中为参数的默认值）：
>可以通过exp help=y或者imp help=y查看exp或imp的详细参数，下面以exp为例解释参数意义
USERID：用户名/口令

>FULL：导出整个数据库，只有拥有exp_full_database角色的用户或者特权用户如sys，system等才能进行全库导出。 示例如下
exp "'/ as sysdba'" full=y

>BUFFER：制定数据缓冲区大小，主要用于提高exp/imp速度，该单位为字节，不能写成buffer=1m的形式，应写成字节为单位的参数，如buffer=1048576
exp hr/hr file=t_b.dmp buffer= 1048576 tables=T

>OWNER：需要导出的用户，示例如下
exp "'/ as sysdba'" owner=\(hr,ITPUX02\) file=hr_ITPUX02.dmp
上例中由于是在linux平台进行测试的，需要对 owner=\(hr,ITPUX02\)使用\进行转义

>FILE：输出文件

>TABLES：需要导出的表

>COMPRESS：导入到一个区 (Y) 。主要目的是为了消除存储碎片，以保证某张表的所有记录都存储在连续的空间里。 但是负面效应很明显， 如果该参数值为y，则会将高水位线以下的所有extent导入到一个区中， 因此在导入时很有可能出现，明明表中数据很少，但是却花了很多时间在建立的extent上。 且自oracle9i开始，使用了本地管理的表空间，存储碎片的问题应该比低版本好多了，个人建议将compress设为n。

* DIRECT：直接路径 (N)。
>传统模式导出和直接路径导出的原理
传 统模式导出相当于使用select语句从表中取出数据，数据从磁盘上先读到buffer cache中，记录被转移到一个评估检测的缓冲区中，数据经过语法检测后没有问题，将数据传给PGA，最后写入导出的文件中。如果使用Direct Path模式导出，数据直接从磁盘上读取到导出的PGA中：记录直接被转换导出会话的私有buffer中。这也就是意味着SQL语句处理层被忽略掉了，因 为数据已经是符合导出的格式了，不需要其他的转换处理了。数据直接被传送给导出的客户端，最后写入导出文件。过程可概况如下
```shell
direct=n datafile---->sga----->pga----->dump
direct=y datafile---->pga----->dump
```
>传统路径导出VS直接路径导出(oracle exp direct=y)两者的差异
>a、 Conventional path Export
&emsp;&emsp; 传统路径模式使用SQL SELECT语句抽取表数据。数据从磁盘读入到buffer cache缓冲区中，行被转移到评估缓冲区。在此之后根据SQL表达式，将记录返回给导出客户端，然后写入到dump文件。
>b、Direct path Export
&emsp;&emsp; 直接导出模式，数据直接从磁盘中读取到导出session的PGA中，行被直接转移到导出session的私有缓冲区，从而跳过SQL命令处理层。
避免了不必要的数据转换。最后记录返回给导出客户端，写到dump文件。

* 传统路径导出VS直接路径导出(oracle exp direct=y)性能问题
>&emsp;&emsp; a、直接路径导出方式比传统路径方式具有更优的性能，速度更快，因为绕过了SQL命令处理部分。
&emsp;&emsp; b、直接路径导出方式支持RECORDLENGTH参数(最大为64k)，该参数值通常建议设置为系统I/O或者DB_BLOCK_SIZE的整数倍
&emsp;&emsp; c、影响直接路径导出的具体因素(DB_BLOCK_SIZE，列的类型，I/O性能，即数据文件所在的磁盘驱动器是否单独于dump文件所在的磁盘驱动器)
&emsp;&emsp; d、无论是直接路径导出还是传统路径导出产生的dump，在使用imp方式导入时，会耗用相同的时间.

* 传统路径导出VS直接路径导出(oracle exp direct=y)直接路径导出的限制
>a、直接路径导出不支持交互模式
b、不支持表空间传输模式(即transport_tablespaces=y不被支持)，支持的是full,owner,tables导出方式
c、不支持query查询方式，如exp scott/tiger tables=emp query=\"where job=\'salesman\' \" 不被支持
d、直接路径导出使用recordlength设置一次可以导出数据的量，取代传统路径使用buffer的设置
e、直接路径导出要求nls_lang环境参数等于数据库字符集，负责收到exp-41警告及exp-0终止错误

>GRANTS：导出权限 (Y)
INCTYPE：增量导出类型，已废除
INDEXES：导出索引 (Y)
RECORD：跟踪增量导出 (Y) ，已废除
TRIGGERS：导出触发器 (Y)
LOG：屏幕输出的日志文件

>STATISTICS：在导出文件中保留对象的统计信息，默认值ESTIMATE，还可以为compute或者none。如果导出时出现
EXP-00091: Exporting questionable statistics
可以考虑将 STATISTICS设置为NONE
>ROWS：确定表中的数据行是否导出，默认为Y，导出

>QUERY：用于导出表的子集的select子句，示例如下
exp hr/hr file=emp_q.dmp tables=employees query=\"where hire_date \>to_date\(\'1999-01-01\'\,\'yyyy-mm-dd\'\)\"

* PARFILE：参数文件名，可以用如下方式导出
```sql
exp hr/hr parfile=parfile
$ cat parfile
file=t_p.dmp
compress=y
rows=y
tables=(t,empl%s)
```

>使用parfile参数可以对频繁进行的导出操作进行反复调用，同时也可以避免不同操作系统之间需要对特定字符进行转义的烦恼，如下例
```sql
exp hr/hr parfile=parfile
$cat parfile
file=t_p.dmp
compress=y
rows=y
tables=employees
statistics=none
query="where hire_date>to_date('1999-01-01','yyyy-mm-dd')"
```
>CONSISTENT：在导出时，将影响正在导出的表的事务设为只读，主要作用于嵌套表和分区表，默认为N。
CONSTRAINTS：导出的约束条件 (Y)
OBJECT_CONSISTENT:只在对象导出期间设置为只读的事务处理 (N)
FEEDBACK:每 x 行显示进度，默认为0
FILESIZE：每个导出文件的最大大小
FLASHBACK_SCN：用于将会话快照设置回以前状态的SCN
FLASHBACK_TIME：用于获取最接近指定时间的SCN的时间
RESUMABLE：遇到空间不足时的错误时挂起，默认为N，需与 RESUMABLE_NAME和 RESUMABLE_TIMEOUT一起使用
RESUMABLE_NAME：用于标示哪个会话需要使用 RESUMABLE选项，格式为 User USERNAME (USERID), Session SESSIONID, Instance INSTANCEID
RESUMABLE_TIMEOUT：RESUMABLE的等待时间，默认为7200s，如果在指定时间内未解决问题，则操作中断
>TTS_FULL_CHECK：对TTS执行完整或部分相关性检查
TABLESPACES：要导出的表空间列表，示例如下
exp "'/ as sysdba'" file=t_ts.dmp tablespaces=(users,example)
TRANSPORT_TABLESPACE 导出可传输的表空间元数据 (N)
直接备份到磁带上
exp icdmain/icd rows=y indexes=n compress=n buffer=65536 feedback=100000 file=/dev/rmt0 log=exp.log tables=(tab1,tab2,tab3)
注：在磁盘空间允许的情况下，应先备份到本地服务器，然后再拷贝到磁带。出于速度方面的考虑，尽量不要直接备份到磁带设备

### 2.2 import参数详解
```sql
[oracle@orcl:/]$imp -help

Import: Release 11.2.0.4.0 - Production on Tue Jan 30 19:01:13 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.



You can let Import prompt you for parameters by entering the IMP
command followed by your username/password:

     Example: IMP SCOTT/TIGER

Or, you can control how Import runs by entering the IMP command followed
by various arguments. To specify parameters, you use keywords:

     Format:  IMP KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
     Example: IMP SCOTT/TIGER IGNORE=Y TABLES=(EMP,DEPT) FULL=N
               or TABLES=(T1:P1,T1:P2), if T1 is partitioned table

USERID must be the first parameter on the command line.

Keyword  Description (Default)       Keyword      Description (Default)
--------------------------------------------------------------------------
USERID   username/password           FULL         import entire file (N)
BUFFER   size of data buffer         FROMUSER     list of owner usernames
FILE     input files (EXPDAT.DMP)    TOUSER       list of usernames
SHOW     just list file contents (N) TABLES       list of table names
IGNORE   ignore create errors (N)    RECORDLENGTH length of IO record
GRANTS   import grants (Y)           INCTYPE      incremental import type
INDEXES  import indexes (Y)          COMMIT       commit array insert (N)
ROWS     import data rows (Y)        PARFILE      parameter filename
LOG      log file of screen output   CONSTRAINTS  import constraints (Y)
DESTROY                overwrite tablespace data file (N)
INDEXFILE              write table/index info to specified file
SKIP_UNUSABLE_INDEXES  skip maintenance of unusable indexes (N)
FEEDBACK               display progress every x rows(0)
TOID_NOVALIDATE        skip validation of specified type ids 
FILESIZE               maximum size of each dump file
STATISTICS             import precomputed statistics (always)
RESUMABLE              suspend when a space related error is encountered(N)
RESUMABLE_NAME         text string used to identify resumable statement
RESUMABLE_TIMEOUT      wait time for RESUMABLE 
COMPILE                compile procedures, packages, and functions (Y)
STREAMS_CONFIGURATION  import streams general metadata (Y)
STREAMS_INSTANTIATION  import streams instantiation metadata (N)
DATA_ONLY              import only data (N)
VOLSIZE                number of bytes in file on each volume of a file on tape

The following keywords only apply to transportable tablespaces
TRANSPORT_TABLESPACE import transportable tablespace metadata (N)
TABLESPACES tablespaces to be transported into database
DATAFILES datafiles to be transported into database
TTS_OWNERS users that own data in the transportable tablespace set

Import terminated successfully without warnings.
```
* IMP的所有参数（括号中为参数的默认值）：
>i&emsp;&emsp;mp的参数和exp的大致相同，下面是常用参数的解释，与exp相同的这就不再赘述
>&emsp;&emsp;ignore：Oracle在恢复数据的过程中，当导入某个表时，该表已经存在，就要根据ignore参数的设置来决定如何操作。若 ignore=y，Oracle不执行CREATE TABLE语句，直接将数据插入到表中，如果插入的记录违背了约束条件，比如主键约束，唯一索引等，则出错的记录不会插入，但合法的记录会添加到表中。若 ignore=n，Oracle不执行CREATE TABLE语句，同时也不会将数据插入到表中，而是忽略该表的错误，继续导入下一个表。 注
意：如果表中的字段并没有唯一性约束，那么在使用ignore=y的情况下很有可能插入重复数据。
>&emsp;&emsp;indexes：在恢复数据的过程中，若indexes=n，则表上的索引不会被恢复，但是对 LOB 索引, OID索引和 主键索引等系统自动生成的索引将无条件恢复。
>&emsp;&emsp;indexfile：不进行导入操作而是将创建对象的文本保存到文件中，可以通过编辑使用该文本文件创建数据库对象。
>&emsp;&emsp;fromuser,touser:这两个参数可以组合使用，也可以分开使用。他们可以实现将源用户的对象数据，导入到目标用户schema底下的功能。这里要注意，导入时的用户需要有
imp_full_database角色，示例如下
导入一个或一组指定用户所属的全部对象
$imp system/manager file=full_all.dmp log=seapark fromuser=hr
$imp system/manager file=seapark log=seapark fromuser=(seapark,amy,amyc,harold)
将一个或一组指定用户所属的全部对象导入到另一个用户下
$imp hr/hr fromuser=hr touser=czm file=hr_all.dmp
$imp system/manager file=tank log=tank fromuser=(seapark,amy) touser=(seapark1, amy1)
>commit：默认值为 COMMIT=N，及在每插入完一个对象后提交。 当COMMIT=Y时候是根据你BUFFER的大小决定每次提交的数量。对于包含了LONG、RAW、 DATE等类型的表，不论BUFFER设置多大，都是每插入一行进行提交。设置commit=y可以防止减少回滚段的压力，但由于频繁提交，会带来性能 上的影响，推荐使用COMMIT=N。

*  下列关键字仅用于可传输的表空间
>TRANSPORT_TABLESPACE 导入可传输的表空间元数据 (N)
TABLESPACES 将要传输到数据库的表空间
DATAFILES 将要传输到数据库的数据文件
TTS_OWNERS 拥有可传输表空间集中数据的用户

* 关于增量参数的说明：exp/imp的增量并不是真正意义上的增量，所以最好不要使用。

## 3.exp / imp常用语法
### 3.1 exp
* 完全
```shell
exp system/oracle compress=n buffer=4096000 feedback=100000 full=y file=expfull_itpuxdb.dmp log=expfull_itpuxdb.log
--exp \"/ as sysdba\" compress=n buffer=4096000 feedback=100000 full=y file=exp_itpuxdb.dmp log=exp_itpuxdb.log
--backup/backup : exp_full_database,imp_full_database, grant exp_full_database to backup; grant imp_full_database to backup;
```

* 用户
```shell
exp system/oracle compress=n buffer=4096000 feedback=100000 owner=itpux file=expitpux_itpuxdb.dmp log=expitpux_itpuxdb.log
--exp itpux/itpux compress=n buffer=4096000 feedback=100000 file=expitpux_itpuxdb.dmp log=expitpux_itpuxdb.log
```

* 表
```shell
exp itpux/itpux compress=n buffer=4096000 feedback=100000 tables=t1,t2,t2 file=exp_t1_itpuxdb.dmp log=exp_t1_itpuxdb.log
```

* other
```shell
exp system/oracle compress=n buffer=4096000 feedback=100000 rows=n file=expfull_itpuxdb.dmp log=expfull_itpuxdb.log
```


### 3.2 imp
* 完全
```shell
imp system/oracle ignore=y buffer=4096000 feedback=100000 full=y file=expfull_itpuxdb.dmp log=impfull_itpuxdb.log
--exp \"/ as sysdba\" system/oracle ignore=y buffer=4096000 feedback=100000 full=y file=expfull_itpuxdb.dmp log=impfull_itpuxdb.log
--backup/backup : exp_full_database,imp_full_database, grant exp_full_database to backup; grant imp_full_database to backup;
```

* 用户
```shell
imp system/oracle fromuser=itpux touser=itpux ignore=y buffer=4096000 feedback=100000 file=expitpux_itpuxdb.dmp log=impitpux_itpuxdb.log
--imp itpux/itpux fromuser=itpux touser=itpux ignore=y buffer=4096000 feedback=100000 file=expitpux_itpuxdb.dmp log=impitpux_itpuxdb.log
```

* 表
```shell
imp itpux/itpux fromuser=itpux touser=itpux tables=t1,t2,t3 ignore=y buffer=4096000 feedback=100000 file=exp_t1_itpuxdb.dmp log=imp_t1_itpuxdb.log
```

## 4.配置生产环境的逻辑自动备份策略
>要求：每天早上5点做逻辑全备，备份文件保留一周。
```shell
vi /backup/scripts/expfull_db01.sh
export BAKDATE=`date +%Y%m%d`
export ORACLE_SID=db01
nohup exp system/oracle compress=n buffer=4096000 feedback=100000 full=y file=/backup/expfull_db01_$BAKDATE.dmp log=/backup/expfull_db01_$BAKDATE.log &
gzip -f /backup/expfull_db01_$BAKDATE.dmp
find /backup -name expfull_db01_*.dmp.gz -atime +2 -exec rm -rf {} \;
#chown -R oracle:dba /backup/expfull_db01.sh
#chmod 775 /backup/expfull_db01.sh
#crontab -e
00 05 * * * su - oracle -c /backup/scripts/expfull_db01.sh
```

## 5.exp / imp生产环境数据迁移流程和注意事项
### 5.1 数据迁移的目的
>使用exp/imp进行数据迁移的前置条件、操作步骤，降低对对应用造成的影响及避免故障
### 5.2 数据迁移的适用范围
>所有线上库9i ,10g > 11g 12c 
>11g不建用了，
### 5.3 数据迁移的风险评估
>&emsp;&emsp;01.exp导出数据时没有使用compress=n参数，会导致所有数据被压缩在一个extent里，导入可能由于没有连续的blocks满足需要，导致imp失败。
&emsp;&emsp;02.有些os对文件大小有限制，exp数据时需要使用filesize参数来分割导出文件
&emsp;&emsp;03.exp导出数据时没有正确估计dmp文件所需空间，导致主机磁盘满。
&emsp;&emsp;05.imp导入数据时没有使用ignore=y参数，目标库上存在表的情况下数据无法导入
&emsp;&emsp;05.imp导入大量数据时没有使用commit=y参数，导致事务太久，undo资源占用过大无法及时回收。
&emsp;&emsp;06.跨字符集的数据迁移，由于字符集不兼容导致数据迁移失败。
&emsp;&emsp;07.imp跨schema进行数据迁移时，没有正确指定fromuser、touser，导致数据没有正确导入
&emsp;&emsp;09.导入表空间不存在或者空间不足，导致表创建失败或者数据导入失败，导致其他应用报错
### 5.4 数据迁移的准备工作
>&emsp;&emsp;01.检查源数据库和目标库的版本、字符集，如果目标库版本低于源库，使用目标库的软件做导出。如果字符集不一致，不建议使用exp/imp迁移数据。
&emsp;&emsp;02.检查目标库上表结构和源结构是否一致，如果不一致，先修复结构，保证一致。
&emsp;&emsp;03.user_segments里查出导出表所占的空间大小，检查os对文件大小的限制。如果表大小超出文件大小，exp导出时加上这两个参数：filesize=小于文件限制的数值m,file=exp01.dmp,exp02.dmp,…多个dmp文件
&emsp;&emsp;04.表比较多的情况下，建议用parfile。各个参数在parfile里写好。
tables=(tab1,tab2,tab3,..)
&emsp;&emsp;05.根据不同的需要要书写query子句时，这个参数跟direct=y冲突
### 5.5 数据迁移的执行过程
>01.如果目标表是已存在数据，跟应用确认后，可以先进行导出备份，以防后面需要回退。
先根据需求编辑exp、imp的参数文件：’-‘后面是参数说明，实际使用时去掉cat exp_itpux.par
userid=itpux/itpux@db01
direct=y –直接路径导出，加快导出速度
compress=n –避免数据全部压缩在一个数据块上
file=exp_itpux01.dmp
log=exp_itpux01.log
recordlength=65535 –写dmp文件时一次IO的大小，上限是65535，可以加快导出速度
buffer=4096000 -数据缓冲区大小
tables=itpux01

exp parfile=exp_itpux.par –进行数据导出
```shell
cat imp_itpux.par
userid=itpux/itpux
commit=y –开启批量提交，避免长事务
ignore=y –如果目标表已经存在，只导入数据
fromuser=itpux
touser=itpux
tables=itpux01
file=imp_itpux01.dmp
log=imp_itpux01.log
buffer=100000 –大小控制导入速度的，设置过大会导致日志产生很快
imp parfile=imp_itpux.par –进行数据导入
```
>注意上面的fromuser和touser。如果将表导入到两个schema:itpux01,itpux02
需要按照这种格式配置参数：
fromuser和touser一一对应，即使导出时只有一个schema.
fromuser=itpux01,itpux01
touser=itpux02,itpux02

* 02 exp导出数据时，检查exp的日志，如果报错，一般是参数配置错误，参考官方文档调整参数。
* 03 imp导入之前，需要创建相应的用户，权限，表空间等对象，否则报错：
```sql
spool itpux_object_create_scripts.sql
SET LONG 2000000 PAGESIZE 0 head off verify off feedback off linesize 180
select dbms_metadata.get_ddl('USER','ITPUX02') from dual;
select dbms_metadata.get_granted_ddl('OBJECT_GRANT','ITPUX02') from dual;
select dbms_metadata.get_granted_ddl('ROLE_GRANT','ITPUX02') from dual;
select dbms_metadata.get_granted_ddl('SYSTEM_GRANT','ITPUX02') from dual;
select dbms_metadata.get_ddl('TABLESPACE','ITPUX02') from dual;
spool off;
```
* 04 imp导入数据过程，需要监控下数据库事务和日志产生速度。
* 05 对导入的表收集统计信息。

### 5.6 数据迁移后的验证方案
>对比exp、imp的日志，确认导出导入数据量是否一致。并在数据库上检查数据量。
比如上面的数据迁移，检查数据量跟日志显示是否一致。
select count(*) from itpux.itpux01;
>跨schema或者数据库迁移数据时，除检查日志外，还需要检查源和目标的对象数据量、是否有失效对象。
```sql
select object_type s_object_type,count(*) from dba_objects where owner='ITPUX01' group by object_type ;
select object_type t_object_type,count(*) from dba_objects where owner='ITPUX01' group by object_type ;
select * from dba_objects where status <> 'VALID' and owner='ITPUX01';
```
* dblink两个库一起查询;
```sql
SELECT s.s_object_type,
       s.s_count,
       t.t_object_type,
       t.t_count
  FROM (  SELECT object_type s_object_type, COUNT (*) s_count
            FROM dba_objects@db01
           WHERE owner = 'ITPUX01'
        GROUP BY object_type) s,
       (  SELECT object_type t_object_type, COUNT (*) t_count
            FROM dba_objects
           WHERE owner = 'ITPUX01'
        GROUP BY object_type) t;
```
>自动生成编译无效对象SQL及编译过程

* 统计当前用户无效对象数量:
```sql
  SELECT owner,
         object_type,
         status,
         COUNT (*)
    FROM dba_objects
   WHERE status <> 'VALID'
GROUP BY owner, object_type, status
ORDER BY owner, object_type;
```
* 生成编译无效对象SQL
```sql
SELECT    'ALTER '
       || OBJECT_TYPE
       || ' '
       || OWNER
       || '.'
       || OBJECT_NAME
       || ' COMPILE;'
  FROM dba_objects
 WHERE     status <> 'VALID'
       AND object_type IN ('PACKAGE',
                           'PACKAGE BODY',
                           'FUNCTION',
                           'PROCEDURE',
                           'TRIGGER',
                           'VIEW');
```
>通过复制以上SQL语句,直接手动执行编译执行,也可以采用如下方式在oracle用户下进行手工编译
```sql
# su - oracle
$ sqlplus / as sysdba
SQL> @$ORACLE_HOME/rdbms/admin/utlrp.sql
```

* 如果有序列要处理
```sql
exclude=SEQUENCE
```
>目标库
方法1:
1.源库： 查出sequence的最大值
2.目标库： DROP SEQUENCE .... 再 CREATE SEQUENCE START最大值 其他不变
方法2：
目标库： DROP SEQUENCE .... 再 CREATE SEQUENCE START 超大值 （保证比之前的都大，不会发生冲突的 ）

* 可以用以下脚本来实现 ：
```sql
/* Formatted on 2018/1/30 19:49:33 (QP5 v5.313) */
script FROM THE SOURCE database. USE this script TO RESET THE proper starting VALUE FOR sequences ON THE TARGET database.
cr_tts_create_seq.SQL
SET HEADING OFF FEEDBACK OFF TRIMSPOOL ON ESCAPE OFF
SET LONG 1000 LINESIZE 1000 PAGESIZE 0
COL SEQDDL FORMAT A300
SPOOL tts_create_seq.sql
PROMPT /* ========================= */
PROMPT /* Drop and create sequences */
PROMPT /* ========================= */

SELECT REGEXP_REPLACE (
           DBMS_METADATA.get_ddl ('SEQUENCE', sequence_name, sequence_owner),
           '^.*(CREATE SEQUENCE.*CYCLE).*$',
              'DROP SEQUENCE "'
           || sequence_owner
           || '"."'
           || sequence_name
           || '";'
           || CHR (10)
           || '\1;')
           SEQDDL
  FROM dba_sequences
 WHERE sequence_owner NOT IN (SELECT name
                                FROM SYSTEM.logstdby$skip_support
                               WHERE action = 0);

SPOOL OFF
```

### 5.7 数据迁移核心对象风险
>由于核心表访问、变更频繁，不宜直接使用imp对核心表大量导入数据。

### 5.8 数据迁移回退方案
>exp对应用无影响，不需要回退。
imp后可能数据有误，需要进行回退操作。
如果目标表本来就是空表，跟应用确认后，直接清空即可。
如果目标表原有数据，跟应用确认是否使用原有备份数据进行恢复。若需要，先exp备份当前数据，然后清空再导入前面的备份数据。

## 6.exp / imp数据迁移案例
### 6.1 迁移目的
>&emsp;&emsp;将linux系统oracle服务器上schema（itpux01,itpux02）全部通过Exp迁移到另一台oracle服务器，并能正常查询到相关数据。
### 6.2 迁移流程
```sql
1.准备数据：linux系统oracle服务器上创建用户,表空间.并创建相关表.数据等。
2.linux本地用Exp做导出;
3.再到远程主机创建相关对象；
4.远程主机用imp做导入(导入);
5.验证远程本地数据合法性;
```
### 6.3 迁移过程
* 创建表空间
```sql
CREATE TABLESPACE itpux11
    DATAFILE '/oracle/oradata/db01/itpux01.dbf'
                 SIZE 100 M
                 AUTOEXTEND OFF
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;
	
CREATE TABLESPACE itpux12
    DATAFILE '/oracle/oradata/db01/itpux02.dbf'
                 SIZE 100 M
                 AUTOEXTEND OFF
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;
```

* 创建用户并授权
```sql
CREATE USER itpux01 IDENTIFIED BY itpux01
    DEFAULT TABLESPACE itpux01;
CREATE USER itpux02 IDENTIFIED BY itpux02
    DEFAULT TABLESPACE itpux02;
GRANT DBA TO itpux01;
GRANT DBA TO itpux02;

ALTER USER itpux01
    QUOTA UNLIMITED ON itpux01;

ALTER USER itpux02
    QUOTA UNLIMITED ON itpux02;
```

* moon用户登录并创建测试表
```sql
CONN itpux01 / itpux01
CREATE TABLE itpux01
(
    id    NUMBER (30) PRIMARY KEY NOT NULL,
    name DATE
);

CONN itpux02 / itpux02
CREATE TABLE itpux02
(
    id    NUMBER (30) PRIMARY KEY NOT NULL,
    name DATE
);

```

* 创建2个存储过程并执行
```sql
CONN itpux01 / itpux01

CREATE OR REPLACE PROCEDURE p_itpux01
IS
BEGIN
    EXECUTE IMMEDIATE 'select count(*) from itpux01';

    FOR i IN 1 .. 1000
    LOOP
        INSERT INTO itpux01 (id, name)
             VALUES (i, SYSDATE);

        COMMIT;
    END LOOP;

    EXECUTE IMMEDIATE 'select count(*) from itpux01';
END p_itpux01;
/


BEGIN
    p_itpux01;
END;
/

SELECT COUNT (*) FROM itpux01;

SELECT *
  FROM itpux01
 WHERE id > 990;

CONN itpux02 / itpux02
CREATE OR REPLACE PROCEDURE p_itpux02
IS
BEGIN
    EXECUTE IMMEDIATE 'select count(*) from itpux02';

    FOR i IN 1 .. 2000
    LOOP
        INSERT INTO itpux02 (id, name)
             VALUES (i, SYSDATE);

        COMMIT;
    END LOOP;

    EXECUTE IMMEDIATE 'select count(*) from itpux02';
END p_itpux02;
/

BEGIN
    p_itpux02;
END;
/
```

* 查询验证数据
```sql
CONN itpux01 / itpux01

SELECT COUNT (*) FROM itpux01;

SELECT *
  FROM itpux01
 WHERE id > 990;

CONN itpux02 / itpux02

SELECT COUNT (*) FROM itpux02;

SELECT *
  FROM itpux02
 WHERE id > 1990;
```

* 本地用Exp做导出
```sql
EXP SYSTEM/oracle OWNER=itpux01,itpux02 COMPRESS=n FILE=exp_2table.dmp LOG=exp_2table.LOG direct=y recordlength=65535 buffer=4096000
```

* imp导入之前，需要创建相应的用户，权限，表空间等对象，否则报
```sql
/* Formatted on 2018/1/30 20:03:14 (QP5 v5.313) */
SPOOL itpux_object_create_scripts.sql
SET LONG 2000000 PAGESIZE 0 HEAD OFF VERIFY OFF FEEDBACK OFF LINESIZE 180

SELECT DBMS_METADATA.get_ddl ('USER', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('OBJECT_GRANT', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('ROLE_GRANT', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('SYSTEM_GRANT', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_ddl ('TABLESPACE', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_ddl ('USER', 'ITPUX02') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('OBJECT_GRANT', 'ITPUX02') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('ROLE_GRANT', 'ITPUX02') FROM DUAL;

SELECT DBMS_METADATA.get_granted_ddl ('SYSTEM_GRANT', 'ITPUX02') FROM DUAL;

SELECT DBMS_METADATA.get_ddl ('TABLESPACE', 'ITPUX02') FROM DUAL;

SPOOL OFF;
```

* 异机用imp做导入
```sql
imp SYSTEM/oracle fromuser=itpux01,itpux02 touser=itpux01,itpux02 FILE=exp_2table.dmp LOG=exp_2table.LOG COMMIT=y IGNORE=y buffer=4096000
```

* 验证导入后的数据合法性
```sql

```

### 6.4 数据迁移后的验证方案
>对比exp、imp的日志，确认导出导入数据量是否一致。并在数据库上检查数据量。
比如上面的数据迁移，检查数据量跟日志显示是否一致。
```sql
SELECT COUNT (*) FROM itpux.itpux01;
```
>跨schema或者数据库迁移数据时，除检查日志外，还需要检查源和目标的对象数据量、是否有失效对象。
```sql
  SELECT object_type s_object_type, COUNT (*)
    FROM dba_objects
   WHERE owner = 'ITPUX01'
GROUP BY object_type;

  SELECT object_type t_object_type, COUNT (*)
    FROM dba_objects
   WHERE owner = 'ITPUX01'
GROUP BY object_type;

SELECT *
  FROM dba_objects
 WHERE status <> 'VALID' AND owner = 'ITPUX01';
```
* dblink两个库一起查
```sql
SELECT s.s_object_type,
       s.s_count,
       t.t_object_type,
       t.t_count
  FROM (  SELECT object_type s_object_type, COUNT (*) s_count
            FROM dba_objects@db01
           WHERE owner = 'ITPUX01'
        GROUP BY object_type) s,
       (  SELECT object_type t_object_type, COUNT (*) t_count
            FROM dba_objects
           WHERE owner = 'ITPUX01'
        GROUP BY object_type) t;
```
* 自动生成编译无效对象SQL及编译过程
>统计当前用户无效对象数量
```sql
  SELECT owner,
         object_type,
         status,
         COUNT (*)
    FROM dba_objects
   WHERE status <> 'VALID'
GROUP BY owner, object_type, status
ORDER BY owner, object_type;
```

>生成编译无效对象SQL
```sql
SELECT    'ALTER '
       || OBJECT_TYPE
       || ' '
       || OWNER
       || '.'
       || OBJECT_NAME
       || ' COMPILE;'
  FROM dba_objects
 WHERE     status <> 'VALID'
       AND object_type IN ('PACKAGE',
                           'PACKAGE BODY',
                           'FUNCTION',
                           'PROCEDURE',
                           'TRIGGER',
                           'VIEW');
```
>通过复制以上SQL语句,直接手动执行编译执行.
也可以采用如下方式在oracle用户下进行手工编译
```sql
su - oracle
$ sqlplus / as sysdba
SQL> @$ORACLE_HOME/rdbms/admin/utlrp.sql
```
## 7.exp / imp迁移过程字符集的处理
> 进行数据的导入导出时，我们要注意关于字符集的问题。在EXP/IMP过程中我们需要注意四个字符集的参数：
```shell
01.导出端的客户端字符集。
02.导出端数据库字符集。
03.导入端的客户端字符集。
04.导入端数据库字符集。
```
>我们首先需要查看这四个字符集参数。
查看数据库的字符集的信息：
```sql
SQL> select * from nls_database_parameters;
//SQL> select * from props$;
SQL> SET PAGESIZE 30
SQL> select * from nls_database_parameters;

PARAMETER                      VALUE
------------------------------ ------------------------------
NLS_LANGUAGE                   AMERICAN
NLS_TERRITORY                  AMERICA
NLS_CURRENCY                   $
NLS_ISO_CURRENCY               AMERICA
NLS_NUMERIC_CHARACTERS         .,
NLS_CHARACTERSET               ZHS16GBK
NLS_CALENDAR                   GREGORIAN
NLS_DATE_FORMAT                DD-MON-RR
NLS_DATE_LANGUAGE              AMERICAN
NLS_SORT                       BINARY
NLS_TIME_FORMAT                HH.MI.SSXFF AM
NLS_TIMESTAMP_FORMAT           DD-MON-RR HH.MI.SSXFF AM
NLS_TIME_TZ_FORMAT             HH.MI.SSXFF AM TZR
NLS_TIMESTAMP_TZ_FORMAT        DD-MON-RR HH.MI.SSXFF AM TZR
NLS_DUAL_CURRENCY              $
NLS_COMP                       BINARY
NLS_LENGTH_SEMANTICS           BYTE
NLS_NCHAR_CONV_EXCP            FALSE
NLS_NCHAR_CHARACTERSET         UTF8
NLS_RDBMS_VERSION              11.2.0.4.0

20 rows selected.

```
>我们再来查看客户端的字符集信息：
客户端字符集的参数NLS_LANG=_< territory >.
language：指定oracle消息使用的语言，日期中日和月的显示。
Territory：指定货币和数字的格式，地区和计算星期及日期的习惯。
Characterset：控制客户端应用程序使用的字符集。通常设置或等于客户端的代码页。ZHS16GBK、UTF8。

* 如数据库语言环境
>os是windows,使用命令
set NLS_LANG=AMERICAN_AMERICA.ZHS16GBK

>os是linux or unix,使用命令
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
在unix中：
\

>$ env|grep NLS_LANG
NLS_LANG=AMERICAN_AMERICA.ZHS16GBK

>当前修改可用：
$ export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK

>永久修改可用：
vi bash_profile

* 通常在导出时最好把客户端字符集设置得和数据库端相同。当进行数据导入时，主要有以下两种情况：
>&emsp;&emsp;(1) 源数据库和目标数据库具有相同的字符集设置。
这时，只需设置导出和导入端的客户端NLS_LANG等于数据库字符集即可。
&emsp;&emsp;(2) 源数据库和目标数据库字符集不同。
&emsp;&emsp;先将导出端客户端的NLS_LANG设置成和导出端的数据库字符集一致，导出数据，然后将导入端客户端的NLS_LANG设置成和导出端一致，导入数据，这样转换只发生在数据库端，而且只发生一次。
&emsp;&emsp;这种情况下，只有当导入端数据库字符集为导出端数据库字符集的严格超集时，数据才能完全导成功，否则，可能会有数据不一致或乱码出现。
## 8.exp / imp优化的方法
>当需要exp/imp的数据量比较大时，这个过程需要的时间是比较长的，我们可以用一些方法来优化exp/imp的操作。

* 01.exp:使用直接路径 direct=y
>&emsp;&emsp;oracle会避开sql语句处理引擎,直接从数据库文件中读取数据,然后写入导出文件.
&emsp;&emsp;可以在导出日志中观察到: exp-00067: table xxx will be exported in conventional path
&emsp;&emsp;如果没有使用直接路径,必须保证buffer参数的值足够大.
&emsp;&emsp;有一些参数于direct=y不兼容,无法用直接路径导出可移动的tablespace,或者用query参数导出数据库子集.
&emsp;&emsp;当导入导出的数据库运行在不同的os下时,必须保证recordlength参数的值一致.

* 02.exp: 清空回收站
```sql
SQL> select count(*) from dba_recyclebin;
SQL> purge dba_recyclebin;
```

* 03.imp:避免磁盘排序
>涉及到sort_area_size参数,也就是我们的PGA要够大。

* 05.imp:优化日志缓冲区
>比如将log_buffer容量扩大10倍(最大不要超过5M),也就是我们的SGA要够大。

* 05.imp:避免日志切换等待
>增加重做日志组的数量,增大日志文件大小.

* 06.imp:使用commit
>commit = y
注意:这个方式不能处理包含LOB和LONG类型的表,对于这样的table,如果使用commit = y,每插入一行,就会执行一次提交.

* 07.imp:使用NOLOGGING方式减小重做日志大小
>在导入时指定参数indexes=n,只导入数据而忽略index,在导完数据后在通过脚本创建index,指定NOLOGGING选项

## 9.exp / imp 常见问题及解决方法
### 9.1 关于imp/exp版本的问题
>一般来说，从低版本导入到高版本问题不大，麻烦的是将高版本的数据导入到低版本中。
* 可以跨版本的使用EXP/IMP，但必须正确地使用EXP和IMP的版本（官方可查）：
```sql
1、使用IMP的版本匹配数据库的版本，如：要导入到11.2中，使用11.2的IMP工具。
2、使用EXP的版本匹配两个数据库中最低的版本，如：从11.2往10.2中导入，则使用10.2版本的EXP工具。
3、高版本的Export导出来的转储文件，低版本的Import读不了；低版本的Export导出来的转储文件，高版本的Import可以进行读取。
4、从Oracle低版本的Export数据可以Import到Oracle高版本中，但限于Oracle的相邻版本，两个不相邻版本间进行转换应借助中间版本。
5、exp/imp可以做到在不同版本Oracle、不同数据库上的迁移。
```

### 9.2 数据库对象已经存在
>&emsp;&emsp;一般情况, 导入数据前应该彻底删除目标数据下的表, 序列, 函数/过程,触发器等; 数据库对象已经存在, 按缺省的imp参数, 则会导入失败如果用了参数ignore=y, 会把exp文件内的数据内容导入如果表有唯一关键字的约束条件, 不合条件将不被导入如果表没有唯一关键字的约束条件, 将引起记录重复
### 9.3 数据库对象有主外键约束
>不符合主外键约束时, 数据会导入失败,
解决办法:
先导入主表, 再导入依存表
disable目标导入对象的主外键约束, 导入数据后, 再enable它们
### 9.4 权限不够
>如果要把A用户的数据导入B用户下, A用户需要有imp_full_database权限
### 9.5 导入大表存储分配失败
>&emsp;&emsp;默认的EXP时, compress = y, 也就是把所有的数据压缩在一个数据块上.
导入时, 如果不存在连续一个大数据块, 则会导入失败. 导出大表时, 记得compress= n, 则不会引起这种错误.
### 9.6 imp和exp使用的字符集不同
>&emsp;&emsp;如果字符集不同, 导入会失败, 可以改变unix环境变量或者NT注册表里NLS_LANG相关信息. 导入完成后再改回来.

### 9.7 imp和exp版本不能往上兼容
>可以从低版本导入高版本，但不能从高版本导入到低版本。
如果遇到迁移因版本不同的问题，可以用低版本的export 导出，到导入到低版本。

### 9.8 导出统计信息的问题。
>&emsp;&emsp;如果导出统计信息,在只导出部分数据,或不导出数据时,导出统计信息会报错.另如果未导出统计信息,但导入时,需导入统计信息
,那此时,导入后,统计信息会被锁住,而无法更新统计信息.
&emsp;&emsp;此时,我们可使用包dbms_stats.unlock_schema_stats来解锁.最好的办法是,在exp,imp时,加入参数statistics=none,不exp,imp统计信息,在导入完成后,在重新收集统计信息.

```sql
BEGIN
    DBMS_STATS.gather_database_stats;
END;
/

BEGIN
    DBMS_STATS.gather_schema_stats ('ITPUX02');
END;
/
```