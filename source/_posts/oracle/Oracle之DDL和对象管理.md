---
title: Oracle之DDL和对象管理
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
  - DDL
---



# SQL语言之DDL和对象管理

[Oracle 11G R2 官方文档][1]

## 1 介绍用户模拟下可以访问的模式对象
```sql
  SELECT object_type, COUNT (*)
    FROM user_objects
GROUP BY object_type;

  SELECT object_type, COUNT (*)
    FROM all_objects
GROUP BY object_type;

SELECT DISTINCT owner
  FROM all_objects
```

## 2 研究模式中的数据类型
```sql
-- 1.DESC
-- 2.SQL
SELECT column_name, data_type
  FROM user_tab_columns
 WHERE table_name = 'EMPLOYEES';

SELECT DISTINCT data_type
  FROM user_tab_columns
 WHERE table_name = 'EMPLOYEES';
```
## 3 表的创建与管理
### 3.1 表的创建
* 语法

```sql
Create table [schema,] table_name(
column_name data_type [default express] [constraint]
[,column_name data_type [default express] [constraint]]
[,column_name data_type [default express] [constraint]]
);


--1.表的创建与结构变更
--01.工具
--A：创建表
--B：写入数据
--C：更改结构
```
* SQL 语法

```sql
--创建sql02,sql03,sql04
 --创建sql02,sql03,sql04

CREATE TABLE SQL02
(
    sql02_id        NUMBER (8) NOT NULL,
    sql02_id2       NUMBER (8) NOT NULL,
    sql02_date      DATE NOT NULL,
    sql02_status    VARCHAR2 (8) NOT NULL,
    sql02_number NUMBER (10, 2)
);

CREATE TABLE SQL03
(
    sql03_id1    NUMBER (8) NOT NULL,
    sql03_id2    NUMBER (8) NOT NULL,
    sql03_id3    NUMBER (8) NOT NULL
);

ALTER TABLE sql03
    MODIFY (sql03_id3 VARCHAR2 (10));

CREATE TABLE SQL04
(
    sql04_id        NUMBER (8) NOT NULL,
    sql04_desc      VARCHAR2 (20) NOT NULL,
    sql04_status    VARCHAR2 (8) NOT NULL,
    sql04_price     NUMBER (10, 2) NOT NULL,
    sql04_date      DATE NOT NULL,
    sql04_count     NUMBER (8) NOT NULL
);

```
### 3.2 表的重命名
```sql
RENAME sql044 TO sql04;

ALTER TABLE SQL04
    MODIFY sql04_count NUMBER (18);

ALTER TABLE SQL04
    RENAME COLUMN sql04_count TO SQL04_COUNT2;

ALTER TABLE SQL04
    RENAME COLUMN sql04_count02 TO SQL04_COUNT;

SELECT * FROM tab 
SELECT * FROM user_tables
```
### 3.3 表的复制
```sql
CREATE TABLE itpux05
AS
    SELECT * FROM sql04;

SELECT * FROM sql01;

CREATE TABLE itpux06
AS
    SELECT *
      FROM sql01
     WHERE sql01_id = 1;

SELECT * FROM itpux06
```
### 3.4 表的截断
```sql
--100g delete 2h,truncate 2 分钟。
TRUNCATE TABLE itpux07;
SELECT * FROM itpux07
```
### 3.4 表的删除
```sql
DROP TABLE itpux06;
```
### 3.4 .删除表的定义
```sql
DROP TABLE table_name CASCADE CONSTRAINTS;              --相关的视图，约束，等相关所有的关系对象。
DROP TABLE table_name PURGE;                            --释放资源，不经过回收站。
DROP TABLE itpux05 CASCADE CONSTRAINTS;
DROP TABLE itpux05 PURGE;
```
## 4 簇的概念与介绍
>簇(cluster)其实就是一组表，由一组共享相同数据块的多个表组成，将经常一起使用的表
组合在一起成簇可以提高处理效率；在一个簇中的表就叫做簇表。

* 不宜用聚簇表的情况

