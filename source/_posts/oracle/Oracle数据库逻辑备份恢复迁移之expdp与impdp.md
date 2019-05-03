---
title: Oracle数据库逻辑备份恢复迁移之expdp与impdp
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
  - Expdp
  - Impdp
  - 备份容灾
---

# Oracle数据库逻辑备份恢复迁移之expdp与impdp

## 注意

[Oracle 11G R2 官方文档][1]

## 1.Oracle数据泵expdp/impdp概念
* Oracle Database 10g引入了最新的数据泵(Data Dump)技术，数据泵导出导入(EXPDP和IMPDP)的作用
> 1）实现逻辑备份和逻辑恢复.
2）在数据库用户之间移动对象.
3）在数据库之间移动对象
4）实现表空间搬移.
* 数据泵导出导入与传统导出导入的区别
>&emsp;&emsp;在10g之前,传统的导出和导入分别使用exp工具和imp工具,从10g开始,不仅保留了原有的exp和imp工具,还提供了数据泵导出导入工具expdp和impdp.使用expdp和impdp时应该注意的事项：
1）exp和imp是客户端工具程序,它们既可以在可以客户端使用,也可以在服务端使用。
2）expdp和impdp是服务端的工具程序,他们只能在oracle服务端使用,不能在客户端使用。
3）imp只适用于exp导出文件,不适用于expdp导出文件;impdp只适用于expdp导出文件,而不适用于exp导出文件。

* 数据泵导出包括导出表,导出方案,导出表空间,导出数据库4种方式.

* oracle数据泵的工作流程如下
>1、在命令行执行命令
2、expdp/impdp命令调用dbms_datapump pl/sql包。 这个api提供高速的导出导入功能。
3、 当data 移动的时候， data pump 会自动选择direct path 或者external table mechanism 或者 两种结合的方式。
&emsp;&emsp;当metadata（对象定义） 移动的时候，data pump会使用dbms_metadata pl/sql包。 metadata api 将metadata（对象定义）存储在xml里。所有的进程都能load 和unload 这些metadata. 因为data pump 调用的是服务端的api, 所以当一个任务被调度或执行，客户端就可以退出连接，任务job 会在server 端继续执行，随后通过客户端实用程序从任何地方检查任务的状态和进行修改。

