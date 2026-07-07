# Lec16 Recovery System

> 来源：`ch19 - Recovery System.pptx`  
> 说明：隐藏/省略 PPT 已在对应小节标明。隐藏页包括：Slides 7-11, 16, 19, 21, 29-31, 36, 41-43, 48-52, 63, 68, 86。

隐藏页处理原则：

- 对第 7-11 页这类基础图示页，只保留核心概念。
- 对后续隐藏页，如果内容属于本章主线，例如 checkpoint、buffering、remote backup、ARIES optimization，则仍整理进笔记，但在标题或段落中标注来源为隐藏 slides。

## 1. 本讲主线

Recovery system 要解决的是：即使事务失败、系统崩溃、磁盘损坏，数据库也要尽量保持 **atomicity** 和 **durability**。

本讲可以串成这条线：

- 先区分 failure 类型，以及 volatile / nonvolatile / stable storage。
- 再看事务如何通过 buffer、private work-area、`read(X)`、`write(X)` 访问数据。
- 核心机制是 **log-based recovery**：先写日志，再允许数据库页面被写回。
- 基本恢复算法采用 **repeating history**：先 redo 到崩溃前状态，再 undo 未完成事务。
- Checkpoint、buffering、fuzzy checkpoint 用来降低恢复成本。
- 更高级的恢复需要支持 early lock release、logical undo。
- ARIES 是实际系统中重要的恢复算法框架。

---

## 2. Failure Classification

数据库系统可能遇到的 failure 主要有三类。

### 2.1 Transaction Failure

Transaction failure 指某个事务自身无法正常完成。

- **Logical errors**：事务内部逻辑条件不满足，导致无法完成。
- **System errors**：数据库系统主动终止某个事务，例如死锁处理时选择某个事务作为 victim。

这类 failure 通常只需要回滚该事务，不一定影响整个系统。

### 2.2 System Crash

System crash 指电源故障、硬件故障、软件故障等导致系统崩溃。

课件采用 **fail-stop assumption**：

- 系统崩溃会使内存内容丢失；
- 但非易失存储上的内容假设不会被系统崩溃破坏；
- 数据库系统会用完整性检查来减少磁盘数据被悄悄损坏的风险。

### 2.3 Disk Failure

Disk failure 指磁头损坏等磁盘故障导致部分或全部磁盘内容丢失。

课件假设这种破坏是可检测的：

- 磁盘使用 checksum 检测块是否损坏；
- 如果磁盘真的丢失数据，就需要从备份和日志恢复。

---

## 3. Storage Structure

恢复系统讨论三类存储。

| 类型 | 特点 | 例子 |
| --- | --- | --- |
| Volatile storage | 系统崩溃后内容丢失 | main memory, cache |
| Nonvolatile storage | 系统崩溃后通常保留内容，但仍可能损坏 | disk, tape, flash, battery-backed RAM |
| Stable storage | 理想化的“永不丢失”存储 | 通过多份副本近似实现 |

Stable storage 是一个理论抽象。实际系统通常通过在不同非易失介质上保存多个副本来近似它。

### 3.1 Stable Storage Implementation

实现 stable storage 的基本思想：

- 对每个 block 维护多个副本；
- 副本最好放在不同磁盘，甚至远程站点；
- 这样可以抵抗单个磁盘损坏，甚至火灾、洪水等灾难。

难点在于：写入多个副本时，系统可能在写到一半时崩溃，导致副本不一致。

课件给出的处理思路：

1. 假设每个 block 有两个副本。
2. 先写第一个物理块。
3. 第一个写成功后，再写第二个物理块。
4. 两个副本都写成功后，才认为输出完成。

如果恢复时发现副本不一致：

- 简单但昂贵的方案：比较所有 block 的两个副本。
- 更好的方案：记录正在进行的 disk write，恢复时只检查这些可能不一致的 block。
- 若一个副本 checksum 错误，用另一个副本覆盖它。
- 若两个副本 checksum 都正确但内容不同，可以用第一个副本覆盖第二个副本。

### 3.2 隐藏页简略说明：Slides 7-11

Slides 7-11 是隐藏/省略页，保留核心含义即可：

- 第 7-8 页继续讲 stable storage：多副本写入可能出现 partial failure / total failure，需要用写入记录和 checksum 找出并修复不一致副本。
- 第 9-10 页介绍 data access 图示：磁盘上的 physical block 通过 `input(B)` 进入内存 buffer，内存 buffer 通过 `output(B)` 写回磁盘；事务在自己的 private work-area 中读写局部变量。
- 第 11 页补充 private work-area、`read(X)`、`write(X)` 与 `output(BX)` 的关系。

---

## 4. Data Access

课件区分两类 block：

- **Physical blocks**：磁盘上的 block。
- **Buffer blocks**：主存中临时保存的 block。

block 在磁盘和内存之间移动有两个基本操作：

