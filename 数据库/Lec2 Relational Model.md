# Lec2 Relational Model

关系模型是本课程的核心数据模型。它把数据表示为关系，也就是二维表；用关系代数描述查询；用键和完整性约束描述数据语义。

---

## 1. Relation 的基本结构

关系数据库中：

```text
relation  -> table
tuple     -> row
attribute -> column
domain    -> attribute 的取值范围
```

形式化地，给定域 $D_1,D_2,\dots,D_n$，关系 $r$ 是笛卡尔积的一个子集：

$$
r \subseteq D_1 \times D_2 \times \cdots \times D_n
$$

关系中的每个元素是一个 $n$ 元组：

$$
(a_1,a_2,\dots,a_n), \quad a_i \in D_i
$$

例如：

```text
instructor(ID, name, dept_name, salary)
```

可以有元组：

```text
(10101, Srinivasan, Comp. Sci., 65000)
```

---

## 2. Relation Schema and Instance

### 2.1 Relation Schema

relation schema 描述关系的结构：

```text
R = (A1, A2, ..., An)
```

例如：

```text
instructor = (ID, name, dept_name, salary)
```

### 2.2 Relation Instance

relation instance 是某一时刻该关系中的实际元组集合，记作：

$$
r(R)
$$

schema 相对稳定，instance 随插入、删除、更新而变化。

---

## 3. Attributes 与 Domains

每个 attribute 都有一个 domain，即允许取值集合。

关系模型中通常要求 attribute value 是 **atomic** 的，也就是不可再分。

例如：

- `salary` 的 domain 可以是数字；
- `dept_name` 的 domain 可以是字符串；
- `phone_numbers` 如果一个单元格中放多个电话号码，就不满足 atomic。

### Null

`null` 是每个 domain 的特殊成员，表示：

- unknown；
- not applicable；
- missing。

但 null 会使很多运算变复杂，尤其是比较、聚合和完整性约束。

---

## 4. Relations are Unordered

关系是集合，因此：

- tuple 没有顺序；
- attribute 在数学定义中也可按名字引用；
- 表中显示顺序只是物理或展示层面的结果。

SQL 实际采用 multiset / bag 语义，允许重复行；但纯关系模型中关系是 set，不允许重复 tuple。

---

## 5. Database Schema

database schema 是整个数据库的逻辑结构，由多个 relation schemas 组成。

例如大学数据库可以包含：

```text
student(ID, name, dept_name, tot_cred)
instructor(ID, name, dept_name, salary)
course(course_id, title, dept_name, credits)
takes(ID, course_id, sec_id, semester, year, grade)
teaches(ID, course_id, sec_id, semester, year)
department(dept_name, building, budget)
```

database instance 是某一时刻所有关系实例的集合。

---

## 6. Keys

### 6.1 Superkey

设 $K \subseteq R$。

如果 $K$ 的值足以在任意合法关系实例中唯一标识一个 tuple，则 $K$ 是 $R$ 的 superkey。

例如在：

```text
instructor(ID, name, dept_name, salary)
```

如果 `ID` 唯一，则：

```text
{ID}
{ID, name}
{ID, dept_name}
```

都是 superkey。

### 6.2 Candidate Key

candidate key 是最小的 superkey。

“最小”指不能再删属性，否则就不再是 superkey。

如果 `{ID}` 已经能唯一标识 tuple，则 `{ID, name}` 不是 candidate key，因为它不是最小的。

### 6.3 Primary Key

从 candidate keys 中选一个作为 primary key。

primary key 用于数据库主要标识 tuple，通常不允许 null。

### 6.4 Foreign Key

foreign key 描述两个关系之间的引用。

如果关系 $r_1$ 的属性 $A$ 引用关系 $r_2$ 的主键 $B$，则 $r_1.A$ 的每个非空值必须出现在 $r_2.B$ 中。

例如：

```text
teaches(ID, course_id, ...)
instructor(ID, name, ...)
```

`teaches.ID` 是引用 `instructor.ID` 的 foreign key。

### 6.5 Referential Integrity

referential integrity 要求引用关系中的外键值必须能在被引用关系中找到。

直觉：

> 不能出现一条授课记录指向一个不存在的教师。

---

## 7. Schema Diagram

schema diagram 用图表示：

