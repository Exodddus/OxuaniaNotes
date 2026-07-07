# Lec6 E-R Model

## 1. 数据库设计的整体流程

数据库设计不是直接建表，而是从现实需求逐步落到可实现的模式：

1. **需求分析（requirement specification）**：完整描述用户需要保存的数据、业务规则与操作。
2. **概念设计（conceptual design）**：选择数据模型，用 E-R 图描述实体、联系和约束。
3. **逻辑设计（logical design）**：把概念模式转换成关系模式，并检查模式是否合理。
4. **物理设计（physical design）**：决定文件组织、索引、分区等存储细节。

设计时要避免两类主要问题：

- **冗余（redundancy）**：同一事实重复存储，浪费空间，并可能导致多份副本不一致。
- **信息不完整（incompleteness）**：模式无法表达某些业务事实或约束。

E-R 模型主要服务于**概念设计**：先正确理解现实世界，再考虑如何建表。

---

## 2. E-R 模型的基本组成

### 2.1 Entity 与 Entity Set

- **实体（entity）**：现实世界中可以与其他对象区分开的“事物”或“对象”。
  - 例如某位学生、某门课程、某个院系。
- **实体集（entity set）**：具有相同类型、共享相同属性的一组实体。
  - 例如所有学生构成 `student` 实体集。

实体由属性描述。例如：

```text
instructor = (ID, name, salary)
course     = (course_id, title, credits)
```

E-R 图中：

- 矩形表示实体集；
- 属性列在矩形内部；
- 主键属性用下划线标出。

### 2.2 Relationship 与 Relationship Set

- **联系（relationship）**：若干实体之间的关联。
- **联系集（relationship set）**：同一类型联系的集合。

例如 `advisor` 表示学生与导师之间的指导关系：

```text
student -- advisor -- instructor
```

若联系集 $R$ 涉及实体集 $E_1,E_2,\dots,E_n$，则可以把它看成：

$$
R \subseteq E_1 \times E_2 \times \cdots \times E_n
$$

E-R 图中用**菱形**表示联系集。

### 2.3 联系也可以有属性

有些属性描述的不是某一个实体，而是“一次联系本身”。

例如学生从何时开始由某位教师指导，`date` 应属于 `advisor` 联系，而不是固定属于 `student` 或 `instructor`：

```text
student -- advisor(date) -- instructor
```

判断方法：如果属性值依赖于“哪几个实体发生了这次联系”，它通常应是联系属性。

### 2.4 Role（角色）

一个实体集可以在同一个联系中出现多次，此时要用**角色名**区分。

例如课程的先修关系是 `course` 与自身之间的联系：

```text
course(course_id) -- prereq -- course(prereq_id)
```

`course_id` 与 `prereq_id` 表示同一实体集在联系中扮演的不同角色。

### 2.5 联系的度（degree）

联系集中参与的实体集数量称为联系的度：

- 二元联系（binary relationship）：两个实体集参与，最常见；
- 三元联系（ternary relationship）：三个实体集参与；
- $n$ 元联系：$n$ 个实体集参与。

不要仅因为能把一个三元联系拆成若干二元联系，就认为二者语义一定等价。三元联系表达的是“三个实体共同参与同一事实”。

---

## 3. 属性的类型

### 3.1 Simple 与 Composite Attribute

- **简单属性（simple attribute）**：不可再分，例如 `salary`。
- **复合属性（composite attribute）**：可以拆成更小的组成部分。

例如：

```text
name    = (first_name, middle_initial, last_name)
address = (street, city, state, zip_code)
```

复合属性适合在需要分别查询其组成部分时使用。

### 3.2 Single-Valued 与 Multivalued Attribute

- **单值属性（single-valued attribute）**：一个实体只有一个值，例如学生的 `ID`。
- **多值属性（multivalued attribute）**：一个实体可以有多个值，例如教师的多个 `phone_number`。

课件的矩形内记法常用花括号表示多值属性：

```text
{ phone_number }
```

### 3.3 Derived Attribute

**派生属性（derived attribute）**可以由其他属性计算得到。例如 `age` 可以由 `date_of_birth` 与当前日期计算。

派生值通常不必重复存储，否则会产生更新不一致。

### 3.4 Domain

每个属性都有一个允许取值的集合，即**域（domain）**。例如：

- `salary` 的域可以是非负数；
- `semester` 的域可以是 `{Spring, Summer, Fall, Winter}`。

---

## 4. Mapping Cardinality（映射基数）

映射基数描述：一个实体通过某个联系最多能与另一侧多少个实体关联。对二元联系有四种情况。