```sql
1)如果预料到聚簇中的表会大量修改,聚簇表会对DML 的性能产生负面影响
2)非常不适合对单表的全表扫描,因为只能引起对其它表的全表扫描
3)频繁对表进行TRUNCATE 和加载,因为聚簇中的表是不能TRUNCATE 的，只能
TRUNCATE 簇
4)如果表只是偶尔被连接或者它们的公共列经常被修改，则不要聚簇表
5)如果经常从所有有相同聚簇键值的表查询出的结果数据超过一个或两个Oracle 块，
则不要聚簇表
6)如果空间不够，并且不能为将要插入的新记录分配额外的空间，那么不要使用聚簇
```
## 5 临时表的创建与使用
### 5.1 会话级临时表
```sql
--会话结束自动清空数据
CREATE GLOBAL TEMPORARY TABLE table_name
(
    col1 type1,
    col2 type2...
)
on commit preserve rows;
```

### 5.2 事务级临时表
```sql
--事务结束（commit,rollback） ，清除数据。
CREATE GLOBAL TEMPORARY TABLE table_name
(
    col1 type1,
    col2 type2...
)
on commit delete rows;
```

## 6 索引的创建与管理
## 6.1 索引分类
* 逻辑分类
```sql
1.单列或多列
2.唯一索引和非唯一索引
3.函数索引
Doman
```

* 物理分类
```sql
物理分类:
B-TREE
Bitmap

B*Tree 索引
B*Tree 索引是最常见的索引结构，默认建立的索引就是这种类型的索引。

DML 语句：
Create index indexname on tablename(columnname[columnname...])
Bitmap 索引
位图索引主要用于决策支持系统或静态数据，不支持行级锁定
```
## 6.2 索引的日常管理
* 基本语法
```sql
CREATE [UNIQUE] | [BITMAP] INDEX index_name --unique 表示唯一索引
ON table_name([column1 [ASC|DESC],column2 --bitmap，创建位图索引
[ASC|DESC],…] | [express])
[TABLESPACE tablespace_name]
[PCTFREE n1] --指定索引在数据块中空闲空间
[STORAGE (INITIAL n2)]
[NOLOGGING] --表示创建和重建索引时允许
对表做DML 操作，默认情况下不应该使用
[NOLINE]
[NOSORT]; --表示创建索引时不进行排序，
默认不适用，如果数据已经是按照该索引顺序排列的可以使用
```

* 1.添加/创建索引
```sql
--工具

CREATE INDEX cust_name_i1
    ON SQL01 (sql01_name);       --单列索引

CREATE INDEX cust_name_i1
    ON SQL01 (sql01_name)
    TABLESPACE ITPUX;            --单列
索引指定表空间
--命令

CREATE INDEX cust_name_i2
    ON sql01 (sql01_name, sql01_status);    --组合索引

CREATE BITMAP INDEX sql01_level_i3
    ON sql01 (sql01_level);      --位图索引
```

* 2.修改索引
```sql
--重命名索引

ALTER INDEX cust_name_i1
    RENAME TO idx1_sql01_name;

--合并索引

ALTER INDEX idx1_sql01_name
    COALESCE;
```

* 3.重建索引
```sql
--重命名索引

--删除原来的索引，重新建立索引。
DROP INDEX idx1_sql01_name;

CREATE INDEX idx1_sql01_name
    ON SQL01 (sql01_name);

--直接rebuild

ALTER INDEX idx1_sql01_name
    REBUILD;                                    --会锁表。

ALTER INDEX idx1_sql01_name
    REBUILD ONLINE;
```

* 4.删除索引
```sql
DROP INDEX idx1_sql01_name;
```

* 5.查看索引
```sql
--select * from user_indexes;

SELECT *
  FROM dba_indexes
 WHERE owner = 'ITPUX';

SELECT *
  FROM dba_indexes
 WHERE table_name = 'SQL01';

--select * from all_indexes;
--select * from user_ind_columns;

SELECT *
  FROM dba_ind_columns
 WHERE index_owner = 'ITPUX' AND table_name = 'SQL01';

--select * from all_ind_columns;

SELECT index_name,
       column_name,
       index_type,
       uniqueness,
       tablespace_name
  FROM dba_indexes NATURAL JOIN dba_ind_columns
 WHERE table_name = 'SQL01';
```
## 7 约束的创建与管理
### 7.1  5种约束