- relations；
- attributes；
- primary keys；
- foreign-key constraints；
- referential integrity constraints。

通常：

- 每个 relation 是一个框；
- primary key 用下划线；
- 箭头表示 foreign key 指向 referenced relation。

---

## 8. Relational Query Languages

查询语言可以分为：

- procedural：说明怎么得到结果；
- non-procedural / declarative：说明要什么结果。

三种纯理论语言：

- relational algebra；
- tuple relational calculus；
- domain relational calculus。

它们表达能力等价。

本章重点是 **relational algebra**。

---

## 9. Relational Algebra 总览

关系代数是一种过程式查询语言。

每个操作：

- 输入一个或两个关系；
- 输出一个新关系。

六个基本操作：

- select：$\sigma$；
- project：$\Pi$；
- union：$\cup$；
- set difference：$-$；
- Cartesian product：$\times$；
- rename：$\rho$。

附加操作：

- intersection；
- natural join；
- theta join；
- outer join；
- assignment；
- division。

扩展操作：

- generalized projection；
- aggregation。

---

## 10. Select Operation

select 从关系中选出满足谓词的 tuples。

记法：

$$
\sigma_p(r)
$$

定义：

$$
\sigma_p(r)=\{t \mid t\in r \land p(t)\}
$$

例子：

$$
\sigma_{dept\_name='Physics'}(instructor)
$$

$$
\sigma_{dept\_name='Physics' \land salary>90000}(instructor)
$$

select 作用在“行”上，对应 SQL 的 `where`。

---

## 11. Project Operation

project 从关系中选取指定 attributes。

记法：

$$
\Pi_{A_1,A_2,\dots,A_k}(r)
$$

例子：

$$
\Pi_{ID,name,salary}(instructor)
$$

注意：

> 关系是集合，所以 projection 后会自动去重。

project 作用在“列”上，对应 SQL 的 `select` 属性列表。

---

## 12. Union Operation

union 合并两个兼容关系。

记法：

$$
r \cup s
$$

定义：

$$
r \cup s=\{t \mid t\in r \lor t\in s\}
$$

要求：

1. $r$ 和 $s$ 有相同 arity；
2. 对应属性的 domain compatible。

例子：找 Fall 2009 或 Spring 2010 开设的课程：

$$
\Pi_{course\_id}(\sigma_{semester='Fall' \land year=2009}(section))
\cup
\Pi_{course\_id}(\sigma_{semester='Spring' \land year=2010}(section))
$$

---

## 13. Set Difference Operation

set difference 找出在一个关系中但不在另一个关系中的 tuples。

记法：

$$
r-s
$$

定义：

$$
r-s=\{t \mid t\in r \land t\notin s\}
$$

同样要求两个关系 compatible。

例子：Fall 2009 开设但 Spring 2010 未开设的课程：

$$
\Pi_{course\_id}(\sigma_{semester='Fall' \land year=2009}(section))
-
\Pi_{course\_id}(\sigma_{semester='Spring' \land year=2010}(section))
$$

---

## 14. Cartesian Product

笛卡尔积把两个关系的 tuple 做所有可能组合。

记法：

$$
r \times s
$$

如果：

- $r$ 有 $n$ 个 tuples；
- $s$ 有 $m$ 个 tuples；

则：

$$
|r \times s| = n \times m
$$

笛卡尔积本身通常不直接有意义，但和 selection 结合后可以表达 join。

例如：

$$
\sigma_{instructor.ID=teaches.ID}(instructor \times teaches)
$$

---

## 15. Rename Operation

rename 用于给关系或属性改名。

常见记法：

$$
\rho_X(E)
$$

表示把表达式 $E$ 的结果命名为 $X$。

也可以给属性改名：

$$
\rho_{X(A_1,A_2,\dots,A_n)}(E)
$$

用途：

- 简化长表达式；
- 处理自连接；
- 避免属性名冲突。

---

## 16. Composition of Operations

关系代数操作可以嵌套组合。

例如找 Physics 系教师姓名：

$$
\Pi_{name}(\sigma_{dept\_name='Physics'}(instructor))
$$

对应 SQL：

```sql
select name
from instructor
where dept_name = 'Physics';
```

---

## 17. Additional Operations

附加操作不增加表达能力，因为它们可以由六个基本操作定义，但使用起来更方便。

