# Lec5 Advanced SQL

本章关注 SQL 与程序语言的结合、函数/过程、触发器、递归查询，以及 OLAP 扩展聚合。

---

## 1. 为什么需要从程序语言访问 SQL

单独使用 SQL 不足以完成所有数据库应用，因为：

- SQL 不是通用编程语言，表达能力有限；
- 用户交互、图形界面、打印报表、网络通信等非声明式操作不能只靠 SQL 完成；
- 复杂业务逻辑通常需要宿主语言配合。

从通用编程语言访问数据库有两类方式：

- API：程序通过函数库连接数据库、发送 SQL、读取结果；
- Embedded SQL：把 SQL 嵌入宿主语言，编译时由预处理器转换为函数调用。

---

## 2. JDBC and ODBC

JDBC 和 ODBC 都是应用程序访问数据库的 API。

| 接口 | 主要语言 |
| --- | --- |
| JDBC | Java |
| ODBC | C / C++ / C# 等 |

应用程序通常要做：

1. 打开数据库连接；
2. 发送 SQL 命令；
3. 一条条取回查询结果；
4. 处理异常；
5. 关闭资源。

---

## 3. JDBC

JDBC 是 Java 访问 SQL 数据库的 API。

基本流程：

```text
open connection
create Statement
execute query/update
fetch ResultSet
close Statement
close Connection
```

连接示意：

```java
Connection conn = DriverManager.getConnection(url, userid, passwd);
Statement stmt = conn.createStatement();
...
stmt.close();
conn.close();
```

### 3.1 Execute Update

用于执行 insert、delete、update、DDL 等不返回结果集的语句：

```java
stmt.executeUpdate(
  "insert into instructor values('77987', 'Kim', 'Physics', 98000)"
);
```

### 3.2 Execute Query

用于执行 select，返回 `ResultSet`：

```java
ResultSet rset = stmt.executeQuery(
  "select dept_name, avg(salary) from instructor group by dept_name"
);

while (rset.next()) {
  System.out.println(rset.getString("dept_name") + " " + rset.getFloat(2));
}
```

读取列可以按列名，也可以按列号：

```java
rset.getString("dept_name")
rset.getString(1)
```

### 3.3 Null Values in JDBC

如果用 `getInt()` 等方法读取 null，返回值本身可能无法区分 null。

需要调用：

```java
rset.wasNull()
```

判断刚刚读取的值是否为 SQL null。

---

## 4. Prepared Statement

prepared statement 先把 SQL 模板发送给数据库编译，之后多次绑定参数执行。

```java
PreparedStatement pStmt = conn.prepareStatement(
  "insert into instructor values(?,?,?,?)"
);

pStmt.setString(1, "88877");
pStmt.setString(2, "Perry");
pStmt.setString(3, "Finance");
pStmt.setInt(4, 125000);
pStmt.executeUpdate();
```

优点：

- 避免重复编译；
- 参数绑定更安全；
- 防止 SQL injection；
- 自动处理字符串转义等问题。

---

## 5. SQL Injection

危险写法：

```java
"select * from instructor where name = '" + name + "'"
```

如果用户输入：

```text
X' or 'Y' = 'Y
```

拼接后变成：

```sql
select * from instructor where name = 'X' or 'Y' = 'Y'
```

条件永真，可能泄露所有数据。

更恶意的输入还可能包含 update/delete。

原则：

> 只要用户输入会进入 SQL，就应使用 prepared statement，而不是字符串拼接。

---

## 6. Metadata Features

### 6.1 ResultSet Metadata

查询结果的元数据：

```java
ResultSetMetaData rsmd = rs.getMetaData();
for (int i = 1; i <= rsmd.getColumnCount(); i++) {
  System.out.println(rsmd.getColumnName(i));
  System.out.println(rsmd.getColumnTypeName(i));
}
```

可以获得：

- 列数；
- 列名；
- 列类型。

### 6.2 Database Metadata

数据库级元数据：

```java
DatabaseMetaData dbmd = conn.getMetaData();
ResultSet rs = dbmd.getColumns(null, "univdb", "department", "%");
```

可以查询：

- 数据库中有哪些表；
- 某表有哪些列；
- 列名和类型；
- 约束、索引等信息。

---

## 7. Transaction Control in JDBC

JDBC 默认通常是 auto-commit：

> 每条 SQL 语句单独作为一个事务并自动提交。

多步更新时这很危险。

关闭自动提交：

```java
conn.setAutoCommit(false);
```

