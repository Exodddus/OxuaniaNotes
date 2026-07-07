# Lec9 Physical Storage System

本节关注数据库最底层的物理存储：数据放在哪里、读写一次要付出什么代价，以及系统如何减少磁盘/SSD I/O。

一个很重要的直觉是：数据库很多优化不是为了少算几次 CPU，而是为了**少访问几次外存**，尤其是少做随机 I/O。

---

## 1. Physical Storage Media 的分类

存储介质可以从三个角度比较：

- **是否易失**
  - volatile storage（易失存储）：断电后内容丢失，如 cache、主存。
  - non-volatile storage（非易失存储）：断电后内容仍保留，如磁盘、SSD、磁带、NVM。
- **访问速度**
  - 越靠近 CPU，访问越快。
  - 越远离 CPU，容量通常越大、单位成本越低。
- **可靠性**
  - 可能因为断电、系统崩溃、设备物理损坏导致数据丢失。

数据库系统设计时必须同时考虑：

```text
速度、容量、成本、持久性、可靠性
```

---

## 2. Storage Hierarchy（存储层次）

存储层次可以大致分为：

| 层次 | 典型介质 | 特点 |
| --- | --- | --- |
| Primary storage | cache、main memory | 最快，但通常易失 |
| Secondary storage | SSD、magnetic disk | 在线存储，非易失，数据库主要存放位置 |
| Tertiary storage | magnetic tape、optical storage | 离线或近线存储，慢但便宜，常用于备份归档 |

典型访问时间数量级：

- cache / main memory：纳秒级；
- SSD / NVM：微秒级到更低；
- 磁盘：毫秒级；
- 磁带等三级存储：更慢。

所以，数据库系统会尽量把频繁访问的数据留在主存 buffer 中，避免反复访问外存。

---

## 3. Magnetic Disk（磁盘）

### 3.1 磁盘的基本结构

磁盘由多个 platter（盘片）组成，每个盘片表面被划分为：

- **track（磁道）**：盘面上的同心圆；
- **sector（扇区）**：磁道进一步切分后的最小读写单位，典型大小为 512B；
- **cylinder（柱面）**：多个盘片上半径相同的一组磁道。

读写一个扇区时：

1. disk arm 移动读写头到目标磁道；
2. 盘片持续旋转；
3. 目标扇区转到读写头下方时进行读写。

### 3.2 Disk Controller（磁盘控制器）

磁盘控制器负责连接计算机系统和磁盘硬件。它会：

- 接受读写扇区的高级命令；
- 控制磁臂移动和实际读写；
- 给扇区附加 checksum，用于检测数据是否损坏；
- 写入后读回检查，确认写入成功；
- 对坏扇区进行 remapping。

---

## 4. 磁盘性能指标

### 4.1 Access Time（访问时间）

访问时间指从发出读写请求到开始传输数据之间的时间，主要包括：

$$
Access\ Time = Seek\ Time + Rotational\ Latency
$$

- **Seek time（寻道时间）**：磁臂移动到目标磁道的时间。
  - 平均寻道时间约为最坏情况的一半；
  - 典型值约 4 到 10 ms。
- **Rotational latency（旋转延迟）**：等待目标扇区转到读写头下方的时间。
  - 平均旋转延迟约为最坏情况的一半；
  - 典型值约 4 到 11 ms。

磁盘慢，主要不是因为传输慢，而是因为 seek 和旋转等待很贵。

### 4.2 Data Transfer Rate（数据传输率）

数据传输率表示磁盘实际读出或写入数据的速度。顺序访问时传输率较高，随机访问时大量时间浪费在 seek 上。

### 4.3 Disk Block（磁盘块）

数据库通常不直接以 sector 为单位管理数据，而是以 **disk block** 为单位进行存储分配和数据传输。

- 典型 block 大小：4KB 到 16KB。
- block 太小：需要更多 I/O 次数。
- block 太大：可能浪费空间，因为一个块可能只被部分填满。

---

## 5. Sequential Access 与 Random Access

### 5.1 Sequential Access（顺序访问）

连续请求相邻 block：

- 第一个 block 需要 seek；
- 后续 block 通常可以连续读；
- I/O 效率高。

### 5.2 Random Access（随机访问）

每次请求可能落在磁盘任意位置：

- 每次访问都可能需要 seek；
- 传输率低；
- IOPS 成为关键指标。

### 5.3 IOPS

**IOPS（I/O operations per second）** 表示每秒可以完成多少次随机块读写。

传统磁盘大约只有几十到几百 IOPS；SSD 的随机读写能力通常高得多。

---

## 6. MTTF（Mean Time To Failure）

MTTF 表示设备连续运行到发生故障的平均时间。

课件中的一个重要理解：

