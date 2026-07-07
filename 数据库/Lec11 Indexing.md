# Lec11 Indexing

索引的目标是加速数据访问。它本质上是一种额外的数据结构，用空间和维护成本换取更快的查找、范围查询、排序或连接。

可以把索引理解成图书馆目录：不需要从第一本书翻到最后一本书，而是先查目录，再定位目标位置。

---

## 1. Basic Concepts

### 1.1 Search Key

**search key** 是用于查找记录的一个属性或一组属性。

注意：

- search key 不一定是 candidate key；
- search key 可以有重复值；
- 例如按 `salary` 建索引时，很多教师可能有相同 salary。

### 1.2 Index Entry

索引文件由 index entries 组成，每个 entry 通常形如：

```text
search-key value -> pointer
```

pointer 可以指向：

- 数据记录；
- 包含若干记录指针的 bucket；
- B+-tree 的子节点；
- 其他定位结构。

索引文件通常比原始数据文件小很多。

---

## 2. Index Evaluation Metrics

评价一个索引时看：

- **支持的访问类型**
  - point query：查某个具体值；
  - range query：查某个范围。
- **access time**
- **insertion time**
- **deletion time**
- **space overhead**

索引并非越多越好。索引会加速查询，但插入、删除、更新时也要维护索引。

---

## 3. Ordered Indices

ordered index 中，index entries 按 search key 排序。

### 3.1 Primary Index / Clustering Index

在一个已经按某 search key 顺序存储的文件上，如果索引的 search key 也决定文件的顺序，则该索引称为：

- primary index；
- clustering index。

注意：

> primary index 的 search key 通常是 primary key，但不一定必须是 primary key。

### 3.2 Secondary Index / Non-Clustering Index

如果索引的 search key 顺序和文件的物理顺序不同，则称为：

- secondary index；
- non-clustering index。

secondary index 对点查询很有用，但如果匹配多个记录，可能产生大量随机 I/O。

### 3.3 Index-Sequential File

带有 primary index 的 ordered sequential file 称为 index-sequential file。

---

## 4. Dense Index 与 Sparse Index

### 4.1 Dense Index（稠密索引）

dense index 为文件中每一个 search-key value 建一个 index record。

如果 search key 唯一，则每条记录对应一个 key entry。

如果 search key 不唯一，entry 可以指向一组记录指针。

优点：

- 查找快；
- 适合 secondary index。

缺点：

- 空间开销大；
- 插入删除维护成本高。

### 4.2 Sparse Index（稀疏索引）

sparse index 只为部分 search-key value 建 index record。

适用条件：

> 数据文件必须按 search key 顺序存储。

查找 key 为 $K$ 的记录时：

1. 在 index 中找到小于等于 $K$ 的最大 search-key value；
2. 从该 entry 指向的位置开始顺序扫描数据文件。

优点：

- 空间少；
- 维护成本低。

缺点：

- 查找通常比 dense index 慢。

常见折中：

> 每个数据 block 建一个 sparse index entry，记录该 block 中最小的 search-key value。

---

## 5. Multilevel Index

如果 primary index 太大，无法放入内存，则访问 index 本身也会变贵。

解决方法：

> 把磁盘上的 primary index 当成一个 sequential file，再在它上面建 sparse index。

术语：

- **inner index**：原 primary index；
- **outer index**：建在 inner index 上的 sparse index。

如果 outer index 仍然太大，可以继续增加层次。

B+-tree 可以看成 multilevel index 的系统化形式。

---

## 6. B+-Tree Index

B+-tree 是数据库系统中最重要的索引结构之一。

### 6.1 B+-Tree 的性质

设一个 B+-tree 节点最多有 $n$ 个指针。

B+-tree 满足：

- 所有从 root 到 leaf 的路径长度相同；
- 非根非叶节点有 $\lceil n/2\rceil$ 到 $n$ 个 children；
- leaf node 有 $\lceil (n-1)/2\rceil$ 到 $n-1$ 个 search-key values；
- 如果 root 不是 leaf，至少有 2 个 children；
- 如果 root 是 leaf，可有 0 到 $n-1$ 个 values。

这保证了树始终平衡，因此查找、插入、删除都是对数级。

### 6.2 B+-Tree Node Structure

一个节点通常形如：

```text
P1 K1 P2 K2 P3 ... Kn-1 Pn
```

其中：

- $K_i$ 是 search-key value；
- $P_i$ 是 pointer；
- key 有序：

$$
K_1 < K_2 < \cdots < K_{n-1}
$$

### 6.3 Leaf Node

leaf node 中：

