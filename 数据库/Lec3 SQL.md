# Lec3 Introduction to SQL

本章介绍 SQL 的基础：数据定义、基本查询、集合操作、空值、聚合、嵌套子查询，以及数据库修改语句。

SQL 的核心特点是 declarative：你说明“要什么结果”，不说明“怎么执行”。

---

## 1. SQL 简史

SQL 起源于 IBM System R 项目的 SEQUEL，后来改名为 SQL（Structured Query Language）。

标准化历程包括：

- SQL-86；
- SQL-89；
- SQL-92；
- SQL:1999；
- SQL:2003；
- SQL:2006；
- SQL:2008；
- SQL:2011；
- SQL:2016；
- SQL:2019。

实际数据库系统通常支持 SQL-92 的大部分内容，并加入各自扩展。

---

## 2. Data Definition Language

DDL 用于定义 relation 及其相关信息，包括：

- relation schema；
- attribute domain；
- integrity constraints；
- indices；
- authorization；
- physical storage information。

---

## 3. SQL Domain Types

常见内建类型：

| 类型 | 含义 |
| --- | --- |
| `char(n)` | 定长字符串 |
| `varchar(n)` | 变长字符串，最大长度为 n |
| `int` | 整数 |
| `smallint` | 较小整数 |
| `numeric(p,d)` / `decimal(p,d)` | 定点数，总共 p 位，小数点右边 d 位 |
| `real` | 单精度浮点 |
| `double precision` | 双精度浮点 |
| `float(n)` | 至少 n 位精度的浮点数 |
| `date` | 日期 |
| `time` | 时间 |
| `timestamp` | 日期 + 时间 |
| `interval` | 时间间隔 |

例子：

```sql
date '2005-7-27'
time '09:00:30.75'
timestamp '2005-7-27 09:00:30.75'
interval '1' day
```

常见时间函数：

```sql
current_date()
current_time()
year(x), month(x), day(x)
hour(x), minute(x), second(x)
```

---

## 4. Create Table

基本格式：

```sql
create table r (
  A1 D1,
  A2 D2,
  ...
  An Dn,
  integrity_constraint1,
  ...
);
```

例子：

```sql
create table instructor (
  ID        char(5),
  name      varchar(20) not null,
  dept_name varchar(20),
  salary    numeric(8,2),
  primary key (ID),
  foreign key (dept_name) references department
);
```

注意：

> primary key 自动隐含 not null。

---

## 5. Integrity Constraints in SQL

常见约束：

- `not null`；
- `primary key (...)`；
- `foreign key (...) references ...`；
- `default`；
- `check`。

例子：

```sql
create table student (
  ID        varchar(5),
  name      varchar(20) not null,
  dept_name varchar(20),
  tot_cred  numeric(3,0) default 0,
  primary key (ID),
  foreign key (dept_name) references department
);
```

复合主键例子：

```sql
create table takes (
  ID        varchar(5),
  course_id varchar(8),
  sec_id    varchar(8),
  semester  varchar(6),
  year      numeric(4,0),
  grade     varchar(2),
  primary key (ID, course_id, sec_id, semester, year),
  foreign key (ID) references student,
  foreign key (course_id, sec_id, semester, year) references section
);
```

外键删除/更新动作：

```sql
on delete cascade | set null | restrict | no action | set default
on update cascade | set null | restrict | no action | set default
```

---

## 6. Drop and Alter

删除表：

```sql
drop table student;
```

删除表中所有数据但保留表结构：

```sql
delete from student;
```

增加属性：

```sql
alter table student add resume varchar(256);
```

新增属性对已有 tuple 通常为 null。

删除属性：

```sql
alter table r drop A;
```

并非所有数据库都支持 drop attribute。

---

## 7. SQL 与关系代数

典型 SQL：

```sql
select A1, A2, ..., An
from r1, r2, ..., rm
where P;
```

对应：

$$
\Pi_{A_1,\dots,A_n}
(\sigma_P(r_1 \times r_2 \times \cdots \times r_m))
$$

但注意 SQL 默认是 multiset 语义，允许重复。

---

## 8. Basic Query Structure

基本查询格式：

