---
title: Oracle SQL之常用函数
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
  - SQL
---


# OracleSQL语言之常用函数

## 注意

[Oracle 11G R2 官方文档][1] 

## 1. 聚合函数-数据统计
```sql
--01.max 取列和表达式的最大值

SELECT MAX (salary) FROM employees;
SELECT MAX (DISTINCT salary) FROM employees;

--02.min 取列和表达式的最小值

SELECT MIN (salary) FROM employees;
SELECT MIN (DISTINCT salary) FROM employees;

--03.avg 取列和表达式的平均值

SELECT AVG (salary) FROM employees;
SELECT AVG (DISTINCT salary) FROM employees;
SELECT * FROM employees;

--04.sum 求列和表达式的总和

SELECT SUM (salary) FROM employees;
SELECT SUM (salary + 1000) FROM employees;
SELECT SUM ((salary + 1000) - salary) FROM employees;

SELECT department_id, SUM (salary)
    FROM employees
GROUP BY department_id
ORDER BY 2 DESC;

--05.count 求行数总和

SELECT COUNT (*) FROM employees;

SELECT COUNT (*)
  FROM employees
 WHERE manager_id = 120;

SELECT COUNT (email) FROM employees;

SELECT COUNT (DISTINCT manager_id) FROM employees;

--06.other
--stddev 标准差,variance 协方差，median 求中位数

SELECT STDDEV (salary) FROM employees;
SELECT VARIANCE (salary) FROM employees;
SELECT MEDIAN (salary) FROM employees;
```