| 类型 | 含义 |
|---|---|
| 1:1 | A 中一个实体至多对应 B 中一个实体，反之亦然 |
| 1:N | A 中一个实体可对应 B 中多个实体；B 中一个实体至多对应 A 中一个实体 |
| N:1 | A 中多个实体可对应 B 中一个实体；A 中一个实体至多对应 B 中一个实体 |
| M:N | 两侧实体都可对应另一侧多个实体 |

在课件采用的箭头记法中：

- 联系到实体的**箭头**表示该侧为“一”；
- 普通线表示该侧为“多”。

例如 `advisor` 的箭头指向 `instructor`，表示每个学生至多有一位导师，而一位教师可以指导多个学生，因此是：

```text
student N : 1 instructor
```

---

## 5. Participation Constraint（参与约束）

参与约束，描述的是实体集中的实体是否必须参加某个联系。

- **全部参与（total participation）**：每个实体至少参与一次，用双线表示。
- **部分参与（partial participation）**：允许某些实体一次也不参与，用单线表示。

例如，如果规定每个学生必须有导师，则 `student` 对 `advisor` 是全部参与；教师可以暂时不指导任何学生，因此 `instructor` 可以是部分参与。

### 5.1 最小-最大基数记法

可以在连线旁标注 `l..h`：

- $l$：最小参与次数；
- $h$：最大参与次数；
- `*`：无上限。

课件中的例子：

```text
instructor 0..* -- advisor -- 1..1 student
```

含义是：

- 一位教师可以指导 0 个或多个学生；
- 每个学生必须恰好有 1 位导师。

因此：

- 最小值为 1 表示全部参与；
- 最大值为 1 表示至多参与一个联系；
- `0..*` 表示可不参与，也可参与任意多次。

### 5.2 三元联系的基数约束

三元或更高元联系中的箭头语义必须结合其他参与者一起理解。

例如在 `proj_guide(student, project, instructor)` 中，箭头指向 `instructor` 表示：

> 对每一组给定的 `(student, project)`，至多存在一个 `instructor`。

课件约定高元联系最多画一个箭头，以避免产生含混的组合约束。

---

## 6. Key（键）

### 6.1 实体集的键

数据库必须通过属性值区分不同实体。

- **超键（superkey）**：能唯一标识实体的属性集合；
- **候选键（candidate key）**：最小超键；
- **主键（primary key）**：被选作主要标识的候选键。

### 6.2 联系集的键

设联系集 $R$ 涉及实体集 $E_1,E_2,\dots,E_n$。各实体集主键的并集总能形成 $R$ 的一个超键；实际主键由映射基数决定。

对二元联系：

- **M:N**：两侧主键的并集是联系集主键；
- **1:N / N:1**：“多”侧实体的主键可唯一决定一次联系，因此可作为联系集主键；
- **1:1**：任意一侧的主键都可以作为联系集候选键。

若联系本身允许同一组实体之间出现多次，还需要额外属性（例如时间或序号）才能唯一标识每次联系。

---

## 7. Weak Entity Set（弱实体集）

### 7.1 定义

没有足够属性形成自身主键的实体集称为**弱实体集**。

弱实体依赖某个**标识实体集（identifying entity set）** 存在，并通过**标识联系（identifying relationship）** 与它关联。

典型约束是：

- 弱实体对标识联系全部参与；
- 从标识实体到弱实体通常是一对多。

### 7.2 Discriminator / Partial Key

**分辨符（discriminator，也称 partial key）** 只能在已知标识实体的前提下区分弱实体。

弱实体的主键为：

$$
\text{标识实体主键} \cup \text{弱实体分辨符}
$$

例如 `section` 依赖 `course`：

```text
course 主键: course_id
section 分辨符: (sec_id, semester, year)
section 主键: (course_id, sec_id, semester, year)
```

### 7.3 E-R 图记法

- 弱实体集：
	- 标识联系：双菱形；
	- 分辨符：虚线下划线；
	- 弱实体到标识联系：双线，表示必须全部参与。

虽然 `course_id` 在 E-R 图的弱实体框中可以不显式出现，但转换成关系表后必须作为外键和主键组成部分存入 `section` 表。

---

## 8. Redundant Attributes（冗余属性）

如果某个事实已经由联系表达，一般不要再在实体中重复保存同一信息。

例如：

```text
student(ID, name, tot_cred, dept_name)
department(dept_name, building, budget)
student -- stud_dept -- department
```

若 `stud_dept` 已表示学生所属院系，概念模型中的 `student.dept_name` 就是冗余的。重复表达会导致：

- 属性值与联系不一致；
- 修改时要同时更新两处；
- 业务约束难以维护。

