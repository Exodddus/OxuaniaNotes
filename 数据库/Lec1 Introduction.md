# Lec1 Introduction

本章是数据库系统的总览：为什么需要 DBMS、数据库从哪些层次抽象数据、数据库语言和数据库引擎分别负责什么，以及不同用户如何与数据库系统交互。

一句话概括：

> 数据库系统的目标，是用方便、高效、安全、可靠的方式存储、共享和管理大量相关数据。

---

## 1. Database System 是什么

### 1.1 Database 与 DBMS

- **database**：关于某个 enterprise 的一组相互关联的数据。
- **DBMS（Database Management System）**：管理数据库的软件系统。
- **database system**：数据库、DBMS、应用程序、用户等组成的整体。

传统文件系统应用通常是：

```text
applications -> files
```

数据库系统应用通常是：

```text
applications -> DBMS -> database
```

DBMS 提供的核心能力：

- 定义数据的存储结构；
- 提供数据查询和修改机制；
- 保证数据安全；
- 在系统崩溃后恢复；
- 多用户共享时进行并发控制。

---

## 2. Database Applications

数据库应用几乎出现在所有信息系统中。

典型场景：

- 企业销售：customers、products、purchases；
- 财务会计：payments、receipts、assets；
- 人力资源：employees、salaries、payroll taxes；
- 制造业：production、inventory、orders、supply chain；
- 银行：customers、accounts、loans、transactions；
- 大学：students、instructors、courses、registration、grades；
- 航空：reservations、schedules；
- Web 服务：订单追踪、推荐系统、广告系统。

课件还强调了数据库与 Big Data / AI 的关系：

- Big Data 的典型特征：
  - Volume；
  - Variety；
  - Velocity；
  - Value。
- 数据驱动 AI，尤其是生成式 AI 和大语言模型，也依赖大规模数据管理能力。

---

## 3. 为什么不用普通文件系统

早期数据库应用直接建立在 file system 上，会产生很多问题。

### 3.1 Data Redundancy and Inconsistency

同一数据可能在多个文件中重复保存。

后果：

- 浪费空间；
- 更新时必须改多份；
- 如果某些文件更新了、某些没更新，就产生 inconsistency。

### 3.2 Data Isolation

数据分散在多个文件和格式中。

不同程序各自维护自己的数据格式，导致“数据孤岛”，跨文件访问困难。

### 3.3 Difficulty in Accessing Data

每产生一个新需求，就可能要写一个新程序。

例如临时想查“某类账户余额超过某值的客户”，文件系统下可能需要手写程序扫描文件；数据库则可以直接写查询。

### 3.4 Integrity Problems

完整性约束被埋在程序代码里，而不是显式声明在数据库中。

例如：

```text
account balance >= 1
```

如果这个约束散落在多个应用程序中，修改和维护都会很困难。

### 3.5 Atomicity Problems

原子性要求一组操作要么全部发生，要么全部不发生。

例如转账：

```text
A.balance = A.balance - 50
B.balance = B.balance + 50
```

如果第一步执行后系统崩溃，第二步没有执行，数据库就进入不一致状态。

数据库系统通过 transaction 和 recovery 机制保证 atomicity。

### 3.6 Concurrent Access Anomalies

多用户同时访问是必要的，但如果没有控制会产生异常。

例如两个事务同时读到账户余额 100，各自加 50 并写回：

```text
T1 read balance = 100
T2 read balance = 100
T1 write 150
T2 write 150
```

最终余额是 150，而不是 200。这是典型并发异常。

数据库系统通过 concurrency control 解决。

### 3.7 Security Problems

文件系统很难细粒度控制：

- 谁能读哪些数据；
- 谁能修改哪些表；
- 谁能创建表或索引；
- 如何审计操作。

数据库系统提供 authentication、privilege、audit 等机制。

---

## 4. 数据库的核心特征

数据库系统通常具有：

- **data persistence**：数据持久保存；
- **convenience in accessing data**：访问方便；
- **data integrity**：维护完整性约束；
- **concurrency control**：支持多用户并发；
- **failure recovery**：故障恢复；
- **security control**：安全控制。

---

## 5. View of Data：三层抽象

数据库通过三层抽象隐藏复杂性。

```text
view schema
   |
logical schema
   |
physical schema
```

### 5.1 Physical Schema

描述数据实际如何存储。

例如：

- 文件组织方式；
- 索引；
- block 布局；
- 存储位置。

这是最底层。

### 5.2 Logical Schema

描述数据库中保存什么数据，以及数据之间的关系。

例如：

```text
instructor(ID, name, dept_name, salary)
student(ID, name, dept_name, tot_cred)
```

逻辑模式是数据库设计最核心的一层。

### 5.3 View Schema

描述不同用户看到的数据视图。

例如普通用户可能只能看到：

```text
instructor(ID, name, dept_name)
```

看不到 `salary`。

---

## 6. Schema and Instance

### 6.1 Schema

schema 是数据库的结构设计，相对稳定。

例如：

