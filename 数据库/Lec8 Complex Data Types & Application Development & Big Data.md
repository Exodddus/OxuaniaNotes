# Lec8 Complex Data Types & Application Development & Big Data

对应课件：

- `ch8 - Complex Data Types(2).pdf`
- `ch9 - Application Development.pdf`
- `ch10 - Big Data.pdf`

这一讲把三块内容放在一起：数据库如何表示更复杂的数据，应用程序如何围绕数据库搭起来，以及大数据系统为什么和传统数据库不完全一样。

---

## 1. Complex Data Types

### 1.1 为什么需要复杂数据类型

传统关系模型要求属性值尽量是 atomic value，但很多高级数据库应用天然需要更复杂的数据：

- 一个图书馆系统中，一本书可能有多个作者、一个出版社、多个关键字。
- 一个用户画像中，一个用户可能有多个兴趣标签。
- Web 服务和移动应用经常在前后端之间传输嵌套对象。
- 空间数据库需要表示点、线、多边形、地图、CAD 设计对象。

应用程序往往用面向对象语言编写，但关系数据库的类型系统和对象语言不完全匹配。常见解决路线有三种：

- **OODB**：直接构建面向对象数据库，原生支持对象数据和编程语言访问。
- **ORDB**：在关系数据库中加入对象特性，即 object-relational database。
- **ORM**：在对象模型和关系模型之间自动转换，例如 Hibernate、Django ORM。

---

### 1.2 Nested Relation 与 1NF

以图书馆系统为例，`books` 可以自然表示为嵌套关系：

- `title`
- `authors`，一组作者
- `publisher`
- `keywords`，一组关键字

这种表示更贴近现实对象，但它不是 1NF，因为属性中出现集合值。若强行转成 1NF，可能得到 `flat-books(title, author, keyword, pub-name, pub-branch)` 这样的扁平表。

扁平表的问题是会产生大量重复和组合爆炸。例如一本书有 `m` 个作者和 `n` 个关键字，可能产生 `$m \times n$` 行。

课件中用多值依赖解释如何消除这种 awkwardness：

- `$title \twoheadrightarrow author$`
- `$title \twoheadrightarrow keyword$`
- `$title \twoheadrightarrow pub\text{-}name, pub\text{-}branch$`

可以把 `flat-books` 分解成 4NF：

- `(title, author)`
- `(title, keyword)`
- `(title, pub-name, pub-branch)`

直觉：多值属性之间相互独立时，不应该把它们硬塞进同一张 1NF 表里产生重复组合。

---

### 1.3 SQL:1999 对复杂类型的支持

SQL:1999 为复杂类型加入了若干扩展：

- **Collection types**：集合类型，nested relation 就是一种集合类型。
- **Structured types**：结构化类型，类似 composite attributes。
- **Inheritance**：类型继承和表继承。
- **Object orientation**：对象标识符和引用。
- **Large object types**：如 `BLOB`、`CLOB`。

课件提醒：SQL:1999 的复杂类型标准没有被所有数据库完整实现，但主流数据库通常支持其中一部分，语法也常常不同。

数组和 multiset 示例：

```sql
create type Publisher as (
  name varchar(20),
  branch varchar(20)
);

create type Book as (
  title varchar(20),
  author_array varchar(20) array[10],
  pub_date date,
  publisher Publisher,
  keyword_set varchar(20) multiset
);

create table books of Book;
```

这里 `author_array` 是数组，`keyword_set` 是 multiset，`publisher` 是结构化类型。

---

### 1.4 Object-Relational Database Systems

对象关系数据库支持用户自定义类型、表类型、数组、multiset、继承和引用。

用户自定义类型：

```sql
create type Person as (
  ID varchar(20) primary key,
  name varchar(20),
  address varchar(20)
) ref from (ID);

create table people of Person;
```

表类型：

```sql
create type Interest as table (
  topic varchar(20),
  degree_of_interest int
);

create table users (
  ID varchar(20),
  name varchar(20),
  interests Interest
);
```

类型继承：

```sql
create type Student under Person (
  degree varchar(20)
);

create type Teacher under Person (
  salary integer
);
```

PostgreSQL 风格的表继承：

```sql
create table students (
  degree varchar(20)
) inherits people;

create table teachers (
  salary integer
) inherits people;
```