## 2. 分组函数-使用group by 与having
```sql
-01.简单的分组函数应用
--统计各个国家名字的长度
--length,avg,round

  SELECT country_name, LENGTH (country_name)
    FROM countries
GROUP BY country_name;

SELECT ROUND (AVG (LENGTH (country_name))) FROM countries;

--工资

  SELECT manager_id, MIN (salary) min_sal, MAX (salary) max_sal
    FROM employees
GROUP BY manager_id;

--02.多列分组数据
--统计简历表中每年辞职的员工数
--to_char,count

SELECT * FROM jl;

  SELECT TO_CHAR (end_date, 'yyyy') year, COUNT (*)
    FROM job_history
GROUP BY TO_CHAR (end_date, 'yyyy')
ORDER BY 1;

  SELECT TO_CHAR (end_date, 'yyyy') year, job_id, COUNT (*)
    FROM job_history
GROUP BY TO_CHAR (end_date, 'yyyy'), job_id
ORDER BY 1;

--03.使用having 子句
--限制：1.对行进行分组，2.应用组函数，3.显示符合having 子句条件的组
--查招聘多于等于15 个员工在星期几。

SELECT * FROM employees;

--to_char,count

  SELECT TO_CHAR (hire_date, 'Day') 星期几, COUNT (*)
    FROM employees
GROUP BY TO_CHAR (hire_date, 'Day')
  HAVING COUNT (*) >= 15;

--统计部门里最大工资大于10000 的
--max

SELECT department_id, MAX (salary)
    FROM employees
GROUP BY department_id
  HAVING MAX (salary) > 10000
ORDER BY 2;

```
## 3. 字符函数-运算与截取
```sql
--01.round（n,[m]） 四舍五入
--m=0 整数，m<0 小数点的前m 位,m>0 小数点的后m 位
--四舍

SELECT salary, salary + 0.2 FROM employees;
SELECT salary, ROUND (salary + 0.2) FROM employees;
SELECT salary, ROUND ((salary + 0.2), 0) FROM employees;

--五入

SELECT salary, salary + 0.8 FROM employees;
SELECT salary, ROUND (salary + 0.8) FROM employees;

--小数位

SELECT salary, salary + 0.842 FROM employees;
SELECT salary, ROUND (salary + 0.842) FROM employees;
SELECT salary, ROUND ((salary + 0.842), 2) FROM employees;
SELECT salary, ROUND ((salary + 0.867), 2) FROM employees;
SELECT salary, ROUND ((salary + 0.867), -3) FROM employees;
SELECT salary, ROUND ((salary + 0.867), -4) FROM employees;

--02.trunc(n,[m]) 截取数字.
--m=0 去掉小数位，m<0 小数点的前m 位,m>0 小数点的后m 位

SELECT salary, salary + 0.2 FROM employees;
SELECT salary, TRUNC (salary + 0.2) FROM employees;
SELECT salary, TRUNC ((salary + 0.2), 0) FROM employees;
SELECT salary, TRUNC (salary + 0.842) FROM employees;
SELECT salary, TRUNC ((salary + 0.867), 2) FROM employees;
SELECT salary, TRUNC ((salary + 0.867), -3) FROM employees;
SELECT salary, TRUNC ((salary + 0.867), -4) FROM employees;

--03.ceil(n) 返回>=数字n 的最小整数

SELECT salary, salary + 0.8567 FROM employees;
SELECT salary, CEIL (salary + 0.8567) FROM employees;
SELECT salary, CEIL (salary + 0.2567) FROM employees;

--04.floor(n) 返回<=数字n 的最小整数

SELECT salary, salary + 0.8567 FROM employees;
SELECT salary, FLOOR (salary + 0.8567) FROM employees;
SELECT salary, FLOOR (salary + 0.2567) FROM employees;

--05.length(n) 返回字符串的长度

SELECT country_name, LENGTH (country_name) FROM countries;
```
## 4. 转换函数-大小写转换
```sql
--01.lower 转换为小写
SELECT * FROM employees;

SELECT first_name, last_name
  FROM employees
 WHERE first_name LIKE '%li%';

--Li,lI,LI,li

SELECT first_name, last_name
  FROM employees
 WHERE LOWER (first_name) LIKE '%li%';

SELECT first_name, last_name
  FROM employees
 WHERE first_name = LOWER ('ITPUX01');

SELECT LOWER (first_name), last_name FROM employees;

--02.upper 转换为大写

SELECT UPPER (first_name), last_name FROM employees;

--03.initcap 将第一个字母大写，其余的小写。

SELECT first_name, last_name FROM employees;

SELECT INITCAP (first_name), last_name FROM employees;

--04.综合运用

SELECT UPPER (first_name), LOWER (last_name), INITCAP (job_id) FROM employees;
```
## 5. 转换函数-日期字符数字转换
```sql
--01.to_date 字符串>日期类型
--Year: yy 16,yyy 016, yyyy 2016
--Month: mm 11,mon 11 月/nov,month 11 月/november
--Day: dd 当月第几天02, ddd 当年第几天02, dy 当周第几天星期五/fri,day 当周第几天星期五/friday
--Hour: hh 12 小时进制01，hh24 24 小时进制13
--24 小时的时间范围： 0:00:00 - 23:59:59,12 小时时间范围:1:00:00-12:59:59
--Minute: mi 60 进制45
--Second: ss 60 进制25
--其它: Q 季度4， WW 当年第几周44，W 当月第几周1

SELECT TO_DATE ('2016-12-18 19:02:02', 'YYYY-MM-DD HH24:MI:SS') FROM DUAL;
SELECT TO_DATE ('2016-12-18 19:02', 'YYYY-MM-DD HH24:MI') FROM DUAL;
SELECT TO_DATE ('2016-12-18 19', 'YYYY-MM-DD HH24') FROM DUAL;
SELECT TO_DATE ('2016-12-18', 'YYYY-MM-DD') FROM DUAL;
SELECT TO_DATE ('2016-12', 'YYYY-MM') FROM DUAL;
SELECT TO_DATE ('2016', 'YYYY') FROM DUAL;
SELECT TO_DATE ('20161218', 'YYYY-MM-DD') FROM DUAL;

--02.to_char 日期/数字>字符串

SELECT TO_CHAR (SYSDATE, 'yyyy-mm-dd hh24:mi:ss') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'yyyy') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'dd') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'hh24') FROM DUAL;
SELECT TO_CHAR ('23432.4') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'yyyymmdd') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'mmdd') FROM DUAL;
SELECT TO_CHAR (SYSDATE, 'yyyymm') FROM DUAL;

--03.to_number 字符>数字

SELECT SYSDATE - (TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) FROM DUAL;
SELECT TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) FROM DUAL;
SELECT TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) / 365 FROM DUAL;
SELECT TRUNC (TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) / 365) FROM DUAL;
SELECT TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) * 24 FROM DUAL;
SELECT TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) * 24 * 60 FROM DUAL;
SELECT TO_NUMBER (SYSDATE - TO_DATE ('2006-11-02 15:55:55', 'yyyy-mm-dd hh24:mi:ss')) * 24 * 60 * 60 FROM DUAL;

```
## 6. 日期函数-使用日期函数使用
```sql
--01.sysdate

SELECT SYSDATE FROM DUAL;

--02.add_months 增加/减去月份

SELECT ADD_MONTHS (SYSDATE, -1) FROM DUAL;

SELECT ADD_MONTHS (SYSDATE, +1) FROM DUAL;

--03.last_day 返回日期的最后一天

SELECT LAST_DAY (SYSDATE) FROM DUAL;
SELECT LAST_DAY (ADD_MONTHS (SYSDATE, -1)) FROM DUAL;

--04.months_between 时间2-时间1 的月份

SELECT MONTHS_BETWEEN (TO_DATE ('2016/11/30', 'YYYY-MM-DD'),
                       TO_DATE ('2016/05/30', 'YYYY-MM-DD'))
    FROM DUAL;

--05.next_day
--星期日=1，一=2 二=3，三=4，四=5，五=6，六=7

SELECT NEXT_DAY (SYSDATE, 5) FROM DUAL;
SELECT NEXT_DAY (SYSDATE, 1) FROM DUAL;

--06.current_date 当前日期

SELECT CURRENT_DATE FROM DUAL;

--07.current_timestamp 当前日期

SELECT CURRENT_TIMESTAMP FROM DUAL;

--08.sessiontimezone 时区

SELECT SESSIONTIMEZONE FROM DUAL;

--09.trunc( for date)

SELECT TRUNC (SYSDATE, 'YYYY') FROM DUAL;
SELECT TRUNC (SYSDATE, 'MM') FROM DUAL;
SELECT TRUNC (SYSDATE, 'D') FROM DUAL;
SELECT TRUNC (SYSDATE, 'DD') FROM DUAL;
```

