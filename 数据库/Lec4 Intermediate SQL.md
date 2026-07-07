# Lec4 Intermediate SQL

本章在基础 SQL 之上，补充 join 语法、更多数据类型与约束、view、index、transaction、authorization 等内容。

---

## 1. Joined Relations

join 操作接受两个 relation，返回一个新 relation。

在 SQL 中，join 通常出现在 `from` 子句中。

一个 join 需要区分两件事：

- **join condition**：哪些 tuple 能匹配，以及结果中保留哪些属性；
- **join type**：对于没有匹配的 tuple 如何处理。

常见连接：

- natural join；
- inner join；
- outer join；
- join ... on；
- join ... using。

---

## 2. Inner Join

inner join 只保留满足连接条件的 tuple。

例子：

```sql
select *
from course inner join prereq
  on course.course_id = prereq.course_id;
```

等价于基于条件的普通连接。

如果用 natural join：

```sql
course natural join prereq
```

会自动按所有同名属性连接，并只保留一份同名属性。

---

## 3. Outer Join

outer join 是普通 join 的扩展，用于避免信息丢失。

它先做 join，然后把没有匹配的 tuple 也加入结果，并用 null 填充另一侧属性。

### 3.1 Left Outer Join

保留左表中所有 tuple。

```sql
course natural left outer join prereq;
```

如果某门课没有 prereq，也会出现在结果中，prereq 相关属性为 null。

### 3.2 Right Outer Join

保留右表中所有 tuple。

```sql
course natural right outer join prereq;
```

可以用来发现右表中没有对应左表记录的数据。

### 3.3 Full Outer Join

保留两边所有 tuple。

```sql
course natural full outer join prereq;
```

未匹配的一侧用 null 补齐。

---

## 4. Join Conditions: on 与 using

### 4.1 join ... on

显式指定连接条件：

```sql
select *
from course join prereq
  on course.course_id = prereq.course_id;
```

`on` 更清楚，不会因为无关同名属性产生误连接。

### 4.2 join ... using

当连接属性同名时，可以写：

```sql
select *
from course full outer join prereq using (course_id);
```

`using(course_id)` 表示只用该同名属性连接，并只保留一份该属性。

---

## 5. SQL Data Types and Schemas

### 5.1 Date and Time Types

常见类型：

- `date`；
- `time`；
- `timestamp`；
- `interval`。

例子：

```sql
date '2005-7-27'
time '09:00:30.75'
timestamp '2005-7-27 09:00:30.75'
interval '1' day
```

常见函数：

```sql
current_date()
current_time()
year(x), month(x), day(x)
hour(x), minute(x), second(x)
```

### 5.2 User-Defined Types

自定义类型：

```sql
create type Dollars as numeric(12,2) final;
```

使用：

```sql
create table department (
  dept_name varchar(20),
  building varchar(15),
  budget Dollars
);
```

### 5.3 Domains

domain 是带约束的用户定义域。

```sql
create domain person_name char(20) not null;
```

带 check 的 domain：

```sql
create domain degree_level varchar(10)
constraint degree_level_test
check (value in ('Bachelors', 'Masters', 'Doctorate'));
```

type 和 domain 类似，但 domain 可以直接附加约束。

### 5.4 Large Object Types

大对象用于存储照片、视频、CAD 文件等。

- `blob`：binary large object；
- `clob`：character large object。

查询返回大对象时，系统通常返回 pointer，而不是把整个对象直接返回。

---

## 6. Integrity Constraints

完整性约束用于防止授权修改破坏数据一致性。

单表约束包括：

- `not null`；
- `primary key`；
- `unique`；
- `check(P)`；
- `foreign key`。

---

## 7. Not Null and Unique

### 7.1 not null

```sql
name varchar(20) not null
budget numeric(12,2) not null
```

表示该属性不能取 null。

### 7.2 unique

```sql
unique(A1, A2, ..., Am)
```

表示这些属性构成 superkey。

注意：

> unique 允许 null，而 primary key 不允许 null。

---

## 8. Check Clause

`check(P)` 要求每个 tuple 满足谓词 $P$。

例子：

```sql
create table section (
  course_id varchar(8),
  sec_id varchar(8),
  semester varchar(6),
  year numeric(4,0),
  building varchar(15),
  room_number varchar(7),
  time_slot_id varchar(4),
  primary key(course_id, sec_id, semester, year),
  check (semester in ('Fall', 'Winter', 'Spring', 'Summer'))
);
```

---

## 9. Referential Integrity and Cascading Actions