不过在把 E-R 图转换成关系模式时，`dept_name` 会作为外键加入 `student` 表。这不是逻辑事实被表示两次，而是对联系的关系实现。

---

## 9. E-R 图转换为关系模式

### 9.1 强实体集

简单属性的强实体集直接变成同名关系：

```text
course(course_id, title, credits)
```

实体主键成为关系主键。

### 9.2 弱实体集

弱实体关系包含：

- 标识实体的主键；
- 弱实体的所有简单属性；
- 其中“标识实体主键 + 分辨符”构成主键。

```text
section(course_id, sec_id, semester, year)
PK = (course_id, sec_id, semester, year)
FK course_id -> course(course_id)
```

没有描述性属性的标识联系通常不需要单独建表。

### 9.3 M:N 联系

M:N 联系通常单独建表，包含：

- 两侧实体的主键，作为外键；
- 联系自身的描述性属性；
- 两侧主键的并集通常构成联系表主键。

例如课程注册 `takes` 连接 `student` 与弱实体 `section`：

```text
takes(s_id, course_id, sec_id, semester, year, grade)
PK = (s_id, course_id, sec_id, semester, year)
```

其中 `grade` 是联系属性，两端实体主键的并集构成联系表主键。

### 9.4 1:N / N:1 联系

通常把“一”侧主键作为外键加入“多”侧关系，无须再为联系单独建表。

例如许多教师属于一个院系：

```text
department(dept_name, building, budget)
instructor(ID, name, salary, dept_name)
FK instructor.dept_name -> department(dept_name)
```

若联系有属性，也可以把它一并放入“多”侧关系。若“多”侧是部分参与，外键可能为 `NULL`；此时也可以保留独立联系表来避免空值。

### 9.5 1:1 联系

可以把任意一侧主键作为外键加入另一侧。选择时通常考虑：

- 优先放在**全部参与**的一侧，以减少 `NULL`；
- 把联系属性放在同一侧；
- 用 `UNIQUE` 约束保证 1:1，而不是退化为 N:1。

### 9.6 Composite Attribute

复合属性在关系模式中展开成各个简单分量。

```text
name(first_name, last_name)
```

转换后写作：

```text
instructor(ID, first_name, last_name, ...)
```

### 9.7 Multivalued Attribute

多值属性必须单独建表。若实体 $E$ 的主键为 $K$，多值属性为 $M$，则建立：

```text
E_M(K, M)
PK = (K, M)
```

例如：

```text
instructor_phone(ID, phone_number)
PK = (ID, phone_number)
```

若一个实体除了主键外只有一个多值复合属性，实体本身的独立关系可能可以省略。例如：

```text
time_slot_detail(time_slot_id, day, start_time, end_time)
```

已经包含了 `time_slot_id`，可能无须再建立只有 `time_slot_id` 一列的 `time_slot` 表。

---

## 10. 非二元联系的设计

### 10.1 何时保留三元联系

如果事实天然由三个实体共同决定，就应保留三元联系。例如项目指导可能同时涉及：

```text
(student, project, instructor)
```

把它拆成三组二元联系，可能只说明实体两两有关，却无法说明它们属于**同一次三方事实**。

### 10.2 何时改成二元联系

有些表面上的三元联系实际是两种独立关系。例如 `parents(child, father, mother)` 更适合拆成：

```text
father(child, person)
mother(child, person)
```

这样还能记录“只知道母亲、不知道父亲”的部分信息。

### 10.3 用人工实体转换

任意三元联系 $R(A,B,C)$ 都可以通过创建人工实体 $E$ 转为三个二元联系：

```text
R_A(E, A)
R_B(E, B)
R_C(E, C)
```

并把原联系 $R$ 的属性移到 $E$ 中。每个原关系实例 $(a_i,b_i,c_i)$ 对应一个新的 $e_i$。

但必须同步翻译原来的参与和基数约束；有些约束无法仅靠这三个二元联系完整表达。因此“能转换”不等于“设计更好”。

---

## 11. E-R 设计中的常见决策

### 11.1 实体还是属性

适合作为实体的情况：

- 对象还有自己的属性；
- 对象需要参与其他联系；
- 一个主体可以关联多个该对象；
- 对象需要独立标识与管理。

例如把 `phone` 建模为实体后，可以继续保存号码类型、运营商、状态等信息；若只关心一个号码值，属性可能更简单。

### 11.2 实体还是联系

通常：

- 有独立身份、生命周期的对象建模为实体；
- 描述实体之间发生的动作或关联，建模为联系。

如果一个“联系”本身需要参加其他联系，往往应把它提升为实体。

### 11.3 联系属性放在哪里