* 字符串与日期转换

```sql
select to_char(sysdate,'yyyy-MM-dd HH24:mi:ss') from dual;
select to_date('2018-02-12 16:35:36','yyyy-MM-dd HH24:mi:ss') from dual;
```
## 7. 集合函数-合并两张表的数据
```sql
--01.union (无重并集，去重排序)
SELECT * FROM employees;
SELECT * FROM scott.salgrade;
SELECT COUNT (1), SUM (salary)
  FROM (SELECT y.salary
          FROM employees y
        UNION
        SELECT s.hisal
          FROM scott.salgrade s);

SELECT * FROM sql01;
SELECT * FROM sql06;
CREATE TABLE sql06
AS
    SELECT * FROM sql01;
    
DELETE FROM sql06
      WHERE sql01_id = 3;

COMMIT;

SELECT * FROM sql01
UNION
SELECT * FROM sql06;

--02.union all（有重并集，不去重排序）

SELECT COUNT (1), SUM (salary)
  FROM (SELECT y.salary
          FROM employees y
        UNION ALL
        SELECT s.hisal
          FROM scott.salgrade s);

SELECT * FROM sql01
UNION ALL
SELECT * FROM sql06;

--03.intersect （交集,显示相同的）

SELECT * FROM sql01
INTERSECT
SELECT * FROM sql06;

--04.minus （差集，显示不同的）

SELECT * FROM sql01
MINUS
SELECT * FROM sql06;
```
## 8. 分析函数-使用DECODE 函数
```sql
--decode (条件,值1，返回值1，值2，返回值2,...，缺省值）
SELECT * FROM employees;

  SELECT DECODE (job_id,  'AD_PRES', 'CEO',  'HR_PEP', 'HR',  'SA') JOB,
         COUNT (*)                                                JOB_COUNT
    FROM employees
GROUP BY DECODE (job_id,  'AD_PRES', 'CEO',  'HR_PEP', 'HR',  'SA');

--判断输出

SELECT *
  FROM employees
 WHERE salary IN (3100, 2800);

SELECT DECODE (salary, 3100, 'yes', 'no'), DECODE (salary, 2800, 'yes', 'no')
  FROM employees
 WHERE salary IN (3100, 2800);
```


  [1]: https://docs.oracle.com/cd/E11882_01/index.htm