- `input(B)`：把磁盘上的 physical block `B` 读入主存。
- `output(B)`：把主存中的 buffer block `B` 写回磁盘，替换原来的 physical block。

为简化讨论，课件假设每个 data item 都能完整放在一个 block 内。

### 4.1 Transaction Private Work-Area（Hidden Slide 11）

每个事务 `Ti` 都有自己的 private work-area。

- 事务访问和更新的数据项，会在 private work-area 中保存本地副本。
- `Ti` 对数据项 `X` 的本地副本记为 `xi`。

数据项在 system buffer 和事务 work-area 之间移动：

- `read(X)`：把数据项 `X` 的值赋给事务本地变量 `xi`。
- `write(X)`：把本地变量 `xi` 的值写回 buffer block 中的 `X`。

注意：

- `write(X)` 只是写到 buffer，不代表马上写回磁盘。
- `output(BX)` 不一定紧跟 `write(X)`。
- 系统可以在认为合适的时候把 buffer block 写回磁盘。

事务规则：

- 第一次访问 `X` 前必须先 `read(X)`。
- 后续访问可以直接使用本地副本。
- `write(X)` 可以在事务提交前的任意时间执行。

---

## 5. Database Recovery 的目标

Recovery algorithm 的目标是在 failure 之后仍保证：

- **Atomicity**：未完成事务的影响不能永久留下。
- **Consistency**：恢复后数据库回到一致状态。
- **Durability**：已经提交的事务结果不能丢失。

恢复算法有两部分：

1. 正常事务处理期间做的事：记录足够的信息，让以后能够恢复。
2. failure 后做的事：根据这些信息恢复数据库内容。

课件还强调两点：

- 假设 strict two-phase locking 保证没有 dirty read。
- 恢复算法最好是 **idempotent**：执行多次和执行一次效果相同。

---

## 6. Log-Based Recovery

Log-based recovery 的核心思想是：

> 修改数据库本身之前，先把描述修改的信息写到 stable storage 上的 log 中。

Log 是一串 log records，用来记录数据库上的 update activities。

### 6.1 Log Records

常见 log record：

```text
<Ti start>
<Ti, X, V1, V2>
<Ti commit>
<Ti abort>
```

含义：

- `<Ti start>`：事务 `Ti` 开始。
- `<Ti, X, V1, V2>`：事务 `Ti` 修改 `X`，旧值为 `V1`，新值为 `V2`。
- `<Ti commit>`：事务 `Ti` 成功提交。
- `<Ti abort>`：事务 `Ti` 已完成回滚。

### 6.2 WAL: Write-Ahead Logging

**Write-Ahead Logging (WAL)** 是恢复系统的基本规则：

> 在内存中的数据被输出到数据库之前，与该数据有关的 log records 必须已经输出到 stable storage。

也就是说：

- 可以延迟数据页写回；
- 也可以让未提交事务的更新先进入 buffer，甚至被写到磁盘；
- 但只要数据页要写回磁盘，相关 undo 信息必须已经安全写入 log。

WAL 是支持 steal 策略和 no-force 策略的前提。

### 6.3 Log File Examples（Hidden Slide 16）

Slide 16 是隐藏页，主要用日志片段说明：

- 已提交事务在恢复中需要 redo。
- 已经 abort 的事务也可能在 repeating history 中被 redo，因为它的 compensation log records 也要重放。
- 崩溃时未完成的事务最终进入 undo phase。

---

## 7. Concurrency Control and Recovery

并发事务共享：

- 一个 disk buffer；
- 一个 log file。

同一个 buffer block 可能包含多个事务更新过的数据项。

课件假设：

- 如果事务 `Ti` 修改了某个 item，其他事务在 `Ti commit/abort` 前不能修改同一 item。
- 未提交事务的更新不应对其他事务可见。

这通常由 **strict two-phase locking** 保证：

- 更新对象时获取 exclusive lock；
- 直到事务结束才释放。

后面讲 logical undo 时，会说明如何支持 early lock release。

---

## 8. Database Modification（Hidden Slides 19, 21）

### 8.1 Immediate Modification

Immediate-modification scheme 允许未提交事务的更新在事务提交前进入 buffer，甚至写到磁盘。

要求：

- update log record 必须先于 database item 写入。
- 更新过的 block 可以在事务提交前或提交后写回 stable storage。
- block 输出顺序可以不同于事务中 `write` 的顺序。

这种方法需要 undo 和 redo 两类能力。

### 8.2 Deferred Modification

Deferred-modification scheme 只在事务提交时才把更新写入 buffer / disk。

优点：

- 恢复逻辑更简单；
- 未提交更新不会提前污染数据库。

缺点：

- 需要保存本地副本，开销更大。

### 8.3 Transaction Commit

事务提交的判定标准：

> 当 `<Ti commit>` log record 被输出到 stable storage 时，事务才算 committed。