* 1.not null (非空约束)
```sql
--1.1 创建表的时候

CREATE TABLE itpux01
(
    id    NUMBER NOT NULL PRIMARY KEY,
    name VARCHAR2 (20)
);

--1.2 在已经创建的表加添加

CREATE TABLE itpux02
(
    id NUMBER,
    name VARCHAR2 (20)
);

ALTER TABLE itpux02
    MODIFY id NOT NULL;

--1.3 验证。

INSERT INTO itpux02
     VALUES (NULL, 'name');

INSERT INTO itpux02
     VALUES (1, 'name');

SELECT * FROM itpux02;

--1.4 删除

ALTER TABLE itpux02
    MODIFY id NULL;

INSERT INTO itpux02
     VALUES (NULL, 'name');

SELECT * FROM itpux02;

INSERT INTO itpux02
     VALUES (2, 'name');
```
* 2.primary key 约束
```sql
DROP TABLE itpux01;
CREATE TABLE itpux01
(
    id    NUMBER NOT NULL PRIMARY KEY,
    name VARCHAR2 (20)
);
--1.2 在已经创建的表加添加
DROP TABLE itpux02;

CREATE TABLE itpux02
(
    id NUMBER,
    name VARCHAR2 (20)
);

ALTER TABLE itpux02
    ADD CONSTRAINT itpux02_id_con PRIMARY KEY (id); --建议命名
--alter table itpux02 drop constraint itpux02_id_con;

ALTER TABLE itpux02
    ADD PRIMARY KEY (id);

--1.3 删除

ALTER TABLE itpux02
    DROP CONSTRAINT itpux02_id_con;

ALTER TABLE itpux02
    DROP CONSTRAINT SYS_C0011169;
```

* 3.unique 约束
```sql
--3.1 创建表的时候

CREATE TABLE itpux03
(
    id      NUMBER NOT NULL,
    name    VARCHAR2 (20) UNIQUE
);

INSERT INTO itpux03
     VALUES (1, 'name1');

INSERT INTO itpux03
     VALUES (2, 'name2');

--3.2 删除

ALTER TABLE itpux03
    DROP UNIQUE (name);

INSERT INTO itpux03
     VALUES (3, 'name2');

--3.3 后期添加

ALTER TABLE itpux03
    ADD UNIQUE (name);

DELETE FROM itpux03
      WHERE id = 3;

ALTER TABLE itpux03
    ADD UNIQUE (name);
```

* 4.Check 约束
```sql
  --4.1 新建表添加约束

CREATE TABLE itpux04
(
    id     NUMBER NOT NULL PRIMARY KEY,
    name VARCHAR2 (20),
    age    NUMBER CONSTRAINT itpux04_age_con CHECK (age > 17)
);

INSERT INTO itpux04
     VALUES (1, 'mm', 16);

INSERT INTO itpux04
     VALUES (2, 'mm', 18);

--4.2 删除

ALTER TABLE itpux04
    DROP CONSTRAINT itpux04_age_con;

--4.3 后期
alter table itpux04 add constraint itpux04_age_con check(age>17);
```

* 5.Foreign key 约束
```sql
CREATE TABLE itpux051
(
    jobid    NUMBER NOT NULL PRIMARY KEY,
    name1 VARCHAR2 (20),
    age NUMBER (3)
);

CREATE TABLE itpux052
(
    no      NUMBER NOT NULL PRIMARY KEY,
    dcname VARCHAR2 (20),
    dcid    NUMBER REFERENCES itpux051 (jobid)
);

INSERT INTO itpux052
     VALUES (1, 'dc1008', 20161218);                                                  --error

INSERT INTO itpux051
     VALUES (20161218, '风哥', 31);

INSERT INTO itpux052
     VALUES (1, 'dc1008', 20161218);

SELECT * FROM itpux051;

SELECT * FROM itpux052;

--删除

ALTER TABLE itpux052
    DROP CONSTRAINT SYS_C0011182;

DROP TABLE itpux051;
DROP TABLE itpux051 CASCADE CONSTRAINTS;
--后期增加

ALTER TABLE itpux052
    ADD CONSTRAINT itpux052_dcid_con FOREIGN KEY (dcid)
        REFERENCES itpux051 (jobid);
```

