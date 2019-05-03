---
title: Oracle DML日常操作和事务处理
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


# Oracle DML日常操作和事务处理

## 注意

[Oracle 11G R2 官方文档][1] 

## 1 使用INSERT 命令
### 1.1 语法
```sql
INSERT INTO <TABLE> [(colum[,column.....])]VALUES (values[,value....])
INSERT INTO 表名(列名列表) VALUES(值列表);
table： 指定表或视图
column：指定列名，多列之间用,分开
value： 指定待插入的数据，多值之间依,分开
```
### 1.2 基本使用
```sql
/* Formatted on 2018/2/12 14:59:34 (QP5 v5.313) */
--1.insert 语句
--1.1

SELECT * FROM sql02;                     -- for update
--1.2
SELECT * FROM sql03;
INSERT INTO sql03 (sql03_id1, sql03_id2, sql03_id3)
     VALUES (1, 11, 'sql03_11');
COMMIT;
--1.3

SELECT * FROM sql04;
INSERT INTO sql04
     VALUES (1,
             '11g sql04 01',
             'active',
             60,
             SYSDATE,
             20);
INSERT INTO sql04
     VALUES (2,
             '11g sql04 02',
             'active',
             100,
             SYSDATE,
             40);
COMMIT;
--1.4

SELECT * FROM sql03;
INSERT INTO sql03
     VALUES (&sql03_id1, &sql03_id2, &sql03_id3);

COMMIT;
--1.5

CREATE TABLE sql05
AS
    SELECT * FROM sql04;
DELETE FROM sql05;
COMMIT;

SELECT * FROM sql05;
INSERT INTO sql05
    SELECT * FROM sql04;
COMMIT;

SELECT * FROM sql05;

--1.6

INSERT INTO sql04
     VALUES (3,
             '11g sql04 03',
             'active',
             160,
             SYSDATE,
             20);

INSERT INTO sql04
     VALUES (4,
             '11g sql04 04',
             'active',
             220,
             SYSDATE,
             40);

COMMIT;

SELECT * FROM sql04;

INSERT INTO sql05
    SELECT *
      FROM sql04
     WHERE sql04_price > 200;

COMMIT;

SELECT * FROM sql05;
```
## 2 使用UPDATE 命令
### 2.1 语法
```sql
UPDATE 表名称SET 列名称= 新值<WHERE 条件>
```
### 2.2 基本使用
```sql
/* Formatted on 2018/2/12 15:00:37 (QP5 v5.313) */
--2.1

SELECT *
  FROM sql02
FOR UPDATE;

--2.2

UPDATE sql02
   SET sql02_number = 20002
 WHERE sql02_id = 2;

UPDATE sql02
   SET sql02_status = 'no'
 WHERE sql02_id = 2;

SELECT *
  FROM sql02 commit;

--2.3

SELECT *
  FROM sql02 update sql02 set sql02_id2=222,sql02_status='OK' where sql02_id=2;
--2.4

SELECT *
  FROM sql04 update sql04
set (sql04_desc, sql04_count) =
(select sql04_desc, sql04_count from sql04 where sql04_id = 2)
where sql04_id = 3;
COMMIT;

SELECT * FROM sql04
```

## 3 使用DELETE 命令
### 3.1 语法
```sql
DELETE FROM <table/view> [WHERE <condition>]
```
### 3.2 基本使用
```sql
--3.1

SELECT *
  FROM sql02
FOR UPDATE;

select * from sql02
--3.2
select * from sql04
delete from sql04 where sql04_id=1;
select * from sql04
--3.3
delete from sql04 where sql04_id=2 and sql04_id=3;

DELETE FROM sql04
      WHERE sql04_id IN (2, 3);

DELETE FROM sql04
      WHERE sql04_price IN ('160', '100');

select * from sql04
--3.4
select * from sql03
delete from sql03;
COMMIT;
--truncate table
```
## 4 创建PL/SQL 对象
### 4.1 语法
```sql
/* Formatted on 2018/2/12 15:01:26 (QP5 v5.313) */
SELECT * FROM yg;

--创建

CREATE OR REPLACE PROCEDURE yg_count
AS
    v_yg_count1   NUMBER (10);
    v_yg_count2   NUMBER (10);
BEGIN
    SELECT COUNT (*) INTO v_yg_count1 FROM yg;
    DELETE FROM yg
          WHERE job_id = 'SH_CLERK';
    SELECT COUNT (*) INTO v_yg_count2 FROM yg;
    DBMS_OUTPUT.put_line ('离职前的员工数：' || v_yg_count1);
    DBMS_OUTPUT.put_line ('离职后的员工数：' || v_yg_count2);
END;
/

SET SERVEROUTPUT ON;
EXECUTE yg_count;
--离职前的员工数：107
--离职后的员工数：97

--删除
DROP PROCEDURE yg_count;
```