注意：

- 该事务之前的所有 log records 必须已经输出。
- 事务的 data writes 可能仍在 buffer 中，提交后才被写回磁盘。

---

## 9. Undo and Redo

对 update log record：

```text
<Ti, X, V1, V2>
```

定义：

- **Undo**：把 `X` 恢复成旧值 `V1`。
- **Redo**：把 `X` 设置为新值 `V2`。

### 9.1 undo(Ti)

`undo(Ti)` 从 `Ti` 的最后一条 log record 开始向前扫描：

- 对每个 `<Ti, X, V1, V2>` 执行 undo，把 `X` 写回 `V1`。
- 每撤销一个数据项，就写一条 compensation log record：

```text
<Ti, X, V>
```

其中 `V` 是恢复后的旧值。

事务全部撤销后，写：

```text
<Ti abort>
```

### 9.2 redo(Ti)

`redo(Ti)` 从 `Ti` 的第一条 update log record 开始向后扫描：

- 对每个 `<Ti, X, V1, V2>` 执行 redo，把 `X` 写成 `V2`。
- redo 本身不需要额外写 log。

---

## 10. Failure Recovery: 谁 undo，谁 redo

系统崩溃后，恢复时根据 log 判断每个事务状态。

事务 `Ti` 需要 **undo**，如果 log 中：

- 有 `<Ti start>`；
- 但没有 `<Ti commit>`；
- 也没有 `<Ti abort>`。

事务 `Ti` 需要 **redo**，如果 log 中：

- 有 `<Ti start>`；
- 且有 `<Ti commit>` 或 `<Ti abort>`。

这里看起来 `<Ti abort>` 也要 redo 很奇怪，但原因是恢复算法采用 **repeating history**。

### 10.1 Repeating History

如果某个事务之前已经被 undo，并写入了 `<Ti abort>`，随后系统又崩溃，那么恢复时会 redo 这个事务的历史。

redo 的内容包括：

- 事务原本做过的更新；
- 之前 undo 过程中写入的 compensation log records。

这叫 **repeating history**。

它看起来浪费，但大大简化了恢复过程：

1. 先把数据库恢复到崩溃瞬间应有的状态。
2. 再统一撤销崩溃时仍未完成的事务。

---

## 11. Immediate Modification Recovery Example

课件的例子里：

```text
<T0 start>
<T0, A, 1000, 950>
<T0, B, 2000, 2050>
<T0 commit>
<T1 start>
<T1, C, 700, 600>
<T1 commit>
```

不同崩溃时间对应不同恢复动作：

- 如果 `T0` 尚未提交：undo `T0`，把 `B` 恢复为 `2000`，把 `A` 恢复为 `1000`。
- 如果 `T0` 已提交而 `T1` 未提交：redo `T0`，undo `T1`。
- 如果 `T0` 和 `T1` 都已提交：redo `T0` 和 `T1`。

核心判断：

- committed transaction 的效果必须保留；
- incomplete transaction 的效果必须撤销。

---

## 12. Checkpoints

如果每次恢复都从 log 开头开始 undo / redo，会非常慢。

原因：

- 系统运行时间越长，log 越大；
- 很多早已写入数据库的事务没有必要再处理。

**Checkpointing** 用来缩短恢复需要扫描的 log 范围。

### 12.1 基本 checkpoint 步骤

周期性执行：

1. 把内存中的所有 log records 输出到 stable storage。
2. 把所有 modified buffer blocks 输出到磁盘。
3. 写一条 checkpoint record：

```text
<checkpoint L>
```

其中 `L` 是 checkpoint 时仍 active 的事务列表。

缺点：

- 做 checkpoint 时所有 update 都要停止。
- 停顿时间可能很长。

### 12.2 Checkpoint 后的恢复

恢复时：

1. 从 log 末尾向前找最近的 `<checkpoint L>`。
2. 只需要考虑：
   - `L` 中的事务；
   - checkpoint 之后开始的事务。
3. checkpoint 之前已经 commit/abort 的事务，其更新已经写入 stable storage，可以忽略。
4. 对 `L` 中的事务，还需要继续向前找它们的 `<Ti start>`，因为 undo 可能需要更早的 log。

因此，早于这些 start records 的 log 可以在合适时删除。

### 12.3 Checkpoint Example（Hidden Slides 29-30）

课件例子：

- `T1` 在 checkpoint 前已经完成并且更新已写盘，可以忽略。
- `T2` 和 `T3` 需要 redo。
- `T4` 崩溃时未完成，需要 undo。

记忆方式：

- checkpoint 前完成的事务通常不再处理；
- checkpoint 时 active 的事务进入关注范围；
- checkpoint 后开始的事务也进入关注范围。

---

## 13. Basic Recovery Algorithm（Hidden Slide 31）

### 13.1 Normal Operation: Logging

正常运行时记录：