引用类型可以让一个对象引用另一个对象：

```sql
create type Department as (
  dept_name varchar(20),
  head ref(Person) scope people
);

create table departments of Department;
```

使用引用时可以写路径表达式：

```sql
select head->name, head->address
from departments;
```

---

### 1.5 ORM

ORM 允许应用代码使用对象模型，同时把数据存储在传统关系数据库中。它做的事包括：

- 指定对象和关系元组之间的映射。
- 创建对象时自动创建数据库元组。
- 更新或删除对象时自动更新或删除数据库元组。
- 支持按条件查询对象，底层实际查询数据库元组，再构造对象。

Hibernate 例子：

```java
@Entity
public class Student {
  @Id String ID;
  String name;
  String department;
  int tot_cred;
}
```

`@Entity` 表示这个类映射到数据库关系，默认关系名是类名，默认属性名是类属性名。`@Id` 表示主键。默认表名和列名可用 `@Table`、`@Column` 改写。

保存对象：

```java
Session session = getSessionFactory().openSession();
Transaction txn = session.beginTransaction();
Student stud = new Student("12328", "John Smith", "Comp. Sci.", 0);
session.save(stud);
txn.commit();
session.close();
```

查询对象：

```java
Session session = getSessionFactory().openSession();
Transaction txn = session.beginTransaction();

Student stud1 = session.get(Student.class, "12328");
List students = session
  .createQuery("from Student as s order by s.ID asc")
  .list();

txn.commit();
session.close();
```

---

## 2. Semi-Structured Data

### 2.1 半结构化数据

半结构化数据适合模式经常变化、层次结构复杂、前后端需要交换对象的场景。

关系模型要求 atomic data types，有时会显得过重。例如把用户兴趣拆成独立关系当然规范，但在用户画像里直接存成集合或 JSON 属性可能更自然。

课件重点提到两种半结构化模型：

- **XML**：Extensible Markup Language，早期常用，现在仍在很多系统中使用。
- **JSON**：JavaScript Object Notation，现在 Web 服务中非常常见。

---

### 2.2 XML

XML 用 tag 标记文本：

```xml
<course>
  <course_id>CS-101</course_id>
  <title>Intro. to Computer Science</title>
  <dept_name>Comp. Sci.</dept_name>
</course>
```

XML 文档可以看作一棵树，节点包括元素、属性和文本。它适合表示嵌套结构。

常见 XML 查询和约束工具：

- **DTD**：定义 XML 文档的合法结构。
- **XPath**：用路径表达式定位 XML 树中的节点。
- **XQuery**：查询嵌套 XML 结构的语言。

XPath 例子：

```text
bookstore/book
bookstore/book[1]/title
bookstore/book/author
bookstore/book[price > 30]/year
bookstore/book[1]/title/text()
```

XQuery 例子：

```xquery
<book_pairs>
  for $a in /bookstore/book,
      $b in /bookstore/book
  where $a/author[1] = $b/author[1]
    and $a/price > $b/price
  return <book_pair>
    <first_author>{data($a/author[1])}</first_author>
    {$a/title}
    {$b/title}
  </book_pair>
</book_pairs>
```

SQL 系统也常扩展支持 XML 数据的存储、从关系数据生成 XML、从 XML 类型中提取数据。

---

### 2.3 JSON

JSON 是轻量、语言无关的文本数据交换格式，今天广泛用于 Web services。

JSON 例子：

```json
{
  "ID": "22222",
  "name": {
    "firstname": "Albert",
    "lastname": "Einstein"
  },
  "deptname": "Physics",
  "children": [
    { "firstname": "Hans", "lastname": "Einstein" },
    { "firstname": "Eduard", "lastname": "Einstein" }
  ]
}
```

数据库对 JSON 的常见支持：

- JSON 类型，用于存储 JSON 数据。
- 路径表达式，用于从 JSON 对象中提取字段，例如 `v->ID` 或 `v.ID`。
- 从关系数据生成 JSON，例如 PostgreSQL 的 `json_build_object`。
- 用聚合函数生成 JSON collection，例如 PostgreSQL 的 `json_agg`。
- 压缩或二进制表示，例如 BSON，用于更高效存储。

MySQL JSON 示例：