### 17.1 Set Intersection

交集：

$$
r \cap s
$$

可用差集定义：

$$
r \cap s = r - (r-s)
$$

### 17.2 Natural Join

natural join 自动按两个关系中同名属性相等进行连接，并只保留一份同名属性。

记法：

$$
r \bowtie s
$$

例如：

```text
instructor(ID, name, dept_name, salary)
teaches(ID, course_id, sec_id, semester, year)
```

则：

$$
instructor \bowtie teaches
$$

会自动按 `ID` 相等连接。

### 17.3 Theta Join

theta join 使用任意连接条件：

$$
r \bowtie_\theta s = \sigma_\theta(r \times s)
$$

例如：

$$
instructor \bowtie_{instructor.ID=teaches.ID} teaches
$$

### 17.4 Outer Join

outer join 保留未匹配的 tuples，用 null 填充另一侧属性。

包括：

- left outer join；
- right outer join；
- full outer join。

它用于避免普通 join 丢失未匹配数据。

---

## 18. Assignment Operation

assignment 用于把中间结果赋给临时关系变量：

$$
temp \leftarrow \Pi_{ID,name}(instructor)
$$

它不增加表达能力，但能让复杂表达式更清晰。

---

## 19. Division Operator

division 常用于表达“对所有”的查询。

若：

```text
r(R)
s(S)
S ⊆ R
```

则：

$$
r \div s
$$

返回在 $R-S$ 上的 tuples，使其与 $s$ 中所有 tuples 组合后都出现在 $r$ 中。

典型问题：

> 找出选修了所有 Biology 系课程的学生。

这种“所有”语义经常可以用 division 或 SQL 中的 `not exists` 表达。

---

## 20. Extended Relational Algebra

### 20.1 Generalized Projection

允许 projection 中出现表达式。

例如：

$$
\Pi_{ID,name,salary/12}(instructor)
$$

得到月薪。

### 20.2 Aggregate Functions

常见聚合函数：

- `avg`；
- `min`；
- `max`；
- `sum`；
- `count`。

聚合操作一般记为：

$$
_{G_1,G_2,\dots}\mathcal{G}_{F_1(A_1),F_2(A_2),\dots}(E)
$$

其中 $G_i$ 是分组属性。

例子：按院系统计平均工资：

$$
_{dept\_name}\mathcal{G}_{avg(salary)}(instructor)
$$

---

## 21. Modification of the Database

关系代数也可以描述数据库修改：

- deletion；
- insertion；
- update。

删除可以看成用差集替换原关系：

$$
r \leftarrow r - E
$$

插入可以看成并集：

$$
r \leftarrow r \cup E
$$

更新通常可表示为删除旧 tuple 再插入新 tuple。

---

## 22. Multiset Relational Algebra

纯关系代数基于 set，不允许重复。

SQL 默认基于 multiset / bag，允许重复。

差异：

- projection 在关系代数中自动去重；
- SQL 中 `select` 默认不去重；
- SQL 要去重需写 `distinct`。

例如：

```sql
select dept_name
from instructor;
```

可能返回重复院系名。

```sql
select distinct dept_name
from instructor;
```

才去重。

---

## 23. SQL and Relational Algebra

典型 SQL：

```sql
select A1, A2, ..., An
from r1, r2, ..., rm
where P;
```

对应 multiset relational algebra：

$$
\Pi_{A_1,A_2,\dots,A_n}
(\sigma_P(r_1 \times r_2 \times \cdots \times r_m))
$$

带 group by 的 SQL 可以用扩展关系代数中的 aggregation 表示。

---

## 24. 本章速记

- relation 是域的笛卡尔积的子集。
- relation schema 是结构，relation instance 是当前数据。
- attribute values 通常要求 atomic。
- superkey 能唯一标识 tuple；candidate key 是最小 superkey；primary key 是选定的 candidate key。
- foreign key 保证 referential integrity。
- 关系代数六大基本操作：$\sigma,\Pi,\cup,-,\times,\rho$。
- selection 选行，projection 选列并去重。
- natural join 自动按同名属性连接，但要小心无关同名属性。
- division 常用于表达“对所有”。
- SQL 默认是 multiset 语义，不自动去重，除非使用 `distinct`。