显式提交或回滚：

```java
conn.commit();
conn.rollback();
```

重新开启：

```java
conn.setAutoCommit(true);
```

---

## 8. SQLJ

SQLJ 是 Java 中的 embedded SQL。

相比 JDBC：

- JDBC 更动态，很多 SQL 错误只能运行时发现；
- SQLJ 把 SQL 嵌入 Java，部分错误可以编译期发现。

示意：

```java
#sql iterator deptInfoIter(String dept_name, int avgSal);
```

SQLJ 现在实际使用不如 JDBC 普遍，但它体现了 embedded SQL 的思想。

---

## 9. ODBC

ODBC 是 Open Database Connectivity，是应用程序访问数据库的标准 API。

基本流程：

1. 分配 SQL environment；
2. 分配 database connection handle；
3. 调用 `SQLConnect()` 连接数据库；
4. 调用 `SQLExecDirect()` 发送 SQL；
5. 调用 `SQLFetch()` 逐行取结果；
6. 调用 `SQLBindCol()` 把结果列绑定到 C 变量；
7. 释放 statement、connection、environment。

ODBC 每个数据库系统通常提供自己的 driver library。

---

## 10. ODBC Prepared Statements

prepared statement 在 ODBC 中也存在。

准备语句：

```c
SQLPrepare(stmt, sqlString);
```

绑定参数：

```c
SQLBindParameter(stmt, parameterNumber, ...);
```

执行：

```c
SQLExecute(stmt);
```

---

## 11. Embedded SQL

embedded SQL 把 SQL 语句嵌入宿主语言。

宿主语言可以是：

- C；
- C++；
- Java；
- Fortran；
- PL/1 等。

基本形式：

```c
EXEC SQL <embedded SQL statement>;
```

Java 中常见形式：

```java
#sql { ... };
```

### 11.1 Host Variables

宿主语言变量在 SQL 中用冒号区分：

```sql
:credit_amount
```

变量需要声明在 declare section 中：

```sql
EXEC SQL BEGIN DECLARE SECTION;
  int credit_amount;
EXEC SQL END DECLARE SECTION;
```

### 11.2 Cursor

嵌入式 SQL 查询通常使用 cursor。

声明：

```sql
EXEC SQL
declare c cursor for
select ID, name
from student
where tot_cred > :credit_amount;
```

打开：

```sql
EXEC SQL open c;
```

取一行：

```sql
EXEC SQL fetch c into :si, :sn;
```

关闭：

```sql
EXEC SQL close c;
```

当没有更多数据时，`SQLSTATE` 被设为 `'02000'`。

### 11.3 Updates Through Cursor

可以声明 cursor 可更新：

```sql
EXEC SQL
declare c cursor for
select *
from instructor
where dept_name = 'Music'
for update;
```

然后对当前 cursor 指向的 tuple 更新：

```sql
update instructor
set salary = salary + 1000
where current of c;
```

---

## 12. Procedural Extensions and Stored Procedures

SQL 提供过程化扩展：

- if-then-else；
- while；
- repeat；
- for loop；
- local variables。

stored procedure 可以存储在数据库中，通过 `call` 执行。

好处：

- 把业务逻辑放在数据库端；
- 外部应用不用知道内部细节；
- 可减少网络交互；
- 便于统一管理逻辑。

---

## 13. SQL Functions

函数返回一个值。

例子：给定院系名，返回该院系教师数：

```sql
create function dept_count(dept_name varchar(20))
returns integer
begin
  declare d_count integer;
  select count(*) into d_count
  from instructor
  where instructor.dept_name = dept_count.dept_name;
  return d_count;
end;
```

使用：

```sql
select dept_name, budget
from department
where dept_count(dept_name) > 12;
```

---

## 14. Table Functions

SQL:2003 支持返回 relation 的函数。

例子：

```sql
create function instructors_of(dept_name char(20))
returns table (
  ID varchar(5),
  name varchar(20),
  dept_name varchar(20),
  salary numeric(8,2)
)
return table (
  select ID, name, dept_name, salary
  from instructor
  where instructor.dept_name = instructors_of.dept_name
);
```

使用：

```sql
select *
from table(instructors_of('Music'));
```

---

## 15. SQL Procedures

procedure 不一定直接返回值，可以有 `in` / `out` 参数。

```sql
create procedure dept_count_proc(
  in dept_name varchar(20),
  out d_count integer
)
begin
  select count(*) into d_count
  from instructor
  where instructor.dept_name = dept_count_proc.dept_name;
end;
```