- $K_i$ 是实际 search-key value；
- $P_i$ 指向数据记录或记录集合；
- 最后一个 pointer 通常指向下一个 leaf node。

leaf nodes 之间按 key 顺序串联，因此范围查询很方便。

### 6.4 Non-Leaf Node

non-leaf node 不直接指向数据记录，而是形成 leaf nodes 上方的 sparse index。

若一个 non-leaf node 中有：

```text
P1 K1 P2 K2 ... Kn-1 Pn
```

则：

- $P_1$ 指向 key 小于 $K_1$ 的子树；
- $P_i$ 指向 key 大于等于 $K_{i-1}$ 且小于 $K_i$ 的子树；
- $P_n$ 指向 key 大于等于 $K_{n-1}$ 的子树。

---

## 7. B+-Tree 查询

查找值 $v$：

1. 从 root 开始；
2. 在当前 non-leaf node 中找到应该进入的 child pointer；
3. 重复直到 leaf；
4. 在 leaf 中查找 $v$；
5. 若找到，则跟随 record pointer；否则不存在。

若有 $K$ 个 search-key values，fanout 为 $n$，树高大约为：

$$
\lceil \log_{\lceil n/2\rceil}(K)\rceil
$$

课件例子：

- 若 $n = 100$；
- 有 1,000,000 个 search-key values；
- 查找最多访问约 4 个节点。

这就是 B+-tree 相比二叉树适合磁盘的原因：每个节点对应一个 disk block，fanout 大，树高低。

---

## 8. B+-Tree 插入

插入 `(v, pointer)`：

1. 找到 $v$ 应该出现的 leaf node；
2. 如果 leaf 有空间，直接插入；
3. 如果 leaf 已满，split leaf；
4. 把新 leaf 的最小 key 插入 parent；
5. 如果 parent 也满，继续向上 split；
6. 最坏情况下 split 到 root，使树高增加 1。

### 8.1 Leaf Split

把包含新 entry 在内的 $n$ 个 `(key, pointer)` 排序：

- 前 $\lceil n/2\rceil$ 个留在原节点；
- 剩余放入新节点；
- 新节点的最小 key 复制到 parent 中作为分隔 key。

---

## 9. B+-Tree 删除

删除 `(v, pointer)`：

1. 从 leaf 中移除对应 entry；
2. 如果节点仍满足最小 occupancy，结束；
3. 如果节点太空：
   - 能和 sibling 合并，则 merge；
   - 不能合并，则 redistribute；
4. parent 中对应 key/pointer 也要更新；
5. 删除可能向上传播；
6. 如果 root 最后只剩一个 child，则删除 root，让 child 成为新 root。

### 9.1 Merge

如果当前节点和 sibling 的 entries 能放进一个节点，就合并。

合并后要从 parent 中删除指向被删除节点的 pointer 和分隔 key。

### 9.2 Redistribution

如果两个节点合并后放不下，就在当前节点和 sibling 之间重新分配 entries，使二者都满足最小 occupancy。

同时更新 parent 中的分隔 key。

---

## 10. B+-Tree 更新复杂度

插入和删除单个 entry 的 I/O 数与树高成正比：

$$
O(\log_{\lceil n/2\rceil}(K))
$$

实际系统中通常更快，因为：

- internal nodes 经常在 buffer 中；
- split / merge 并不频繁；
- 大多数插入删除只影响 leaf node。

平均 node occupancy 与插入顺序有关：

- 随机插入约 $2/3$；
- 按排序顺序插入可能约 $1/2$。

---

## 11. B+-Tree 高度与大小估算

课件例子：

```text
person(
  pid char(18) primary key,
  name char(8),
  age smallint,
  address char(40)
)
```

假设：

- block size = 4KB；
- 1,000,000 persons；
- record size = `18 + 8 + 2 + 40 = 68` bytes。

数据文件 records per block：

$$
\lfloor 4096 / 68 \rfloor = 60
$$

存储 1M 条记录大约需要：

$$
\lceil 1000000 / 60 \rceil = 16667
$$

若索引 entry 大小约为 key 18B + pointer 4B，则 non-leaf fanout 约：

$$
n = \lfloor (4096 - 4) / (18 + 4)\rfloor + 1 = 187
$$

由此可以估算 B+-tree 大概 3 层左右即可容纳百万级记录。

考试中重点不是死记数字，而是会按：

```text
block size / entry size -> fanout
记录数 / 每叶节点容量 -> leaf 数
leaf 数 / fanout -> 上层节点数
```

逐层估算。

---

## 12. B+-Tree File Organization

B+-tree index 的 leaf node 存的是 record pointers。

