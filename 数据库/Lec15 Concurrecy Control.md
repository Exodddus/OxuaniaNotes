# Lec15 Concurrecy Control

## 1. Lock-Based Protocols

- 这一讲的核心问题是：如何用 **locking protocol** 在并发执行下保证正确性。
- 有两种lock，分别为 X-lock 和 S-lock
	- S-lock：只允许读操作，多个事务可以同时持有同一个数据项的 S 锁
	- X-lock：允许读写操作，只要一个事务拿到 X 锁，其他任何事务都不能再加 S 锁或 X 锁。
- 仅仅“加锁”本身并不够，关键在于：
  - 锁加在什么对象上；
  - 什么时候申请锁；
  - 什么时候释放锁；
  - 是否允许升级/降级；
  - 如何处理死锁与恢复性。

一个简单的反例是：

```text
T2: lock-S(A); read(A); unlock(A);
    lock-S(B); read(B); unlock(B);
    display(A+B)
```

- 上面的做法不能保证 serializability。
- 原因是：在 `read(A)` 和 `read(B)` 之间，别的事务可能更新了 `A` 或 `B`，最终 `A+B` 显示出来的和就是错的。
- 所以，数据库系统真正需要的是一整套 **locking protocol**，而不是零散地给单个操作上锁。

---

## 2. Deadlock

- 当两个或多个事务都在等待对方释放资源时，就会出现 **deadlock**。
- 典型场景：
  - `T3` 持有 `B` 的锁，想申请 `A`；
  - `T4` 持有 `A` 的锁，想申请 `B`；
  - 两者都无法继续执行。

### 2.1 Wait-for Graph

- 死锁通常用 **wait-for graph** 描述。
- 图中：
  - 顶点是事务；
  - 有向边 `Ti -> Tj` 表示 `Ti` 正在等待 `Tj` 释放某个数据项。

结论：

- 系统处于死锁状态，当且仅当 wait-for graph 中存在环。
- 因此系统需要周期性运行死锁检测算法，检查图中是否有 cycle。

### 2.2 死锁的处理

- 一旦检测到死锁，就必须选择至少一个事务回滚。
- 回滚后释放其持有的锁，其他事务才能继续前进。
- 本质上，这是用“牺牲一个事务”的代价来打破循环等待。

---

## 3. Two-Phase Locking (2PL)

- **Two-phase locking** 是最经典的基于锁的并发控制协议。
- 它保证生成的调度是 **conflict-serializable**。

### 3.1 两个阶段

#### Phase 1: Growing Phase

- 事务可以申请锁；
- 事务不能释放锁。

#### Phase 2: Shrinking Phase

- 事务可以释放锁；
- 事务不能再申请新锁。

- 一个事务从“还能加锁”切换到“只能释放锁”的那个时刻，叫做 **lock point**。

### 3.2 为什么 2PL 能保证可串行化

课件给出的核心思路是：

- 取 lock point 最早的事务 `Ti`；
- 如果其他事务 `Tj` 的某个操作阻塞了 `Ti` 的操作，那么 `Tj` 的 lock point 必定晚于 `Ti`；
- 由此可以建立一个等价的串行顺序。

直观理解就是：

- 2PL 通过限制“加锁/释放锁的顺序”，限制了合法 schedule 的集合；
- 因而冲突关系可以被组织成某个串行顺序。

2PL只是提供了一种可行的串行化顺序，而不是唯一的方法。

---

## 4. Extensions of Basic 2PL

- 基本 2PL 保证 conflict serializability；
- 但它还不一定保证 **recoverability**，也不一定避免 **cascading rollback**；
- 所以课件进一步介绍了更强的变体。

### 4.1 Strict 2PL

- **Strict 2PL**：事务必须一直持有所有 **exclusive locks**，直到 `commit/abort`。

作用：

- 保证 recoverability；
- 避免 cascading rollback。

### 4.2 Rigorous 2PL

- **Rigorous 2PL**：事务必须一直持有 **所有锁**，直到 `commit/abort`。
- 不仅 X-lock 不提前释放，S-lock 也不提前释放。

性质：

- 比 strict 2PL 更强；
- 事务可以按 **commit 的顺序** 串行化；
- 很多数据库系统实现的其实更接近 rigorous 2PL，但口头上仍然简称为 2PL。

---

## 5. Lock Conversion