调用：

```sql
declare d_count integer;
call dept_count_proc('Physics', d_count);
```

---

## 16. Procedural Constructs

### 16.1 Compound Statement

```sql
begin
  ...
end
```

可以包含多条 SQL 和局部变量声明。

### 16.2 While

```sql
declare n integer default 0;
while n < 10 do
  set n = n + 1;
end while;
```

### 16.3 Repeat

```sql
repeat
  set n = n - 1;
until n = 0
end repeat;
```

### 16.4 For Loop

```sql
for r as
  select budget from department
  where dept_name = 'Music'
do
  set n = n - r.budget;
end for;
```

### 16.5 If

```sql
if boolean_expression then
  statement;
elseif boolean_expression then
  statement;
else
  statement;
end if;
```

不同 DBMS 的语法差异很大，实际使用要查系统手册。

---

## 17. External Language Functions / Procedures

SQL:1999 允许用 C、C++、Java 等外部语言写函数/过程。

示意：

```sql
create procedure dept_count_proc(
  in dept_name varchar(20),
  out count integer
)
language C
external name '/usr/avi/bin/dept_count_proc';
```

优点：

- 表达能力更强；
- 某些操作效率更高。

缺点：

- 安全风险；
- 可能破坏数据库进程内存；
- 可能访问未授权数据。

安全策略：

- 使用 sandbox，比如 Java；
- 在独立进程中运行外部代码，通过进程间通信传参和返回结果；
- 直接在数据库进程内运行仅适合极度重视性能且信任代码的场景。

---

## 18. Triggers

trigger 是数据库发生某种修改时自动执行的语句。

它可看作 ECA rule：

- Event：insert / delete / update；
- Condition：满足什么条件；
- Action：执行什么动作。

设计 trigger 要说明：

1. 什么时候触发；
2. 触发后做什么。

---

## 19. Trigger Example

账户余额大额变化时记录日志：

```sql
create trigger account_trigger
after update of account on balance
referencing new row as nrow
referencing old row as orow
for each row
when nrow.balance - orow.balance >= 200000
  or orow.balance - nrow.balance >= 50000
begin
  insert into account_log
  values (nrow.account_number,
          nrow.balance - orow.balance,
          current_time());
end;
```

---

## 20. Triggers for Constraints

有些完整性约束不能用普通 foreign key 或 check 表达，可以用 trigger。

例如 `time_slot_id` 不是 `time_slot` 的主键，不能直接建 foreign key，可以用 trigger 检查 section 插入是否合法。

插入 section 后检查：

```sql
create trigger timeslot_check1
after insert on section
referencing new row as nrow
for each row
when (nrow.time_slot_id not in (
  select time_slot_id from time_slot
))
begin
  rollback;
end;
```

删除 time_slot 后检查是否仍被 section 引用，也可以用 trigger。

---

## 21. Triggering Events and Actions

trigger event 可以是：

- insert；
- delete；
- update。

update trigger 可以限制到特定属性：

```sql
after update of takes on grade
```

可以引用：

- `old row`：delete / update 前的 tuple；
- `new row`：insert / update 后的 tuple。

trigger 可以在事件前触发：

```sql
before update
```

例如把空白 grade 转成 null。

---

## 22. Row-Level and Statement-Level Triggers

### 22.1 Row-Level Trigger

每影响一行执行一次：

```sql
for each row
```

适合逐行检查或逐行维护。

### 22.2 Statement-Level Trigger

整个 SQL 语句执行一次：

```sql
for each statement
```

可使用 transition table：

- `old table`；
- `new table`。

对一次更新大量行的语句，statement-level trigger 更高效。

---

## 23. When Not To Use Triggers

trigger 不应滥用。

过去常用 trigger 做：

- 维护 summary data；
- 复制数据库时记录变化。

现在更推荐：

- materialized view；
- 数据库内建 replication；
- 封装方法或过程。

trigger 风险：

- 备份恢复或复制时意外触发；
- 错误 trigger 导致关键事务失败；
- cascading execution，触发器连锁触发，难以调试。

---

## 24. Recursive Queries

SQL:1999 支持 recursive view definition。

典型用途：transitive closure。

例如找直接或间接先修课：

```sql
with recursive rec_prereq(course_id, prereq_id) as (
  select course_id, prereq_id
  from prereq
  union
  select rec_prereq.course_id, prereq.prereq_id
  from rec_prereq, prereq
  where rec_prereq.prereq_id = prereq.course_id
)
select *
from rec_prereq;
```