属性应放在能唯一决定其值的对象上。

例如账户开户日期：

- 若日期描述“某客户何时获得某账户的访问权”，它属于 `access(customer, account)`；
- 若日期描述“账户本身何时创建”，它属于 `account`。

### 11.4 强实体还是弱实体

若对象只能在某个所属实体内部被标识，并且存在依赖明显，可建模为弱实体；若对象已有全局唯一标识，通常使用强实体更清晰。

---

## 12. Specialization 与 Generalization

### 12.1 基本概念

- **特化（specialization）**：自顶向下，把高层实体集划分为具有特殊性质的子实体集。
- **概化（generalization）**：自底向上，把若干具有共同属性的实体集合并为高层实体集。

两者本质上互为逆过程，在 E-R 图中采用同一种表示。

例如：

```text
person(ID, name, address)
  |- employee(salary)
  |    |- instructor(rank)
  |    `- secretary(hours_per_week)
  `- student(tot_credits)
```

### 12.2 Attribute Inheritance

低层实体集继承高层实体集的：

- 全部属性；
- 参与的联系；
- 主键。

因此 `instructor` 自动拥有 `employee.salary` 和 `person` 的 `ID、name、address`。

### 12.3 Disjoint 与 Overlapping

- **不相交（disjoint）**：一个高层实体最多属于一个子类。
- **重叠（overlapping）**：一个高层实体可以同时属于多个子类。

例如 `employee` 可要求必须是 `instructor` 或 `secretary` 之一，且不能同时属于二者；而 `person` 可以同时是 `employee` 和 `student`。

### 12.4 Total 与 Partial

完全性约束说明高层实体是否必须属于至少一个子类：

- **全部特化（total）**：每个高层实体都必须属于某个子类；
- **部分特化（partial）**：允许高层实体不属于任何已列出的子类。

这两组约束彼此独立，因此会形成四种组合：

- disjoint + total；
- disjoint + partial；
- overlapping + total；
- overlapping + partial。

---

## 13. 特化结构转换为关系模式

### 方法一：高层表 + 每个低层表

```text
person(ID, name, address)
employee(ID, salary)
student(ID, tot_credits)
```

- 子表主键同时是指向父表的外键；
- 不会重复存储父类属性；
- 查询完整子类对象时需要连接父表和子表。

这是最通用、最规范的做法。

### 方法二：每个具体子类存全部属性

```text
employee(ID, name, address, salary)
student(ID, name, address, tot_credits)
```

- 查询单个子类方便；
- overlapping 情况下，同一人属于多个子类会重复存储 `name、address`；
- partial 情况下，还需要额外表保存不属于任何子类的父类实体。

### 方法三：单表 + 类型属性

```text
person(ID, name, address, person_type, salary, tot_credits)
```

- 查询不需要连接；
- 不适用属性会产生大量 `NULL`；
- overlapping 时单一 `person_type` 不够，需要多个标志位或集合值；
- 约束更难表达。

---

## 14. ER 与 UML 类图

UML 类图与 E-R 图都能描述对象、属性、关联和泛化，但重点不同：

- E-R 图面向数据库概念建模；
- UML 类图面向整个软件系统，还可描述方法、可见性等。

常见对应关系：

| E-R | UML |
|---|---|
| 实体集 | 类 |
| 实体属性 | 类属性 |
| 二元联系 | association |
| 联系属性 | association class |
| 特化/概化 | inheritance / generalization |

课件特别提醒：**E-R 与 UML 在连线两端标注基数的位置语义相反**。读图时应先确认使用的是哪套记法。

---

## 15. 本章解题与设计主线

看到一段业务描述时，可以按下面顺序建模：

1. 找出需要独立标识的对象，建立实体集与主键。
2. 给实体补充简单、复合、多值或派生属性。
3. 找出实体间的业务事实，建立联系及联系属性。
4. 确定每侧的最小、最大基数，即参与约束和映射基数。
5. 检查是否存在依赖其他实体才能标识的弱实体。
6. 检查是否重复表达同一事实，是否把三元联系错误拆成二元联系。
7. 最后再按映射规则转换为关系表、主键和外键。

最重要的记忆点：

- 实体是“对象”，联系是“对象之间的事实”。
- 箭头指向“一”侧，双线表示全部参与。
- 联系属性属于一次关联，而不是随意放到某个实体上。
- 弱实体主键 = 标识实体主键 + 分辨符。
- M:N 联系单独建表；1:N 联系通常把“一”侧键放入“多”侧。
- 多值属性单独建表，复合属性展开。
- 三元联系不能在不检查语义和约束的情况下直接拆成二元联系。