## 2.expdp/impdp 命令参数详解
### 2.1关于expdp
```shell
[oracle@db01:/home/oracle]$expdp -help
Export: Release 11.2.0.3.0 - Production on Mon Aug 22 00:16:30 2016
Copyright (c) 1982, 2011, Oracle and/or its affiliates. All rights reserved.
The Data Pump export utility provides a mechanism for transferring data objects
between Oracle databases. The utility is invoked with the following command:
Example: expdp scott/tiger DIRECTORY=dmpdir DUMPFILE=scott.dmp
You can control how Export runs by entering the 'expdp' command followed
by various parameters. To specify parameters, you use keywords:
Format: expdp KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
Example: expdp scott/tiger DUMPFILE=scott.dmp DIRECTORY=dmpdir SCHEMAS=scott
or TABLES=(T1:P1,T1:P2), if T1 is partitioned table
USERID must be the first parameter on the command line.
The available keywords and their descriptions follow. Default values are listed within square brackets.

ATTACH
Attach to an existing job.
For example, ATTACH=job_name.

该选项用于在客户会话与已存在导出作用之间建立关联.语法如下
attach=[schema_name.]job_name
schema_name用于指定方案名,job_name用于指定导出作业名.注意,如果使用attach选项,在命令行除了连接字符串和attach选项外,不能指定任何其他选项,示例如下:
expdp scott/tiger attach=scott.export_job

CLUSTER
Utilize cluster resources and distribute workers across the Oracle RAC.
Valid keyword values are: [Y] and N.
而在11GR2后EXPDP和IMDP的WORKER进程会在多个INSTANCE启动，所以DIRECTORY必须在共享磁盘上，如果没有设置共享磁盘还是指定cluster=no来防止报错

COMPRESSION
Reduce the size of a dump file.
Valid keyword values are: ALL, DATA_ONLY, [METADATA_ONLY] and NONE.
这个压缩比例可以和操作系统“gzip -9”相媲美，某些特例下有可能比gzip还要高效：1/7。
10g中的COMPRESSION参数只提供METADATA_ONLY和NONE两个选项，基本上没有提供压缩功能。
11g中的COMPRESSION参数提供四个选项，分别是ALL、DATA_ONLY、METADATA_ONLY和NONE，非常的丰富，稍后我们将使用ALL参数进行操作。

Oracle 11g的EXPDP工具提供了真正意义上的“备份压缩”，这个技术在备份空间不足的情况下非常实用。all 压缩元数据和对象数据 data_only 只压缩对象数据 metadata_only 只压缩元数据 none 不压缩任何数据。

CONTENT
Specifies data to unload.
Valid keyword values are: [ALL], DATA_ONLY and METADATA_ONLY.
该选项用于指定要导出的内容.默认值为all
content={all | data_only | metadata_only}
当设置content为all 时,将导出对象定义及其所有数据.为data_only时,只导出对象数据,为metadata_only时,只导出对象定义。
expdp scott/tiger directory=dump dumpfile=a.dump content=metadata_only

DATA_OPTIONS
Data layer option flags.
Valid keyword values are: XML_CLOBS.
用于为某些类型的数据提供选项

DIRECTORY
Directory object to be used for dump and log files.
指定转储文件和日志文件所在的目录，directory=directory_object
directory_object用于指定目录对象名称.需要注意,目录对象是使用create directory语句建立的对象,而不是os 目录。

expdp scott/tiger directory=dump dumpfile=a.dump
先在对应的位置创建物理文件夹，如d:/backup
建立目录:
create or replace directory backup as '/opt/oracle/utl_file'
sql>create directory backup as ‘d:/backup’;
sql>grant read,write on directory backup to system;
查询创建了那些子目录:
select * from dba_directories;

DUMPFILE
Specify list of destination dump file names [expdat.dmp].
For example, DUMPFILE=scott1.dmp, scott2.dmp, dmpdir:scott3.dmp.
用于指定转储文件的名称,默认名称为expdat.dmp
dumpfile=[directory_object:]file_name [,….]
directory_object用于指定目录对象名,file_name用于指定转储文件名.需要注意,如果不指定directory_object,导出工具会自动使用directory选项指定的目录对象：expdp scott/tiger directory=dump1 dumpfile=dump2:a.dmp

ENCRYPTION
Encrypt part or all of a dump file.
Valid keyword values are: ALL, DATA_ONLY, ENCRYPTED_COLUMNS_ONLY, METADATA_ONLY and NONE.
是否加密导出数据 默认none
encryption={all | data_only |metadata_only | encrypted_columns_only | none}
all 对象和元数据 data_only 对象加密 metadata_only 元数据加密 encryption_columns_only 只加密加密列 none 不加密

ENCRYPTION_ALGORITHM
Specify how encryption should be done.
Valid keyword values are: [AES128], AES192 and AES256.
加密算法 默认aes128
encryption_algorithm= {aes128 | aes 192 | aes 256}

ENCRYPTION_MODE
Method of generating encryption key.
Valid keyword values are: DUAL, PASSWORD and [TRANSPARENT].
加密和解密所使用的安全类型
encryption_mode={dual | password |transparent }
dual 表示用 oracle wallet 或指定口令建立导出文件password 指定口令建立导出文件 transparent oracle wallet建立导出文件

ENCRYPTION_PASSWORD
Password key for creating encrypted data within a dump file.
指定加密和解密口令 encryption_password=password

ESTIMATE
Calculate job estimates.
Valid keyword values are: [BLOCKS] and STATISTICS.
指定估算被导出表所占用磁盘空间分方法.默认值是blocks。
extimate={blocks | statistics}
设置为blocks时,oracle会按照目标对象所占用的数据块个数乘以数据块尺寸估算对象占用的空间,设置为statistics时,根据最近统计值估算对象占用空间: expdp scott/tiger tables=emp estimate=statistics directory=dump dumpfile=a.dump

ESTIMATE_ONLY
Calculate job estimates without performing the export.
指定是否只估算导出作业所占用的磁盘空间,默认值为n
extimate_only={y | n}
设置为y时,导出作用只估算对象所占用的磁盘空间,而不会执行导出作业,为n时,不仅估算对象所占用的磁盘空间,还会执行导出操作.
expdp scott/tiger estimate_only=y nologfile=y

EXCLUDE
Exclude specific object types.
For example, EXCLUDE=SCHEMA:"='HR'".
该选项用于指定执行操作时释放要排除对象类型或相关对象 ITPUX

exclude=object_type[:name_clause] [,….]
object_type用于指定要排除的对象类型,name_clause用于指定要排除的具体对象.exclude和include不能同时使用。
expdp scott/tiger directory=dump dumpfile=a.dup exclude=view
FILESIZE
Specify the size of each dump file in units of bytes.
指定导出文件的最大尺寸,默认为0,(表示文件尺寸没有限制)
FLASHBACK_SCN
SCN used to reset session snapshot.
指定导出特定scn时刻的表数据。flashback_scn=scn_value
scn_value用于标识scn值.flashback_scn和flashback_time不能同时使用： expdp
scott/tiger directory=dump dumpfile=a.dmp flashback_scn=358523

FLASHBACK_TIME
Time used to find the closest corresponding SCN value.
指定导出特定时间点的表数据
flashback_time=”to_timestamp(time_value)”
expdp scott/tiger directory=dump dumpfile=a.dmp flashback_time= “to_timestamp(’25-08-2004 14:35:00’,’dd-mm-yyyy hh24:mi:ss’)”

FULL
Export entire database [N].
指定数据库模式导出,默认为n。 full={y | n} 。为y时,标识执行数据库导出.
HELP
Display Help messages [N].
指定是否显示expdp命令行选项的帮助信息,默认为n。当设置为y时,会显示导出选项的帮助信息. expdp help=y

INCLUDE
Include specific object types.
For example, INCLUDE=TABLE_DATA.
指定导出时要包含的对象类型及相关对象。include = object_type[:name_clause] [,… ]

JOB_NAME
Name of export job to create.
指定要导出作用的名称,默认为sys_xxx 。job_name=jobname_string

LOGFILE
Specify log file name [export.log].
指定导出日志文件文件的名称,默认名称为export.log
logfile=[directory_object:]file_name
directory_object用于指定目录对象名称,file_name用于指定导出日志文件名.如果不指定directory_object.导出作用会自动使用directory的相应选项值.
expdp scott/tiger directory=dump dumpfile=a.dmp logfile=a.log

NETWORK_LINK
Name of remote database link to the source system.
指定数据库链接名,如果要将远程数据库对象导出到本地例程的转储文件中,必须设置该选项.

NOLOGFILE
Do not write log file [N].
该选项用于指定禁止生成导出日志文件,默认值为n.

PARALLEL
Change the number of active workers for current job.
指定执行导出操作的并行进程个数,默认值为1
一般是cpu的2倍，可以被文件个数整除

PARFILE
Specify parameter file name.
指定导出参数文件的名称。parfile=[directory_path] file_name

QUERY
Predicate clause used to export a subset of a table.
For example, QUERY=employees:"WHERE department_id > 10".
用于指定过滤导出数据的where条件
query=[schema.] [table_name:] query_clause
schema用于指定方案名,table_name用于指定表名,query_clause用于指定条件限制子句.query选项不能与connect=metadata_only,extimate_only,transport_tablespaces等选项同时使用.
expdp scott/tiger directory=dump dumpfiel=a.dmp tables=emp query=’where deptno=20’

REMAP_DATA
Specify a data conversion function.
For example, REMAP_DATA=EMP.EMPNO:REMAPPKG.EMPNO.
用于转换列的数据函数， 并将转换值导出到文件中
remap_data=[schema1.]tablename.column_name:[schema2.]pkg.func
REUSE_DUMPFILES
Overwrite destination dump file if it exists [N].
覆盖已处在的导出文件，默认N

SAMPLE
Percentage of data to be exported.
用于指定被采样数据块的百分比
sample=[[schema_name.]table_name:]sample_percent (采用比率)

SCHEMAS
List of schemas to export [login schema].
该方案用于指定执行方案模式导出,默认为当前用户方案.

SERVICE_NAME
Name of an active Service and associated resource group to constrain Oracle RAC resources.

SOURCE_EDITION
Edition to be used for extracting metadata.
用于提取源数据的版本

STATUS
Frequency (secs) job status is to be monitored where
the default [0] will show new status when available.
指定显示导出作用进程的详细状态,默认值为0

TABLES
Identifies a list of tables to export.
For example, TABLES=HR.EMPLOYEES,SH.SALES:SALES_1995.
指定表模式导出
tables=[schema_name.]table_name[:partition_name][,…]
schema_name用于指定方案名,table_name用于指定导出的表名,partition_name用于指定要导出的分区名.

TABLESPACES
Identifies a list of tablespaces to export.
指定要导出表空间列表

TRANSPORTABLE
Specify whether transportable method can be used.
Valid keyword values are: ALWAYS and [NEVER].
指定是否可以使用可传输方法

TRANSPORT_FULL_CHECK
Verify storage segments of all tables [N].
	该选项用于指定被搬移表空间和未搬移表空间关联关系的检查方式,默认为n. 当设置为y时,导出作用会检查表空间直接的完整关联关系,如果表空间所在表空间或其索引所在的表空间只有一个表空间被搬移,将显示错误信息.当设置为n时,导出作用只检查单端依赖,如果搬移索引所在表空间,但未搬移表所在表空间,将显示出错信息,如果搬移表所在表空间,未搬移索引所在表空间,则不会显示错误信息.

TRANSPORT_TABLESPACES
List of tablespaces from which metadata will be unloaded.
指定执行表空间模式导出 ，用于指定搬移的的表空间
VERSION
Version of objects to export.
Valid keyword values are: [COMPATIBLE], LATEST or any valid database version.
用于指定被导出对象的数据库版本 默认 compatible
version={compatable | latest |version_string}
compatible 根据compatible参数生成对象 latest 根据数据库的实际版本 version_string 指定数据库版本（>9.2）
------------------------------------------------------------------------------
The following commands are valid while in interactive mode.
Note: abbreviations are allowed.
ADD_FILE
Add dumpfile to dumpfile set. 

向转储文件集中添加转储文件

CONTINUE_CLIENT
Return to logging mode. Job will be restarted if idle.
返回到记录模式。如果处于空闲状态, 将重新启动作业。

EXIT_CLIENT
Quit client session and leave job running.
退出客户机会话并使作业处于运行状态

FILESIZE
Default filesize (bytes) for subsequent ADD_FILE commands.
后续 ADD_FILE 命令的默认文件大小 (字节)。

HELP
Summarize interactive commands.
总结交互命令。

KILL_JOB
Detach and delete job.
分离和删除作业
PARALLEL
Change the number of active workers for current job.
更改当前作业的活动 worker 的数目。

REUSE_DUMPFILES
Overwrite destination dump file if it exists [N].
是否覆盖dumpfiles

START_JOB
Start or resume current job.

Valid keyword values are: SKIP_CURRENT.
启动/恢复当前作业。
STATUS
Frequency (secs) job status is to be monitored where
the default [0] will show new status when available.
在默认值 (0) 将显示可用时的新状态的情况下,要监视的频率 (以秒计) 作业状态。
STATUS[=interval]
STOP_JOB
Orderly shutdown of job execution and exits the client.
Valid keyword values are: IMMEDIATE.
顺序关闭执行的作业并退出客户机。STOP_JOB=IMMEDIATE 将立即关闭数据泵作业。
```
## 2.2 关于impdp
```shell
[oracle@db01:/home/oracle]$impdp -help
Import: Release 11.2.0.3.0 - Production on Mon Aug 22 00:16:43 2016
Copyright (c) 1982, 2011, Oracle and/or its affiliates. All rights reserved.
The Data Pump Import utility provides a mechanism for transferring data objects
between Oracle databases. The utility is invoked with the following command:
Example: impdp scott/tiger DIRECTORY=dmpdir DUMPFILE=scott.dmp
You can control how Import runs by entering the 'impdp' command followed
by various parameters. To specify parameters, you use keywords:
Format: impdp KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
Example: impdp scott/tiger DIRECTORY=dmpdir DUMPFILE=scott.dmp
USERID must be the first parameter on the command line.
------------------------------------------------------------------------------
The available keywords and their descriptions follow. Default values are listed within square brackets.
ATTACH
Attach to an existing job.
For example, ATTACH=job_name.
CLUSTER
Utilize cluster resources and distribute workers across the Oracle RAC.
Valid keyword values are: [Y] and N.
CONTENT
Specifies data to load.
Valid keywords are: [ALL], DATA_ONLY and METADATA_ONLY.
DATA_OPTIONS
Data layer option flags.
Valid keywords are: SKIP_CONSTRAINT_ERRORS.
DIRECTORY
Directory object to be used for dump, log and SQL files.
DUMPFILE
List of dump files to import from [expdat.dmp].
For example, DUMPFILE=scott1.dmp, scott2.dmp, dmpdir:scott3.dmp.
ENCRYPTION_PASSWORD
Password key for accessing encrypted data within a dump file.
Not valid for network import jobs.
ESTIMATE
Calculate job estimates.
Valid keywords are: [BLOCKS] and STATISTICS.
EXCLUDE
Exclude specific object types.
For example, EXCLUDE=SCHEMA:"='HR'".
FLASHBACK_SCN
SCN used to reset session snapshot.
FLASHBACK_TIME
Time used to find the closest corresponding SCN value.
FULL
Import everything from source [Y].
HELP
Display help messages [N].
INCLUDE
Include specific object types.
For example, INCLUDE=TABLE_DATA.
JOB_NAME
Name of import job to create.
LOGFILE
Log file name [import.log].
NETWORK_LINK
Name of remote database link to the source system.
NOLOGFILE
Do not write log file [N].
PARALLEL
Change the number of active workers for current job.
PARFILE
Specify parameter file.
PARTITION_OPTIONS
Specify how partitions should be transformed.
Valid keywords are: DEPARTITION, MERGE and [NONE].
QUERY
Predicate clause used to import a subset of a table.
For example, QUERY=employees:"WHERE department_id > 10".
REMAP_DATA
Specify a data conversion function.
For example, REMAP_DATA=EMP.EMPNO:REMAPPKG.EMPNO.
REMAP_DATAFILE
Redefine data file references in all DDL statements.
REMAP_SCHEMA
Objects from one schema are loaded into another schema.
REMAP_TABLE
Table names are remapped to another table.
For example, REMAP_TABLE=HR.EMPLOYEES:EMPS.
REMAP_TABLESPACE
Tablespace objects are remapped to another tablespace.
REUSE_DATAFILES
Tablespace will be initialized if it already exists [N].
SCHEMAS
List of schemas to import.
SERVICE_NAME
Name of an active Service and associated resource group to constrain Oracle RAC resources.
SKIP_UNUSABLE_INDEXES
Skip indexes that were set to the Index Unusable state.
SOURCE_EDITION
Edition to be used for extracting metadata.
SQLFILE
Write all the SQL DDL to a specified file.
STATUS
Frequency (secs) job status is to be monitored where
the default [0] will show new status when available.
STREAMS_CONFIGURATION
Enable the loading of Streams metadata
TABLE_EXISTS_ACTION
Action to take if imported object already exists.
Valid keywords are: APPEND, REPLACE, [SKIP] and TRUNCATE.
TABLES
Identifies a list of tables to import.
For example, TABLES=HR.EMPLOYEES,SH.SALES:SALES_1995.
TABLESPACES
Identifies a list of tablespaces to import.
TARGET_EDITION
Edition to be used for loading metadata.
TRANSFORM
Metadata transform to apply to applicable objects.
Valid keywords are: OID, PCTSPACE, SEGMENT_ATTRIBUTES and STORAGE.
TRANSPORTABLE
Options for choosing transportable data movement.
Valid keywords are: ALWAYS and [NEVER].
Only valid in NETWORK_LINK mode import operations.
TRANSPORT_DATAFILES
List of data files to be imported by transportable mode.
TRANSPORT_FULL_CHECK
Verify storage segments of all tables [N].
TRANSPORT_TABLESPACES
List of tablespaces from which metadata will be loaded.
Only valid in NETWORK_LINK mode import operations.
VERSION
Version of objects to import.
Valid keywords are: [COMPATIBLE], LATEST or any valid database version.
Only valid for NETWORK_LINK and SQLFILE.
------------------------------------------------------------------------------
The following commands are valid while in interactive mode.
Note: abbreviations are allowed.
CONTINUE_CLIENT
Return to logging mode. Job will be restarted if idle.
EXIT_CLIENT
Quit client session and leave job running.
HELP
Summarize interactive commands.
KILL_JOB
Detach and delete job.
PARALLEL
Change the number of active workers for current job.
START_JOB
Start or resume current job.
Valid keywords are: SKIP_CURRENT.
STATUS
Frequency (secs) job status is to be monitored where
the default [0] will show new status when available.
STOP_JOB
Orderly shutdown of job execution and exits the client.
Valid keywords are: IMMEDIATE.

其实IMPDP命令行选项与EXPDP有很多相同的,下面我们只介绍不同的部分：
（1）REMAP_DATAFILE
该选项用于将源数据文件名转变为目标数据文件名,在不同平台之间搬移表空间时可能需要该选项.
REMAP_DATAFIEL=source_datafie:target_datafile

（2）REMAP_SCHEMA
该选项用于将源方案的所有对象装载到目标方案中.
REMAP_SCHEMA=source_schema:target_schema

（3）REMAP_TABLESPACE
将源表空间的所有对象导入到目标表空间中
REMAP_TABLESPACE=source_tablespace:target_tablespace

（4）REUSE_DATAFILES
该选项指定建立表空间时是否覆盖已存在的数据文件.默认为N。
REUSE_DATAFIELS={Y | N}

（5）SKIP_UNUSABLE_INDEXES
指定导入是是否跳过不可使用的索引,默认为N

（6）SQLFILE
指定将导入要指定的索引DDL操作写入到SQL脚本中。
SQLFILE=[directory_object:]file_name
Impdp scott/tiger DIRECTORY=dump DUMPFILE=tab.dmp SQLFILE=a.sql

（7）STREAMS_CONFIGURATION
指定是否导入流元数据(Stream Matadata),默认值为Y.

（8）TABLE_EXISTS_ACTION
该选项用于指定当表已经存在时导入作业要执行的操作,默认为SKIP
TABBLE_EXISTS_ACTION={SKIP | APPEND | TRUNCATE | FRPLACE }
当设置该选项为SKIP时,导入作业会跳过已存在表处理下一个对象;当设置为APPEND时,会追加数据,为TRUNCATE时,导入作业会截断表,然后为其追加新数据;当设置为REPLACE时,导入作业会删除已存在表,重建表病追加数据,注意,TRUNCATE选项不适用与簇表和NETWORK_LINK选项

（9）TRANSFORM
该选项用于指定是否修改建立对象的DDL语句
TRANSFORM=transform_name:value[:object_type]
Transform_name用于指定转换名,其中SEGMENT_ATTRIBUTES用于标识段属性(物理属性,存储属性,表空间,日志等信息),STORAGE用于标识段存储属性,VALUE用于指定是否包含段属性或段存储属性,object_type用于指定对象类型.
Impdp scott/tiger directory=dump dumpfile=tab.dmp Transform=segment_attributes:n:table

（10）TRANSPORT_DATAFILES
该选项用于指定搬移空间时要被导入到目标数据库的数据文件。
TRANSPORT_DATAFILE=datafile_name
Datafile_name用于指定被复制到目标数据库的数据文件
Impdp system/manager DIRECTORY=dump DUMPFILE=tts.dmp TRANSPORT_DATAFILES=’/user01/data/tbs1.f’
```
## 3.expdp/impdp 使用方法详解