```sql
create table users (
  id int auto_increment primary key,
  info json
);

insert into users(info)
values ('{"name":"Alice", "age":30}');

select *
from users
where info->'$.age' > 28;

update users
set info = json_set(info, '$.age', 31)
where id = 1;
```

---

### 2.4 RDF 与 SPARQL

RDF，即 Resource Description Format，用 triples 表示事实：

- `(subject, predicate, object)`
- 例子：`(NBA-2019, winner, Raptors)`
- 例子：`(Washington-DC, capital-of, USA)`
- 例子：`(Washington-DC, population, 6200000)`

RDF 可以看作灵活 schema 的知识图谱表示。对象、属性、对象之间的关系都可以用三元组表示。

SPARQL 用 triple patterns 查询 RDF：

```sparql
select ?name
where {
  ?cid title "Intro. to Computer Science" .
  ?sid course ?cid .
  ?sid name ?name .
}
```

RDF triples 适合表示二元关系。若要表示 n-ary relationship，可以：

- 创建人工实体，把它分别连到 n 个实体。
- 使用 quads，加上 context entity。

RDF 常用于知识库，如 DBPedia、Yago、Freebase、WikiData。Linked open data 的目标是连接不同知识图谱，让查询可以跨数据库进行。

---

## 3. Textual Data

### 3.1 信息检索

Textual data 多数是非结构化数据。信息检索的简单模型是：给定 keyword query，返回包含这些关键字的文档。

由于匹配文档通常很多，系统必须做 relevance ranking。

---

### 3.2 TF-IDF

课件给出的 term frequency 定义为：`$TF(d,t)=\log(1+n(d,t)/n(d))$`。

其中：

- `$n(d,t)$`：term `t` 在文档 `d` 中出现的次数。
- `$n(d)$`：文档 `d` 中 term 的总数。

课件给出的 inverse document frequency 形式是：`$IDF(t)=1/n(t)$`，其中 `$n(t)$` 是包含 term `t` 的文档数。实际系统中也常见 `$IDF(t)=\log(N/n(t))$` 这样的变体。

直觉：

- 一个词在当前文档中出现越多，越可能重要。
- 一个词在所有文档中越常见，区分度越低。

---

### 3.3 PageRank

超链接可以提供页面重要性的线索。PageRank 的直觉是：

- 被很多页面链接的页面更重要。
- 被重要页面链接的页面更重要。

设 `$T[i,j]$` 为随机游走者从页面 `i` 点击到页面 `j` 的概率，`$P[i]$` 为页面 `i` 的 PageRank，`$N$` 为页面总数，`$\delta$` 为随机跳转参数。课件中的迭代公式为：`$P[j]=\delta/N+(1-\delta)\sum_{i=1}^{N}T[i,j]P[i]$`。

常见计算方式：

- 初始化所有 `$P[i]=1/N$`。
- 反复用上式更新。
- 当变化很小或迭代达到上限时停止。

---

### 3.4 Retrieval Effectiveness

衡量检索效果常用：

- **Precision**：返回结果中真正相关的比例，`$precision=\frac{|returned \cap relevant|}{|returned|}$`。
- **Recall**：所有相关结果中被返回的比例，`$recall=\frac{|returned \cap relevant|}{|relevant|}$`。
- **precision@k / recall@k**：只看前 `k` 个返回结果。

结构化数据和知识库也可以做 keyword search，适合用户不知道 schema 或根本没有固定 schema 的情况。

---

## 4. Spatial Data

### 4.1 空间数据库

空间数据库存储和空间位置相关的信息，并支持空间数据的高效存储、索引和查询。

典型场景：

- 地理数据：道路地图、土地使用图、地形图、行政边界图。
- 几何数据：CAD、工程设计对象。
- GIS：专门存储和查询地理数据的系统。

地理坐标常用 round-earth coordinate system，例如 `(latitude, longitude, elevation)`。

---

### 4.2 几何表示

基本表示：

- 点可以用坐标表示。
- 线段可以用端点坐标表示。
- polyline / linestring 是连续线段序列。
- 多边形可以用边界线段序列表示。
- 3D 点比 2D 点多一个 `z` 坐标。
- 复杂多面体可以分解成 tetrahedrons，类似把多边形 triangulate。

许多数据库支持 `geometry` 和 `geography` 类型。

---

### 4.3 Design Database