如果某型号新磁盘理论 MTTF 为 1,200,000 小时，并不意味着一块磁盘真的能稳定用 136 年。它的含义更接近：

> 如果有 1000 块相对新的磁盘，平均每 1200 小时约有一块发生故障。

因此在大规模数据库系统中，单盘故障并不罕见，必须通过冗余、备份、日志、恢复机制来处理。

---

## 7. Optimization of Disk-Block Access

数据库系统常用以下策略降低磁盘访问成本。

### 7.1 Buffering

在内存中设置 buffer，缓存最近或频繁访问的 disk block。

如果请求的 block 已在 buffer 中，就不需要访问磁盘。

### 7.2 Read-ahead / Prefetch

预读可能马上会用到的额外 block。

适合顺序扫描，因为如果当前正在读某个连续文件，后续 block 很可能马上被访问。

### 7.3 Disk-arm Scheduling

重新排列磁盘请求顺序，减少磁臂移动。

典型算法是 **elevator algorithm（电梯算法）**：

- 磁臂像电梯一样朝一个方向移动；
- 顺路处理该方向上的请求；
- 到达边界后再反向。

### 7.4 File Organization 与 Extent

尽量把同一个文件的 block 连续分配。

- **extent（盘区）**：一组连续 block，系统以 extent 为单位分配空间。
- 如果文件 block 分散在磁盘各处，就会产生 fragmentation（碎片）。
- 顺序访问碎片化文件时，磁臂移动增加，性能下降。
- defragmentation 可以把碎片整理得更连续。

### 7.5 Nonvolatile Write Buffer

非易失性写缓存使用带电池保护的 RAM 或 flash memory。

写入请求可以先安全地写入非易失 buffer，然后数据库继续执行，不必等待真正落到磁盘。

好处：

- 降低写延迟；
- 可以把多个写操作重新排序，减少磁臂移动；
- 即使断电，buffer 中的数据仍可恢复。

### 7.6 Log Disk

日志磁盘专门顺序写入 block update log。

优点是：

- 顺序写几乎不需要 seek；
- 可达到类似非易失写缓存的效果；
- 不一定需要特殊硬件。

---

## 8. Flash Storage 与 SSD

### 8.1 NAND Flash

NAND flash 是 SSD 常用存储介质。

特点：

- 按 page 读，page 大小常见为 512B 到 4KB；
- 随机读和顺序读差距远小于磁盘；
- page 只能写一次，重写前必须先 erase；
- erase 以 erase block 为单位，通常 256KB 到 1MB；
- erase 较慢，约 2 到 5 ms；
- 每个 erase block 有擦写寿命限制。

### 8.2 SSD 与磁盘的差异

| 指标 | Magnetic Disk | SSD |
| --- | --- | --- |
| 随机访问 | 慢，seek 很贵 | 快得多 |
| 读 page | 毫秒级 | 微秒级 |
| IOPS | 约 50 到 200 | 可达上万甚至更高 |
| 更新方式 | in-place update | erase 后 rewrite |
| 功耗 | 较高 | 较低 |
| 磨损问题 | 不明显 | erase block 有寿命限制 |

SSD 的核心问题不再是机械 seek，而是：

- erase 成本；
- 写放大；
- 磨损均衡。

### 8.3 Flash Translation Layer

SSD 内部通过 **Flash Translation Layer（FTL）** 维护逻辑 page 到物理 page 的映射。

当逻辑 page 被更新时，系统通常不原地覆盖，而是：

1. 把新版本写到新的物理 page；
2. 更新映射表；
3. 旧 page 之后再统一回收。

### 8.4 Wear Leveling（磨损均衡）

wear leveling 的目标是让 erase 操作尽量均匀分布到不同物理 block 上，避免某些 block 过早损坏。

---

## 9. Storage Class Memory / NVM

NVM（Non-Volatile Memory）介于 DRAM 和 SSD 之间。

它的特点是：

- 非易失；
- 延迟低于 SSD；
- 可接近主存速度；
- 某些 NVM 支持 byte-addressable；
- 但写入寿命和延迟仍不同于 DRAM。

可以把它理解成一种“像内存一样快、又能持久保存”的存储层，但成本、寿命和系统支持会影响实际使用方式。

---

## 10. 本节速记

- 数据库物理存储优化的核心是减少 I/O，尤其减少随机 I/O。
- 磁盘访问时间主要由 seek time 和 rotational latency 决定。
- 顺序访问远快于随机访问。
- block 是数据库存储和传输的基本单位。
- buffering、read-ahead、disk scheduling、连续文件组织、非易失写缓存、log disk 都是为了降低磁盘访问代价。
- SSD 随机访问更强，但写入涉及 erase 和 wear leveling。
- NVM 同时具备持久性和更低访问延迟，是介于 DRAM 与 SSD 之间的新层次。