```text
<Ti start>
<Ti, Xj, V1, V2>
<Ti commit>
```

### 13.2 Normal Operation: Transaction Rollback

回滚事务 `Ti`：

1. 从 log 末尾向前扫描。
2. 遇到 `<Ti, Xj, V1, V2>`：
   - 把 `Xj` 写回 `V1`；
   - 写 compensation log record `<Ti, Xj, V1>`。
3. 遇到 `<Ti start>` 时停止。
4. 写 `<Ti abort>`。

### 13.3 Failure Recovery: Two Phases

系统崩溃后的恢复分两阶段。

#### Redo Phase

目标：重复历史，把所有相关更新重放一遍。

步骤：

1. 找最近的 `<checkpoint L>`。
2. 初始化 `undo-list = L`。
3. 从 checkpoint 向后扫描：
   - 遇到 `<Ti, Xj, V1, V2>`，redo：把 `Xj` 写成 `V2`。
   - 遇到 compensation log record `<Ti, Xj, V2>`，redo：把 `Xj` 写成 `V2`。
   - 遇到 `<Ti start>`，把 `Ti` 加入 `undo-list`。
   - 遇到 `<Ti commit>` 或 `<Ti abort>`，把 `Ti` 从 `undo-list` 删除。

Redo phase 结束后，数据库被恢复到崩溃时的状态。

#### Undo Phase

目标：撤销崩溃时仍未完成的事务。

步骤：

1. 从 log 末尾向前扫描。
2. 遇到属于 `undo-list` 中事务的 `<Ti, Xj, V1, V2>`：
   - undo：把 `Xj` 写回 `V1`；
   - 写 compensation log record `<Ti, Xj, V1>`。
3. 遇到属于 `undo-list` 中事务的 `<Ti start>`：
   - 写 `<Ti abort>`；
   - 从 `undo-list` 删除 `Ti`。
4. 当 `undo-list` 为空时停止。

之后系统可以恢复正常事务处理。

### 13.4 Recovery Example（Hidden Slide 36）

Slide 36 是隐藏页，展示了一段包含 checkpoint、commit、abort 和 compensation log 的恢复例子。读这个例子时重点看三件事：

- 从最近 checkpoint 建立初始 `undo-list`。
- Redo phase 先把 checkpoint 后的历史重放。
- Undo phase 只撤销最终仍在 `undo-list` 中的未完成事务。

---

## 14. Log Buffer and Database Buffer

实际系统中，log 和 database page 都会使用 buffer。

### 14.1 Log Record Buffering

Log record 可以先缓存在内存中，而不是每条都立即输出到 stable storage。

输出时机：

- log buffer 满；
- 执行 log force；
- 事务 commit 时必须 force 该事务所有 log records，包括 commit record。

**Group commit**：

- 多个事务的 log records 可以合并一次 I/O 输出；
- 降低每个事务 commit 的 I/O 成本。

### 14.2 Buffered Log 的规则

如果 log records 被 buffer，必须满足：

1. Log records 按产生顺序输出到 stable storage。
2. 事务 `Ti` 只有在 `<Ti commit>` 输出到 stable storage 后，才进入 commit state。
3. 数据 block 输出到数据库前，与该 block 中数据有关的所有 log records 必须已经输出到 stable storage。

第 3 条就是 WAL rule。

严格来说，WAL 至少要求 undo information 先写入 stable storage。

---

## 15. Database Buffering（Hidden Slides 41-43）

数据库维护内存中的 data block buffer。

当需要读入新 block 且 buffer 已满时，系统必须选择某个 block 移出 buffer：

- 如果该 block 没被修改，可以直接丢弃。
- 如果该 block 是 dirty block，必须写回磁盘。

### 15.1 No-Force Policy

本章恢复算法支持 **no-force policy**：

> 事务 commit 时，不要求立刻把该事务更新过的 block 写到磁盘。

优点：

- commit 更快；
- 可以减少同步 I/O。

代价：

- 崩溃后需要 redo committed transaction。

对比：

- **force policy** 要求 commit 时把更新 block 写盘，commit 成本更高。

### 15.2 Steal Policy

本章恢复算法也支持 **steal policy**：

> 含有未提交事务更新的 block 也可以在事务 commit 前写回磁盘。

优点：

- buffer 管理更灵活；
- 不必因为未提交更新长期占住 buffer。

代价：

- 崩溃后需要 undo uncommitted transaction。

### 15.3 Latches

如果一个含未提交更新的 block 要写回磁盘，必须先保证相关 undo log 已经写入 stable storage。

输出 block 时还要求没有事务正在更新该 block。

课件给出的做法：

1. 写数据项前，事务对包含该数据项的 block 获取 exclusive latch。
2. 写完后释放 latch。
3. 输出 block 到磁盘时：
   - 先获取该 block 的 exclusive latch；
   - 执行 log flush；
   - 输出 block；
   - 释放 latch。