- 课件还介绍了带锁转换的 2PL。

### 5.1 升级与降级

在 Growing Phase：

- 可以申请 `lock-S` 或 `lock-X`；
- 可以把 `S` 锁升级为 `X` 锁，也就是 **lock upgrade**。

在 Shrinking Phase：

- 可以释放锁；
- 可以把 `X` 锁降级为 `S` 锁，也就是 **lock downgrade**。

### 5.2 直观意义

- upgrade 适合“先读后改”的事务；
- downgrade 适合“先独占修改，再允许别人读”的场景。

- 这个协议仍然保证 serializability，因为它仍保留了 2PL 的核心结构：先扩张，再收缩。

---

## 6. Deadlock Prevention

slides P22

除了“检测后回滚”，还可以主动设计规则，尽量不让死锁发生。

### 6.1 Wait-Die Scheme

- 基于时间戳。
- **Older transaction may wait for younger**。
- **Younger transaction never waits for older**，而是直接回滚。
- 这是 **non-preemptive** 的方案。

特点：

- 年轻事务可能反复回滚很多次；
- 但由于重启时保留原时间戳，老事务总是优先，因此不会永久饥饿。

### 6.2 Wound-Wait Scheme

- 也是基于时间戳。
- **Older transaction wounds younger**：老事务不会等年轻事务，而是强制对方回滚。
- **Younger transaction may wait for older**。
- 这是 **preemptive** 的方案。

特点：

- 可能比 wait-die 产生更少的回滚；
- 同样通过“保留原时间戳”避免 starvation。

### 6.3 核心思想

- 两种方案本质上都在用“按年龄建立偏序”；
- 一旦等待关系只能沿某个固定方向发生，就不容易形成 cycle。

---

## 7. Graph-Based Protocols

- **Graph-based protocols** 是 2PL 的一种替代方案。
- 基本思路是：先对所有数据项建立一个 **partial ordering**。

设数据项集合为：

```text
D = {d1, d2, ..., dh}
```

如果规定：

```text
di < dj
```

那么任何同时访问 `di` 和 `dj` 的事务，都必须先访问 `di` 再访问 `dj`。

结果：

- 整个数据集合可以看成一个有向无环图；
- 事务只允许按图的方向访问数据；
- 因而不会形成某些冲突结构。

直观上，它是在“访问顺序”层面预先消灭一部分危险调度。

---

## 8. Multiple Granularity Locking

- 锁并不一定只加在 tuple 上。
- 数据项可以有不同粒度，因此需要 **multiple granularity locking**。

### 8.1 层次结构

典型的粒度层次从粗到细可以是：

- database
- area
- file
- relation / table
- record

这些粒度形成一棵层次树：

- 上层粒度大；
- 下层粒度小；
- 显式锁住某个节点时，它的后代也会被隐式锁住相应模式。

### 8.2 粗粒度 vs 细粒度

**Fine granularity**：

- 更高并发；
- 更高的加锁开销。

**Coarse granularity**：

- 更低加锁开销；
- 更低并发性。

所以多粒度锁的核心 trade-off 就是：

- `concurrency` 和 `locking overhead` 之间的平衡。

### 8.3 Intention Lock Modes

多粒度锁里只使用 `S` 和 `X` 还不够，因为一个事务如果想锁住某个较低层节点，例如某条 record，系统需要在它的祖先节点上留下“我要在下面加锁”的标记。这个标记就是 **intention lock**。

直观上：

- 普通 `S / X` 锁表示“我正在锁这个节点本身”；
- intention lock 表示“我准备在这个节点的某些后代上加锁”。

课件中给出的三种主要 intention lock modes 是：

- **IS**：Intention Shared。表示事务打算在该节点的某些后代上加 `S` 锁。
- **IX**：Intention Exclusive。表示事务打算在该节点的某些后代上加 `X` 锁。
- **SIX**：Shared and Intention Exclusive。表示事务已经以 `S` 模式锁住当前节点，同时还打算在某些后代上加 `X` 锁。

举例来说，如果一个事务要对某个 record 加 `S` 锁，那么它需要先在从 root 到该 record 父节点的路径上加 `IS` 锁，再在 record 上加 `S` 锁。类似地，如果要对某个 record 加 `X` 锁，就需要先在祖先节点上加 `IX` 锁，再在目标 record 上加 `X` 锁。