referential integrity 要求外键值必须出现在被引用关系中。

例如 `course.dept_name` 引用 `department.dept_name`：

```sql
foreign key (dept_name) references department
```

级联动作：

```sql
on delete cascade
on update cascade
```

其他动作：

- `set null`；
- `set default`；
- `restrict`；
- `no action`。

---

## 10. Constraint Checking During Transactions

有些约束在单条语句后可能暂时违反，但整个事务结束后可以满足。

例如：

```sql
create table person (
  ID char(10),
  name char(40),
  mother char(10),
  father char(10),
  primary key(ID),
  foreign key(father) references person,
  foreign key(mother) references person
);
```

插入一个人时，可能父母记录还没插入。

解决思路：

- 先插入父母；
- 先把 father/mother 设为 null，之后 update；
- defer constraint checking 到 transaction end。

---

## 11. Complex Check Clauses and Assertions

理论上可以写复杂 check，例如：

```sql
check (time_slot_id in (
  select time_slot_id from time_slot
))
```

但现实中，多数数据库不支持 check 中含 subquery。

替代方式：

- trigger；
- assertion。

assertion 语法：

```sql
create assertion assertion_name check predicate;
```

例如约束 `student.tot_cred` 必须等于已通过课程 credits 之和，可以用 assertion 表达。

实际系统中 assertion 支持也并不普遍。

---

## 12. Views

view 是虚拟关系，不一定实际存储数据。

用途：

- 隐藏某些数据；
- 简化复杂查询；
- 提供逻辑数据独立性；
- 控制用户可见内容。

例子：隐藏教师工资：

```sql
create view faculty as
select ID, name, dept_name
from instructor;
```

使用 view：

```sql
select name
from faculty
where dept_name = 'Biology';
```

### 12.1 View Definition

格式：

```sql
create view v as <query expression>;
```

view definition 保存的是查询表达式，不是立即计算出的结果表。

使用 view 时，系统会做 view expansion，把 view 替换为其定义。

---

## 13. Views Defined Using Other Views

view 可以基于其他 view 定义。

```sql
create view physics_fall_2009 as
select course.course_id, sec_id, building, room_number
from course, section
where course.course_id = section.course_id
  and course.dept_name = 'Physics'
  and section.semester = 'Fall'
  and section.year = 2009;
```

再定义：

```sql
create view physics_fall_2009_watson as
select course_id, room_number
from physics_fall_2009
where building = 'Watson';
```

查询时可以层层展开。

---

## 14. Update of a View

简单 view 有时可以更新。

例如：

```sql
create view faculty as
select ID, name, dept_name
from instructor;
```

插入：

```sql
insert into faculty values ('30765', 'Green', 'Music');
```

可转换为：

```sql
insert into instructor
values ('30765', 'Green', 'Music', null);
```

### 14.1 不能唯一翻译的更新

如果 view 来自 join，更新可能不唯一。

```sql
create view instructor_info as
select ID, name, building
from instructor, department
where instructor.dept_name = department.dept_name;
```

如果插入：

```sql
insert into instructor_info values ('69987', 'White', 'Taylor');
```

如果 Taylor 楼里有多个 department，就无法判断该教师属于哪个 department。

### 14.2 Updatable View 常见条件

多数数据库只允许简单 view 更新：

- `from` 中只有一个 base relation；
- `select` 只包含属性名，没有表达式、聚合或 distinct；
- 未出现在 view 中的属性可以设为 null；
- 没有 `group by` 或 `having`。

---

## 15. Materialized Views

materialized view 把 view 查询结果实际存成物理表。

优点：

- 查询快；
- 适合复杂聚合和分析。

缺点：

- base relations 更新后，materialized view 可能过期；
- 需要维护 view。

例子：

```sql
create materialized view departments_total_salary(dept_name, total_salary) as
select dept_name, sum(salary)
from instructor
group by dept_name;
```

---

## 16. View and Logical Data Independence

如果把：

```text
S(a,b,c)
```

拆成：

```text
S1(a,b)
S2(a,c)
```

为了让旧查询仍能使用 `S`，可以创建 view：

```sql
create view S(a,b,c) as
select a,b,c
from S1 natural join S2;
```

这样用户仍然可以写：

```sql
select *
from S
where ...;
```

这就是 view 支持 logical data independence 的例子。

---

## 17. Indexes

index 是加速访问的数据结构。

创建索引：

```sql
create index studentID_index on student(ID);
```

索引用于快速定位具有特定属性值的 records。

例如：