```text
instructor(ID, name, dept_name, salary)
```

### 6.2 Instance

instance 是某一时刻数据库中的实际数据。

类比：

- schema 像变量类型；
- instance 像变量当前值。

---

## 7. Data Independence

数据独立性是三层抽象带来的重要好处。

### 7.1 Physical Data Independence

修改 physical schema，不影响 logical schema。

例如：

- 增加索引；
- 改变文件组织；
- 改变存储位置。

应用程序不需要跟着改。

### 7.2 Logical Data Independence

修改 logical schema，不影响 view schema。

例如把一个关系拆成两个关系后，可以通过 view 维持原来的用户接口。

逻辑数据独立性通常比物理数据独立性更难做到。

---

## 8. Data Models

数据模型用于描述：

- 数据；
- 数据之间的联系；
- 数据语义；
- 数据约束。

常见模型：

- relational model；
- entity-relationship model；
- object-based data model；
- semistructured data model；
- network / hierarchical model。

本课程核心使用 **relational model**。

关系模型中：

```text
table -> relation
row   -> tuple
column -> attribute
```

---

## 9. Database Languages

数据库语言主要分两类。

### 9.1 DDL

DDL（Data Definition Language）用于定义数据库结构。

例如：

```sql
create table instructor (
  ID char(5),
  name varchar(20),
  dept_name varchar(20),
  salary numeric(8,2),
  primary key (ID)
);
```

DDL 的结果通常存入 data dictionary / system catalog。

### 9.2 DML

DML（Data Manipulation Language）用于查询和修改数据。

包括：

- insert；
- delete；
- update；
- select。

DML 又可以分为：

- procedural：用户说明“怎么做”；
- declarative / nonprocedural：用户说明“要什么”，不说明“怎么做”。

SQL 是典型 declarative language。

---

## 10. SQL Query Language

典型 SQL 查询：

```sql
select name
from instructor
where dept_name = 'Comp. Sci.';
```

更复杂的例子：

```sql
select instructor.name, department.building
from instructor, department
where instructor.dept_name = department.dept_name;
```

SQL 只描述需要的结果，具体如何执行由 query processor 和 optimizer 决定。

---

## 11. Database Access from Application Program

应用程序通常不能只靠 SQL 完成全部工作，因为还需要：

- 用户交互；
- 图形界面；
- 报表输出；
- 网络通信；
- 复杂业务逻辑。

因此应用程序需要通过接口访问数据库。

常见方式：

- ODBC；
- JDBC；
- embedded SQL；
- ORM 框架。

---

## 12. Database Engine

数据库引擎大致包含：

- storage manager；
- query processor；
- transaction manager。

### 12.1 Storage Manager

storage manager 是数据库和底层文件系统之间的接口。

负责：

- 数据文件管理；
- buffer management；
- authorization and integrity manager；
- transaction manager；
- file manager；
- index manager。

相关数据结构：

- data files；
- data dictionary；
- indices。

### 12.2 Query Processor

query processor 负责：

- DDL interpretation；
- DML compilation；
- query optimization；
- query evaluation。

SQL 查询一般会经历：

```text
SQL -> relational algebra -> evaluation plan -> execution
```

### 12.3 Transaction Management

transaction manager 保证：

- atomicity；
- consistency；
- isolation；
- durability。

这些统称为 ACID。

---

## 13. Database Users

不同用户以不同方式与数据库交互。

### 13.1 Naive Users

通过预定义应用程序访问数据库。

例如银行柜员、订票系统用户。

### 13.2 Application Programmers

编写应用程序，通过 SQL/API 访问数据库。

### 13.3 Sophisticated Users / Data Analysts

直接使用查询语言和分析工具访问数据库。

### 13.4 Database Administrators

DBA 负责数据库整体管理。

职责包括：

- schema 定义；
- storage structure 和 access method 定义；
- 权限管理；
- integrity constraints 管理；
- 性能监控；
- backup and recovery；
- 数据库调优和扩容。

---

## 14. History of Database Systems

数据库系统发展大致经历：

- 1950s-1960s：文件处理系统；
- 1960s-1970s：层次模型、网状模型；
- 1970s：关系模型提出，Ted Codd 获图灵奖；
- 1980s：关系数据库商业化；
- 1990s：SQL 标准化、对象关系数据库；
- 2000s 以后：Web 数据、大数据、NoSQL、云数据库、分布式数据库；
- 现在：数据库与 AI、分析系统、云原生系统深度结合。

---

## 15. 本章速记

- DBMS 用来方便、高效、安全、可靠地存储和访问数据。
- 文件系统的问题包括冗余、不一致、数据孤立、访问困难、完整性难维护、原子性问题、并发异常和安全问题。
- 数据库有三层抽象：physical、logical、view。
- schema 是结构，instance 是某时刻的数据。
- data independence 分为 physical 和 logical。
- SQL 包含 DDL 和 DML。
- 数据库引擎主要包括 storage manager、query processor、transaction manager。
- DBA 负责权限、存储、备份、恢复、调优等管理工作。