设计数据库通常把设计组件表示为对象，组件之间的连接表示设计结构。

- 简单 2D 对象：points、lines、triangles、rectangles、polygons。
- 复杂 2D 对象：由 union、intersection、difference 组合而成。
- 复杂 3D 对象：由 spheres、cylinders、cuboids 等组合而成。
- wireframe models：用一组简单对象表示三维表面。

设计数据库还存储非空间信息，例如材料、颜色等。空间完整性约束很重要，例如管道不应相交、电线之间距离不能太近。

---

### 4.4 Geographic Data

两类常见表示：

- **Raster data**：位图或像素图，例如卫星云图。额外维度可以表示高度、温度、时间等。
- **Vector data**：由点、线段、多边形、球体、长方体等基本几何对象构成。地图数据常用 vector format。

空间查询包括：

- **Region queries**：查询部分或完全位于某区域内的对象，例如 PostGIS 的 `ST_Contains()`、`ST_Overlaps()`。
- **Nearness queries**：查询靠近某位置的对象。
- **Nearest neighbor queries**：找满足条件的最近对象。
- **Spatial graph queries**：基于空间图的查询，例如道路网络中的最短路径。

---

## 5. Application Development

### 5.1 应用程序与数据库

大多数数据库用户不会直接写 SQL，而是通过应用程序访问数据库。应用程序是用户和数据库之间的中介。

典型应用分层：

- **front-end**：用户界面，例如表单、GUI、Web 界面。
- **middle layer**：业务逻辑层。
- **backend**：数据库和数据访问层。

应用架构演化：

- Mainframe era：1960s 到 1970s。
- Personal computer era：1980s。
- Web era：1990s 中期以后。
- Web and Smartphone era：2010 年以后。

---

### 5.2 Web Interface

浏览器已经成为数据库应用的事实标准界面。

优点：

- 大量用户可从任何地方访问数据库。
- 用户不需要安装专门客户端。
- JavaScript 等脚本可透明下载并在浏览器运行。

常见例子：

- 银行业务
- 机票和租车预订
- 大学选课和成绩系统

---

### 5.3 Three-Layer Web Architecture

三层 Web 架构通常包含：

- 浏览器 / presentation layer
- Web server / application server / business logic layer
- Database server / data layer

应用层常再分：

- **Presentation or user interface**：负责展示和交互。
- **Business logic layer**：抽象实体并执行业务规则。
- **Data access layer**：连接业务层与底层数据库，把对象模型映射到关系模型。

MVC 是常见 UI 架构：

- **Model**：业务逻辑和数据。
- **View**：数据展示，依赖显示设备。
- **Controller**：接收事件，执行动作，返回 view。

---

### 5.4 HTTP、Session 与 Cookie

HTTP 是 connectionless 的：

- 服务器回复请求后关闭连接。
- 服务器默认忘记这次请求。
- 这样可以降低服务器负载。

但应用需要 session information，例如用户登录一次后不应每个请求都重新认证。

解决方法是 cookie：

- 服务器第一次交互时向浏览器发送 cookie。
- 浏览器后续请求同一服务器时带上 cookie。
- 服务器用 cookie 找到 session 信息，如用户身份和偏好。
- cookie 可以永久保存，也可以只保存一段时间。

Servlet API 中 session 的典型逻辑：

```java
HttpSession session = request.getSession(true);
session.setAttribute("userid", userid);

HttpSession existing = request.getSession(false);
String userid = (String) existing.getAttribute("userid");
```

---

### 5.5 Servlets、JSP、PHP、JavaScript

**Servlets** 是 Java Web 应用服务器中的程序。它们运行在 application server 中，每个请求通常由服务器中的线程处理。程序员继承 `HttpServlet` 并重写 `doGet`、`doPost`。

Servlet 可以：

- 从 Web form 获取参数。
- 用 JDBC 访问数据库。
- 生成 HTML 返回给客户端。

示例：

```java
String persontype = request.getParameter("persontype");
String name = request.getParameter("name");
```

**Server-side scripting** 把 HTML、可执行代码和 SQL 查询放在一起，由服务器执行并生成最终 HTML。常见语言包括 JSP、PHP、VBScript、Perl、Python。

JSP：

- HTML 页面中嵌入 Java 代码。
- JSP 会被编译成 Java + Servlet。
- JSP tag library 可用于构建复杂 UI。