Latch 是短时间持有的轻量级互斥机制，不同于事务级 lock。

### 15.4 Database Buffer in Virtual Memory

Database buffer 可以实现为：

- 数据库专门保留的一块真实主存；
- 或虚拟内存中的区域。

保留真实主存的问题：

- 内存需要预先在数据库和应用之间分割；
- OS 不能根据实时负载灵活调整。

虚拟内存方式更灵活，但可能出现 **dual paging problem**：

- OS 想淘汰某个 modified buffer page 时，把它写到 swap space；
- 数据库之后要把该 page 写回数据库文件时，可能还要先从 swap space 读回来；
- 产生额外 I/O。

理想情况下，OS 淘汰数据库 buffer page 时应把控制权交给数据库，由数据库在遵守 WAL 的前提下直接写回数据库文件，但常见 OS 并不完全支持这种机制。

---

## 16. Fuzzy Checkpointing

普通 checkpoint 要停止所有更新，停顿时间可能过长。

**Fuzzy checkpointing** 允许 checkpoint 期间事务继续更新。

步骤：

1. 暂停所有事务更新。
2. 写 `<checkpoint L>`，并 force log 到 stable storage。
3. 记录当前 modified buffer blocks 列表 `M`。
4. 允许事务继续执行。
5. 把 `M` 中的 modified blocks 输出到磁盘。
   - 输出某个 block 时，该 block 不能正在被更新。
   - 仍然必须遵守 WAL。
6. 在磁盘固定位置 `last_checkpoint` 保存指向该 checkpoint record 的指针。

恢复时：

- 从 `last_checkpoint` 指向的 checkpoint record 开始扫描。
- 该 checkpoint 之前的 log records 其更新已经反映到磁盘，不需要 redo。
- 如果系统在 checkpoint 过程中崩溃，不完整 checkpoint 也可以安全处理。

---

## 17. Loss of Nonvolatile Storage

前面大多假设 nonvolatile storage 没有丢失。

如果磁盘本身损坏，需要更强的备份机制。

### 17.1 Dump

处理 nonvolatile storage loss 的方法类似 checkpoint：

周期性把整个数据库内容 dump 到 stable storage。

dump 过程：

1. 不允许有 active transaction。
2. 把内存中的所有 log records 输出到 stable storage。
3. 把所有 buffer blocks 输出到磁盘。
4. 把数据库内容复制到 stable storage。
5. 在 stable storage 上的 log 中写 `<dump>`。

### 17.2 Disk Failure Recovery

磁盘故障后：

1. 从最近一次 dump 恢复数据库。
2. 查看 log。
3. Redo dump 之后所有已经提交的事务。

也可以允许 dump 期间事务仍 active，这叫 **fuzzy dump** 或 **online dump**，思想类似 fuzzy checkpointing。

---

## 18. Remote Backup Systems（Hidden Slides 48-52）

Remote backup systems 用于提高 availability。

核心目标：

> 即使 primary site 被摧毁，transaction processing 也能在 backup site 继续。

### 18.1 Failure Detection

Backup site 必须检测 primary site 是否失败。

难点：

- primary site 真失败；
- 或者只是 communication link failure。

常用方法：

- 在 primary 和 backup 之间维护多条通信链路；
- 使用 heartbeat messages。

### 18.2 Transfer of Control

Backup 接管前：

1. 使用自己持有的 database copy；
2. 加上已经从 primary 收到的 log records；
3. 执行 recovery：
   - redo completed transactions；
   - rollback incomplete transactions。

接管后：

- backup 变成 new primary。

旧 primary 恢复后如果要重新接管：

- 必须从旧 backup 接收 redo logs；
- 在本地应用所有更新。

### 18.3 Hot-Spare

为了缩短 takeover 时间，backup 可以周期性处理 redo log，执行 checkpoint，并删除更早 log。

**Hot-spare configuration** 更进一步：

- backup 持续处理 primary 发来的 redo log；
- 在本地不断应用更新；
- primary 失败后，只需要 rollback incomplete transactions，就可以处理新事务。

### 18.4 Durability Modes

为了保证 durability，可以延迟 commit，直到更新也被 backup 记录。

课件列出三种模式：

| 模式 | Commit 条件 | 问题 / 特点 |
| --- | --- | --- |
| One-safe | commit log record 写到 primary 就提交 | backup 接管前可能没收到更新，可能丢已提交事务 |
| Two-very-safe | commit log record 写到 primary 和 backup 才提交 | 任一站点不可用都会影响提交，可用性较低 |
| Two-safe | 双方都 active 时按 two-very-safe；只有 primary active 时按 one-safe | 在 durability 和 availability 之间折中 |

---

## 19. Early Lock Release and Logical Undo

一些高并发结构，例如 B+ tree concurrency control，会提前释放锁。

这会给恢复带来问题：

