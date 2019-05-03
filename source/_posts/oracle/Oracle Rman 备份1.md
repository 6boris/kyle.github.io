---
title: Oracle Rman 备份1
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

# Oracle Rman 的备份 （基础）


 [Oracle 11G R2 官方文档][1] 
## 1. RMAN作用与体系架构

## 2. nocatalog 余catalog 的介绍与catalog的配置

### 2.1 nocatalog介绍
>&emsp;&emsp;nocatalog方式 就是用control file作为catalog，每一次备份都要往控制文件里面写好多备份信息，控制文件里面会有越来越多的备份信息。因此，当使用rman nocatalog方式备份时，备份controlfile是非常重要的。

>&emsp;&emsp;由于nocatalog时利用controlfile存放备份信息，建议将oracle参数文件中的control_file_record_keep_time值加大（缺省为7天）, 参数在$oracle_home/dbs/initsid.ora中，该参数control_file__record_keep_time设置备份信息保存时间，到规定时间就自动清除以前的备份信息。

```sql
--查看参数
show parameter control
--修改时间为半个月
alter system set control_file_record_keep_time=14 scope=both;
--查看参数
select name,value,issys_modifiable from v$parameter where name='control_file_record_keep_time';
```

### 2.2 catalog介绍
>&emsp;&emsp;恢复目录存储的是与rman 备份有关的元数据。在某种意义上，恢复目录可以看做是保存rman备份和恢复所需的相关信息的副本。
>&emsp;&emsp;我们可以在oracle 数据库中在用户模式下创建恢复目录，这个恢复目录仅仅是一些数据包，表，索引和视图。
>&emsp;&emsp;rman中的再同步命令会使得目标数据库控制文件中的内容刷新这些表中的数据。当然，区别在于恢复目录可以包含企业中所有数据库的信息，而控制文件只包含关于它自己的数据库的信息。

### 2.3 catalog恢复目录的配置过程

* 创建时需要的表空间
```sql
create tablespace rman_tbs 
	datafile '/u01/app/oracle/oradata/orcl/rman_tbs.dbf'
	size 1G 
	autoextend off;
```

* 创建用户并授权
```sql
create user rman 
	identified by rman 
	default tablespace rman_tbs;
	
grant connect,resource,recovery_catalog_owner to rman;
```

* 创建恢复目录
```sql
rman catalog rman/rman
create catalog tablespace rman_tbs;
--注意这里tablespace不能为rman
```
* 配置rman 专用的监听
```sql
rman =
	(description =
		(address_list =
			(address = (protocol = tcp)(host = orcl)(port = 1521))
		)
		(connect_data =
			(sid = orcl)
		)
	)
```

## 3. 详解 RMAN 的使用
### 3.1 设置环境变量

>&emsp;&emsp;  修改oracle用户环境变量为如下，在linux系统中还有一个rman命令不是oracle的，如果在win环境下，可以直接使用cmd。
path=$oracle_home/bin:/sbin:$path
才可以使用rman命令