### 3.1 oracle expdp 使用方法介绍
```shell
使用expdp工具时,其转储文件只能被存放在directory对象对应的os目录中,而不能直接指定转储文件所在的os目录.因此,使用expdp工具时,必须首先建立directory对象.并且需要为数据库用户授予使用directory对象权限.
--创建directory:
create directory itpuxbak_dir as '/backup';
grant read,write on directory itpuxbak_dir to system;
grant read,write on directory itpuxbak_dir to itpux01;

--01)导出整个数据库
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_full_db01.dmp logfile=expdp_full_db01.log full=y
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_full_db01_%U.dmp logfile=expdp_full_db01.log full=y parallel=4

--02)导出方案-schema-用户
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_u_itpux01.dmp logfile=expdp_u_itpux01.log schemas=itpux01,itpux02

--03)导出表空间
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_ts_itpux01.dmp logfile=expdp_ts_itpux01.log tablespaces=itpux01,itpux02

--04)导出表
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=expdp_tb_itpux01.log tables=itpux01,itpux02
expdp itpux01/itpux01 directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=expdp_tb_itpux01.log tables=itpux01

--05)按表查询条件导出
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_q_itpux01.dmp logfile=expdp_q_itpux01.log tables=itpux01 query='where id=5'
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_q_itpux01.dmp logfile=expdp_q_itpux01.log QUERY=employees:"WHERE department_id > 10"
```
### 3.2 oracle impdp 使用方法介绍
>使用impdp工具时,如果数据库中不存在相应的directory，必须首先建立directory对象.并且需要为数据库用户授予使用directory对象权限.
```sql
--创建directory:
create directory itpuxbak_dir as '/backup';
grant read,write on directory itpuxbak_dir to system;
grant read,write on directory itpuxbak_dir to itpux01;

--01)导入整个数据库
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_full_db01.dmp logfile=impdp_full_db01.log full=y
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_full_db01_%U.dmp logfile=impdp_full_db01.log full=y parallel=4

--02)导入方案-schema-用户
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_u_itpux01.dmp logfile=impdp_u_itpux01.log schemas=itpux01,itpux02
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_u_itpux01.dmp logfile=impdp_u_itpux01.log schemas=itpux01,itpux02 remap_schema=itpux02:itpux002

--03)导入表空间
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_ts_itpux01.dmp logfile=impdp_ts_itpux01.log tablespaces=itpux01,itpux02

--04)导入表
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01,itpux02
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01,itpux02 remap_schema=system:itpux
impdp itpux01/itpux01 directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log TABLES=HR.EMPLOYEES,SH.SALES:SALES_1995

--05)按表查询条件导入
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01,itpux02 query='where id=5'
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01,itpux02 remap_schema=system:itpux query='where id=5'

impdp itpux01/itpux01 directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log tables=itpux01 query='where id=5'
impdp system/oracle directory=itpuxbak_dir dumpfile=expdp_tb_itpux01.dmp logfile=impdp_tb_itpux01.log TABLES=HR.EMPLOYEES,SH.SALES:SALES_1995 QUERY=HR.EMPLOYEES:"WHERE department_id > 10"
```