- 如果某个操作提前释放锁，其他事务可能已经基于修改后的结构继续更新。
- 这时再用旧值物理恢复，可能破坏其他事务的更新。

因此需要 **logical undo**。

### 19.1 Physical Undo vs Logical Undo

以 B+ tree 插入为例：

- 插入操作完成后释放锁；
- 其他事务可能继续修改同一个 B+ tree；
- 如果回滚时简单恢复 old value，可能覆盖其他事务的合法更新。

所以插入的 undo 应该是执行一个“删除对应 entry”的逻辑操作。

类似地：

- 插入 tuple 的 undo 是 delete tuple。
- deposit 的 undo 可以是 subtract amount。

这种记录 undo operation 的日志叫 **logical undo logging**。

### 19.2 Physical Redo

即使 undo 使用 logical undo，redo 仍然用 physical redo。

原因：

- 崩溃恢复刚开始时，磁盘上的数据库状态可能不是 operation-consistent。
- Logical redo 很复杂。
- Physical redo 不妨碍 early lock release。

---

## 20. Operation Logging

对于需要 logical undo 的操作，课件给出 operation logging。

操作开始时：

```text
<Ti, Oj, operation-begin>
```

其中 `Oj` 是操作实例的唯一标识。

操作执行过程中：

- 写正常的物理 redo / undo log records。

操作完成时：

```text
<Ti, Oj, operation-end, U>
```

其中 `U` 是 logical undo 所需的信息。

### 20.1 Example

向索引 `I9` 插入 `(K5, RID7)`：

```text
<T1, O1, operation-begin>
...
<T1, X, 10, K5>
<T1, Y, 45, RID7>
<T1, O1, operation-end, (delete I9, K5, RID7)>
```

含义：

- redo：仍然用物理 redo 重放插入中的具体页面修改。
- undo：如果操作已经完成，执行 logical undo：`delete I9, K5, RID7`。

### 20.2 Crash / Rollback Cases

如果 crash 或 rollback 发生在 operation 完成前：

- 找不到 `operation-end`；
- 使用 physical undo 撤销已经做过的部分。

如果 crash 或 rollback 发生在 operation 完成后：

- 找到 `operation-end`；
- 使用其中的 `U` 做 logical undo；
- 忽略该操作内部的 physical undo 信息。

Redo 永远使用 physical redo。

---

## 21. Transaction Rollback with Logical Undo

回滚事务 `Ti` 时向后扫描 log。

### 21.1 遇到普通 update record

如果遇到：

```text
<Ti, X, V1, V2>
```

执行 physical undo，并写：

```text
<Ti, X, V1>
```

### 21.2 遇到 operation-end

如果遇到：

```text
<Ti, Oj, operation-end, U>
```

说明该 logical operation 已经完成。

处理：

1. 使用 `U` 执行 logical undo。
2. logical undo 过程中的更新像正常操作一样写 log。
3. operation rollback 完成后，不写新的 `operation-end`，而写：

```text
<Ti, Oj, operation-abort>
```

4. 跳过该操作之前的所有内部 log records，直到 `<Ti, Oj, operation-begin>`。

跳过很重要：避免同一个 logical operation 被重复 rollback。

### 21.3 Rollback Example（Hidden Slide 63）

Slide 63 是隐藏页，给出一个同时包含 complete operation 和 incomplete operation 的 rollback 例子：

- 对已经写出 `operation-end` 的操作，使用其中的 logical undo 信息，并写 `operation-abort`。
- 对没有完成的操作，使用物理 undo 撤销已经写出的部分。
- 如果系统在 rollback 中途再次崩溃，后续恢复会通过 redo-only / compensation records 避免重复撤销同一操作。

### 21.4 遇到 redo-only 或 operation-abort

- 遇到 redo-only record：忽略。
- 遇到 `<Ti, Oj, operation-abort>`：说明该操作的 logical undo 已完成，继续跳过直到对应 `operation-begin`。
- 遇到 `<Ti start>`：停止扫描，写 `<Ti abort>`。

### 21.5 Failure Recovery with Logical Undo

带 logical undo 的系统恢复仍然遵循两阶段思想：

1. Redo phase：从 checkpoint 向后物理 redo，重复历史，并建立 `undo-list`。
2. Undo phase：对 `undo-list` 中未完成事务向前撤销；遇到 logical operation 时按上面的规则处理。

Redo 结束后，数据库到达崩溃瞬间状态；Undo 结束后，未完成事务的影响被撤销。

---

## 22. ARIES Recovery Algorithm

**ARIES** 是实际系统中非常重要的 recovery method。

前面讲的基本恢复算法可以看成 ARIES 的简化版。

ARIES 的特点：

- 用 **LSN (Log Sequence Number)** 标识 log records。
- 在 page 中保存 **PageLSN**，表示该 page 已经反映到哪个 log record。
- 使用 **physiological redo**。
- 使用 **Dirty Page Table** 避免不必要 redo。
- 使用低开销 fuzzy checkpoint。