PHP：

- 广泛用于 Web server scripting。
- 有丰富库，包括通过 ODBC 访问数据库。

JavaScript：

- 支持前端表单校验。
- 可以修改 DOM。
- 可以通过 AJAX 向服务器取数据并更新页面，不必刷新整个页面。

---

### 5.6 Business Logic Layer

业务逻辑层负责：

- 提供实体抽象，例如 students、instructors、courses。
- 执行业务规则，例如学生只有满足先修课和缴费条件才能选课。
- 支持 workflow，定义多参与者任务的执行过程。

这一层的价值是把业务规则集中起来，避免散落在 UI 或 SQL 中。

---

### 5.7 Application 层 ORM

应用开发中的 ORM 允许代码基于对象模型编写，同时数据库仍使用传统关系模型。

ORM 需要 schema designer 指定对象数据和关系 schema 的映射。Hibernate 的特点：

- 支持复杂查询语言。
- 查询会翻译成 SQL。
- 可以把 relationship 映射成对象中的集合，例如一个 `Student` 对象包含它选过的课程集合。

---

### 5.8 Web Services

Web services 允许 Web 上的数据通过远程过程调用机制访问。

两类常见方式：

- **REST**：使用标准 HTTP 请求访问 URL，返回 XML 或 JSON。
- **Big Web Services**：请求和返回都用 XML，有建立在 HTTP 之上的标准协议层。

今天最常见的后端 API 通常是 REST 风格，返回 JSON。

---

### 5.9 Disconnected Operations 与 RAD

Disconnected operations：应用连接 Web 时使用网络，断网时可以本地运行，常使用 HTML5 local storage。

Rapid Application Development 的目标是加快 Web 应用开发：

- 用函数库生成 UI 元素。
- IDE 拖拽生成 UI。
- 从 declarative specification 生成 UI 代码。
- 框架支持快速 CRUD，例如 Ruby on Rails。

---

### 5.10 Application Performance

热门网站可能每天有数百万用户、高峰期每秒数千请求，因此性能是核心问题。

提升 Web server performance 的常见方式是 caching：

- 在服务器端缓存 JDBC connections。
- 缓存查询结果或部分页面。
- 在客户端或代理处缓存静态资源。

直觉：很多请求有共性，缓存可以避免重复计算和重复访问数据库。

---

### 5.11 Application Security

#### Cross Site Scripting / XSRF

课件例子：某页面里有 HTML 或脚本触发另一个站点上的转账 URL。如果用户正登录银行站点，请求可能以用户身份成功执行。

防护思路：

- 使用 HTTP referer 检查请求是否来自本站合法页面。
- 检查请求 IP 是否和认证时一致。
- 对更新操作使用额外 token。
- 不要让简单 GET 请求执行敏感更新。

#### Password Leakage

不要把数据库密码等明文放在用户可能访问到的脚本中。即使服务器通常执行 `.jsp`、`.php` 而不是返回源码，编辑器备份文件如 `file.jsp~`、`.file.jsp.swp` 也可能被错误暴露。

还应限制数据库服务器只接受 application server 的 IP 访问。

#### Authentication

单因素密码认证对关键应用风险较高：

- 密码可能被猜到。
- 未加密传输可能被嗅探。
- 用户在多个站点复用密码。
- 恶意软件可能窃取密码。

两因素认证可以把密码和一次性验证码结合起来。

#### Single Sign-On

SSO 允许用户认证一次，多个应用通过认证服务验证身份。相关标准和系统包括：

- SAML：跨安全域交换认证和授权信息。
- OpenID：允许用户选择身份提供方。
- LDAP / Active Directory：常用于组织内部认证。

#### Application-Level Authorization

SQL 标准难以表达细粒度授权，例如“学生只能看自己的成绩，不能看其他人的成绩”。

原因：

- 数据库通常不知道最终应用用户是谁。
- SQL 授权多在表或列级别，不容易做到行级别。

一种 workaround 是使用 view：

```sql
create view studentTakes as
select *
from takes
where takes.ID = syscontext.user_id();
```

更通用的方向是 fine-grained / row-level authorization，例如 Oracle VPD 可以给查询自动加 predicate。

#### Audit Trails

应用必须记录 audit trail，用来追踪谁更新了数据或访问了敏感数据。