B+-tree file organization 的 leaf node 直接存 records。

优点：

- 数据本身按 key 聚簇；
- 插入、删除后仍能维持较好的顺序；
- 范围查询友好。

区别：

| 类型                        | leaf 中存什么                   |
| ------------------------- | --------------------------- |
| B+-tree index             | search key + record pointer |
| B+-tree file organization | actual records              |

因为 record 比 pointer 大，所以 leaf node 能容纳的记录数更少，空间利用率更重要。

课件提到可以在 split/merge 时引入更多 sibling 参与 redistribution，使节点至少约 $2n/3$ 满，提高空间利用率。

---

## 13. Record Relocation 与 Secondary Index

如果 secondary index 中直接保存 record pointer，那么当 record 移动时，所有指向它的 secondary index 都要更新。

这在 B+-tree file organization 中尤其麻烦，因为 split 可能移动记录。

解决办法：

> secondary index 不直接存 record pointer，而是存 primary-index search key。

查找时：

1. 用 secondary index 找到 primary key；
2. 再用 primary index 定位 record。

代价：

- 查询多一次 primary index traversal；
- 但 record 移动时不需要更新所有 secondary index。

如果 primary-index search key 不唯一，需要额外加 record-id 保证唯一。

---

## 14. Indexing Strings

字符串作为 key 时有两个问题：

- key 长度可变，fanout 也变；
- key 太长会降低一个节点能放的 entries 数。

常见优化是 **prefix compression**：

- internal node 只保存足以区分子树的前缀；
- leaf node 中共享公共前缀。

例如：

```text
Silas
Silberschatz
```

可能只需要前缀 `Silb` 就能作为分隔信息。

---

## 15. Multiple-Key Access

查询：

```sql
select ID
from instructor
where dept_name = 'Finance' and salary = 80000;
```

如果只有单属性索引，有三种策略：

1. 用 `dept_name` 索引找 Finance，再检查 salary；
2. 用 `salary` 索引找 80000，再检查 dept_name；
3. 分别用两个索引得到 record pointers，再求交集。

### 15.1 Composite Search Key

复合索引 search key 包含多个属性，例如：

```text
(dept_name, salary)
```

它按 lexicographic ordering 排序：

$$(a_1,a_2) < (b_1,b_2)$$

当且仅当：

- $a_1 < b_1$；或
- $a_1 = b_1$ 且 $a_2 < b_2$。

### 15.2 复合索引的使用规律

索引 `(dept_name, salary)` 可以高效支持：

```sql
where dept_name = 'Finance' and salary = 80000
```

也可以支持：

```sql
where dept_name = 'Finance' and salary < 80000
```

但不能高效支持：

```sql
where dept_name < 'Finance' and salary = 80000
```

原因是第一个属性是范围条件后，第二个属性难以继续精确定位。

这就是复合索引中常说的“左前缀”思想。

---

## 16. Non-Unique Search Keys

如果 search key 不唯一，可以把它扩展成唯一复合 key：

```text
(ai, Ap)
```

其中 $A_p$ 可以是：

- primary key；
- record ID；
- 其他能保证唯一性的属性。

查找 `ai = v` 可以变成范围查询：

$$
(v, -\infty) \text{ 到 } (v, +\infty)
$$

优点：

- 插入删除代码更简单；
- 每个 index entry 唯一。

代价：

- key 更长；
- 存储开销更大；
- 非聚簇索引可能需要更多随机 I/O。

---

## 17. Indexing in Main Memory

主存随机访问比磁盘便宜得多，但 cache miss 仍然昂贵。

传统大 B+-tree node 在内存中做 binary search 可能导致多次 cache miss。

内存索引设计要 cache conscious：

- 节点大小适合 cache line；
- 或者在大节点内部用小树结构组织 key；
- 尽量让一次 cache line 读取带来更多有用数据。

---

## 18. Bulk Loading 与 Bottom-Up Build

逐条插入大量 entries 到 B+-tree 非常低效，因为每条可能至少产生一次 I/O。

### 18.1 Sorted Insertion

先排序，再按顺序插入。

优点：

- I/O 性能比随机插入好很多。

缺点：

- leaf nodes 可能只有半满。

### 18.2 Bottom-Up B+-Tree Construction

更高效的方法：

1. 先排序所有 index entries；
2. 从 leaf level 开始逐层构造 B+-tree；
3. 顺序写入磁盘。

这种方式是数据库 bulk-load 工具常用方法。

---

## 19. Indexing on Flash

Flash / SSD 上随机 I/O 成本比磁盘低很多，但写入有特殊问题：