这会不断扩展：

```text
直接先修
先修的先修
先修的先修的先修
...
```

直到没有新结果。

---

## 25. The Power of Recursion

没有 recursion 或 iteration 的 SQL 查询只能写固定层数的 join。

例如你可以手写：

- 直接先修；
- 两层先修；
- 三层先修；

但无法用固定查询表达任意深度。

recursive query 可以表达 transitive closure，这是非递归 SQL 做不到的。

另一个例子：

```text
manager(employee_name, manager_name)
```

找直接或间接上级，也需要递归。

---

## 26. OLAP and Multidimensional Data

OLAP（Online Analytical Processing）用于在线交互式数据分析。

特点：

- 快速汇总；
- 多角度查看；
- 支持聚合和下钻。

多维数据包含：

- **measure attributes**：可度量、可聚合的值，例如 sales 数量；
- **dimension attributes**：观察 measure 的维度，例如 item、color、size、time、region。

---

## 27. Cross Tab and Data Cube

cross-tab / pivot table 用一个维度作为行，另一个维度作为列，单元格中放聚合值。

data cube 是 cross-tab 的多维推广。

如果有三个维度：

```text
item_name, color, size
```

就可以从不同维度组合查看销售数量。

---

## 28. Hierarchy, Rollup and Drill Down

维度可以有层次。

例如：

```text
item -> category
city -> province -> country
day -> month -> year
```

操作：

- **rollup**：从细粒度汇总到粗粒度；
- **drill down**：从粗粒度展开到细粒度；
- **slicing**：固定某个维度值；
- **dicing**：固定多个维度值；
- **pivoting**：改变 cross-tab 中使用的行列维度。

---

## 29. Cube Operation

`cube` 计算指定属性所有子集上的 group by。

例子：

```sql
select item_name, color, size, sum(number)
from sales
group by cube(item_name, color, size);
```

如果有 3 个属性，会生成 $2^3=8$ 种 grouping：

```text
(item_name, color, size)
(item_name, color)
(item_name, size)
(color, size)
(item_name)
(color)
(size)
()
```

空 grouping `()` 表示总体汇总。

SQL 标准中，不在某个 grouping 中的属性通常用 null 表示。

---

## 30. Rollup Operation

`rollup` 只生成指定属性列表的前缀分组。

```sql
select item_name, color, size, sum(number)
from sales
group by rollup(item_name, color, size);
```

生成：

```text
(item_name, color, size)
(item_name, color)
(item_name)
()
```

rollup 适合层次汇总。

例如：

```sql
select category, item_name, sum(number)
from sales, itemcategory
where sales.item_name = itemcategory.item_name
group by rollup(category, item_name);
```

可以同时得到：

- 每个 item 的销售；
- 每个 category 的销售；
- 总销售。

---

## 31. Multiple Rollups and Cubes

一个 `group by` 中可以有多个 rollup / cube。

例如：

```sql
group by rollup(item_name), rollup(color, size)
```

产生的 grouping 是两个集合的笛卡尔积：

```text
{item_name, ()} × {(color, size), (color), ()}
```

即：

```text
(item_name, color, size)
(item_name, color)
(item_name)
(color, size)
(color)
()
```

---

## 32. Merge Statement

`merge` 用于批量处理更新。

例子：把 `funds_received` 中的存款批量加入 `account`。

```sql
merge into account as A
using (
  select *
  from funds_received as F
) on (A.account_number = F.account_number)
when matched then
  update set balance = balance + F.amount;
```

`merge` 更完整的形式还可以处理 not matched 时插入新 tuple。

---

## 33. 本章速记

- JDBC/ODBC 是程序访问数据库的 API。
- JDBC 基本流程：connection -> statement -> execute -> result set -> close。
- 用户输入进入 SQL 时必须使用 prepared statement，避免 SQL injection。
- metadata 可以查询结果列信息和数据库结构信息。
- auto-commit 对多步事务不安全，应显式 commit/rollback。
- embedded SQL 使用 host variables 和 cursor。
- stored procedures/functions 把业务逻辑放到数据库端。
- trigger 是 ECA rule：Event、Condition、Action。
- trigger 可用于复杂约束，但不应滥用，容易产生连锁和调试问题。
- recursive query 能表达 transitive closure。
- OLAP 使用 dimension 和 measure；cube 生成所有子集聚合，rollup 生成前缀层次聚合。