审计日志可用于：

- 事后发现安全破坏。
- 修复安全破坏造成的损失。
- 追踪破坏者。

审计既需要数据库层，也需要应用层。

---

## 6. Big Data

### 6.1 Big Data 的动机

大数据来自 Web、社交媒体、IoT 等场景，早期典型来源之一是 Web logs。

Big Data 通常用 3V 描述：

- **Volume**：数据量远大于传统数据库处理规模。
- **Velocity**：插入和生成速度更高。
- **Variety**：数据类型更多，不只是关系数据。

大数据系统常见需求：

- 事务处理系统需要极高可扩展性。
- 查询处理系统需要支持非关系数据。
- 一些应用愿意牺牲 ACID 或传统数据库特性来换取可扩展性。

---

### 6.2 大数据存储系统

课件列出四类：

- Distributed file systems
- Sharding across multiple databases
- Key-value storage systems
- Parallel and distributed databases

---

### 6.3 Distributed File Systems 与 HDFS

分布式文件系统把数据存储在大量机器上，但对外提供单一文件系统视图。

特点：

- 支持大规模数据密集型应用。
- 在廉价且不可靠机器上冗余存储。
- 通过复制处理硬件故障。
- 例子：Google File System、HDFS。

HDFS 架构：

- 整个 cluster 有单一 namespace。
- 文件被切分成 blocks，典型 block size 是 `64 MB`。
- 每个 block 复制到多个 DataNodes。
- Client 从 NameNode 获取 block 位置，再直接访问 DataNode。

NameNode：

- 把 filename 映射到 Block ID 列表。
- 把 Block ID 映射到包含副本的 DataNodes。

DataNode：

- 把 Block ID 映射到磁盘上的物理位置。

HDFS 数据一致性模型：

- write-once-read-many。
- client 只能 append 到已有文件。

分布式文件系统适合大量大文件，但处理 billions of small tuples 时开销高、性能差。

---

### 6.4 Sharding

Sharding 是把数据分区到多个数据库中。通常按照 partitioning attribute / shard key 分区，例如 `user_id`。

例子：

- key `1` 到 `100000` 放数据库 1。
- key `100001` 到 `200000` 放数据库 2。

优点：

- 扩展性好。
- 实现相对简单。

缺点：

- 对应用不透明，应用需要知道数据在哪个 shard。
- 跨 shard 查询和更新更复杂。
- 某个 shard 过载后迁移负载不容易。
- 数据库数量更多，故障概率上升，因此需要复制来保证可用性。

---

### 6.5 Key-Value Storage Systems

Key-value store 存储大量小记录，记录大小常为 KB 到 MB。

基本特点：

- 记录按 key 分区到多台机器。
- 系统把查询路由到合适机器。
- 记录复制到多台机器，以保证机器故障时仍可用。

可存储的数据形式：

- uninterpreted bytes，例如 Amazon S3、Amazon Dynamo。
- wide-table，即同一 key 下有任意多属性名，例如 BigTable、Cassandra、HBase、DynamoDB。
- JSON 文档，例如 MongoDB、CouchDB。

基本操作：

```text
put(key, value)
get(key)
delete(key)
```

有些系统支持 key 上的 range queries，document stores 还支持非 key 属性查询。

限制：

- 通常不是完整数据库系统。
- 对事务更新支持有限或没有支持。
- 应用需要自己管理更多查询处理逻辑。

这些限制换来更容易构建大规模可扩展存储系统，所以它们也常被称为 NoSQL systems。

---

### 6.6 Replication、Consistency 与 CAP

并行和分布式数据库需要 availability，即使部分机器故障也能运行。复制是实现 availability 的常用方法。

一致性要求：

- 所有 live replicas 有相同值。
- 每次 read 能看到最新版本。

常见做法是 majority protocol，例如有 3 个副本时，读写都访问 2 个副本。

网络分区出现时，系统可能分成多个不能互相通信的部分。CAP 的核心含义是：在 partition 存在时，不能同时保证 consistency 和 availability。

取舍：

- 传统数据库通常选择 consistency。
- 很多 Web 应用通常选择 availability。
- 订单处理等关键部分可能仍选择更强一致性。

---

## 7. MapReduce

### 7.1 MapReduce 思想