### 7.1  约束的管理

* 1.约束表级定义与列级定义
```sql
  --1.1 表级定义
CREATE TABLE itpux08
(
    id NUMBER (8),
    name VARCHAR2 (50),
    CONSTRAINT pk_itpux08_id PRIMARY KEY (id)
);

--1.2 列级定义
CREATE TABLE itpux09
(
    id    NUMBER (8) CONSTRAINT pk_itpux09_id PRIMARY KEY,
    name VARCHAR2 (50)
);
```

* 2.约束的disable/enable,validate/novalidate 的区别
```sql
--disable 禁用
--disable validate --关闭后，不能增删改
--disable novalidate --关闭后，可以增删改
--enable 启用
--enable validate --启用后，进行所有约束，有的，新加的
--enable novalidate --启用后，对新加入的约束，对旧的不约束
--2.1 非主键的

CREATE TABLE itpux06
(
    id     NUMBER NOT NULL PRIMARY KEY,
    name VARCHAR2 (20),
    age    NUMBER CONSTRAINT itpux06_age_con CHECK (age > 17)
);

ALTER TABLE itpux06
    DISABLE CONSTRAINT itpux06_age_con;   --novalidate

INSERT INTO itpux06
     VALUES (2, 'name2', 18);

ALTER TABLE itpux06
    ENABLE CONSTRAINT itpux06_age_con;     --validate

ALTER TABLE itpux06
    DISABLE VALIDATE CONSTRAINT itpux06_age_con;                                                                 --validate

INSERT INTO itpux06
     VALUES (3, 'name3', 18);              --error

ALTER TABLE itpux06
    ENABLE CONSTRAINT itpux06_age_con;     --validate

INSERT INTO itpux06
     VALUES (3, 'name3', 18);

ALTER TABLE itpux06
    DISABLE CONSTRAINT itpux06_age_con;    --novalidate

INSERT INTO itpux06
     VALUES (4, 'name4', 16);

ALTER TABLE itpux06
    ENABLE CONSTRAINT itpux06_age_con;     --errpr

ALTER TABLE itpux06
    ENABLE NOVALIDATE CONSTRAINT itpux06_age_con;

--2.2 主键的

CREATE TABLE itpux07
(
    id NUMBER,
    name VARCHAR2 (20)
);

ALTER TABLE itpux07
    ADD CONSTRAINT itpux07_id_con PRIMARY KEY (id);

INSERT INTO itpux07
     VALUES (1, 'name1');

ALTER TABLE itpux07
    DISABLE CONSTRAINT itpux07_id_con;    --novalidate

INSERT INTO itpux07
     VALUES (1, 'name2');

SELECT * FROM itpux07;

ALTER TABLE itpux07
    ENABLE CONSTRAINT itpux07_id_con;    --error

ALTER TABLE itpux07
    ENABLE NOVALIDATE CONSTRAINT itpux07_id_con;                                                                 --error

CREATE INDEX idx_itpux07_id
    ON itpux07 (id);

ALTER TABLE itpux07
    ENABLE NOVALIDATE CONSTRAINT itpux07_id_con;                                                                 --ok

SELECT * FROM itpux07;
```
### 7.3 显示约束的信息
* 1.约束的数据字典
```sql
select * from all_constraints;
select * from dba_constraints;
select * from user_constraints;
select distinct constraint_type from user_constraints ;
-- R references : column
-- U unique : column
-- P primary key : column
-- C check : column
-- O read only on a view : object
-- V Check on a view : object
select * from all_cons_columns;
select * from dba_cons_columns;
select * from user_cons_columns;
select * from dba_constraints where table_name='ITPUX08';
```