### 3.3 oracle expdp/impdp补充
```shell
--01)directory
create directory itpuxbak_dir as '/backup';
--drop directory itpuxbak_dir
select * from dba_directories
grant read,write on directory itpuxbak_dir to system; --system:itpux01:other users
grant create any directory to system;
select * from dba_sys_privs where grantee='SYSTEM';

--02) sysdba full=y
expdp "'/as sysdba'" directory=itpuxbak_dir dumpfile=expdp_full_db01.dmp logfile=expdp_full_db01.log full=y --sys

--03)查询datapump jobs
select * from dba_datapump_jobs;
expdp "'/as sysdba'" directory=itpuxbak_dir dumpfile=expdp_full_db01.dmp logfile=expdp_full_db01.log full=y job_name=itpuxexpjob;
```

## 4.配置生产环境的逻辑自动备份策略
>每天做逻辑备份，数据量大小500G，保留2天，空间准备2TB
我的库不开归档的，不能在线做RMAN。
```shell
create directory itpuxbak_dir as '/backup';
grant read,write on directory itpuxbak_dir to system;
grant create any directory to system;

#vi expdpfull_db01.sh
export BAKDATE=`date +%Y%m%d`
expdp system/oracle directory=itpuxbak_dir dumpfile=expdp_full_db01.$BAKDATE.%U.dmp logfile=expdp_full_db01.$BAKDATE.log full=y parallel=4
find /backup -name expdp_full_db01.*.dmp -atime +2 -exec rm -rf {} \;

#--cluster=N
#chown -R oracle:dba /backup
#chmod -R 775 /backup
#crontab -e
0 20 * * * su - oracle -c /backup/scripts/expdpfull_db01.sh
```
* --报错的处理：
```shell
ORA-39181: Only partial table data may be exported due to fine grain access control on "OE"."PURCHASEORDER"
select count(*) from "OE"."PURCHASEORDER"
grant EXEMPT ACCESS POLICY to system;
expdp system/oracle directory=itpuxbak_dir dumpfile=test-b.dmp logfile=test-b.log tables=OE.PURCHASEORDER
```

