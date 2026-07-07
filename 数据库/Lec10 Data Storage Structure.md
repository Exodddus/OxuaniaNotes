# Lec10 Data Storage Structures

本节讨论数据库文件内部如何组织 record、block、free space，以及 buffer manager 如何在主存和磁盘之间调度数据。

如果 Lec9 关注“存储设备本身”，那么本节关注“数据库怎样把关系表真正放到文件和页里”。

---

## 1. File Organization 的基本概念

数据库在物理层通常存储为一组 files。

层次关系可以写成：

```text
Database
  -> files
    -> records
      -> fields
```

- 一个 file 是一串 records；
- 一个 record 是一串 fields；
- 一个 relation 通常对应一个或多个 file；
- block 是磁盘和内存之间传输的基本单位。

课件先从最简单情况开始：

- record 长度固定；
- 一个 file 只存一种 record；
- record 小于一个 disk block。

---

## 2. Fixed-Length Records（定长记录）

如果每条记录长度都是 $n$ bytes，那么第 $i$ 条记录可以从下面位置开始存：

$$
n \times (i-1)
$$

这种方式访问简单，但有一个问题：record 可能跨越 block 边界。

常见修改是：

> 不允许一条 record 跨 block 存储。

这样会浪费一点空间，但读写更简单。

### 2.1 删除定长记录

删除第 $i$ 条记录后，有三种常见处理方式。

#### 方法一：整体前移

把第 $i+1,\dots,n$ 条记录依次向前移动。

优点：

- 文件保持紧凑；
- 顺序不变。

缺点：

- 移动成本高。

#### 方法二：用最后一条记录填补空洞

把最后一条记录移动到第 $i$ 条的位置。

优点：

- 移动成本低。

缺点：

- 原有顺序被破坏。

#### 方法三：free list

不移动记录，而是把空闲位置链接起来，形成 free list。

优点：

- 删除成本低；
- 插入新记录时可以复用空位。

缺点：

- 需要额外维护空闲链表。

---

## 3. Variable-Length Records（变长记录）

变长记录出现的原因：

- 一个 file 中存储多种 record type；
- 某些字段长度不固定，例如 `varchar`；
- 某些旧数据模型中存在 repeating fields；
- 存在 null 值。

### 3.1 变长属性的表示

常见做法是：

- 定长字段直接放在记录前部；
- 变长字段用固定大小的 `(offset, length)` 指向实际内容；
- 实际变长数据放在记录后部；
- null 值用 **null-value bitmap（空值位图）** 表示。

例如：

```text
instructor(ID, name, dept_name, salary)
```

如果 `name` 或 `dept_name` 是变长字符串，记录中可以先保存它们的偏移和长度，再在后部保存实际字符串。

---

## 4. Slotted Page Structure（分槽页结构）

变长记录通常用 **slotted page** 管理。

一个 page/block 内部包含：

- page header；
- free space；
- records。

header 中记录：

- 当前 page 中 record entries 的数量；
- free space 的结束位置；
- 每条 record 的位置和大小。

### 4.1 为什么需要 slotted page

变长记录删除或更新后，页内会出现碎片。slotted page 允许系统在页内移动 record，使它们保持连续。

关键点：

> 外部指针不应该直接指向 record 的物理位置，而应该指向 header 中的 slot entry。

这样 record 在 page 内移动时，只需要更新 slot entry，不需要修改所有外部指针。

---

## 5. Organization of Records in Files

课件列出几种典型文件组织方式：

- heap file organization；
- sequential file organization；
- multitable clustering file organization；
- B+-tree file organization；
- hashing file organization。

---

## 6. Heap File Organization

heap file 中，record 可以放在文件中任何有空闲空间的位置。

特点：

- 插入灵活；
- record 一旦分配后通常不移动；
- 需要高效找到可用 free space。

### 6.1 Free-Space Map

free-space map 用于记录每个 block 的空闲程度。