- 不能原地覆盖；
- 最终需要 erase；
- page size 的最优选择可能比磁盘小；
- bulk loading 仍然有用，因为能减少 page erase。

因此 flash 上经常使用 write-optimized index，例如 LSM-tree。

---

## 20. Write Optimized Indices

B+-tree 对写密集 workload 可能表现不好：

- 假设 internal nodes 在内存中；
- 每次插入仍可能需要访问一个 leaf；
- 磁盘上随机写代价高；
- flash 上每次覆盖可能引发 page rewrite / erase。

两类写优化结构：

- LSM-tree；
- Buffer tree。

---

## 21. LSM-Tree

LSM-tree 的核心思想：

> 先把写操作积累在内存中，再批量顺序写入磁盘。

### 21.1 基本流程

1. 新记录先插入内存中的 $L_0$ tree；
2. $L_0$ 满后，与磁盘上的 $L_1$ tree merge；
3. $L_1$ 超过阈值后，再 merge 到 $L_2$；
4. 依此类推。

一般 $L_{i+1}$ 的阈值是 $L_i$ 的 $k$ 倍。

### 21.2 优点

- 插入主要使用顺序 I/O；
- leaf 更满，空间浪费少；
- 写放大在一定范围内可控；
- 适合写密集系统和 SSD。

### 21.3 缺点

- 查询可能要查多个 levels；
- 同一内容在 merge 过程中可能被复制多次。

### 21.4 Stepped-Merge Index

stepped-merge index 是 LSM 的变体：

- 每层允许多个 trees；
- 当某层有 $k$ 个 trees 后，再合并到下一层。

优点：

- 写成本更低。

缺点：

- 查询更贵，因为要查更多 trees。

优化：

- 对每个 tree 建 Bloom filter；
- point lookup 时只有 Bloom filter 可能命中的 tree 才需要真正查。

### 21.5 删除和更新

LSM 中删除通常通过特殊 delete entry 表示。

查询时如果同时找到原记录和 delete entry，则认为记录已删除。

merge 时遇到匹配的原记录和 delete entry，可以一起丢弃。

更新可以看成：

```text
delete old + insert new
```

---

## 22. Buffer Tree

Buffer tree 的思想：

> 在 B+-tree 的每个 internal node 中加一个 buffer，用来暂存 insert 操作。

当 buffer 满时，批量把记录推向下一层。

优点：

- 相比普通 B+-tree，单条写入的 I/O 分摊成本更低；
- 查询开销比 LSM 小；
- 可和多种 tree index 结合。

缺点：

- 比 LSM-tree 有更多 random I/O。

---

## 23. Bitmap Indices

bitmap index 适合取值种类较少的属性，例如：

- gender；
- country；
- state；
- income level。

### 23.1 基本形式

对某个属性的每个取值建一个 bitmap。

如果 relation 有 $N$ 条记录，则每个 bitmap 有 $N$ bits。

某记录对应 bit 为：

- 1：该记录属性值等于该 bitmap 对应值；
- 0：否则。

### 23.2 位运算查询

bitmap index 特别适合多属性组合查询。

例如：

```text
100110 AND 110011 = 100010
100110 OR  110011 = 110111
NOT 100110 = 011001
```

可以用 AND / OR / NOT 快速完成交、并、补。

### 23.3 优点

- 对低基数属性空间很小；
- 位运算极快；
- 统计匹配条数很快。

例如一个 record 为 100 bytes，一个 bitmap 对 relation 大小约为：

$$
1 / 800
$$

如果属性只有 8 个不同值，所有 bitmap 加起来约占 relation 大小的 1%。

### 23.4 高效实现

bitmap 会打包成 machine word。

一个 CPU 指令可以同时对 32 或 64 bits 做 AND/OR。

统计 1 的个数可以用预计算表：

- 对每个 byte 的 256 种可能预存其中 1 的数量；
- 扫描 bitmap 时查表并求和。

---

## 24. 本节速记

- index 用空间和维护成本换访问速度。
- primary index / clustering index 的顺序与数据文件顺序一致。
- secondary index 顺序与文件物理顺序不同，可能带来随机 I/O。
- dense index 查找快但空间大；sparse index 空间小但要求文件有序。
- B+-tree 所有数据入口在 leaf，leaf 顺序相连，适合范围查询。
- B+-tree fanout 大，所以树高很低，适合磁盘。
- 插入可能 split，删除可能 merge 或 redistribute。
- B+-tree file organization 的 leaf 直接存 records。
- composite index 要注意属性顺序和左前缀。
- LSM-tree 适合写密集场景，但查询可能要查多个层级。
- bitmap index 适合低基数属性和多条件组合查询。