```sql
select A1, A2, ..., An
from r1, r2, ..., rm
where P;
```

- `select`：要输出的属性，对应 projection；
- `from`：涉及的关系，对应 Cartesian product；
- `where`：筛选条件，对应 selection。

---

## 9. Select Clause

查询教师姓名：

```sql
select name
from instructor;
```

SQL 允许重复结果。

去重：

```sql
select distinct dept_name
from instructor;
```

显式保留重复：

```sql
select all dept_name
from instructor;
```

查询所有属性：

```sql
select *
from instructor;
```

select 中可以使用算术表达式：

```sql
select ID, name, salary / 12 as monthly_salary
from instructor;
```

---

## 10. Where Clause

`where` 指定 tuple 必须满足的条件。

例子：

```sql
select name
from instructor
where dept_name = 'Comp. Sci.' and salary > 80000;
```

比较可以和：

- `and`；
- `or`；
- `not`

组合。

### between

```sql
select name
from instructor
where salary between 90000 and 100000;
```

等价于：

```sql
where salary >= 90000 and salary <= 100000
```

### tuple comparison

```sql
select name, course_id
from instructor, teaches
where (instructor.ID, dept_name) = (teaches.ID, 'Biology');
```

---

## 11. From Clause and Joins

`from` 列出参与查询的 relations。

```sql
select *
from instructor, teaches;
```

表示笛卡尔积。

通常和 `where` 结合表达 join：

```sql
select name, course_id
from instructor, teaches
where instructor.ID = teaches.ID;
```

---

## 12. Natural Join

natural join 会自动匹配所有同名属性，并只保留一份同名属性。

```sql
select *
from instructor natural join teaches;
```

如果：

```text
instructor(ID, name, dept_name, salary)
teaches(ID, course_id, sec_id, semester, year)
```

则 natural join 会按 `ID` 连接。

### 小心 natural join 的坑

如果两个表有无关但同名的属性，也会被自动相等连接。

错误例子：

```sql
select name, title
from instructor natural join teaches natural join course;
```

因为 `instructor` 和 `course` 都有 `dept_name`，会错误要求教师院系等于课程院系。

更稳的写法：

```sql
select name, title
from (instructor natural join teaches) join course using (course_id);
```

或显式写连接条件：

```sql
select name, title
from instructor, teaches, course
where instructor.ID = teaches.ID
  and teaches.course_id = course.course_id;
```

---

## 13. Rename Operation

SQL 使用 `as` 重命名 attribute 或 relation。

```sql
select ID, name, salary / 12 as monthly_salary
from instructor;
```

自连接时必须使用别名：

```sql
select distinct T.name
from instructor as T, instructor as S
where T.salary > S.salary
  and S.dept_name = 'Comp. Sci.';
```

`as` 通常可以省略：

```sql
from instructor T, instructor S
```

---

## 14. String Operations

SQL 使用 `like` 做字符串模式匹配。

特殊字符：

- `%`：匹配任意子串；
- `_`：匹配任意单个字符。

例子：

```sql
select name
from instructor
where name like '%dar%';
```

转义 `%`：

```sql
like '100 \%' escape '\'
```

常见模式：

- `'Intro%'`：以 Intro 开头；
- `'%Comp%'`：包含 Comp；
- `'___'`：恰好三个字符；
- `'___%'`：至少三个字符。

---

## 15. Ordering and Limit

排序：

```sql
select distinct name
from instructor
order by name;
```

降序：

```sql
order by salary desc
```

多属性排序：

```sql
order by dept_name, name
```

限制返回行数：

```sql
select name
from instructor
order by salary desc
limit 3;
```

`limit offset, row_count` 表示跳过 offset 行后取 row_count 行。

---

## 16. Duplicates and Multiset Semantics

SQL 默认保留重复。

如果 tuple 在输入中出现多次，selection 和 projection 会保留相应数量的副本。

去重必须写：

```sql
select distinct ...
```

---

## 17. Set Operations

SQL 集合操作：

- `union`；
- `intersect`；
- `except`。

这些默认去重。

例子：

```sql
(select course_id from section where semester = 'Fall' and year = 2009)
union
(select course_id from section where semester = 'Spring' and year = 2010);
```