### 22.1 Physiological Redo（Hidden Slide 68）

Physiological redo 介于 physical 和 logical 之间：

- 物理地标识受影响的 page；
- page 内部的 action 可以更逻辑化。

例子：

- 删除一条 record 后，page 内其他 record 可能移动以填补空洞。
- 纯 physical redo 可能要记录 page 内大量 old/new values。
- Physiological redo 可以只记录“删除该 record”这一页内操作。

要求：

- page 输出到磁盘必须是 atomic 的；
- 不完整 page output 需要通过 checksum 等技术检测；
- 一旦 page 损坏，按 media failure 处理。

---

## 23. ARIES Data Structures

### 23.1 LSN

每条 log record 有一个递增的 **LSN**。

通常 LSN 可以是 log file 起始处的 offset，这样便于快速访问。

### 23.2 PageLSN

每个 page 保存一个 **PageLSN**：

> PageLSN 是该 page 已经反映的最后一条 log record 的 LSN。

更新 page 时：

1. X-latch page。
2. 写 log record。
3. 修改 page。
4. 把该 log record 的 LSN 记录到 PageLSN。
5. 释放 latch。

Flush page 到磁盘时：

- 先 S-latch page；
- 保证写出的 page 处于 operation-consistent 状态。

PageLSN 的作用：

- 恢复时判断某条 log record 是否已经反映到磁盘 page；
- 避免重复 redo；
- 保证 idempotence。

这里的 **idempotence（幂等性）** 指：同一个恢复动作即使重复执行多次，最终结果也和只执行一次一样。比如某条 log 已经反映到 page 上，就不要再次 redo，否则重复执行可能造成错误；PageLSN 就是用来判断“这条 log 是否已经做过”的标记。

### 23.3 Log Record

ARIES 中每条普通 log record 包含：

```text
LSN, TransID, PrevLSN, RedoInfo, UndoInfo
```

其中：

- `PrevLSN` 指向同一事务上一条 log record；
- 这样 undo 可以沿着事务自己的 log chain 向前跳。

### 23.4 CLR: Compensation Log Record

ARIES 使用 **CLR (Compensation Log Record)** 记录 undo 过程中做过的动作。

CLR 特点：

- 是 redo-only log record；
- 记录的动作不需要再被 undo；
- 包含 `UndoNextLSN` 字段。

`UndoNextLSN` 指出下一条应该被 undo 的更早 log record。

作用：

- 崩溃发生在 undo 过程中时，恢复后可以 redo 已经完成的 undo；
- 并且跳过已经 undo 过的记录，避免重复 undo。

### 23.5 Dirty Page Table

Dirty Page Table 记录 buffer 中已经更新但磁盘版本可能还不是最新的 pages。

每个 dirty page 记录：

- PageID；
- PageLSN；
- **RecLSN**。

`RecLSN` 的含义：

> 某个 LSN，使得该 LSN 之前的 log records 已经反映到磁盘上的 page 版本。即目前最早一个使本 page 变脏的 LSN

当 page 首次进入 Dirty Page Table 时：

- `RecLSN` 设为当前 update log record 的 LSN。

恢复时 `RecLSN` 可用来减少 redo 范围。

### 23.6 Checkpoint Log

ARIES checkpoint log record 包含：

- Dirty Page Table；
- active transaction list；
- 每个 active transaction 的 `LastLSN`。

磁盘固定位置保存最近一次完整 checkpoint 的 LSN。

ARIES checkpoint 不要求 checkpoint 时写出 dirty pages：

- dirty pages 可以后台持续 flush；
- checkpoint 开销低；
- 因此可以频繁执行。

---

## 24. ARIES Recovery: Three Passes

ARIES recovery 分三趟：

1. **Analysis pass**
2. **Redo pass**
3. **Undo pass**

### 24.1 Analysis Pass

Analysis pass 从最近一次完整 checkpoint 开始。

初始步骤：

1. 从 checkpoint log record 读出 Dirty Page Table。
2. 设置：

```text
RedoLSN = min(RecLSN of all dirty pages)
```

如果没有 dirty page，`RedoLSN` 就是 checkpoint record 的 LSN。

3. 从 checkpoint log record 读出 active transactions，作为 undo-list。
4. 读出每个 active transaction 的 LastLSN。

然后从 checkpoint 向后扫描：

- 遇到某个不在 undo-list 中的事务 log record，把该事务加入 undo-list。
- 遇到 update log record：
  - 如果 page 不在 Dirty Page Table，就加入该 page；
  - `RecLSN` 设为该 update log record 的 LSN。
- 遇到 transaction end log record，把该事务从 undo-list 删除。
- 持续维护每个事务的最后一条 log record。

Analysis pass 结束后：