## 5.expdp/impdp生产环境数据迁移流程
* 5.1、做数据迁移流程的目的
>使用expdp/impdp进行数据迁移的前置条件、操作步骤，降低对对应用造成的影响及避免故障

* 5.2、数据迁移的适用范围
>所有线上库>10.2

* 5.3、 数据迁移的风险评估
>01.有些os对文件大小有限制，expdp数据时需要使用filesize参数来分割导出文件
02.expdp导出数据时没有正确估计dmp文件所需空间，导致主机磁盘满。
03.跨字符集的数据迁移，由于字符集不兼容导致数据迁移失败。
04.导入表空间不存在或者空间不足，导致表创建失败或者数据导入失败，导致其他应用报错

* 5.4、数据迁移的准备工作
>01.检查源数据库和目标库的版本、字符集，如果目标库版本低于源库，使用目标库的软件做导出。如果字符集不一致，不建议使用expdp/impdp迁移数据。
02.user_segments里查出导出表所占的空间大小，检查os对文件大小的限制。
03.表比较多的情况下，建议用parfile。各个参数在parfile里写好。
04.提前准备备份脚本（parfile参数）：
```shell
cat expdp_itpux.par
userid=system/oracle
directory=itpuxbak_dir
dumpfile=expdp_full_db01.$BAKDATE.%U.dmp
logfile=expdp_full_db01.$BAKDATE.log
tables=user.tab1,user.tab2,user.tab3
parallel=4

vi expdp.sh

export BAKDATE=`date +%Y%m%d`
expdp parfile=expdp_itpux.par
nohup ./expdp.sh &
```

* 5.5、数据迁移的执行过程


* 5.6、数据迁移后的验证方案
>对比expdp、impdp的日志，确认导出导入数据量是否一致。并在数据库上检查数据量。
编译无效对象