保留重复：

```sql
union all
intersect all
except all
```

若某 tuple 在 $r$ 中出现 $m$ 次，在 $s$ 中出现 $n$ 次：

- `r union all s`：出现 $m+n$ 次；
- `r intersect all s`：出现 $\min(m,n)$ 次；
- `r except all s`：出现 $\max(0,m-n)$ 次。

---

## 18. Null Values

`null` 表示未知、不存在或不适用。

任何涉及 null 的算术表达式结果通常是 null：

```text
5 + null = null
```

检查 null：

```sql
select name
from instructor
where salary is null;
```

不能写：

```sql
where salary = null
```

### Three-Valued Logic

涉及 null 的比较结果为 `unknown`。

逻辑规则：

- `unknown or true = true`
- `unknown or false = unknown`
- `true and unknown = unknown`
- `false and unknown = false`
- `not unknown = unknown`

SQL 查询中，`where` 条件为 `unknown` 时按 false 处理。

---

## 19. Aggregate Functions

聚合函数作用在一列值的 multiset 上，返回一个值。

常见：

- `avg`；
- `min`；
- `max`；
- `sum`；
- `count`。

例子：

```sql
select avg(salary)
from instructor
where dept_name = 'Comp. Sci.';
```

统计教过 Spring 2010 课程的教师数：

```sql
select count(distinct ID)
from teaches
where semester = 'Spring' and year = 2010;
```

统计表中 tuple 数：

```sql
select count(*)
from course;
```

---

## 20. Group By and Having

按院系统计平均工资：

```sql
select dept_name, avg(salary)
from instructor
group by dept_name;
```

规则：

> select 中未被聚合函数包裹的属性，必须出现在 group by 中。

错误例子：

```sql
select dept_name, ID, avg(salary)
from instructor
group by dept_name;
```

因为 `ID` 不在 group by 中，也不是聚合结果。

### Having

`where` 在分组前过滤 tuple；`having` 在分组后过滤 group。

```sql
select dept_name, avg(salary)
from instructor
group by dept_name
having avg(salary) > 42000;
```

完整执行直觉：

```text
from -> where -> group by -> having -> select -> order by -> limit
```

---

## 21. Null and Aggregates

除 `count(*)` 外，聚合函数通常忽略 null。

如果所有值都是 null：

- `count` 返回 0；
- `sum`、`avg`、`min`、`max` 返回 null。

---

## 22. Nested Subqueries

subquery 是嵌套在另一个查询中的 `select-from-where`。

常用于：

- set membership；
- set comparison；
- set cardinality；
- emptiness test。

---

## 23. Set Membership: in / not in

找 Fall 2009 和 Spring 2010 都开设的课程：

```sql
select distinct course_id
from section
where semester = 'Fall' and year = 2009
  and course_id in (
    select course_id
    from section
    where semester = 'Spring' and year = 2010
  );
```

找 Fall 2009 开设但 Spring 2010 未开设的课程：

```sql
select distinct course_id
from section
where semester = 'Fall' and year = 2009
  and course_id not in (
    select course_id
    from section
    where semester = 'Spring' and year = 2010
  );
```

tuple membership：

```sql
select count(distinct ID)
from takes
where (course_id, sec_id, semester, year) in (
  select course_id, sec_id, semester, year
  from teaches
  where teaches.ID = '10101'
);
```

---

## 24. Set Comparison: some / all

找工资高于 Biology 系某位教师的教师：

```sql
select name
from instructor
where salary > some (
  select salary
  from instructor
  where dept_name = 'Biology'
);
```

`> some` 表示大于集合中至少一个值。

找工资高于 Biology 系所有教师的教师：

```sql
select name
from instructor
where salary > all (
  select salary
  from instructor
  where dept_name = 'Biology'
);
```

`> all` 表示大于集合中每一个值。

---

## 25. Scalar Subquery

scalar subquery 用在需要单个值的位置。

例子：

```sql
select name
from instructor
where salary * 10 > (
  select budget
  from department
  where department.dept_name = instructor.dept_name
);
```

如果 scalar subquery 返回多行，会产生运行时错误。

---

## 26. exists / not exists

`exists` 判断子查询结果是否非空：