* 2.约束的可延迟属性
```sql
select * from dba_constraints where table_name='ITPUX08';

--DEFERRABLE(延迟条件): NOT DEFERRABLE(默认，立即验证), DEFERRABLE（提
交时验证）
--DEFERRED ： IMMEDIATE(默认，立即验证)，DEFERRED（提交时才验证）,
--set constraint 名字immediate;
--用途：物化视图，级联更新。

create table itpux10(
id number not null primary key DEFERRABLE initially IMMEDIATE,
name varchar2(20),
age number constraint itpux10_age_con check(age>17) DEFERRABLE
initially DEFERRED);

select * from dba_constraints where table_name='ITPUX10';
```
## 8 视图的创建与管理
### 8.1 语法
```sql
CREATE [OR REPLACE] [FORCE|NOFORCE] VIEW view_name
[(alias[, alias]...)]
AS subquery
[WITH CHECK OPTION [CONSTRAINT constraint]]
[WITH READ ONLY]
OR REPLACE ：若所创建的试图已经存在，ORACLE 自动重建该视图；
FORCE ：不管基表是否存在ORACLE 都会自动创建该视图；
NOFORCE ：只有基表都存在ORACLE 才会创建该视图：
alias ：为视图产生的列定义的别名；
subquery ：一条完整的SELECT 语句，可以在该语句中定义别名；
WITH CHECK OPTION ：插入或修改的数据行必须满足视图定义的约束；
WITH READ ONLY ：该视图上不能进行任何DML 操作。
-- O read only on a view : object
-- V Check OPTION on a view : object
```
## 9 同义词创建和使用
```sql
--1.创建同义词
create synonym yg_s for yg_v;
create synonym bm_s for bm_v;
create synonym bm_sum_s for bm_sum_v;
select * from yg_s;
select * from bm_s;
drop view yg_v;
drop view bm_v;
alter view bm_sum_v compile;
drop view bm_sum_v;
alter synonym yg_s compile;

--2.删除同义词
drop synonym yg_s;
drop synonym bm_s;
drop synonym bm_sum_s;

--3.查看同义词
select * from dba_synonyms where owner='ITPUX';
select * from all_synonyms
select * from user_synonyms
select * from scott.emp;
create synonym emp_s for scott.emp;
select * from emp_s;
create table emp as select * from scott.emp;
select * from emp;

--4.权限
grant create synonym to itpux01;
grant create any synonym to itpux01;
grant create public synonym to itpux01;
```

## 10 序列创建和使用.
### 10.1 语法
```sql
创建序列：
1、要有创建序列的权限create sequence 或create any sequence
2、创建序列的语法
CREATE SEQUENCE sequence //创建序列名称
		[INCREMENT BY n] //递增的序列值是n 如果n 是正数就递增,如果是负数
		就递减默认是1
		[START WITH n] //开始的值,递增默认是minvalue 递减是maxvalue
		[{MAXVALUE n | NOMAXVALUE}] //最大值
		[{MINVALUE n | NOMINVALUE}] //最小值
		[{CYCLE | NOCYCLE}] //循环/不循环
		[{CACHE n | NOCACHE}];//分配并存入到内存中
	NEXTVAL 返回序列中下一个有效的值，任何用户都可以引用
	CURRVAL 中存放序列的当前值
	NEXTVAL 应在CURRVAL 之前指定，二者应同时有效
```

### 10.2 日常使用
```sql
--创建普通序列
create sequence seql start with 10 nocache maxvalue 15 cycle;
create sequence seql1
--创建一个带主键的表
drop table seqtest
create table seqtest(
	c1 number,
	c2 varchar2(10)
);
alter table seqtest add constraint seqtest_pk primary key (c1);
create sequence seq_test_pk_s
	minvalue 1
	maxvalue 9999999999999999
	start with 1
	increment by 1
	cache 20;
--间断
A:
insert into seqtest values (seq_test_pk_s.nextval,'11');
commit;

B:
insert into seqtest values (seq_test_pk_s.nextval,'22');
A:
insert into seqtest values (seq_test_pk_s.nextval,'33');
commit;
select * from seqtest;
B:
rollback;
select * from seqtest;
--删除
drop table seqtest
drop sequence SEQ_TEST_PK_S
```