MapReduce 是可靠、可扩展的并行计算平台。程序员只提供核心逻辑，系统负责并行化、协调、失败处理等分布式问题。

编程模型：

- 输入是一组 key/value pairs。
- 用户提供 `map(k, v) -> list(k1, v1)`。
- 用户提供 `reduce(k1, list(v1)) -> v2`。
- `(k1, v1)` 是 intermediate key/value pair。
- 输出是 `(k1, v2)` 集合。

MapReduce 通常配合 distributed file system 或 key-value store 使用。

---

### 7.2 Word Count

问题：统计大量文档中每个单词出现次数。

思路：

- 把文档分给多个 workers。
- 每个 worker 解析文档，map 输出 `(word, 1)`。
- 系统按 word 对中间结果分区和 shuffle。
- reduce 对同一个 word 的所有计数求和。

伪代码：

```text
map(String record):
  for each word in record:
    emit(word, 1)

reduce(String key, List value_list):
  count = 0
  for each value in value_list:
    count = count + value
  output(key, count)
```

关键点：`emit` 的第一个属性是 reduce key。系统会根据 reduce key 做类似 `group by` 的操作，并把同 key 的 value list 交给 reduce。

---

### 7.3 Log Processing 例子

目标：给定日志文件，统计 `2013/01/01` 到 `2013/01/31` 之间 `/slide-dir/` 下每个文件被访问的次数。

map：

- 解析每条记录的 date、time、filename。
- 如果日期在范围内，且文件名以 `/slide-dir/` 开头，就 `emit(filename, 1)`。

reduce：

- 对同一个 filename 的记录求和。

适合 MapReduce 的原因：

- 顺序程序处理海量日志太慢。
- 先导入数据库可能成本高。
- 手写分布式并行程序工作量大。
- MapReduce 正好抽象了分布式并行执行。

---

### 7.4 Hadoop MapReduce

Google 推广了可运行在数千机器上的 MapReduce 实现，Hadoop 是开源实现。

Hadoop 通常使用 HDFS：

- 输入可以是 Text / CSV。
- 也可以是 Avro、ORC、Parquet 等压缩或列式格式。
- 也能配合 HBase、Cassandra、MongoDB 等 key-value stores。

Hadoop 中 Mapper 和 Reducer 接口都有四个类型参数：

- map input key type
- map input value type
- map output key type
- map output value type

---

### 7.5 MapReduce vs Databases

MapReduce 广泛用于并行处理，但和数据库系统取向不同。

MapReduce 的优势：

- 容易直接处理文件系统或 key-value store 中的数据。
- 对大规模机器故障更友好。
- 程序员可以写任意处理逻辑。

数据库的优势：

- 关系操作如 select、project、join、aggregation 都有成熟优化。
- 声明式查询更简洁。
- 查询优化器能自动选择执行计划。

许多关系操作可以用 MapReduce 表达，但手写 MapReduce 往往比 SQL 麻烦。

---

## 8. Beyond MapReduce: Spark 与流处理

### 8.1 Algebraic Operations 与 Spark

新一代执行引擎原生支持 joins、aggregation 等代数操作，例如 Apache Tez、Spark。

Spark 的核心抽象：

- **RDD**：Resilient Distributed Dataset，分布在多台机器上的记录集合。
- RDD 可以由其他 RDD 通过代数操作产生。
- RDD lazy computation，只有遇到 `saveAsTextFile()`、`collect()` 等 action 时才真正执行。
- 执行前可以在 operator tree 上做 query optimization。

Spark 支持 Java、Scala、R 等语言，常用 lambda 作为 `map`、`reduce` 等函数参数。

Spark DataFrames / DataSet：

- `DataSet<Row>` 表示有属性名的行类型。
- 支持 `filter`、`join`、`groupBy`、`agg` 等操作。
- 这些操作可以并行执行。

示例：

```java
Dataset<Row> instructor = spark.read().parquet("...");
Dataset<Row> department = spark.read().parquet("...");

instructor
  .filter(instructor.col("salary").gt(100000))
  .join(department,
        instructor.col("dept_name").equalTo(department.col("dept_name")))
  .groupBy(department.col("building"))
  .agg(count(instructor.col("ID")));
```

---

### 8.2 Streaming Data

Streaming data 指连续到达的数据，和 data-at-rest 相对。