### 3.2 rman命令的帮助
```sql
Argument     Value          Description
-----------------------------------------------------------------------------
target       quoted-string  connect-string for target database
catalog      quoted-string  connect-string for recovery catalog
nocatalog    none           if specified, then no recovery catalog
cmdfile      quoted-string  name of input command file
log          quoted-string  name of output message log file
trace        quoted-string  name of output debugging message log file
append       none           if specified, log is opened in append mode
debug        optional-args  activate debugging
msgno        none           show RMAN-nnnn prefix for all messages
send         quoted-string  send a command to the media manager
pipe         string         building block for pipe names
timeout      integer        number of seconds to wait for pipe input
checksyntax  none           check the command file for syntax errors
-----------------------------------------------------------------------------
Both single and double quotes (' or ") are accepted for a quoted-string.
Quotes are not required unless the string contains embedded white-space.
```
### 3.3 rman连接信息介绍
* 连接到目标数据库
```sql
rman>connect target user/pwd@db_name
```
* 注意
>1、connect不能简写为conn
2、连接user必须具备sysdba权限
3、连接的db_name必须在tnsnames.ora中有配置，且有效(即通过sqlplus可以连接）
4、target database必须为archivelog 模式
5、如果是本地可以采用os认证，如果是远程需要使用密码文件认证。
6、rman工具版本与目标数据库必须是同一版本。

### 3.4 在rman里执行操作系统命令
```sql
rman> run{host "ls -artl";}
rman> run{host "ifconfig";}
rman> run{host "pwd";}
/home/oracle
host commandcomplete
rman> run{host "ls";}
rman> exit
```
### 3.5 在rman里执行数据库命令及sql语句
```sql
rman> shutdown immediate
rman> startup
rman> sql 'select * fromuser_tablespaces';
```
### 3.6 在rman中保存脚本或执行脚本
>在RMAN中，我们可以创建一个命令文件，里面包含rman命令，然后在RMAN的中调用这个文件。如：
Rman target usr/pwd cmdfile=backup.cmd
或者，也可以直接在RMAN 中直接运行

* 创建存储的脚本
>使用create script RMAN 命令可以在恢复目录中存储脚本。 创建每个存储的脚本时，都要为脚本指定一个名称。 可以创建执行数据库备份，恢复和维护操作的脚本。在脚本中，RMAN 允许使用comment 参数存储与存储脚本相关的注释。 注意： 必须连接到恢复目录。 如：
```sql
RMAN> create script my_backup_script
comment 'itpux'
{
	backup database plus archivelog;
}
```
* 修改存储脚本
>使用replace script 命令可以替换恢复目录中的存储脚本。
```sql
RMAN> replace script my_backup_script
comment 'bl'
{
	backup database plus archivelog delete input;
}
```
* 删除存储脚本
>使用delete script命令可以删除一个存储脚本。
```sql
RMAN> Delete script my_backup_script;
```
* 使用存储脚本
>创建一些存储过程脚本后，可以执行execute script命令来使用这些脚本。如：
```sql
run { execute script my_backup_script; }
```

* 打印存储的脚本
```sql
RMAN> Print script my_backup_script;
正在打印存储的脚本: my_backup_script
{backup database plus archivelog;}
```

>还可以使用RC_STORED_SCRIPT_LINE恢复目录视图来显示存储的脚本的内容，如：
```sql
SQL> select script_name,text from rc_stored_script_line order by script_name,line;
SCRIPT_NAME TEXT
------------------------------ -------------------------------------------------
my_backup_script {
	my_backup_script backup database plus archivelog delete input;
my_backup_script }
```
### 3.7 查看并修改单个参数
```sql
rman> show all;
using target database control file instead of recoverycatalog
rman configuration parameters are:
```
### 3.8 查看并修改单个参数
```sql
rman>show retention policy;
rman>show backup optimization;
rman>show controlfile autobackup; -----查看当前参数的值，是off。
rman>configure controlfile autobackup on; --修改为on，自动备份控制文件
rman> show controlfile autobackup; ---查看修改后的，已经修改成功。
rman> configurechannel 1 device type disk format '/dbbak/bak_%d_%m_%d_%u'; --注释：配置数据文件的备份路径.
rman> configurecontrolfile autobackup format for device type disk to '%f'; 注释：配置控制文件的备份路径
rman> show all; ------查看所有信息，看到已经更改了默认备份路径
```
### 3.8 备份文件的格式
```sql
备份文件可定义的格式符号如下
使用format参数时可使用的各种替换变量，如下（注意大小写）所示：
%a：Oracle数据库的activation ID即RESETLOG_ID。
%c：备份片段的复制数（从1开始编号，最大不超过256）。
%d：Oracle数据库名称。
%D：当前时间中的日，格式为DD。
%e：归档序号。
%f：绝对文件编号。
%F：基于"DBID+时间"确定的唯一名称，格式的形式为c-IIIIIIIIII-YYYYMMDD-QQ,其中IIIIIIIIII 为该数据库的DBID，YYYYMMDD为日期，QQ是一个1～256的序列。
%h：归档日志线程号。
%I：Oracle数据库的DBID。
%M：当前时间中的月，格式为MM。
%N：表空间名称。
%n：数据库名称，并且会在右侧用x字符进行填充，使其保持长度为8。比如数据库名JSSBOOK，则生成的名称则是JSSBOOKx。
%p：备份集中备份片段的编号，从1开始。
%s：备份集号。
%t：备份集时间戳。
%T：当前时间的年月日格式（YYYYMMDD）。
%u：是一个由备份集编号和建立时间压缩后组成的8字符名称。利用%u可以为每个备份集生成一个唯一的名称。
%U：默认是%u_%p_%c的简写形式，利用它可以为每一个备份片段（即磁盘文件）
```
### 3.9 详解整个rman全备案例
>执行rman备份必须开启数据的归档功能：
```sql
sql> shutdown immediate ---关闭数据库
sql> startup mount; ----启动数据库到mount状态
sql> alter database archivelog; ---开启归档功能
database altered.
sql> archivelog list; ---查看归档状态
database logmode archive mode ---归档已经打开
automatic archival enabled
archive destination use_db_recovery_file_dest
oldest online logsequence 8
next log sequence toarchive 10
current log sequence 10
sql>
```


## 4. 详解 RMAN的常用命令及日常维护
### 4.1 rman所备份的数据库信息查看
* 01. list incarnation
>概述可用的备份
>b 表示backup
a 表示archivelog、 f 表示full backup、 0,1,2 表示incremental level备份
a 表示可用avaliable、 x 表示expired

>这个命令可以派生出很多类似命令，例如
>list backup of database summary
list backup of archivelog all summary
list backup of tablespace users summary;
list backup of datafile n,n,n summary
* 02. list backup summary;
> 列出过期的备份文件

* 03. list backup by file;
> 按照文件类型分别列出
> 分别为：数据文件列表、归档日志列表、控制文件列表、spfile列表

* 04. list backup
> 这个命令列出已有备份集的详细信息。

* 05. list expired backup;
> 列出过期的备份文件

* 06. list copy;
>列出copy文件
```sql
list copy of database;
list copy of controlfile;
list copy of tablespace users;
list copy of datafile n,n,n;
list copy of archivelog all;
list copy of archivelog from scn 10000;
list copy of archivelog until sequence 12;
```
* 07. list backup of spfile;
>服务器参数文件

* 08. list backup of controlfile;
> 控制文件

* 09. list backup of datafle n,n,n,n;
> 数据文件

* 10. list backup of tablespace tablespace_name;
> 表空间对应的backup

* 11. 归档日志
```sql
list backup of archivelog {all, from, high, like, logseq, low, scn, sequence, time, until};
list backup of archivelog all;
list backup of archivelog until time 'sysdate-1';
list backup of archivelog from sequence 10;
list backup of archivelog until sequence 10;
list backup of archivelog from scn 10000;
list backup of archivelog until scn 200000;
list archivelog from scn 1000;
list archivelog until scn 2000;
list archivelog from sequence 10;
list archivelog until sequence 12;
```

### 4.2 report常用命令
>report用于判断数据库当前可恢复状态、以及数据库已有备份的信息。

* 01.report schema
> 报告数据库模式

* 02.report obsolete;
> 报告已丢弃的备份集(配置了保留策略)。

* 03.report unrecoverable;
> 报告当前数据库中不可恢复的数据文件(即没有这个数据文件的备份、或者该数据文件的备份已经过期)

* 04.report need backup;
> 报告需要备份的数据文件(根据条件不同)
```sql
report need backup days=3;
--最近三天没有备份的数据文件(如果出问题的话，这些数据文件将需要最近3天的归档日志才能恢复)
report need backup incremental=3;
--需要多少个增量备份文件才能恢复的数据文件。(如果出问题，这些数据文件将需要3个增量备份才能恢复)
report need backup redundancy=3;
--报告出冗余次数小于3的数据文件
--例如数据文件中包含2个数据文件system01.dbf和users01.dbf.
--在3次或都3次以上备份中都包含system01.dbf这个数据文件，而users01.dbf则小于3次
--那么，报告出来的数据文件就是users01.dbf
--即，报告出数据库中冗余次数小于 n 的数据文件
report need backup recovery window of 2 days;
--报告出恢复需要2天归档日志的数据文件
```

### 4.3 backup常用命令
#### 4.3.1 基本使用
* 01.设置备份标记

* 01.设置备份集大小(一次备份的所有结果为一个备份集，要注意备份集大小)

* 01.设置备份片大小(磁带或文件系统限制)

* 01.备份集的保存策略

* 01.重写configure exclude命令

* 01.检查数据库错误

* 01.跳过脱机，不可存取或只读文件


* 01.强制备份

* 01.基于上次备份时间备份数据文件

* 01.备份操作期间检查逻辑错误

* 01.生成备份副本

* 01.备份控制文件和参数文件

* 01.跳过脱机的，不可读取的或者只读的数据文件

* 01.数据库备份

* 01.表空间备份

* 01.数据文件备份

* 01.归档的重做日志备份

* 01.数据库，表空间和数据文件的映像副本

* 01.控制文件副本

* 01.Archivelog 映像副本

* 01.差异备份与增量备份

#### 4.3.1 差异备份与增量备份
* 块更改跟踪文件
>&emsp;&emsp;默认情况下，当执行增量备份时，发生任何更改的所有数据文件都将备份。 这可能使增量备份花费更长的时间，并且会增加增量备份的大小。 10g中RMAN 提供了只备份更改过的数据块的功能。 这就可以加快增量数据库备份的速度并减少其大小。 执行alter database enable block change tracking 命令可以启用块更改跟踪。
&emsp;&emsp;如果使用Oracle管理文件（OMF），Oracle 将会创建块更改跟踪文件。 如果没有使用OMF，则必须定义块更改跟踪文件的位置和名称。 如：
Alter database enable block change tracking using file '/oracle/backup/block.fil';
&emsp;&emsp;如果跟踪文件已经存在，可以使用reuse参数：
Alter database enable block change tracking using file '/oracle/backup/block.fil' reuse;
&emsp;&emsp;使用alter database block change tracking 命令可以禁用块更改跟踪。 块更改跟踪文件的大小通常预先分片且与数据库大小和重做日志线程的数量有关。 块更改跟踪文件的大小一般是数据库大小的1/30000。 块更改跟踪文件可能会以10MB为增量增长。 块更改跟踪文件的最小尺寸是每个数据文件320k，如果有许多数据文件，则块更改跟踪文件就会较大。 Oracle 会在块更改跟踪文件中存储足够的信息，从而允许最多8天的增量备份。 显而易见，如果增量备份超过8天，则将不使用块跟踪更改跟踪文件，并且无法利用块跟踪文件的有点。
&emsp;&emsp;可以通过检查v$block_change_tracking 视图来确定是否启用了块更改跟踪。 Status 指示了是否启用了块更改跟踪，filename 包含块更改跟踪文件的文件名。可以通过alter database rename file 命令来转移块更改跟踪文件。
```sql
SQL> select status,filename from v$block_change_tracking;
STATUS FILENAME
---------- ------------------------------------------------------
ENABLED /oracle/BACKUP/BLOCK.FIL
```
* 基本备份
>&emsp;&emsp;执行增量备份操作时，首先需要的是增量基本备份（incremental base backup）,以后所有的增量备份都基于这个基本备份。 每次执行数据库备份操作时，都可以通过backup 命令的incremental 参数来为备份指定一个增量级别标识符。 基本备份的增量级别为0，并且必须有基本备份才能够执行其他类型的增量备份操作。 如果没有生成基本备份就尝试执行增量备份操作，RMAN会自动执行基本备份操作。 示例：
```sql
Backup incremental level=0 database;
```
* 差异备份
>&emsp;&emsp;差异备份是RMAN生成的增量备份的默认类型，对于差异备份来说，RMAN会备份自上次同级或者低级差异增量备份以来所发生变化的数据块
```sql
backup incremental level=1 database;
```
* 累积备份
>&emsp;&emsp;累积备份可以使备份集备份前面所有级别的备份以及此次要备份的所有发生变化的数据块。 累积备份是一个可选的备份方法，并要求在backup 命令中使用cumulative 关键字。
```sql
backup incremental level =2 cumulative database;
```
* 增量备份选项
>&emsp;&emsp;Oracle 不仅允许执行数据库的增量备份，还允许执行表空间，数据文件以及数据库文件副本的增量备份操作。 控制文件，归档重做日志以及备份集都不能生成增量备份。 此外，还可以在执行增量备份操作时同时备份归档的重做日志。
```sql
Backup incremental level=0 tablespace users;
Backup incremental level=1 tablespace users;
Backup incremental level=0 datafile 4;
Backup incremental level=1 datafile 4;
Backup incremental level=1 database plus archivelog;
```
* 增量备份更新备份
>RMAN 提供了增量备份更新备份。 这种备份避免了采用数据文件的完整映像副本进行备份的开销，并且具有与映像副本相同的恢复特性。 从某种意义上来说，这种备份类似与使用映像副本的增量备份。
```sql
Run{
Recover copy of database with tag 'Orcl';
Backup incremental level 1 for recover of copy with tag 'Orcl' database;
}
```
>&emsp;&emsp;示例中的recover of copy database 命令并没有真正的恢复数据库，但它使RMAN将任何增量备份应用于与列出标记（Orcl）关联的数据文件副本。 第一次运行该命令时，它将没有任何效果，因为它没有任何可用的增量备份或数据文件副本。 这并不是很严重的问题，并且RMAN 将只显示一条警告消息。 第二次运行该命令时也没有任何效果，因为没有任何增量备份可用。
&emsp;&emsp;执行recover 命令后，就会产生一个增量备份，这个备份第一次运行时，它会创建一个基本备份（如果没有的话）。这实际上增量为1的备份。 第二次执行这个run代码块时，将通过backup 命令执行第一个增量备份。
&emsp;&emsp;一旦该命令运行了2次，第三次执行和后面的执行就能够将前面的增量备份应用与数据文件副本。 注意，recover 和backup命令中将标记赋予相同的名称非常重要。

### 4.4 configure常用命令

* 01. 显示当前的配置信息
```sql
rman> show all;
rman 配置参数为:
configure retention policy to redundancy 1; # default
configure backup optimization off; # default
configure default device type to disk; # default
configure controlfile autobackup off; # default
configure controlfile autobackup format for device type disk to '%f'; # default
configure device type disk parallelism 1 backup type to backupset; # default
configure datafile backup copies for device type disk to 1; # default
configure archivelog backup copies for device type disk to 1; # default
configure maxsetsize to unlimited; # default
configure encryption for database off; # default
configure encryption algorithm 'aes128'; # default
configure archivelog deletion policy to none; # default
configure snapshot controlfile name to '/oracle/product/11.2.0/db_1/database/sncfitpuxdb.ora'; # default
configure retention policy to redundancy 1; # default
注释：配置redundancy配置需要保留几份备份文件，默认是1。也可以改变为其它非零的正整数值。
configure backup optimization off; # default
注释：设备备份优化打开，如果表空间是只读状态，那么在做备份的时候只是第一次会备份，以后不备份。有两个选项off关闭和on打开
configure default device type to disk; # default
注释：配置默认备份设备可以是sbt（磁带），disk（硬盘）；默认是硬盘
configure controlfile autobackup off;# default
注释：配置在备份的时候是否将控制文件一并备份，两个选项,默认是off不备份，也可以是on备份、。
configure controlfile autobackupformat for device type disk to '%f'; # default
注释：配置control file自动备份的路径和文件格式
configure device type disk parallelism1 backup type to backupset; # default
注释：配置磁盘备份的类型，默认是备份集方式（backupset）或镜像拷贝也叫文件拷贝(copy)。
镜像拷贝值适用于磁盘备份，磁带只支持备份集。一般用备份集更多，其效率会更高
configure datafile backupcopies for device type disk to 1; # default
注释：配置生成备份集的个数，默认是1；备份集内会包括（数据文件，控制文件，参数文件）
configure archivelog backupcopies for device type disk to 1; # default
注释：配置归档日志默认的备份路径
configure maxsetsize to unlimited; # default
注释：配置单个备份集大小，缺省是不限制，我们可以配置成1g/100m/1024k or other
configure encryption for database off; # default
注释：配置备份数据加密，是10gr2推出来的新功能，设置为on后。
可以set encryption on identifyed by youpassword only;加密备份，还原的时候需要提供密码。
configureencryption algorithm 'aes128'; # default
注释：指定备份数据加密的类型。
configure archivelogdeletion policy to none; # default
注释：配置归档日志在备份后自动删除
configure snapshot controlfile name to '/oracle/product/10.2.0/db_1/dbs/snapcf_itpuxdb.f'; # default
注释：配置控制文件快照，可以有效的提高控制文件的恢复性。
查询rman设置中非默认值:
sql> select name,value from v$rman_configuration;
```
* 01. 保存策略 (retention policy)
```sql
configure retention policy to recovery window of 7 days;
configure retention policy to redundancy 5;
configure retention policy clear;
configure retention policy to none;
第一种recover window是保持所有足够的备份，可以将数据库系统恢复到最近七天内的任意时刻。任何超过最近七天的数据库备份将被标记为obsolete。
第二种redundancy 是为了保持可以恢复的最新的5份数据库备份，任何超过最新5份的备份都将被标记为redundancy。它的默认值是1份。
第三、四：none 可以把使备份保持策略失效，clear 将恢复默认的保持策略
一般最安全的方法是采用第二种保持策略。
```
* 01. 备份优化 backup optimization
```sql
configure backup optimization on;
configure backup optimization off;
configure backup optimization clear;
默认值为关闭，如果打开，rman将对备份的数据文件及归档等文件进行一种优化的算法。
```
* 01. 默认设备 default device type
```sql
configure default device type to disk;
configure default device type to stb;
configure default device type clear;
是指定所有i/o操作的设备类型是硬盘或者磁带，默认值是硬盘
磁带的设置是configure default device type to sbt;
```
* 01. 控制文件 controlfile
```sql
configure controlfile autobackup on;
configure controlfile autobackup format for device type disk to '/oracle/backup/conf_%F';
configure controlfile autobackup clear;
configrue controlfile autobackup format for device type disk clear;
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/oracle/backup/scontrofile.snp';
--是配置控制文件的快照文件的存放路径和文件名，这个快照文件是在备份期间产生的，用于控制文件的读一致性。
configure snapshot controlfile name clear;
强制数据库在备份文件或者执行改变数据库结构的命令之后将控制文件自动备份，默认值为关闭。这样可以避免控制文件和catalog丢失后，控制文件仍然可以恢复。
```
* 01. 并行数(通道数) device type disk|stb pallelism n;
```sql
configure device type disk|stb parallelism 2;
configure device type disk|stb clear; --用于清除上面的信道配置
configure channel device type disk format '/oracle/backup/rman_%u';
configure channel device type disk maxpiecesize 100m;
configure channel device type disk rate 1200k;
configure channel 1 device type disk format '/oracle/backup/rman_%u';
configure channel 2 device type disk format '/oracle/backup/rman_%u';
configure channel 1 device type disk maxpiecesize 100m;
配置数据库设备类型的并行度。
```
* 01. 生成备份副本 datafile|archivelog backup copies
```sql
configure datafile backup copies for device type disk|stb to 3;
configure archivelog backup copies for device type disk|stb to 3;
--是设置数据库的归档日志的存放设备类型
configure datafile|archivelog backup copies for device type disk|stb clear
backup device type disk database format '/oracle/backup1/%u', '/oracle/backup2/%u', '/oracle/backup3/%u';
是配置数据库的每次备份的copy数量，oracle的每一次备份都可以有多份完全相同的拷贝。
```
* 01. 排除选项 exclude
```sql
configure exclude for tablespace USERS;
configure exclude for tablespace USERS clear;
```
* 01. 备份集大小 maxsetsize
```sql
configure maxsetsize to 1g|1000m|1000000k|unlimited;
configure maxsetsize clear;
```

### 4.5 set 命令
>&emsp;&emsp;使用set 命令可以定义只应用于当前RMAN会话的设置。 set 命令的设置不是永久的，根据实际需求，可以采用两种方式来使用set 命令。
* 在run 代码块外，我们可是执行下面的操作：
```sql
（1）使用set echo 命令在消息日志中显示RMAN 命令。
（2）使用set dbid 命令指定一个数据库的数据库标识符（database identifier： dbid）。
```

* 某些set 命令只能在run代码块的限定范围内使用，常见的有：
>（1）set newname 命令：用于执行表空间时间点恢复（TSPITR）或者数据库复制操作。 该命令允许指定新的数据库数据文件名。 将数据库移动到新的系统中并且文件系统名不同时，我们可以使用这个命令。使用set newname 命令时还需要使用switch 命令。
（2）set maxcorrupt for datafile: 使用该命令可以定义RMAN操作失败前锁允许的数据块讹误的最大数据。
（3）set archivelog destination: 使用该命令可以修改存储归档的重做日志的archive_log_dest_1 目标。
（4）set 命令和until 子句： 使用set命令和set 命令的until 子句可以定义数据库时间点恢复操作锁使用的具体时间点，SCN 或日志序列号。
（5）set backup copies命令： 使用该命令可以定义为备份集中的每个备份片应当创建的副本数。
（6）set command id: 使用该命令可以关联给定的服务器会话和给定的通道。
（7）set controlfile autoback format for device type: 使用该命令可以修改用于控制文件自动备份操作的默认格式。

>&emsp;&emsp;例如： 要执行一个为每个备份片创建两个副本的被操作，并且允许数据文件的最大讹误数为10. 脚本如下：

```sql
run{
	set maxcorrupt for datafile 3 to 5;
	set backup copies=2;
	backup database;
}
set until time "to_date('2016-6-28 17:04:00','yyyy-mm-dd hh24:mi:ss')";
```
### 4.5 crosscheck命令

* 交叉效验RMAN 备份
>&emsp;&emsp;在RMAN目录和物理备份目的地不同步的情况下，我们可以使用crosscheck命令来效验控制文件或恢复目录中的RMAN信息是否与备份介质上的实际物理备份集片相同。
>&emsp;&emsp;使用crosscheck 命令时，我们关心每个备份集或者副本的状态。 如果使用控制文件，用于备份集片的v$backup_set 视图和用于副本的v$databfile_copy 视图中的status列列出了每个备份集或副本的状态码；如果使用恢复目录，则在备份集片的RC_BACKUP_set和副本的RC_DATAFILE_COPY中列出了每个备份集或副本的状态码。 在不同的备份状态码中，我们关心以下两种状态：
>&emsp;&emsp;（1）A（Available:可用）：RMAN 认定该项存在于备份介质上
&emsp;&emsp;（2）X（Expired:不可用）：这个备份集片或副本上存储的RMAN目录（即控制文件或恢复目录）中，但是并没有物理存在于备份介质上。
>&emsp;&emsp;使用crosscheck 命令的目的是将RMAN目录的状态设置为AVAILABLE或者EXPIRED。 执行crosscheck时，RMAN检查目录中列出的每个备份集或副本并且判断他们是否存在与备份介质上。 如果备份集或副本不存在与备份介质上，它就会被标记为expired, 并且不能用于任何还原操作；如果备份集或副本存在与备份介质上，它就会维持available状态。 如果以前被标记为expired 的备份集或副本再次存在于备份介质上，crosscheck 命令就会将它标记回available。
```sql
crosscheck backup;
```
> &emsp;&emsp;可以交叉效验数据文件备份，表空间备份，控制文件备份以及服务器参数文件备份。此外，可以通过识别与备份相关联的标记来选择要交叉效验和特定的备份。 基于使用的设备或者基于一个时间周期，我们甚至可以交叉效验所有的备份。 如：
```sql
crosscheck backup of datafile 1;
crosscheck backup of tablespace users;
crosscheck backup of controlfile;
crosscheck backup of spfile;
crosscheck backup tag='TEST';
crosscheck backup completed after 'sysdate-2';
crosscheck backup completed between 'sysdate-5' and 'sysdate-2';
crosscheck backup device type disk;
```
* 交叉验证归档日志示例：
```sql
RMAN> crosscheck archivelog all;
```
>&emsp;&emsp;我们可以基于一个号码或标准（包括时间，具体的或指定范围的SCN或日志序列号）来交叉效验归档的重做日志备份，甚至还可以使用like参数与通配符来交叉效验特定的归档日志备份。 如：
```sql
crosscheck archivelog like 'ARC001.log';
crosscheck archivelog '/oracle/arch/itpuxdb/archivelog/2016_08_02/o1_mf_1_22_cszw2d4g_.arc';
crosscheck archivelog like '%ARC00012.LOG';
crosscheck archivelog from time "to_date('2016-08-02','yyyy-mm-dd')";
crosscheck archivelog until time "to_date('2016-08-02','yyyy-mm-dd')";
crosscheck archivelog from sequence 12;
crosscheck archivelog until sequence 522;
--使用crosscheck copy命令还可以交叉效验副本。 包括数据文件副本，控制文件副本，归档重做日志副本以及磁盘上的归档的重做日志。 如：
crosscheck copy of datafile 5;
crosscheck datafilecopy '/oracle/oradata/itpuxdb/rman_tbs.dbf;
```

### 4.6 RMAN 备份的验证validate
>&emsp;&emsp;RMAN 提供的validate命令允许查看给定的备份集和进行验证以确保这个备份集能够被还原。注意，validate 命令必须要获得主键ID。 这个可以用list backup summary命令获取
```sql
RMAN> list backup summary;
RMAN> validate backupset 206;
RMAN> validate backupset 206 check logical;
```
### 4.7 change 命令
>&emsp;&emsp;change 命令允许用户修改备份的状态。我们可能会遇到备份介质设备在某个时间爱你段中无效的情况（如突然断电）。这时，我们就可以使用change 命令来指示这个设备上的备份是不可用的。
>&emsp;&emsp;解决硬件故障和修复磁盘后，额可以再次执行change 命令，将备份改为可用的状态。也可以将备份修改为不可用的状态。在还原和恢复操作期间，不会考虑哪些不可用的备份，但在执行delete expired命令期间这些备份记录不会被删除。 相关示例：
```sql
change backup of database tag='TEST' unavailable;
change backup of database like '%TEST%' unavailable;
change backupset 206 unavailable;
change backupset 206 available;
change archivelog '/archivelog/arc0001.log' unavailable;
```
>&emsp;&emsp;可以使用change命令修改归档的重做日志备份的状态。如：将已经备份了指定次数的所有归档的重做日志备份修改为不可用的状态，也可以修改特定设备上的所有备份的状态。
```sql
change archivelog all backed up 5 times to device type disk unavailable;
change backup of database device type disk unavailable;
```
>&emsp;&emsp;可是使用带有delete 参数的change 命令删除备份集，即在备份介质上的物理删除，并且从控制文件和恢复目录中删除。 前提是要知道备份集关键字，可以使用list backup 或 list copy 命令查看。
```sql
RMAN> list backup;
```
>删除备份集1：
```sql
RMAN> change backupset 206 delete;
```
* 使用通道 ORA_DISK_1
```sql
备份片段列表
BP 关键字 BS 关键字 Pc# Cp# 状态 设备类型段名称
------- ------- --- --- ----------- ----------- ----------
1 1 1 1 AVAILABLE DISK /oracle/BACKUP/ITPUX_01LI7BSC_1_1.BAK
是否确定要删除以上对象 (输入 YES 或 NO)? yes
已删除备份片段
备份片段句柄=/oracle/BACKUP/ITPUX_01LI7BSC_1_1.BAK RECID=1 STAMP=723758988
1 对象已删除
在上面的这个示例中，我们删除了备份集和它关联的备份片。 也可以删除一些备份片。 通过 list backup命令我们可以看到备份片的名称，比如：段名:/oracle/BACKUP/ITPUX_01LI7BSC_1_1.BAK。
我们也可以查看备份片的号码，用catalog 用户连接数据库后，查看rc_backup_piece 表，SQL如下：
conn rman/rman
SQL> select bs_key,bp_key,piece#,handle from rc_backup_piece;
BS_KEY BP_KEY PIECE# HANDLE
---------- ---------- ---------- -----------------------------------------------
52 55 1 /BACKUP/ORCL_02LI47UA_1_1
53 56 1 /BACKUP/ORCL_03LI47UF_1_1
75 82 1 /BACKUP/ORCL_04LI4816_1_1
删除备份片，我们可以用BP_KEY，也可以直接用段名（handle）,如：
RMAN> change backuppiece '/oracle/BACKUP/ITPUX_02LI7BSK_1_1.BAK' delete;
使用通道 ORA_DISK_1
备份片段列表
BP 关键字 BS 关键字 Pc# Cp# 状态 设备类型段名称
------- ------- --- --- ----------- ----------- ----------
2 2 1 1 AVAILABLE DISK /oracle/BACKUP/ITPUX_02LI7BSK_1_1.BAK
是否确定要删除以上对象 (输入 YES 或 NO)? yes
已删除备份片段
备份片段句柄=/oracle/BACKUP/ITPUX_02LI7BSK_1_1.BAK RECID=2
STAMP=723758997
1 对象已删除
使用备份集片，如：
change backuppiece 622 delete;
change archivelog until logseq=20 delete;
最后，可以使用change backuppiece uncataog命令从目录中删除备份集片。 如果删除最后剩余的备份集片，那么它也将删除备份集记录。如：
change backuppiece '/oracle/BACKUP/ITPUX_02LI7BSK_1_1.BAK' uncatalog;
```
### 4.8 delete 命令
>&emsp;&emsp;备份集不是永远存在的。我们可以使用保存策略标记备份有效性和生存期的结束。但是，备份策略的实施不会从RMAN目录中删除备份，而只是将这些备份标记为丢弃状态。
>&emsp;&emsp;delete 命令对备份和副本的影响很大。通过delete命令，可以删除基于保存标准被标记为丢弃的任何备份，还可以将恢复目录或控制文件中的备份从expired状态变为

* deleted状态
```sql
delete expired;
delete obsolete;
```
>&emsp;&emsp;执行delete命令时，RMAN会请求用户确认这个删除命令，如果确认了这个删除命令，RMAN 就会完成delete操作。
&emsp;&emsp;注意： 如果一个备份被标记为deleted 状态，就不能恢复这个备份。 如果该备份物理可用，我们仍然可以使用dbms_backup_restore过程来恢复这个备份。

### 4.9 restore命令
> &emsp;&emsp;主要功能是从RMAN备份中还原文件，为恢复做准备。
使用restore 命令时，该命令会在没有认识提示的情况下重写已经存在的任何文件，除非使用set newname命令。
```sql
set newname for datafile '/oracle/app/oradata/itpuxdb/itpux01.dbf' to 'E:/app/Administrator/oradata/orcl';
restore database;

restore controlfile from autobackup;
restore controlfile to '/backup/' from autobackup;
restore controlfile to '/backup';
restore controlfile from autobackup until time "to_date('2016-6-27 13:25:00','yyyy-mm-dd hh24:mi:ss')";
restore controlfile from autobackup maxseq 200 maxdays 100;
restore spfile to pfile '/oracle/backup/spfile.restore';
restore spfile to '/oracle/backup/spfile.restore' from autobackup;
restore spfile from autobackup;
restore spfile from '/oracle/backup/c-1247395743-20160627-00';
restore tablespace tablespace_name ;
restore datafile 5;
restore datafile '/oracle/app/oradata/itpuxdb/USERS01.DBF';
restore database until time "to_date('2016-6-28 17:04:00','yyyy-mm-dd hh24:mi:ss')";
restore database until SCN 1000; --注意： 该示例可以将数据库还原到SCN 1000，但是不会包含SCN.
restore database until sequence 100 thread 1;

restore archivelog all;
restore archivelog from logseq=20 thread=1;
restore archivelog from logseq=20 until logseq=30 thread=1;

run
{
set archivelog destination to "d:/arch";
restore archivelog all;
}

restore database check readonly;
restore (datafile 5) from datafilecopy
```
### 4.10 recover命令
>&emsp;&emsp;recover 命令用于恢复数据库。该命令可以执行数据库的完全恢复或者时间点恢复。 Recover 命令确定需要哪些归档的重做日志，并且析取和应用他们。 一旦完成重做的应用，我们就只需要使用alter database open命令打开数据库即可。
```sql
recover database
recover database noredo; -- 如果联机日志存在，可以用recover database代替
recover tablespace tablespace_name ;
recover datafile 3;
recover datafile '/oracle/app/oradata/itpuxdb/USERS01.DBF';
set until time "to_date('2016-07-05 14:02:00','yyyy-mm-dd hh24:mi:ss')";
restore database;
recover database;
alter database open resetlogs;
recover database until time "to_date('2016-07-05 14:02:00','yyyy-mm-dd hh24:mi:ss')";
recover database until SCN 1000;
recover database until sequence 100 thread 1;
```
### 4.10 Switch 命令
>Switch 命令可以修改数据库控制文件中数据文件的位置，以反映Oracle数据库文件新的位置。
```sql
Switch datafile all; -- 修改控制文件中数据文件位置
```
### 4.10 blockrecover
>一般出现数据块错误时，都会有错误消息：
ORA-01578: ORACLE data block corrupted (file #18,block #88)
如果没有BMR时，我们必须从一个备份中恢复这个数据文件，在恢复过程中，用户不能使用该数据块文件中的所有数据。
用BMR恢复就很简单，只需要执行blockrecover命令即可：
```sql
blockrecover datafile 1 block 88;
```
>如果有必要，可以同时恢复多个数据文件的多个数据块。如：
```sql
blockrecover datafile 18 block 16,17,88,108;
blcokrecover datafile 18 block 88 datafile 19 blcok 188;
```
>&emsp;&emsp;查询v$database_block_corruption视图可以查看讹误数据块的详细信息。 如下所示，使用具有corruption list restore 参数的blockrecover命令可以方便地修正v$database_block_corruption 视图中的讹误数据块。
>&emsp;&emsp;blockrecover corruption list restore until time 'SYSDATE-5';
这条命令将还原讹误列表中最近5天的所有讹误数据块。 在上面的命令中，还可以使用until time 和 until sequence.


### 4.10convert命令
>&emsp;&emsp; Oracle 10g R2以后支持手工跨平台移动数据库，即使这些平台具有不同的尾数格式（endian format）。 尾数格式与字节排序有关，它有两种不同的格式，即大尾数和小尾数。 如果在不同尾数字节格式的平台之间移动数据库，就需要手工操作，并且使用RMAN的convert datafile 或者 convert tablespace命令来将传送的数据文件转换为正确的尾数格式。
```sql
convert tablespace ITPUX to platform='AIX-Based Systems (64-bit)'
db_file_name_convert='/oracle/app/oradata/itpux','/oracle/itpux';
convert datafile '/oracle/app/oradata/itpux/itpux01.DBF' from platform
'AIX-Based Systems (64-bit)' db_file_name_convert '/oracle/app/oradata/itpux','/oracle/app/itpux';
```






  [1]: https://docs.oracle.com/cd/E11882_01/backup.112/e10643/preface.htm#RCMRF89959