## 11 存储过程创建和使用
### 11.1 语法
```sql
CREATE [OR REPLACE] PROCEDURE procedure_name
[(parameter1[model] datatype1, parameter2 [model] datatype2..)]
IS[AS]
BEGIN
PL/SQL;
END [procedure_name];
```
### 11.2日常使用
```sql
--1.编写存储过程
--显示当前系统时间
--通过pl/sql dev 菜单
--手写
create or replace procedure print_sysdate
is
begin
	dbms_output.put_line(sysdate);
	end print_sysdate;
/
--2.存储过程的调用
--sqlplus,pl/sql 代码块
exec print_sysdate;
--set serverout on;
--exec print_sysdate();

--3.存储过程的删除
drop procedure test1;
--4.编译存储过程
--界面
--命令
alter procedure print_sysdate compile;


--5.查询存储过程
select * from dba_objects where object_type='PROCEDURE' and
owner='ITPUX';
select * from all_objects where object_type='PROCEDURE' and owner='ITPUX';
select * from user_objects where object_type='PROCEDURE';

--6.查看存储过程的内容
--界面
--命令
select * from user_source where name='PRINT_SYSDATE';
select * from dba_source where owner='ITPUX' and name='PRINT_SYSDATE';
select * from all_source;
select text from user_source where name='PRINT_SYSDATE';
select * from dba_source where type='PROCEDURE' and text like '%sysdate%';
```

### 11.3 存储过程中事务处理
* 存储过程中事务处理
```sql
--案例1 显示YG 表中员工人数的存储过程。
create or replace procedure yg_count
as
v_total number(10);
begin
	select count(*) into v_total from yg;
	dbms_output.put_line('员工总人数为：'||v_total);
end;
set serverout on;
execute yg_count;
--员工总人数为：107

--案例2 存储过程中的增删改DML 事务操作。
--准备
create table itpux_st(id int,name varchar2(10));
insert into itpux_st values(1,'itpux01');
insert into itpux_st values(2,'itpux02');
commit;

--2.1 增加
--创建过程
create or replace procedure pro_insert
(v_id int,v_name varchar2)
is
begin
	insert into itpux_st values(v_id,v_name);
commit;
end;
--执行
begin
	pro_insert(3,'itpux03');
end;
--检查
select * from itpux_st;

--2.2 删除
--创建过程
create or replace procedure pro_delete
(v_id int)
is
begin
	delete from itpux_st where id=v_id;
commit;
end;
--执行
begin
pro_delete(3);
end;
--检查
select * from itpux_st;

--2.3 更新/更改
--创建过程
create or replace procedure pro_update
(v_id int,v_name varchar2)
is
begin
	update itpux_st set name=v_name where id=v_id;
commit;
end;
--执行
begin
	pro_update(2,'itpux02222');
end;
--检查
select * from itpux_st;

--2.4 查询
--创建过程
create or replace procedure pro_select
(v_id int) --定义输入变量
is
	v_name varchar2(10);--定义输出变量
begin
	select name into v_name from itpux_st where id=v_id;--执行查询
	dbms_output.put_line('学生姓名为：'||v_name);--输出结果
end;
--执行
--set serverout on;
begin
pro_select(2);
end;
```

## 12 触发器创建和使用
* 1.触发器的语法
```sql
CREATE [OR REPLACE] TIGGER 触发器名触发时间触发事件
	ON 表名/视图名
	[FOR EACH ROW] //加上FOR EACH ROW 即为行级触发器，不加时为语句
	级触发器
	BEGIN
	pl/sql 语句
	END
```
### 12.1 DML 触发器及案例
* 语法
```sql
create [or replace] trigger 触发器名称
	{before|after}
	{insert|delete|update[of column [,column...]]}
	or {insert|delete|update[of column [,column...]]}
	on [schema.] 表名|[schema.]视图
	[for each row]
	[when condition]
begin
执行语句;
end;
```

* 1.设置一个添加记录就提示的触发器
```sql
create or replace trigger t1
after insert on itpux.emp
	begin
		dbms_output.put_line('你向ITPUX.EMP 表中添加了一条数据');
	end;
/

--验证
--set serverout on
insert into emp values('8000','itpux15','DBA','8001',sysdate,'15000','0','30');
insert into emp values('8002','itpux16','DBA','8002',sysdate,'15000','0','30');
--你向ITPUX.EMP 表中添加了一条数据
```
* 2.设置一个更改记录就提示的触发器
```sql
create or replace trigger t_update
after update on itpux.emp
for each row --提示每一条修改的记录
	begin
	dbms_output.put_line('你修改了ITPUX.EMP 表中多条数据');
	end;
/
--验证
--set serverout on
select * from emp;
update emp set job='DBA+SA+HR' where ENAME in ('itpux15','itpux16');
--你修改了ITPUX.EMP 表中多条数据
```