- `RedoLSN` 决定 redo pass 从哪里开始。
- Dirty Page Table 的 `RecLSN` 用于减少 redo 工作。
- undo-list 中剩下的事务都需要 rollback。

### 24.2 Redo Pass

Redo pass 从 `RedoLSN` 向后扫描。只 redo 尚未反映到磁盘 page 的 action。

遇到 update log record 时：

1. 如果 page 不在 Dirty Page Table，跳过。
2. 如果 log record 的 LSN 小于该 page 的 RecLSN，跳过。
3. 否则从磁盘取该 page。
4. 如果 page 的 PageLSN 小于 log record 的 LSN，redo。
5. 如果 PageLSN 已经大于等于该 LSN，说明该更新已经反映在 page 上，跳过。

前两个检查可以避免不必要的 page fetch。

### 24.3 Undo Pass

Undo pass 回滚 analysis pass 中得到的 incomplete transactions。

ARIES 不做简单线性后扫，而是利用 LSN 跳转优化：

1. 对每个待 undo 事务，把 next LSN 设为该事务 LastLSN。
2. 每一步选择这些 next LSN 中最大的一个。
3. 跳到该 log record 并 undo。
4. 如果是普通 log record：
   - undo 后写 CLR；
   - next LSN 设为该 log record 的 `PrevLSN`。
5. 如果是 CLR：
   - next LSN 设为 CLR 的 `UndoNextLSN`；
   - 中间记录已经被 undo 过，直接跳过。

Undo pass 完成后，所有 incomplete transactions 都被撤销。

---

## 25. Other ARIES Features

### 25.1 Recovery Independence

ARIES 支持 page 独立恢复：

- 某些 disk pages 失败时，可以只恢复这些 pages；
- 其他 pages 仍然可以被使用。

### 25.2 Savepoints

事务可以设置 savepoint，并回滚到某个 savepoint。

用途：

- 复杂事务中的局部撤销；
- 死锁处理时只回滚足够多的操作，以释放需要的锁。

### 25.3 Fine-Grained Locking

细粒度锁，尤其是 index concurrency algorithms，常需要 logical undo。

例如：

- index 中使用 tuple-level locking；
- B+ tree 操作提前释放锁；
- 此时 physical undo 不可靠，需要 logical undo。

### 25.4 Recovery Optimizations（Hidden Slide 86）

ARIES 还支持多种优化：

- 用 Dirty Page Table 在 redo 时预取 pages。
- Out-of-order redo：
  - 如果某个 page 正在从磁盘读取，可以先推迟该 page 的 redo；
  - 同时继续处理其他 log records；
  - page 读到后再补做 redo。

---

## 26. 对比总结

### 26.1 Steal / No-Force 与 Undo / Redo

| Buffer 策略 | 含义 | 恢复需求 |
| --- | --- | --- |
| Steal | 未提交事务的 dirty page 可以写盘 | 需要 undo |
| No-steal | 未提交事务的 dirty page 不能写盘 | undo 压力较小 |
| Force | commit 时必须把更新 page 写盘 | redo 压力较小，但 commit 慢 |
| No-force | commit 时不必立即写 page | 需要 redo |

本章主要算法支持：

- steal：所以需要 undo；
- no-force：所以需要 redo。

### 26.2 Basic Recovery vs ARIES

| 主题         | Basic Recovery         | ARIES                                |
| ---------- | ---------------------- | ------------------------------------ |
| Log 定位     | 主要按顺序扫描                | 用 LSN 精确定位                           |
| Page 是否已更新 | 不细分                    | PageLSN 判断                           |
| Checkpoint | 可能要求写 dirty blocks     | fuzzy checkpoint，不要求写 dirty pages    |
| Redo 范围    | 从 checkpoint 后扫描       | 从 RedoLSN 开始，用 DPT/RecLSN/PageLSN 剪枝 |
| Undo       | 扫描 log，撤销 undo-list 事务 | 用 PrevLSN / UndoNextLSN 跳转           |
| Undo 日志    | compensation log       | CLR                                  |

---

## 27. 最后要记住的句子

- Recovery system 的核心目标是 failure 后仍保证 atomicity 和 durability。
- WAL 的核心是：数据页写回数据库前，相关 log 必须已经写到 stable storage。
- Commit 的标志不是数据页写盘，而是 commit log record 写到 stable storage。
- Steal 需要 undo，no-force 需要 redo。
- Repeating history 的做法是：先 redo 到崩溃瞬间，再 undo 未完成事务。
- Checkpoint 降低恢复时需要扫描的 log 范围。
- Fuzzy checkpoint 通过允许事务继续执行，减少 checkpoint 停顿。
- Logical undo 用于支持 early lock release，尤其适合 B+ tree 等结构。
- ARIES 的关键词是：LSN、PageLSN、Dirty Page Table、CLR、Analysis/Redo/Undo three passes。