应用：

- 股票交易流。
- 电商购买和搜索行为。
- IoT 传感器读数。
- 网络监控数据。
- 社交媒体帖子流。

流查询用途：

- 监控。
- 告警。
- 自动触发动作。

---

### 8.3 查询流数据

常见方法：

- **Windowing**：把流切成窗口，在窗口上运行查询。窗口可以按时间或 tuple 数量定义。
- **Continuous queries**：持续输出和更新目前为止的部分结果。
- **Algebraic operators on streams**：算子消费输入流并输出新流，可能维护状态。
- **Pattern matching**：检测事件模式并触发动作，属于 CEP。

窗口类型：

- **Tumbling window**：固定窗口，不重叠。例如每小时一个窗口。
- **Hopping window**：窗口大小固定，但按更短间隔滑动。例如一小时窗口每 20 分钟计算一次。
- **Sliding window**：围绕每个新 tuple 的指定大小窗口。
- **Session window**：按用户 session 分组。

Azure Stream Analytics 风格例子：

```sql
select item,
       System.Timestamp as window_end,
       sum(amount)
from orders timestamp by datetime
group by itemid, tumblingwindow(hour, 1);
```

窗口聚合的结果是 relation。许多系统支持 stream-relation join。stream-stream join 通常需要指定匹配 tuple 的 timestamp gap 上界，例如最多相差 30 分钟。

---

## 9. Graph Databases

### 9.1 Graph Data Model

图是一种非常通用的数据模型。企业的 ER 模型也可以看作图：

- entity 是 node。
- binary relationship 是 edge。
- ternary 和更高阶 relationship 可转成二元关系表示。

图也可以用关系表示：

```text
node(ID, label, node_data)
edge(fromID, toID, label, edge_data)
```

但纯关系表示边遍历不够方便。图数据库如 Neo4J 可以提供更自然的图视图，并用图查询语言表达 traversal。

Neo4J 风格例子：

```cypher
match (i:instructor)<-[:advisor]-(s:student)
where i.dept_name = 'Comp. Sci.'
return i.ID as ID, i.name as name, collect(s.name) as advisees;
```

`match` 子句匹配图中的节点和边，也可以做递归边遍历。

---

### 9.2 Parallel Graph Processing

超大图可能有 billions of nodes 和 trillions of edges，例如：

- Web graph：网页是节点，超链接是边。
- Social network graph：用户是节点，好友关系是边。

图计算通常需要多轮迭代。MapReduce 或一般代数框架每轮开销可能较高，因此图处理常使用 BSP。

---

### 9.3 Bulk Synchronous Processing

BSP，即 Bulk Synchronous Processing。

基本思想：

- 每个 vertex 有自己的 state。
- 顶点被分区到多台机器，节点状态通常保存在内存中。
- 程序员提供每个节点执行的方法。
- 节点方法可以向邻居发送消息，也可以接收邻居消息。
- 计算由多个 supersteps 组成。
- 每个 superstep 中，各节点并行执行，然后在全局同步点等待下一轮。

相关系统：

- Google Pregel 推广了 BSP 图处理框架。
- Apache Giraph 是 Pregel 的开源版本。
- Spark GraphX 提供 Pregel-like API。

---

## 10. 总结主线

这三章可以连成一条线：

- **复杂数据类型**：传统关系模型不够直接，于是引入 nested relations、structured types、XML、JSON、RDF、text、spatial data。
- **应用开发**：真实用户通常通过 Web 应用访问数据库，因此需要分层架构、session、ORM、Web services、性能优化和安全控制。
- **大数据**：当数据规模、速度和类型超过传统系统舒适区时，需要 HDFS、sharding、key-value stores、MapReduce、Spark、streaming 和 graph processing。

最容易考的对比：

- OODB / ORDB / ORM 的区别。
- XML / JSON / RDF 的用途差异。
- Precision / Recall、TF-IDF、PageRank。
- Cookie / Session 为什么需要。
- REST 与 Big Web Services。
- Sharding、HDFS、Key-value store 的优缺点。
- CAP 中 partition 下 consistency 和 availability 的取舍。
- MapReduce 的 `map`、`shuffle/group by`、`reduce` 流程。
- Spark 为什么比纯 MapReduce 更适合代数操作和迭代计算。