## 6.expdp/impdp生产环境数据迁移案例
>迁移过程详细请看视频演示，此处只做参考
### 6.1 迁移目的
>将linux系统oracle服务器上schema（itpux01,itpux02）全部通过ExpDP全库迁移到另一台oracle服务器，并能正常查询到相关数据。
>IMPDP虽然加上parallel参数，在做table数据导入时确实速度提高了不少，但是在create index和statistics时，依旧采用单线程的方式，故此一般在迁移过程中这两步操作选择生产DDL脚本增加parallel参数后手工执行，以最大化缩减迁移时间。
### 6.2 迁移流程
```sql
1.linux系统oracle服务器上创建测试数据。
2.linux本地用Expdp做导出;
3.远程主机创建相关对象；
4.远程主机用expdp做导入(导入);
5.验证远程本地数据合法性;
```
### 6.3 演示过程
* 01.准备数据
```sql
/* Formatted on 2018/1/30 21:28:32 (QP5 v5.313) */
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
--创建用户并授权
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

--用户登录并创建测试表
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

--要建两个索引

CREATE INDEX itpux02
    ON itpux02_table_test (id);

--创建2个存储过程并执行。
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
/* Formatted on 2018/1/30 21:31:51 (QP5 v5.313) */
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

--02.获取DDL
--在源主机获取：
SPOOL itpux_tbs_create_ddl.sql
SET LONG 200000 PAGESIZE 0 HEAD OFF VERIFY OFF FEEDBACK OFF LINESIZE 200

SELECT DBMS_METADATA.get_ddl ('TABLESPACE', 'ITPUX01') FROM DUAL;

SELECT DBMS_METADATA.get_ddl ('TABLESPACE', 'ITPUX02') FROM DUAL;

SPOOL OFF;
--在另一台主机执行：
CREATE TABLESPACE "ITPUX01"
    DATAFILE '/oracle/oradata/db01/itpux01.dbf'
                 SIZE 50 M
    LOGGING
    ONLINE
    PERMANENT
    BLOCKSIZE 8192
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    DEFAULT
    NOCOMPRESS
    SEGMENT SPACE MANAGEMENT AUTO;
CREATE TABLESPACE "ITPUX02"
    DATAFILE '/oracle/oradata/db01/itpux02.dbf'
                 SIZE 50 M
    LOGGING
    ONLINE
    PERMANENT
    BLOCKSIZE 8192
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    DEFAULT
    NOCOMPRESS
    SEGMENT SPACE MANAGEMENT AUTO;
--03、源库导出
建立目录（建立directory）
SQL> CREATE OR REPLACE DIRECTORY oradmp AS '/oracle/backup';
SQL> GRANT READ,WRITE ON DIRECTORY oradmp TO SYSTEM;
expdp SYSTEM/oracle DIRECTORY=oradmp FULL=y dumpfile=expdpfull_db01_%U.dmp LOGFILE=expdpfull_db01.LOG PARALLEL=2;
--04、目标库导入
禁止自动维护任务
（禁用数据库自动维护任务，oracle11g中存在）
SQL> EXECUTE DBMS_AUTO_TASK_ADMIN.DISABLE;
建立目录（建立directory）
SQL> CREATE OR REPLACE DIRECTORY oradmp AS '/oracle/backup';
SQL> GRANT READ,WRITE ON DIRECTORY oradmp TO SYSTEM;
生成index、CONSTRAINT的ddl语句
impdp SYSTEM/SYSTEM DIRECTORY=oradmp dumpfile=itpux_01.dmp,itpux_02.dmp LOGFILE=impitpux.LOG PARALLEL=2 SQLFILE=indconddl.SQL INCLUDE=INDEX,CONSTRAINT SCHEMAS=itpux
编辑ddl语句脚本
sed 's/NOLOGGING/ /g;s/LOGGING/ /g' indconddl.SQL>indconddl.sql.nologin
sed '/TABLESPACE/ s/TS_itpux/TS_itpux_INDEX/g;/TABLESPACE/ s/1/32 NOLOGGING/g' indconddl.sql.nologin>indconddl.sql.NEW
设置parallel 4;
执行导入语句
impdp SYSTEM/SYSTEM DIRECTORY=oradmp FULL=y dumpfile=expdpfull_db01_%U.dmp LOGFILE=impdpfull_db01.LOG PARALLEL=2
EXCLUDE=INDEX,CONSTRAINT,STATISTICS SCHEMAS=ITPUX;
检查job，延后迁移时间内发起的job任务

SELECT JOB,
       LOG_USER,
       SCHEMA_USER,
       what,
       LAST_DATE,
       LAST_SEC,
       NEXT_DATE,
       NEXT_SEC,
       FAILURES,
       BROKEN
  FROM DBA_JOBS;

执行ddl语句脚本，建立索引、约束，手工发起统计分析
建立索引、约束
chmod +x indconddl.sql.NEW
 more createindcon.sh
SQLPLUS -s /nolog <<EOS
CONNECT itpux / itpux
SPOOL /tmp/indconcreate.log
@ /oracle/backup/indconddl.sql.new;
SPOOL OFF
EXIT
EOS
执行createindcon.sh
然后再改parallel 4;为NOPARALLEL
--05、收集统计信息
stats.SQL:

BEGIN
    DBMS_STATS.gather_database_stats;
END;
/

nohup sqlplus "/as sysdba" @ stats.SQL &
--06、无效对象的编译
自动生成编译无效对象SQL及编译过程
1) 统计当前用户无效对象数量:
  SELECT OWNER,
         object_type,
         status,
         COUNT (*)
    FROM dba_objects
   WHERE status <> 'VALID'
GROUP BY OWNER, object_type, status
ORDER BY OWNER, object_type;
2) 生成编译无效对象SQL
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
通过复制以上SQL语句,直接手动执行编译执行.
也可以采用如下方式在oracle用户下进行手工编译
# su - oracle
$ sqlplus / as sysdba
SQL> @$ORACLE_HOME/rdbms/ADMIN/utlrp.SQL
--07、验证数据
对比expdp、impdp的日志，确认导出导入数据量是否一致。并在数据库上检查数据量。
比如上面的数据迁移，检查数据量跟日志显示是否一致。

SELECT COUNT (*) FROM itpux.itpux01;

SQL>SELECT OWNER,OBJECT_TYPE,COUNT(*) FROM dba_objects WHERE wner='&owner' GROUP BY OWNER,OBJECT_TYPE ORDER BY OBJECT_TYPE;
跨schema或者数据库迁移数据时，除检查日志外，还需要检查源和目标的对象数据量、是否有失效对象。
检查对象状态
SQL>SELECT OWNER,OBJECT_NAME,SUBOBJECT_NAME,STATUS,OBJECT_TYPE FROM dba_objects WHERE wner='&owner' ORDER BY OBJECT_TYPE, OBJECT_NAME, SUBOBJECT_NAME;

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

--08、收尾
启用数据库自动维护任务
SQL> EXECUTE DBMS_AUTO_TASK_ADMIN.ENABLE;
检查并更正job运行时间

SELECT JOB,
       LOG_USER,
       SCHEMA_USER,
       what,
       LAST_DATE,
       LAST_SEC,
       NEXT_DATE,
       NEXT_SEC,
       FAILURES,
       BROKEN
  FROM DBA_JOBS;

--09、对外应用测试
```