## 5 事务概念与控制
### 5.1 事务说明
```sql
	   
	事务是对数据库操作的逻辑单位，在一个事务中可以包含一条或多条DML （数据操纵语言）、DDL （数据定义语言）和DCL （数据控制语言）语句，这些语句组成一个逻辑整体。
	事务的执行只有两种结果：要么全部执行，把数据库带入一个新的状态，要么全部不执行，对数据库不做任何修改。
	一组SQL,一个逻辑工作单位，执行时整体修改或者整体回退。
```
### 5.2 事务相关概念

```sql
1）事务的提交和回滚：COMMIT/ROLLBACK
2）事务的开始和结束
	开始事务：连接到数据库，执行DML、DCL、DDL 语句
	结束事务：1. 执行DDL(例如CREATE TABLE),DCL(例如GRANT),系统自动
	
执行COMMIT 语句

2. 执行COMMIT/ROLLBACK
3. 退出/断开数据库的连接自动执行COMMIT 语句
4. 进程意外终止，事务自动rollback
5. 事务COMMIT 时会生成一个唯一的系统变化号（SCN）保存
到事务表
3）保存点（savepoint）： 可以在事务的任何地方设置保存点，以便ROLLBACK
4）事务的四个特性ACID :
	1. Atomicity（原子性）: 事务中sql 语句不可分割，要么都做，要么都不做
	2. Consistency(一致性) ： 指事务操作前后，数据库中数据是一致的，数据
满足业务规则约束（例如账户金额的转出和转入），与原子性对应。
	3. Isolation（隔离性）：多个并发事务可以独立运行，而不能相互干扰，一
个事务修改数据未提交前，其他事务看不到它所做的更改。
	4. Durability（持久性）：事务提交后，数据的修改是永久的。
5） 死锁：当两个事务相互等待对方释放资源时，就会形成死锁。
```

### 5.3 oracle 事务隔离级别
* 1 .两个事务并发访问数据库数据时可能存在的问题

```sql
1. 幻想读：
	事务T1 读取一条指定where 条件的语句，返回结果集。此时事务T2 插入一行新记录并commit，恰好满足T1 的where 条件。然后T1 使用相同的条件再次查
询，结果集中可以看到T2 插入的记录，这条新纪录就是幻想。
2. 不可重复读取：
	事务T1 读取一行记录，紧接着事务T2 修改了T1 刚刚读取的记录并commit，然后T1 再次查询，发现与第一次读取的记录不同，这称为不可重复读。
3. 脏读：
	事务T1 更新了一行记录，还未提交所做的修改，这个T2 读取了更新后的数
据，然后T1 执行回滚操作，取消刚才的修改，所以T2 所读取的行就无效，也就是脏数据。
```

* 2.oracle 事务隔离级别

```sql
oracle 支持的隔离级别：（不支持脏读）
READ COMMITTED--不允许脏读，允许幻想读和不可重复读
SERIALIZABLE--以上三种都不允许
sql 标准还支持READ UNCOMMITTED (三种都允许)和REPEATABLE READ（不允
许不可重复读和脏读，只允许幻想读）
```

* 3.事务相关语句

```sql
SET TRANSACTION----设置事务属性
SET CONSTRAINT -----设置约束
SAVEPOINT ------------建立存储点
RELEASE SAVEPOINT --释放存储点
ROLLBACK---------------回滚
COMMIT------------------提交
```

## 6 锁的检测和锁争用	
### 6.1 解决死锁冲突
* 1）执行commit 或者rollback 结束事务
```sql

```
* 2）终止会话
```sql
在等待资源时执行，查找阻塞会话

SELECT sid, serial#, username
  FROM v$session
 WHERE sid IN (SELECT blocking_session FROM v$session);

杀掉会话

ALTER SYSTEM KILL SESSION '111,222';

关于查看锁的一些视图

SELECT * FROM V$SESSION;                        --查看会话和锁的信息
SELECT * FROM V$SESSION_WAIT;                             --查看等待的会话信息
SELECT * FROM V$LOCK;                     --系统中所有锁
SELECT * FROM V$LOCKED_OBJECT;                              --系统中DML 锁
SELECT l.sid, s.SERIAL#, sq.sql_text
  FROM v$lock l, v$session s, v$sql sq
 WHERE l.sid = s.sid AND s.sql_id = sq.sql_id AND s.status = 'ACTIVE'
alter system kill session '139,1431'
```
## 7 了解撤销数据UNDO
### 7.1 查看近来数据库生成的撤销数据量
```sql
ALTER SESSION SET nls_date_format = 'dd-mm-yy hh24:mi:ss';
SELECT begin_time,
end_time,
(undoblks *
(SELECT VALUE FROM v$parameter WHERE NAME = 'db_block_size'))
undo_bytes
FROM v$undostat;
```



  [1]: https://docs.oracle.com/cd/E11882_01/index.htm