```sql
select *
from student
where ID = '12345';
```

如果 `ID` 上有索引，系统不必扫描整个 student 表。

注意：

- 索引加速查询；
- 但插入、删除、更新时需要维护索引；
- 索引会占用额外空间。

---

## 18. Transactions

transaction 是一个工作单元，要求：

> ALL or NONE。

事务可以：

- 全部成功并 commit；
- 失败后 rollback，好像从未发生。

事务通常隐式开始，通过以下语句结束：

```sql
commit work;
rollback work;
```

多数数据库默认每条 SQL 自动提交。

关闭自动提交：

```sql
SET AUTOCOMMIT = 0;
```

转账例子：

```sql
UPDATE account SET balance = balance - 100 WHERE ano = '1001';
UPDATE account SET balance = balance + 100 WHERE ano = '1002';
COMMIT;
```

---

## 19. ACID Properties

事务系统必须保证 ACID。

### 19.1 Atomicity

事务的所有操作要么全部反映到数据库，要么都不反映。

### 19.2 Consistency

事务隔离执行时，应把数据库从一个一致状态带到另一个一致状态。

### 19.3 Isolation

多个事务并发执行时，每个事务不应看到其他事务的中间结果。

对任意两个事务 $T_i,T_j$，在 $T_i$ 看来，$T_j$ 要么在它开始前完成，要么在它结束后开始。

### 19.4 Durability

事务一旦成功提交，即使系统故障，其修改也必须持久保存。

---

## 20. Authorization

authorization 控制用户能对数据库做什么。

### 20.1 数据权限

- `select`：允许读；
- `insert`：允许插入；
- `update`：允许修改；
- `delete`：允许删除。

### 20.2 模式权限

- `resources/create`：创建新关系；
- `alteration`：增删属性；
- `drop`：删除关系；
- `index`：创建或删除索引；
- `create view`：创建 view。

---

## 21. Grant

授权语句：

```sql
grant <privilege list>
on <relation name or view name>
to <user list>;
```

例子：

```sql
grant select on instructor to U1, U2, U3;
grant select on department to public;
grant update(budget) on department to U1, U2;
grant all privileges on department to U1;
```

注意：

> 对 view 授权，不自动意味着对 underlying relations 授权。

---

## 22. Revoke

撤销权限：

```sql
revoke <privilege list>
on <relation name or view name>
from <user list>;
```

例子：

```sql
revoke select on branch from U1, U2, U3;
```

如果某权限依赖被撤销的权限，也可能级联撤销。

---

## 23. Roles

role 是权限集合，可以授予用户，也可以授予其他 role。

```sql
create role instructor;
grant instructor to Amit;
grant select on takes to instructor;
```

role 可以继承：

```sql
create role teaching_assistant;
grant teaching_assistant to instructor;
```

这样 instructor role 拥有 teaching_assistant 的权限。

---

## 24. Authorization on Views

可以只给用户某个 view 的权限，而不给 base table 权限。

```sql
create view geo_instructor as
select *
from instructor
where dept_name = 'Geology';

grant select on geo_instructor to geo_staff;
```

这可以实现行级/列级访问控制。

---

## 25. Other Authorization Features

### 25.1 references privilege

创建 foreign key 可能需要 referenced relation 上的 references 权限。

```sql
grant references(dept_name) on department to Mariano;
```

原因是外键会限制 referenced relation 的删除和更新。

### 25.2 with grant option

允许用户把权限继续授予别人：

```sql
grant select on department to Amit with grant option;
```

撤销时可以选择：

```sql
revoke select on department from Amit cascade;
revoke select on department from Amit restrict;
```

- `cascade`：连带撤销依赖该授权产生的权限；
- `restrict`：如果存在依赖授权，则拒绝撤销。

---

## 26. 本章速记

- `inner join` 只保留匹配 tuple；outer join 会保留未匹配 tuple 并补 null。
- `on` 显式写条件，`using` 指定同名属性连接，natural join 自动匹配所有同名属性。
- `unique` 声明 superkey，但允许 null；primary key 不允许 null。
- 复杂 check 和 assertion 理论上强大，但现实 DBMS 支持有限，常用 trigger 替代。
- view 是虚拟关系，保存查询表达式；materialized view 保存实际结果。
- 简单 view 可更新，join/聚合 view 通常不可唯一更新。
- index 加速查询但增加维护成本。
- transaction 以 commit/rollback 结束，目标是 ACID。
- grant/revoke/role/view authorization 构成 SQL 权限管理基础。