## 7.expdp/impdp迁移过程字符集的处理
>进行数据的导入导出时，我们要注意关于字符集的问题。在EXPDP/IMPDP过程中我们需要注意四个字符集的参数：
```sql
01.导出端的客户端字符集。
02.导出端数据库字符集。
03.导入端的客户端字符集。
05.导入端数据库字符集。
```
* 查看数据库的字符集的信息
```sql
SQL> select * from nls_database_parameters;
//SQL> select * from props$;
NLS_LANGUAGE AMERICAN
NLS_TERRITORY AMERICA
NLS_CURRENCY $
NLS_ISO_CURRENCY AMERICA
NLS_NUMERIC_CHARACTERS .,
NLS_CHARACTERSET ZHS16GBK
NLS_CALENDAR GREGORIAN
NLS_DATE_FORMAT DD-MON-RR
NLS_DATE_LANGUAGE AMERICAN
NLS_SORT BINARY
NLS_TIME_FORMAT HH.MI.SSXFF AM
NLS_TIMESTAMP_FORMAT DD-MON-RR HH.MI.SSXFF AM
NLS_TIME_TZ_FORMAT HH.MI.SSXFF AM TZR
NLS_TIMESTAMP_TZ_FORMAT DD-MON-RR HH.MI.SSXFF AM TZR
NLS_DUAL_CURRENCY $
NLS_COMP BINARY
NLS_LENGTH_SEMANTICS BYTE
NLS_NCHAR_CONV_EXCP FALSE
NLS_NCHAR_CHARACTERSET AL16UTF16
NLS_RDBMS_VERSION 11.2.0.3.0
```

* 我们再来查看客户端的字符集信息
```shell
客户端字符集的参数NLS_LANG=_< territory >.
language：指定oracle消息使用的语言，日期中日和月的显示。
Territory：指定货币和数字的格式，地区和计算星期及日期的习惯。
Characterset：控制客户端应用程序使用的字符集。通常设置或等于客户端的代码页。ZHS16GBK、UTF8。
如数据库语言环境
os是windows,使用命令
set NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
os是linux or unix,使用命令
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
在unix中：
$ env|grep NLS_LANG
NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
当前修改可用：
$ export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
永久修改可用：
vi bash_profile
通常在导出时最好把客户端字符集设置得和数据库端相同。当进行数据导入时，主要有以下两种情况：
```
>(1) 源数据库和目标数据库具有相同的字符集设置。
这时，只需设置导出和导入端的客户端NLS_LANG等于数据库字符集即可。
(2) 源数据库和目标数据库字符集不同。
先将导出端客户端的NLS_LANG设置成和导出端的数据库字符集一致，导出数据，然后将导入端客户端的NLS_LANG设置成和导出端一致，导入数据，这样转换只发生在数据库端，而且只发生一次。
这种情况下，只有当导入端数据库字符集为导出端数据库字符集的严格超集时，数据才能完全导成功，否则，可能会有数据不一致或乱码出现。

## 8.expdp与impdp 版本兼容性与各版本的区别
参考资料：
http://www.itpux.com/thread-288-1-1.html
## 9.如何停止expdp与impdp备份任务的后台进程
参考资料：
http://www.itpux.com/thread-286-1-1.html
## 10.如何清理不需要的数据泵job
参考资料：
http://www.itpux.com/thread-287-1-1.html
## 11.如何对expdp/impdp进行trace跟踪分析问题
### 11.1 使用Trace 480300
>&emsp;&emsp;Data Pump工作原理有两个特点：作业调度，多进程配合协作。对Data Pump的诊断本质上就是对各种Process行为的跟踪。Oracle提供了一个Trace的隐含参数，来帮助我们实现这个目标。
>&emsp;&emsp;Trace并不像其他跟踪过程相同，使用y/n的参数，开启或者关闭。Data Pump的Trace参数是一个7位十六进制组成的数字串。不同的数字串表示不同的跟踪对象方法。7位十六进制数字分为两个部分，前三个数字表示特定的数据泵组件，后四位使用0300就可以。

* 各个组件分别使用不同的三位十六进制数字代表。如下片段所示
```shell
-- Summary of Data Pump trace levels:
-- ==================================
Trace DM DW ORA Lines
level trc trc trc in
(hex) file file file trace Purpose
------- ---- ---- ---- ------ -----------------------------------------------
10300 x x x SHDW: To trace the Shadow process (API) (expdp/impdp)
20300 x x x KUPV: To trace Fixed table
40300 x x x 'div' To trace Process services
80300 x KUPM: To trace Master Control Process (MCP) (DM)
100300 x x KUPF: To trace File Manager
200300 x x x KUPC: To trace Queue services
400300 x KUPW: To trace Worker process(es) (DW)
800300 x KUPD: To trace Data Package
1000300 x META. To trace Metadata Package
--- +
1FF0300 x x x 'all' To trace all components (full tracing)
```
>如果需要同时跟踪多个组件，需要将目标组件的hex值进行累加，后面四位的300相同。
>400300+80300=480300
>&emsp;&emsp;对于跟踪的Trace取值，Oracle建议使用480300就可以应对大部分的情况。480300会跟踪Oracle Dump作业的Master Control Process（MCP）和Work Process。作为初始化跟踪的过程，480300基本就够用了。
>我们先从数据导出Expdp看Trace，导出一个案例。首先清理一下Trace File目录
```sql
expdp \"/ as sysdba\" directory=dumpdir schemas=itpux dumpfile=itpux_dump.dmp parallel=2 trace=480300
```
>&emsp;&emsp;然后检查trc目录，Dm和dw标注的就是MCP和Work Process生成的Trace文件。同时Parallel设置使得dw有00和01两个。
>2个trace 文件在BACKGROUND_DUMP_DEST目录下：
```sql
Master Process trace file: <SID>_dm<number>_<process_id>.trc
Worker Process trace file: <SID>_dw<number>_<process_id>.trc
```
>在导出过程中，我们可以看到两个worker的会话信息。
```sql
SQL> select * from dba_datapump_sessions;
OWNER_NAME JOB_NAME INST_ID SADDR SESSION_TYPE
------------------------------ ------------------------------ ---------- -------- --------------
SYS SYS_EXPORT_SCHEMA_01 1 35EB0580 DBMS_DATAPUMP
SYS SYS_EXPORT_SCHEMA_01 1 35E95280 MASTER
SYS SYS_EXPORT_SCHEMA_01 1 35E8A480 WORKER
SYS SYS_EXPORT_SCHEMA_01 1 35E84D80 WORKER
```
>&emsp;&emsp;此时我们可以从Trace文件中，看到一些Data Pump工作的细节信息。例如：在MCP的Trace文件中，我们看到一系列调用动作过程，如下片段：
--初始化导出动作，整理文件系统；
-日志写入
在Worker Process中，如下片段看出在导出数据。
### 11.2 使用Trace 480301
>&emsp;&emsp;在Trace过程中，我们也可以如10046跟踪过程一样，添加SQL跟踪。Data Pump本质上工作还是一系列的SQL语句，很多时候的性能问题根源都是从SQL着手的。
>切换到SQL跟踪模式也比较简单，一般是在Trace数值后面添加1。
```sql
impdp \"/ as sysdba\" directory=dumpdir dumpfile=itpux_dump.dmp remap_schema=itpux:test trace=480301 parallel=2
```
>&emsp;&emsp;在Trace过程中，我们也可以如10046跟踪过程一样，添加SQL跟踪。Data Pump本质上工作还是一系列的SQL语句，很多时候的性能问题根源都是从SQL着手的。
>切换到SQL跟踪模式也比较简单，一般是在Trace数值后面添加1。我们使用导入过程进行实验。
>&emsp;&emsp;目录生成的Trace文件，都是10046格式的Raw文件。截取片段如下：
10046 生成的trace 文件可读性并不好，所有我们可以使用tkprof工具进行格式化，方便阅读。
```sql
$ tkprof itpux_dm00_17292.trc tkprof_itpux_dm00_17292.out waits=y sort=exeela
$ tkprof itpux_dw01_17294.trc tkprof_itpux__dw01_17294.out waits=y sort=exeela
```