例如每个 block 对应一个几 bit 的 entry，记录该 block 大约有多少比例空闲。

还可以有 second-level free-space map：

- 第一层记录每个 block 的空闲程度；
- 第二层记录若干第一层 entry 的最大值；
- 查找足够大的空闲块时可以更快定位。

free-space map 会周期性写回磁盘。即使其中一些值过期，也可以接受，因为真正插入时会发现并修正。

---

## 7. Sequential File Organization

sequential file 根据某个 **search key** 的值按顺序存储 records。

适合：

- 需要顺序处理整个文件的应用；
- 范围查询；
- 根据 search key 顺序扫描。

### 7.1 删除和插入

删除：

- 可以使用 pointer chain 维护逻辑顺序。

插入：

1. 找到应该插入的位置；
2. 如果目标位置附近有空闲空间，直接插入；
3. 如果没有空间，插入 overflow block；
4. 更新 pointer chain。

问题：

- 随着插入和删除，文件会逐渐失去物理顺序；
- 需要定期 reorganize，恢复顺序。

---

## 8. Multitable Clustering File Organization

multitable clustering 会把多个关系中相关的记录放在同一个文件甚至同一个 block 附近。

例如把：

```text
department
instructor
```

按院系聚簇存储，使某个 department 及其 instructors 放得很近。

### 8.1 优点

适合：

- 经常查询某个 department 及其所有 instructors；
- 经常做 `department join instructor`。

这样可以减少 join 时的 I/O。

### 8.2 缺点

不适合：

- 只扫描 `department`；
- 只扫描某一个单独关系。

因为不同表的记录混在一起，单表扫描可能读入很多不需要的数据。

另外，多表聚簇会导致记录大小更不规则，因此可能需要额外 pointer chain 来链接同一关系的记录。

---

## 9. Table Partitioning

table partitioning 把一个关系的 records 分成若干更小的部分，分别存储。

例如：

```text
transaction_2018
transaction_2019
transaction_2020
```

逻辑上仍然可以把它们看成一个 `transaction` 表。

### 9.1 分区的好处

- 某些查询只访问部分分区，例如 `where year = 2019`；
- 降低 free space 管理成本；
- 不同分区可以放在不同设备上：
  - 当前年份数据放 SSD；
  - 历史数据放磁盘。

### 9.2 注意

如果查询没有分区筛选条件，仍可能需要访问所有分区。

---

## 10. Data Dictionary / System Catalog

data dictionary 又叫 system catalog，用于存储 metadata，即“关于数据的数据”。

它保存：

- relation 信息：
  - relation 名称；
  - attribute 名称、类型、长度；
  - view 的名称和定义；
  - integrity constraints；
- 用户和权限信息；
- 统计信息：
  - 每个 relation 的 tuple 数；
  - distinct value 数等；
- 物理文件组织信息：
  - relation 如何存储；
  - 文件物理位置；
- index 信息。

查询优化器高度依赖 system catalog 中的统计信息。

---

## 11. Storage Access 与 Buffer Manager

### 11.1 Buffer

buffer 是主存中用于保存 disk block 副本的一块区域。

数据库系统希望：

> 尽量减少 disk 和 memory 之间的 block transfer。

### 11.2 Buffer Manager

buffer manager 负责管理主存 buffer。

当程序需要某个 block：

1. 如果 block 已在 buffer 中，直接返回其内存地址；
2. 如果不在：
   1. 为它分配一个 buffer frame；
   2. 如果 buffer 已满，选择某个 block 替换；
   3. 如果被替换 block 是 dirty block，先写回磁盘；
   4. 从磁盘读入目标 block；
   5. 返回内存地址。

---

## 12. Pinned Block 与 Lock

### 12.1 Pinned Block

被 pin 的 block 不允许被替换出 buffer。

典型规则：