* 3.设置一个删除记录就提示的触发器
```sql
create or replace trigger t_delete
before delete on itpux.emp
begin
--select to_char(sysdate,'day') from dual;
if to_char(sysdate,'day') in ('星期六','星期天') then
	dbms_output.put_line('禁止在周末删除ITPUX.EMP 表数据');
	raise_application_error(-20001,'禁止在周末删除ITPUX.EMP 表数据'); --
-20000~20999
end if;
end;
/
--验证
--set serverout on
delete from emp;
--ORA-20001: 禁止在周末删除ITPUX.EMP 表数据
```

### 12.2 DDL 触发器及案例
* 语法
```sql
create or replace trigger ddl 触发器名称
after ddl on 方案名.schema
begin
	执行语句;
end;
```
* 基本使用
```sql
--SYSTEM 用户
drop table itpux_ddl01
create table itpux_ddl01
(
	dbname varchar2(100),
	event varchar2(100),
	username varchar2(30),
	client_ip varchar2(100),
	ddl_time date
);
create or replace trigger t_itpux_ddl01
after ddl on itpux.schema
begin
	insert into itpux_ddl01 values(ora_database_name,ora_sysevent,ora_login_user,ora_client_ip_address,
	sysdate);
end;
--切换至ITPUX 用户
create table itpux012(id number);
select * from system.itpux_ddl01;
```
### 12.3 参考资料
```sql
系统触发器创建基本语法：
create or replace trigger 系统触发器名称
after[before] logon[logoff] on datebase
begin
	执行语句;
end;

详细说明：
after 事件之后触发
before 事件之前触发
logon 登陆触发
logoff 登出触发
startup 开启系统触发
shutdown 关闭系统触发
```
```sql
------------------------------参考资料---------------------------------------
下面介绍一些常用的系统事件属性函数，和建立各种事件触发器的方法在建立系统事
件触发器时，需要使用事件属性函数，
常用的事件属性函数如下：
ora_client_ip_address //返回客户端的ip
ora_database_name //返回数据库名称
ora_login_user //返回登陆用户名
ora_sysevent //返回触发器的系统事件名
ora_des_encrypted_password //返回用户des(md5)加密后的密码
事件属性函数表
Ora_client_ip_address 返回客户端的ip 地址
Ora_database_name 返回当前数据库名
Ora_des_encrypted_password 返回des 加密后的用户口令
Ora_dict_obj_name 返回ddl 操作所对应的数据库对象名
Ora_dict_obj_name_list(name_list out ora_name_list_t)返回在事件中被修改的对
象名列表
Ora_dict_obj_owner 返回ddl 操作所对应的对象的所有者名
Ora_dict_obj_owner_list(owner_list out ora_name_list_t)返回在事件中被修改的对
象的所有者列表
Ora_dict_obj_type 返回ddl 操作所对应的数据库对象的类型
Ora_grantee(user_list out ora_name_list_t)返回授权事件的授权者
Ora_instance_num 返回例程号
Ora_is_alter_column(column_name in varchar2)检测特定列是否被修改
Ora_is_creating_nested_table 检测是否正在建立嵌套表
Ora_is_drop_column(column_name in varchar2)检测特定列是否被删除
Ora_is_servererror(error_number)检测是否返回了特定oracle 错误
Ora_login_user 返回登录用户名
Ora_sysevent 返回触发器的系统事件名
系统触发器的种类和事件出现的时机（前或后）：
事件允许的时机说明
STARTUP AFTER 启动数据库实例之后触发
SHUTDOWN BEFORE 关闭数据库实例之前触发（非正常关闭不触发）
SERVERERROR AFTER 数据库服务器发生错误之后触发
LOGON AFTER 成功登录连接到数据库后触发
LOGOFF BEFORE 开始断开数据库连接之前触发
CREATE BEFORE，AFTER 在执行CREATE 语句创建数据库对象之前、之后
触发
DROP BEFORE，AFTER 在执行DROP 语句删除数据库对象之前、之后触发
ALTER BEFORE，AFTER 在执行ALTER 语句更新数据库对象之前、之后触发
DDL BEFORE，AFTER 在执行大多数DDL 语句之前、之后触发
GRANT BEFORE，AFTER 执行GRANT 语句授予权限之前、之后触发
REVOKE BEFORE，AFTER 执行REVOKE 语句收权限之前、之后触犯发
RENAME BEFORE，AFTER 执行RENAME 语句更改数据库对象名称之前、之
后触犯发
AUDIT/NOAUDIT BEFORE，AFTER 执行AUDIT 或NOAUDIT 进行审计或停
止审计之前、之后触发
特别说明：系统触发器的级别较高，由系统管理员来创建。
------------------------------------------------------------------------------------
```