常见的兼容关系可以记成：

|      | IS | IX | S | SIX | X |
| --- | --- | --- | --- | --- | --- |
| IS | true | true | true | true | false |
| IX | true | true | false | false | false |
| S | true | false | true | false | false |
| SIX | true | false | false | false | false |
| X | false | false | false | false | false |

这个表的直观解释是：

- `IS` 最宽松，因为它只是声明“以后可能在下面读”；
- `IX` 可以和 `IS / IX` 共存，因为多个事务都可以声明将来要在不同后代上修改；
- `S` 和 `IX` 不兼容，因为 `S` 锁住了整个当前节点，而 `IX` 暗示某个后代可能被修改；
- `SIX` 已经包含当前节点上的 `S`，同时又带有向下修改的意图，所以兼容性更弱；
- `X` 最强，几乎和所有其他锁都冲突。

多粒度锁协议通常要求：

- 对某节点加 `S` 锁之前，必须先对其父节点加 `IS` 或更强的锁；
- 对某节点加 `X` 锁之前，必须先对其父节点加 `IX` 或更强的锁；
- 解锁时则按相反方向，从子节点到父节点释放。

这样做的好处是：系统不用扫描一个节点下所有后代是否被锁，只需检查该节点上的 intention lock，就可以判断粗粒度锁请求是否会和细粒度锁冲突。

---

## 9. Index Locking and Phantom

- 课件特别提到 **index locking protocol**，这是处理 phantom problem 的重要办法。

### 9.1 为什么要锁索引

- 关系上的 tuple 往往通过索引来定位；
- 如果只锁住现有记录，而不锁住相关 index leaf nodes，那么范围查询时依然可能有新的元组插入进来；
- 这就会出现 **phantom phenomenon**。

### 9.2 规则

- 查找事务需要对访问到的索引叶子节点加 `S-lock`；
- 即使某个叶子节点里没有满足条件的记录，也可能需要锁住；
- 插入、删除、更新事务要对受影响的索引叶子节点加 `X-lock`；
- 同时仍需遵守 2PL。

结果：

- 幻读问题不会仅靠“锁现有记录”漏掉；
- 因为连“可能插入新记录的位置”也被控制住了。

---

## 10. Timestamp-Ordering Protocol

- 除了锁协议，另一大类并发控制方法是 **timestamp-based protocols**。
- 核心思想：让所有冲突操作都按事务时间戳的顺序执行。

### 10.1 基本对象

每个事务 `Ti` 有时间戳 `TS(Ti)`。

每个数据项 `Q` 维护：

- `R-timestamp(Q)`：读过 `Q` 的事务中最大的时间戳；
- `W-timestamp(Q)`：写过 `Q` 的事务中最大的时间戳。

### 10.2 Read Rule

如果事务 `Ti` 发出 `read(Q)`：

- 若 `TS(Ti) < W-timestamp(Q)`，
  - 说明 `Ti` 想读的是一个已经被“更年轻写入”覆盖掉的旧值；
  - 该读非法，`Ti` 必须回滚。
- 否则，
  - 读操作可以执行；
  - 并更新

```text
R-timestamp(Q) = max(R-timestamp(Q), TS(Ti))
```

### 10.3 Write Rule

写规则与读规则类似，本质也是：

- 如果某个写会违反时间戳顺序，就拒绝并回滚事务；
- 只有不破坏时间先后关系的操作才被允许。

### 10.4 直观理解

- Lock-based protocol 是“通过等待”维持顺序；
- Timestamp ordering 是“通过拒绝非法操作”维持顺序。

---

## 11. Multiversion Concurrency Control (MVCC)

- 课件接着介绍了 **multiversion** 的思想。
- 每次写入不一定覆盖旧版本，而是创建新版本。

### 11.1 基本思路

- 读事务可以读取适合自己的版本；
- 写事务创建新版本；
- 这样读和写之间的冲突显著减少。

### 11.2 多版本的优势

- 特别适合：
  - 大量读的决策支持查询；
  - 与少量更新的 OLTP 事务并发执行。

- 如果所有读都用普通锁，很容易和更新事务冲突，性能差；
- MVCC 的目标就是让“读尽量不阻塞写，写也尽量不阻塞读”。

---

## 12. Snapshot Isolation (SI)