- 读写某个 block 前先 pin；
- 操作完成后 unpin；
- 可以维护 pin count；
- 只有当 `pin_count = 0` 时，block 才能被 evict。

### 12.2 Shared / Exclusive Lock

buffer 中的 page 也需要并发控制。

- shared lock：允许多个读者同时读；
- exclusive lock：写或重组 page 时独占。

规则：

- 同一时刻只能有一个 exclusive lock；
- shared lock 不能和 exclusive lock 并存；
- 多个 shared lock 可以并存。

---

## 13. Buffer-Replacement Policies

当 buffer 满了，需要决定替换哪个 block。

### 13.1 LRU

LRU（Least Recently Used）替换最久没有使用的 block。

直觉：

> 过去最近使用过的数据，将来也可能很快被使用。

但是 LRU 对某些数据库访问模式并不理想。

例如 nested-loop join 中反复扫描一个大表，LRU 可能不断把之后还要用的数据挤出去。

### 13.2 Toss-Immediate

某个 block 的最后一个 tuple 处理完后，立即释放该 block 占用的空间。

适合确定不会再访问该 block 的顺序扫描场景。

### 13.3 MRU

MRU（Most Recently Used）替换最近使用的 block。

对于某些循环扫描场景，MRU 反而比 LRU 更合适。

### 13.4 Query Optimizer Hints

数据库查询有较明确的访问模式，因此 buffer manager 可以使用查询优化器提供的信息，而不是机械地使用单一 LRU。

---

## 14. Clock Algorithm

Clock algorithm 是 LRU 的近似实现。

思想：

- 所有 buffer frame 排成一个环；
- 每个 frame 有一个 `reference_bit`；
- 当 block 的 pin count 降到 0 时，设置 `reference_bit = 1`；
- 替换时扫描环：
  - 如果 `reference_bit = 1`，把它置为 0，给第二次机会；
  - 如果 `reference_bit = 0`，选择该 block 替换。

Clock 的优点：

- 不需要维护完整 LRU 链表；
- 实现简单；
- 性能接近 LRU。

---

## 15. Column-Oriented Storage

传统 row-oriented storage 按行存：

```text
(ID, name, dept_name, salary)
(ID, name, dept_name, salary)
...
```

column-oriented storage 按列存：

```text
ID column
name column
dept_name column
salary column
```

### 15.1 优点

- 如果查询只访问少数属性，可以显著减少 I/O；
- 同一列数据类型相同，压缩效果更好；
- CPU cache 利用率更高；
- 适合 vector processing；
- 对决策支持、分析型查询更高效。

### 15.2 缺点

- 重构完整 tuple 成本高；
- 删除和更新成本高；
- 可能需要解压缩成本。

### 15.3 适用场景

| 存储方式 | 更适合 |
| --- | --- |
| Row-oriented | OLTP，频繁插入、更新、按行访问 |
| Column-oriented | OLAP，分析查询，扫描少量列 |

一些系统同时支持两种表示，称为 hybrid row/column stores。

常见列式文件格式：

- ORC；
- Parquet。

它们广泛用于 Hadoop / HDFS 和大数据分析场景。

---

## 16. 本节速记

- 定长记录访问简单，删除时可前移、尾记录填补或 free list。
- 变长记录常用 `(offset, length)` 和 null bitmap。
- slotted page 通过 slot entry 间接定位 record，方便页内移动记录。
- heap file 插入灵活，但需要 free-space map。
- sequential file 适合顺序/范围访问，但插入删除会产生 overflow，需要重组。
- multitable clustering 对 join 友好，对单表扫描不一定友好。
- partitioning 可以减少部分查询和管理成本。
- data dictionary 存 metadata 和统计信息，是优化器的重要依据。
- buffer manager 负责在内存和磁盘之间缓存 block。
- LRU 不总是适合数据库，Clock 是常见近似算法。
- 列式存储适合分析查询，行式存储适合事务处理。