$$
exists(r) \Leftrightarrow r \ne \emptyset
$$

`not exists` 判断子查询是否为空：

$$
not\ exists(r) \Leftrightarrow r = \emptyset
$$

相关子查询例子：

```sql
select course_id
from section as S
where semester = 'Fall' and year = 2009
  and exists (
    select *
    from section as T
    where semester = 'Spring'
      and year = 2010
      and S.course_id = T.course_id
  );
```

---

## 27. 用 not exists 表达“所有”

找修过 Biology 系所有课程的学生：

```sql
select distinct S.ID, S.name
from student as S
where not exists (
  (select course_id
   from course
   where dept_name = 'Biology')
  except
  (select T.course_id
   from takes as T
   where S.ID = T.ID)
);
```

核心逻辑：

$$
X - Y = \emptyset \Leftrightarrow X \subseteq Y
$$

也就是：

> 不存在一门 Biology 课程是该学生没修过的。

---

## 28. unique

`unique` 判断子查询结果是否没有重复 tuple。

空集上 `unique` 为 true。

找 2009 年最多开设一次的课程：

```sql
select T.course_id
from course as T
where unique (
  select R.course_id
  from section as R
  where T.course_id = R.course_id
    and R.year = 2009
);
```

如果要求“恰好一次”，还要加 `exists`。

---

## 29. With Clause

`with` 定义只在当前查询中可用的临时 view。

找预算最大的院系：

```sql
with max_budget(value) as (
  select max(budget)
  from department
)
select dept_name
from department, max_budget
where department.budget = max_budget.value;
```

复杂查询中可以定义多个临时关系：

```sql
with dept_total(dept_name, value) as (
  select dept_name, sum(salary)
  from instructor
  group by dept_name
),
dept_total_avg(value) as (
  select avg(value)
  from dept_total
)
select dept_name
from dept_total, dept_total_avg
where dept_total.value >= dept_total_avg.value;
```

---

## 30. Modification of the Database

SQL 修改包括：

- delete；
- insert；
- update。

### 30.1 Delete

删除所有教师：

```sql
delete from instructor;
```

删除 Finance 系教师：

```sql
delete from instructor
where dept_name = 'Finance';
```

使用子查询删除：

```sql
delete from instructor
where dept_name in (
  select dept_name
  from department
  where building = 'Watson'
);
```

注意：SQL 会先确定要删除的 tuple 集合，再执行删除，不会边删边重新计算条件。

### 30.2 Insert

插入完整 tuple：

```sql
insert into course
values ('CS-437', 'Database Systems', 'Comp. Sci.', 4);
```

指定属性插入：

```sql
insert into course(course_id, title, dept_name, credits)
values ('CS-437', 'Database Systems', 'Comp. Sci.', 4);
```

从查询结果插入：

```sql
insert into student
select ID, name, dept_name, 0
from instructor;
```

SQL 会先完整执行 `select`，再插入结果。

### 30.3 Update

更新工资：

```sql
update instructor
set salary = salary * 1.03
where salary > 100000;
```

条件更新更适合用 `case`：

```sql
update instructor
set salary = case
  when salary <= 100000 then salary * 1.05
  else salary * 1.03
end;
```

使用 scalar subquery 更新：

```sql
update student S
set tot_cred = (
  select sum(credits)
  from takes natural join course
  where S.ID = takes.ID
    and takes.grade <> 'F'
    and takes.grade is not null
);
```

如果没有修过课，`sum` 可能返回 null，可用 `case` 改成 0。

---

## 31. 本章速记

- `select-from-where` 分别对应 projection、Cartesian product、selection。
- SQL 默认允许重复，去重用 `distinct`。
- `natural join` 方便但危险，要小心无关同名属性。
- `where` 先于 `group by`，`having` 后于 `group by`。
- 除 `count(*)` 外，聚合函数忽略 null。
- `in` 判断属于集合，`some/all` 做集合比较，`exists` 判断非空。
- `not exists (X except Y)` 是表达“所有”的经典模板。
- `with` 可以把复杂查询拆成临时 view。
- 修改数据库用 `insert`、`delete`、`update`，条件更新常用 `case`。