### 12.4 数据库系统触发器

* 1.创建
```sql
--记录登录数据库的用户名和IP
create table login_info (ip varchar(30),username varchar(30));
create or replace trigger logon_ip_info
after logon on database
declare
	ip varchar(30);
	user varchar(30);
begin
	select sys_context('USERENV','SESSION_USER') into user from dual;
	select sys_context('USERENV','ip_address') into ip from dual;
	insert into login_info values(ip,user);
end;
/

select * from login_info;
```
* 2. 管理
```sql
--禁用
alter trigger logon_ip_info disable;
--启用
alter trigger logon_ip_info enable;
--编译
alter trigger logon_ip_info compile;
--删除
drop trigger logon_ip_info;
--查询
select * from dba_objects where owner='ITPUX' and object_type='TRIGGER';
drop trigger ITPUX.T_UPDATE;
drop trigger ITPUX.T_DELETE;
drop trigger system.t_itpux_ddl01;
```
## 13 包的创建和使用

* 1 包定义（PACKAGE）
```sql
2.1 包定义（PACKAGE）
CREATE [OR REPLACE] PACKAGE package_name
	{IS | AS}
	[公有数据类型定义]
	[公有游标声明]
	[公有变量、常量声明]
	[公有子程序声明]
	[package_name];
定义包规范:
CREATE OR REPLACE
package p_stu
as
--定义结构体
type re_stu is record(
	rname student.name%type,
	rage student.age%type
);
--定义游标
type c_stu is ref cursor;
--定义函数
function numAdd(num1 number,num2 number)return number;
--定义过程
procedure GetStuList(cid in varchar2,c_st out c_stu);
end;
```
* 2. 包主体（PACKAGE BODY）
```sql
CREATE [OR REPLACE] PACKAGE BODY package_name
	{IS | AS}
	[私有数据类型定义]
	[私有变量、常量声明]
	[私有子程序声明和定义]
	[公有子程序定义]
BEGIN
执行部分(初始化部分)
END [package_name];

--1 创建包
create or replace package itpux_pkg
is
	procedure update_sal(e_name varchar2,newsal number);
	FUNCTION emp_sal_fun(e_name varchar2) return number;
end;
--2.3 包体
create or replace package body itpux_pkg is
procedure update_sal(e_name varchar2,newsal number)
is
begin
	update emp set sal=newsal where ename=e_name;
end;
function emp_sal_fun(e_name varchar2)
return number is
emp_sal number;
begin
	select sal*12+nvl(comm,0) into emp_sal from emp
where ename=e_name;
return emp_sal;
end;
end;
--2.4 执行包-调过程
set serveroutput on
	exec itpux_pkg.update_sal('itpux14',2000);
commit;
select * from emp;
--查看itpux14 的记录
--2.5 执行包-调函数
declare
	v_emp_sal number(7,2);
begin
	v_emp_sal:=itpux_pkg.emp_sal_fun('itpux14');
	dbms_output.put_line('ITPUX14 的年薪为: '||v_emp_sal);
end;
--ITPUX14 的年薪为: 24000
--2.6 删除包
select * from dba_objects where owner='ITPUX' and object_type='PACKAGE';
drop package ITPUX_PKG;
drop package ITPUXA_PKG;
```




  [1]: https://docs.oracle.com/cd/E11882_01/index.htm