### 11.3.使用10046事件
* 10046 事件有如下级别
```shell
event 10046, level 1 = enable standardSQL_TRACE functionality
event 10046, level 4 = as level 1, plus trace the BIND values
event 10046, level 8 = as level 1, plus trace the WAITs
event 10046, level 12 = as level 1, plus trace the BIND values and the WAITs
```
* 不同级别的使用情况如下
```shell
level 1: lowest level tracing - not alwayssufficient to determine cause of errors;
level 4: useful when an error in DataPump's worker or master process occurs;
level 12: useful when there is an issuewith Data Pump performance.
```
* 注意：
>&emsp;&emsp; 当我们设置10046的级别高于8或者12的时候，需要将TIMED_STATISTICS设置为TRUE. 临时的将这个参数设置为true，可以将trace数据性能的影响降到最低，在11gR2里，该参数默认为true。
>一般只有遇到性能问题时，才会使用8或者12的level。

### 11.4 日常使用
* 01、对当前正在运行的进程进行trace
```sql

--查看数据泵进程的信息：
SET LINES 150 PAGES 100 NUMWIDTH 7
COL username FOR a10
COL spid FOR a7
COL program FOR a25

SELECT TO_CHAR (SYSDATE, 'YYYY-MM-DDHH24:MI:SS') "DATE",
       s.program,
       s.sid,
       s.status,
       s.username,
       d.job_name,
       p.spid,
       s.serial#,
       p.pid
  FROM v$session s, v$process p, dba_datapump_sessions d
 WHERE p.addr = s.paddr
 ands.saddr=d.saddr;
```
>使用sys.dbms_system.set_ev设置10046
>在上节的查询结果里：
Data Pump Master process (DM00)的SID 是58，serial#是85.
Data Pump Worker process (DW01)的SID 是23，serial#是311.

>使用10046 跟踪活动session的语法如下
```sql
Syntax: DBMS_SYSTEM.SET_EV([SID],[SERIAL#],[EVENT],[LEVEL],'')

--在level 4跟踪Worker process进程(Bind values)：
execute sys.dbms_system.set_ev(23,311,10046,4,'');

-- stop tracing:
execute sys.dbms_system.set_ev(23,311,10046,0,'');

--在level 8 跟踪Master进程（Waits）：
execute sys.dbms_system.set_ev(143,50,10046,8,'');

-- stop tracing:
execute sys.dbms_system.set_ev(143,50,10046,0,'');
```
* 02、使用oradebug
>可以在oradebug中设置SPID，来进行trace
```sql
--在level 4跟踪Worker process进程(Bind values)：
oradebug setospid 8173
oradebug unlimit
oradebug event 10046 trace name context forever, level 4
oradebug tracefile_name

--在level 8 跟踪Master进程（Waits）：
oradebug setospid 8171
oradebug unlimit
oradebug event 10046 trace name context forever, level 8
oradebug tracefile_name

--stop tracing:
oradebug event 10046 trace name context off ITPUX
```

* 03、使用tkprof 分析trace文件
>10046 生成的trace 文件可读性并不好，所有我们可以使用tkprof工具进行格式化，方便阅读。
```sql
$ tkprof itpux_dm00_17292.trc tkprof_itpux_dm00_17292.out waits=y sort=exeela
$ tkprof itpux_dw01_17294.trc tkprof_itpux_dw01_17294.out waits=y sort=exeela
```


## 12.expdp/impdp使用总结

* 1.expdp/impdp 默认就是使用直接路径的，所以速度比较快，但是expdp/impdp 是服务端程序，影响它速度的只有磁盘io。
* 2.导出多表时，expdp/impdp用法是tables='table1','table2','table3'。
* 3.dumpfile 参数 ，可以用%u 指定多个数据文件
```sql
expdp xxx/xxx schemas=xxx directory=dump1 dumpfile=xxx_%u.dmp filesize=50g
```
>这样每个文件50g ，xxx_01.dump,xxx_02.dump 这样。
* 4.如果要把用户usera的对象导到用户userb，操作如下：
```shell
impdp system/passwd directory=expdp dumpfile=expdp.dmp remap_schema='usera':'userb' logfile=/oracle/exp.log;
```
* 5.如果导入需要更换表空间，impdp用remap_tablespace='tabspace_old':'tablespace_new'

* 6.关于数据导出时要导出哪些内容
>expdp content（all:对象＋导出数据行，data_only：只导出对象，metadata_only：只导出数据的记录）
* 7、数据泵expdp/impdp 影响速度和性能最大的就是paralle。 所以使用数据泵，要想提高速度，就要设置并行参数。如：
>&emsp;&emsp; expdp full=y directory=dump dumpfile=test_%u.dmp parallel=4
那么expdp将为parallel 创建4个文件： test_01.dmp，test_02.dmp，test_03.dmp，test_04.dmp。 每个进程一个文件。 这样的话，每个文件的大小会因进程而不同。 可以某个文件很大，某个文件却很小。 要解决这个问题，就是设置filesize 参数。 来指定每个文件的最大值。 这样当一个文件达到最大值的之后，就会创建一个新的文件。

>&emsp;&emsp;如：expdp full=y directory=dump dumpfile=test_%u.dmp parallel=4 filesize=50m
导出的dump文件和paralle有关系，那么导入也有关系。 paralle要小于dump文件数。 如果paralle 大于dump文件的个数，就会因为超过的那个进程获取不到文件，就不能对性能提高。一般parall 参数值等于cpu 的个数。而且要小于dump文件的个数。