- 课件重点讲了 **snapshot isolation**。
- 它并不等于真正的 serializable，但在工程实践中很常见。

### 12.1 基本思想

- 每个事务看到的是某个时间点上的 **snapshot**；
- 读操作从快照读数据，因此：
  - 读通常不会被阻塞；
  - 读也通常不会阻塞别的事务。

- 更新事务之间则需要检查并发写冲突。

### 12.2 SI 的优点

课件总结为：

- No dirty read
- No lost update
- No non-repeatable read
- Predicate-based selects are repeatable（课件写成“no phantoms”风格的效果）

而且：

- 性能和 `Read Committed` 接近；
- 读性能通常很好。

### 12.3 SI 的问题

- SI **不一定产生 serializable execution**。
- 在真正的 serializable 中：
  - 并发事务之间，至少有一个会看到另一个的效果。
- 但在 SI 中：
  - 可能两个事务谁也看不到谁的更新；
  - 结果是完整性约束仍可能被破坏。

所以：

- **SI 比 Read Committed 强，但比真正 Serializable 弱。**

### 12.4 First-Committer-Wins / First-Updater-Wins

课件提到：

- 有的系统用 **first committer wins**；
- Oracle 的变体更接近 **first updater wins**：
  - 并发写冲突在写时就检查；
  - 不必等到提交时才发现问题；
  - 可以更早回滚事务。

### 12.5 工程上的提醒

- Oracle 以及 PostgreSQL 9.1 之前的 “serializable” 并不是真正 textbook 意义上的 serializable；
- PostgreSQL 9.1 引入了 **Serializable Snapshot Isolation (SSI)**，才更接近真正可串行化。

---

## 13. Select for Update

- 对某些查询，系统可以通过 `select ... for update` 绕开部分 SI 的问题。

例如：

```sql
select max(orderno) from orders for update;
```

含义：

- 把“读取到的数据”也当成要更新的数据来处理；
- 从而阻止并发事务同时基于同一个旧值继续更新。

但注意：

- `select for update` 并不总能完全保证 serializability；
- phantom problem 仍可能存在；
- 不同数据库对它的实现细节也不同。

---

## 14. Application-Level Concurrency Control

- 现实中的很多应用事务会跨越用户交互过程；
- 这时不能长期持有数据库锁，也不希望长期占用数据库连接；
- 所以需要 **application level concurrency control**。

### 14.1 Version Number

常见做法是：

- 每个 tuple 带一个 `version` 字段；
- 事务读数据时记下当时的 `version`；
- 更新时检查当前版本号是否仍和读时一致。

示意：

```sql
select r.balance, r.version
from r
where acctId = 23;
```

更新时：

```sql
update r
set r.balance = r.balance + :deposit
where acctId = 23
  and r.version = :version;
```

### 14.2 本质

- 这本质上类似一种简化版的 optimistic concurrency control；
- 它通常不验证完整 read set，只验证被写回对象的版本；
- 很多 ORM 框架（如 Hibernate）内部就采用这种思路。

### 14.3 和 SI 的区别

- 版本号控制也可用来支持 first-committer-wins 风格的检查；
- 但它不保证所有读取都来自同一个统一 snapshot；
- 所以它和标准 SI 不完全一样。

---

## 15. This Lecture in One Line

这一讲可以串成下面这条主线：

- **基本锁还不够**，必须有协议；
- **2PL** 解决 conflict serializability；
- **strict / rigorous 2PL** 进一步解决 recoverability 和 cascading rollback；
- **deadlock** 是锁协议的副作用，需要预防或检测；
- **timestamp ordering** 提供了另一套思路；
- **MVCC / SI** 是现代数据库里非常重要的工程实现；
- **真正的高性能并发控制** 往往是 textbook 理论和系统工程折中的结果。

---

## 16. 最后要记住的几个点

- **2PL 保证冲突可串行化，但不天然保证无级联回滚。**
- **Strict 2PL**：X 锁持有到提交/回滚。
- **Rigorous 2PL**：所有锁持有到提交/回滚。
- **Deadlock iff wait-for graph has a cycle.**
- **Timestamp ordering** 不是靠等待，而是靠回滚非法顺序。
- **MVCC** 让读写冲突变少，是现代数据库的核心机制之一。
- **Snapshot Isolation 很强，但不一定真正 serializable。**
- 应用层也可以用 **version number** 做乐观并发控制。
