---
counter: true
---

# Chap 2: Memory Hierarchy

***"本章核心"***

- **存储层次结构**：用少量高速存储器接近处理器，用大量低成本存储器提供容量，依靠局部性让平均访问时间接近上层存储。
- **主存技术**：SRAM 适合 cache，DRAM 适合 main memory；DRAM 通过 row buffer、SDRAM、DDR、多 bank、HBM 等提高带宽。
- **Cache 性能**：核心公式是 `AMAT = Hit time + Miss rate x Miss penalty`，优化都围绕 hit time、miss rate、miss penalty 和 bandwidth。
- **Miss 类型**：compulsory、capacity、conflict 三类 miss 分别对应冷启动、容量不足和映射冲突。
- **基础 cache 优化**：更大块、更大 cache、更高相联度、多级 cache、读 miss 优先、VIPT 地址索引。
- **高级 cache 优化**：小而简单的 L1、way prediction、流水/多 bank/非阻塞 cache、critical word first、write merging、编译器优化、prefetch、HBM cache。
- **虚拟内存**：把主存作为磁盘的 cache，同时提供保护、地址空间隔离和更大的虚拟地址空间。
- **TLB 与地址翻译**：TLB 是地址翻译的 cache；VIPT cache 利用 page offset 并行完成索引和地址翻译。
- **保护与虚拟机**：虚拟内存、特权级和 VMM 共同支持隔离、安全和多 OS 共享硬件。

## Memory Hierarchy

存储层次结构的目标是在速度、容量和成本之间折中：

```text
Registers -> L1 Cache -> L2/L3 Cache -> Main Memory -> Secondary Storage
 fast/small/expensive                         slow/large/cheap
```

它成立的基础是**局部性**：

- **时间局部性**：刚访问过的数据或指令，短时间内可能再次访问。
- **空间局部性**：访问某个地址后，附近地址也可能很快访问。

在体系结构中，cache 可以看作主存的缓存；主存也可以看作 secondary storage 的缓存。虚拟内存正是把这个思想扩展到主存和磁盘之间。

### Latency and Bandwidth

主存性能有两个重要指标：

| 指标 | 含义 | 更关注的场景 |
|---|---|---|
| **Latency** | 从发起读请求到第一个字到达的时间 | cache miss、指针追踪、随机访问 |
| **Bandwidth** | 单位时间能传输的数据量，或取回一个块剩余部分的速度 | 多处理器、I/O、大块 cache line、流式访问 |

课件强调：latency 比 bandwidth 更难降低；bandwidth 更容易通过新组织提升，例如多 bank、宽总线、DDR 和 HBM。

## Main Memory Technologies

### SRAM and DRAM

| 技术 | 典型用途 | 特点 |
|---|---|---|
| **SRAM** | Cache | 每 bit 约 6 个晶体管，不需要刷新，读不会破坏数据，access time 接近 cycle time，速度快但面积大、成本高 |
| **DRAM** | Main memory | 每 bit 约 1 个晶体管，密度高、成本低，但读会破坏信息，需要周期性 refresh，cycle time 大于 access time |

SRAM 更适合小容量高速 cache；DRAM 更适合大容量主存。两者的分工正是存储层次结构的典型体现。

### DRAM Organization

DRAM 通常组织为多个 **bank**，每个 bank 包含许多 row，每个 row 包含许多 column。

一次典型访问包含：

1. **Precharge**：准备或关闭 bank。
2. **Activate**：发送 row address，打开一行并把整行装入 row buffer。
3. **Column access**：通过 column address 从 row buffer 中取出所需数据。

如果连续访问命中同一个 row buffer，就不需要再次执行完整 row access，可以显著提高带宽。这就是 DRAM 利用空间局部性的方式。

### DRAM Improvements

| 改进 | 目标 | 要点 |
|---|---|---|
| **Row buffer** | 提升连续访问性能 | 一次打开一整行，后续 column access 更快 |
| **SDRAM** | 降低同步开销 | 在 DRAM 接口加入 clock signal，让连续传输更规则 |
| **Wider DRAM** | 提高带宽 | 扩大一次传输的数据宽度 |
| **DDR** | 提高峰值数据率 | 在时钟上升沿和下降沿都传输数据 |
| **Multiple banks** | 提供并行访问 | 将 SDRAM 分成多个可独立工作的 bank，类似 interleaving |
| **Power-down mode** | 降低功耗 | 空闲时让 DRAM 忽略 clock，只保留刷新 |

DRAM 功耗包括动态功耗和静态/待机功耗，并且依赖工作电压。读写活动带来动态功耗；待机时仍有刷新和漏电相关功耗。

### Graphics DRAM and HBM

图形处理器需要极高带宽，因此出现了面向 GPU 的 GDRAM/GSDRAM：

- 更宽接口，例如 32-bit；
- 数据引脚更高时钟频率；
- 直接连接 GPU 并安装在板卡上。

**HBM**（High Bandwidth Memory）更多是封装创新而非纯电路创新：把多个 DRAM 堆叠或放在处理器附近，通过大量、更短、更快的连接提高带宽并降低访问延迟。

### Flash and Phase-Change Memory

**Flash memory** 是一种 EEPROM：

- 非易失，断电后仍能保存内容；
- 写入前必须按块 erase；
- 每个 block 可写次数有限；
- 需要 **wear leveling**，让写入均匀分布，避免某些 block 过早磨损；
- 常用于 PMD 和 SSD。

**Phase-change memory** 相比 NAND Flash：

- 不需要写前擦除 page；
- 写性能可高约 10 倍；
- 读延迟可低约 2-3 倍；
- 但具体系统是否采用仍取决于成本、寿命、密度和接口生态。

## Memory Dependability

存储器错误可以分为：

| 错误 | 含义 |
|---|---|
| **Soft error / transient fault** | 单元内容改变，但电路没有永久损坏 |
| **Hard error / permanent fault** | 一个或多个存储单元的工作永久改变 |

课件中的术语位置有些容易混淆，记忆时抓本质即可：soft/transient 是暂态内容错误，hard/permanent 是永久硬件故障。

常见保护方式：

- **Parity**：每组数据加 1 bit 奇偶校验，可检测单 bit 错误，但不能纠正。
- **ECC**：能检测并纠正常见 bit 错误。
- **Chipkill**：类似 RAID，把数据和 ECC 分布在多个 memory chip 上，可处理多个错误或整颗芯片故障。

大型服务器内存容量巨大，单个 bit 出错概率很低也会被系统规模放大，因此依赖 ECC/Chipkill 这类机制。

## Cache Basics

Cache 是主存和处理器之间的小而快的存储器。它保存主存中最近或附近可能会用到的 block。

四个基本问题贯穿所有存储层次：

| 问题 | Cache 中的典型答案 |
|---|---|
| Q1. Where to place a block? | direct mapped、set associative、fully associative |
| Q2. How to find a block? | index 找 set，tag 比较确认 block |
| Q3. Which block to replace? | random、LRU、pseudo-LRU 等 |
| Q4. What happens on a write? | write-through / write-back，write-allocate / no-write-allocate |

### Direct Mapped Cache

Direct mapped cache 中，每个 memory block 只能放到一个 cache line。

地址通常拆成：

```text
tag | index | block offset
```

其中：

$$
\text{block offset bits}=\log_2(\text{block size})
$$

$$
\text{index bits}=\log_2(\text{number of cache lines})
$$

如果两个地址的 index 相同但 tag 不同，它们会映射到同一行，彼此替换，产生 conflict miss。

### Set Associative Cache

Set associative cache 中，每个 memory block 映射到一个 set，但可以放在该 set 的任意 way。

若 cache 容量为 $C$，block size 为 $B$，相联度为 $A$：

$$
\text{number of sets}=\frac{C}{B\times A}
$$

地址拆分为：

```text
tag | set index | block offset
```

相联度越高，conflict miss 通常越少，但 tag 比较、选择器和功耗开销增加，hit time 可能变长。

### Fully Associative Cache

Fully associative cache 中，一个 block 可以放在任意 cache line。

优点是几乎消除映射冲突；缺点是查找必须并行比较许多 tag，硬件复杂、功耗高、延迟大。因此它通常用于容量较小、miss penalty 很高的结构，例如 TLB。

## Cache Performance

### Average Memory Access Time

最重要的公式：

$$
\text{AMAT}
=
\text{Hit time}
+
\text{Miss rate}\times\text{Miss penalty}
$$

完整展开也可以写成：

$$
\text{AMAT}
=(1-\text{Miss rate})\times\text{Hit time}
+
\text{Miss rate}\times(\text{Hit time}+\text{Miss penalty})
$$

化简后得到同一个公式。

Cache 优化本质上是在权衡：

- 降低 **hit time**；
- 降低 **miss rate**；
- 降低 **miss penalty**；
- 提高 **cache bandwidth**；
- 同时控制 power、area、complexity。

### Three Cs of Misses

| Miss 类型 | 原因 | 典型优化 |
|---|---|---|
| **Compulsory miss** | cold-start / first-reference，第一次访问必 miss | 增大 block、prefetch |
| **Capacity miss** | cache 总容量不够，block 被丢弃后又要取回 | 增大 cache、blocking |
| **Conflict miss** | direct mapped 或 set associative 中多个 block 竞争同一 set | 提高相联度、victim cache、way prediction |

注意：cache full 不一定意味着 conflict miss；cache 没满时也可能因为映射限制发生 conflict miss。

## Six Basic Cache Optimizations

课件把基础 cache 优化概括为 6 类。

### Opt 1: Larger Block Size

更大的 block size 可以利用空间局部性，减少 compulsory miss。

代价：

- block 数量减少，可能增加 capacity/conflict miss；
- 每次 miss 要传输更多数据，miss penalty 增大；
- 如果程序空间局部性差，额外数据没有用，还浪费带宽和能耗。

经验上，block size 不是越大越好，而是要用 AMAT 比较：

$$
\text{AMAT}=\text{Hit time}+\text{Miss rate}\times\text{Miss penalty}
$$

### Opt 2: Larger Cache

更大的 cache 可以减少 capacity miss。

代价：

- hit time 可能增加；
- 面积、成本、静态功耗和动态功耗增加；
- L1 cache 太大可能拖慢处理器周期。

### Opt 3: Higher Associativity

提高相联度可以减少 conflict miss。

代价：

- 同一个 set 中要比较更多 tag；
- 数据选择更复杂；
- hit time 和功耗可能增加。

课件提到 **2:1 cache rule of thumb**：

> 一个大小为 $N$ 的 direct-mapped cache，miss rate 大致接近一个大小为 $N/2$ 的 two-way set associative cache。

这只是经验规则，真实结论要看 workload 和 cache 参数。

### Opt 4: Multilevel Cache

多级 cache 用小而快的 L1 保持快 hit time，用更大的 L2/L3 降低 miss penalty。

两级 cache 的 AMAT：

$$
\text{AMAT}
=
\text{Hit time}_{L1}
+
\text{Miss rate}_{L1}
\times
\text{Miss penalty}_{L1}
$$

其中：

$$
\text{Miss penalty}_{L1}
=
\text{Hit time}_{L2}
+
\text{Miss rate}_{L2}
\times
\text{Miss penalty}_{L2}
$$

所以：

$$
\text{AMAT}
=
\text{Hit time}_{L1}
+
\text{Miss rate}_{L1}
\times
(\text{Hit time}_{L2}
+
\text{Miss rate}_{L2}\times\text{Miss penalty}_{L2})
$$

平均每条指令的 memory stall cycles：

$$
\text{Memory stalls per instruction}
=
\text{Misses per instruction}_{L1}\times\text{Hit time}_{L2}
+
\text{Misses per instruction}_{L2}\times\text{Miss penalty}_{L2}
$$

### Local and Global Miss Rate

多级 cache 中必须区分 local miss rate 和 global miss rate。

| 指标 | 定义 |
|---|---|
| **Local miss rate** | 该级 cache 的 miss 数 / 访问该级 cache 的次数 |
| **Global miss rate** | 该级 cache 的 miss 数 / 处理器产生的总 memory access 数 |

对 L1：

$$
\text{Local miss rate}_{L1}
=
\text{Global miss rate}_{L1}
$$

对 L2：

$$
\text{Global miss rate}_{L2}
=
\text{Miss rate}_{L1}\times\text{Local miss rate}_{L2}
$$

课件例子：

- 1000 次 memory references；
- L1 有 40 次 miss；
- L2 有 20 次 miss；
- L2 hit time 为 10 cycles；
- L2 miss penalty 为 200 cycles；
- L1 hit time 为 1 cycle；
- 每条指令 1.5 次 memory references。

则：

$$
\text{L1 miss rate}=40/1000=4\%
$$

$$
\text{L2 local miss rate}=20/40=50\%
$$

$$
\text{L2 global miss rate}=20/1000=2\%
$$

AMAT：

$$
1+0.04\times(10+0.5\times200)=5.4\ \text{cycles}
$$

平均每条指令 stall：

$$
1.5\times0.04\times10+1.5\times0.02\times200
=0.6+6=6.6\ \text{cycles/instruction}
$$

### Inclusion and Exclusion

多级 cache 有两类组织：

- **Inclusive cache**：L1 中的数据也总是在 L2 中。优点是一致性检查更方便，只需查 L2；缺点是有效容量被重复占用。
- **Exclusive cache**：L1 和 L2 尽量不重复保存同一 block。优点是总有效容量更大；缺点是替换和一致性处理更复杂。

### Opt 5: Prioritize Read Misses over Writes

写缓冲区可能保存尚未写入下层存储的写操作。若读 miss 直接等待 write buffer 清空，会增加 miss penalty。

优化思路：

- 读 miss 时先检查 write buffer；
- 如果读地址与 write buffer 无冲突，并且内存系统可用，就让读 miss 先继续；
- 如果冲突，则需要从 write buffer 中转发或等待正确顺序。

这类优化体现了一个事实：多数程序对 read latency 更敏感，因为 load 的结果常在后续指令中被使用。

### Opt 6: Avoid Address Translation During Indexing

处理器产生 virtual address，但物理 cache 通常需要 physical address 来比较 tag。

若先完成地址翻译再访问 cache，会增加 hit time。解决方案是 **VIPT**：

> Virtually Indexed, Physically Tagged

思路：

1. 用 virtual address 的 page offset 部分进行 cache index。
2. TLB 同时翻译出 physical page number。
3. 用 physical tag 做 tag match。

因为 page offset 在虚拟地址和物理地址中相同，所以可以在地址翻译完成前先索引 cache。

对 direct-mapped cache，若要不经翻译就索引 cache，需要：

$$
\text{page offset bits} \ge \text{set index bits}+\text{block offset bits}
$$

也就是：

$$
\text{page size} \ge \text{cache size}
$$

对 $A$-way set associative cache，更一般地：

$$
\text{page size} \ge \frac{\text{cache size}}{A}
$$

这就是为什么提高相联度也能帮助 VIPT cache 做得更大。

## Ten Advanced Cache Optimizations

进阶课件的目标仍然是优化 AMAT，但额外关注 cache bandwidth 和 power。

### Overview

| 目标 | 优化 |
|---|---|
| Reduce hit time | small and simple L1、way prediction |
| Increase cache bandwidth | pipelined access、multibanked cache、nonblocking cache |
| Reduce miss penalty | critical word first、early restart、merging write buffer |
| Reduce miss rate | compiler optimization |
| Reduce miss penalty/rate | hardware prefetching、compiler prefetching |
| Improve high-bandwidth capacity | HBM as L4 cache |

### Opt 1: Small and Simple First-Level Caches

L1 cache 直接影响处理器周期，因此通常设计得小而简单：

- 小容量支持更快 clock cycle；
- 低相联度降低 tag 比较和选择器复杂度；
- direct-mapped cache 可以把 tag check 和 data transmission 部分重叠；
- 还能降低功耗。

代价是 miss rate 可能上升，所以通常用 L2/L3 承接容量需求。

### Opt 2: Way Prediction

Way prediction 用预测位猜测下一次访问会命中 set 中哪个 way。

流程：

1. 先按预测 way 读取数据，减少 hit time 和功耗；
2. tag check 与读取并行进行；
3. 若预测正确，快速返回；
4. 若预测错误，再检查其他 way，增加一个额外延迟。

它适用于 set associative 或 fully associative cache。目标是在保持较低 conflict miss 的同时，接近 direct-mapped cache 的访问速度。

### Opt 3: Pipelined and Multibanked Caches

**Pipelined cache access** 把 cache 访问拆成多个流水阶段，提高吞吐率。

代价：

- 单次访问 latency 增加；
- 分支误预测 penalty 更大；
- load 发出到使用数据之间的周期数增加。

**Multibanked cache** 把 cache 分成多个独立 bank，可支持多个并发访问。

常用方式是 sequential interleaving：连续地址分散到不同 bank 中。若同时访问落在不同 bank，可以并行；若落在同一 bank，仍会 bank conflict。

### Opt 4: Nonblocking Caches

Nonblocking cache 又叫 lockup-free cache。它允许 cache 在一个 miss 尚未完成时继续服务其他 hit 或 miss。

常见能力层次：

- **hit under miss**：miss 未返回时，其他命中的访问仍可继续。
- **miss under miss**：一个 miss 未返回时，还能发起其他 miss。
- **hit under multiple misses**：多个 miss 同时 outstanding 时，hit 仍能继续。

这种优化与乱序执行配合很好，因为处理器可以绕过等待内存的指令，继续执行独立指令。

硬件上通常依赖 MSHR（miss status holding register）记录 outstanding miss 的地址、目标寄存器和需要唤醒的请求。

### Opt 5: Critical Word First and Early Restart

处理器 miss 时通常急需 block 中的某一个 word，而不是整个 block。

**Critical word first**：

- 先向内存请求真正 miss 的 word；
- 该 word 一返回就交给处理器；
- 再继续填充 cache block 的其他 word。

**Early restart**：

- 按正常顺序取回 block；
- 一旦处理器需要的 word 到达，就重启处理器；
- 不必等待整个 block 填完。

二者都降低有效 miss penalty，尤其适合 block size 较大时。

### Opt 6: Merging Write Buffer

Write buffer 中可能有多个连续地址的写入。**Write merging** 把这些相邻写合并成一个 buffer entry。

好处：

- 减少 write buffer entry 占用；
- 减少向下层 memory 发送的独立事务数；
- 提高写带宽利用率。

### Opt 7: Compiler Optimization

编译器可以不改硬件而降低 miss rate。

#### Loop Interchange

Loop interchange 交换循环嵌套顺序，让程序按内存布局顺序访问数据。

例如 C 语言数组按 row-major 存储，连续访问 `a[i][j]` 的 `j` 维通常更友好。

#### Blocking

Blocking 又叫 tiling，把大矩阵运算分成小块，让一个小块的数据在被替换前尽量重复使用。

矩阵乘法中，原始代码可能反复跨行/跨列访问，导致 cache 中刚加载的数据还没充分使用就被替换。Blocking 的目标是：

> maximize accesses to loaded data before they are replaced

它通过增强时间局部性和空间局部性降低 miss rate。

### Opt 8: Hardware Prefetching

Hardware prefetching 在处理器真正请求数据之前，把预测会用到的 block 提前取入 cache 或外部 buffer。

典型 instruction prefetch：

- miss 时一次取两个 block；
- requested block 放入 cache；
- next consecutive block 放入 instruction stream buffer；
- 若后续访问正好需要该 block，就可以减少 miss。

Data prefetch 可以用类似思想，但更难，因为数据访问模式比指令流更不规则。

### Opt 9: Compiler Prefetching

编译器可插入 prefetch 指令，在数据真正使用前提前请求。

两种形式：

- **Register prefetch**：把数据预取到寄存器。
- **Cache prefetch**：把数据预取到 cache。

课件例子的核心思想：

- 原始循环每次迭代 7 cycles；
- cache miss penalty 为 100 cycles；
- 若不 prefetch，miss 数很多，执行时间被 miss penalty 主导；
- 编译器把循环分成预取阶段和计算阶段，使 prefetch 尽量与计算重叠；
- prefetch 成功时，大量 miss penalty 被隐藏。

prefetch 的风险：

- 过早 prefetch，数据可能在使用前被替换；
- 过晚 prefetch，来不及隐藏延迟；
- 预取无用数据会浪费带宽和功耗；
- aggressive prefetch 可能污染 cache。

### Opt 10: HBM as L4 Cache

HBM 可以作为大容量 L4 cache 使用，但标签开销可能很大。

课件例子：

- 1 GiB L4 cache；
- 64 B block size 时 tag 可能达到约 96 MiB；
- 4 KiB block size 时 tag 约 1 MiB。

优化方向：

1. **Place tags and data in the same row**：把 tag 和 data 放在同一 DRAM row。打开 row 很慢，但打开后访问更快；先查 tag，命中再读 data。
2. **Alloy cache**：把 tag 和 data 融合在一起，采用 direct-mapped 结构，直接 index HBM cache 并 burst 读出。

HBM miss 可能需要两次 DRAM 访问：一次拿 tag，一次去 main memory 拿数据。因此快速检测 miss 对降低 miss time 很重要。

## Virtual Memory

虚拟内存可以理解为：

```text
Virtual Memory = Main Memory + Secondary Storage
```

它有两个主要作用：

- 把主存作为磁盘的 cache，让程序拥有比物理内存更大的地址空间；
- 提供保护和隔离，让不同进程只能访问自己允许访问的地址。

### Before Virtual Memory

没有虚拟内存时，进程通常被连续分配到物理内存中。

问题：

- 进程之间容易互相破坏内存；
- 物理内存碎片管理困难；
- 进程需要知道或依赖物理地址；
- 当内存不足时，换入换出粒度和地址重定位都很麻烦。

虚拟内存让每个进程看到独立、连续的 virtual address space，而 OS 和硬件负责把 virtual page 映射到 physical page。

### Four Memory Hierarchy Questions for VM

虚拟内存也回答四个存储层次问题。

#### Q1: Where to Place a Block?

虚拟内存中，block 是 page。

因为 page fault penalty 极高，可能需要访问磁盘，所以 OS 通常允许 virtual page 放到主存任意 physical page frame 中，也就是 fully associative placement。

#### Q2: How to Find a Block?

通过 page table。

地址拆分：

```text
virtual address = virtual page number | page offset
physical address = physical page number | page offset
```

Page offset 不变，VPN 经 page table 翻译成 PPN。

如果没有 TLB，一次数据访问逻辑上需要两次内存访问：

1. 访问 page table entry，得到 physical page number；
2. 用 physical address 访问真正数据。

如果 page table entry 表示该页不在主存，就发生 page fault，由 OS 的 page fault handler 从磁盘取回页面，更新 page table，然后重新执行访问。

#### Q3: Which Block to Replace?

虚拟内存 miss 的代价极高，所以倾向使用接近 LRU 的替换策略。

硬件和 OS 通常配合使用 **reference/use bit**：

- 页面被访问时，硬件逻辑上设置 use bit；
- OS 周期性清除并检查这些 bit；
- 由此估计哪些页面最近较少使用。

#### Q4: What Happens on a Write?

虚拟内存通常采用 **write-back**。

原因是访问磁盘需要数百万个 clock cycles，不能每次写都同步写回磁盘。

Page table entry 中的 **dirty bit** 记录页面是否被修改过：

- dirty page 被替换时必须写回磁盘；
- clean page 可直接丢弃，因为磁盘中已有相同副本。

## Page Tables

Page table 存放从 VPN 到 PPN 的映射，以及保护和状态信息。

### Page Table Size

课件例子：

- 32-bit virtual address；
- 4 KiB page；
- 每个 page table entry 为 4 bytes。

页内偏移：

$$
\log_2(4\text{ KiB})=12
$$

虚拟页数：

$$
2^{32}/2^{12}=2^{20}
$$

页表大小：

$$
2^{20}\times4\text{ bytes}=2^{22}\text{ bytes}=4\text{ MiB}
$$

这只是单个进程的线性页表大小。现实系统通常需要多级页表、反向页表或其他压缩机制。

### TLB

**TLB**（Translation Lookaside Buffer）是地址翻译的 cache。

TLB entry 通常包含：

- tag：virtual address 的一部分，通常是 VPN 或 VPN 的 tag；
- data：physical page frame number；
- protection field：访问权限；
- valid bit；
- use/reference bit；
- dirty bit。

TLB hit 时，地址翻译很快完成；TLB miss 时需要访问 page table，甚至可能触发 page fault。

### TLB Access Flow

典型数据访问路径：

1. CPU 产生 virtual address。
2. TLB 并行比较 tag。
3. 检查访问权限，例如 read/write、user/supervisor。
4. 命中后输出 physical page number。
5. 拼接 page offset，形成 physical address。
6. 用 physical address 访问 cache 或 memory。

如果权限不满足，产生 protection exception；如果页不在内存，产生 page fault。

## Page Size Selection

Page size 会影响页表大小、TLB 覆盖范围、内部碎片、page fault cost 和 cache 设计。

### Larger Page Size

优点：

- 页表项更少，page table 更小；
- 每个 TLB entry 覆盖更多内存，TLB miss 减少；
- 从 secondary storage 传输大页效率通常更高；
- 更容易支持较大的 VIPT cache。

缺点：

- 内部碎片更严重；
- page fault 时传输更多数据；
- 如果程序局部性差，会浪费内存和带宽。

### Smaller Page Size

优点：

- 内部碎片更少；
- 小对象和稀疏地址空间更节省内存；
- page fault 时单次搬运数据更少。

缺点：

- 页表更大；
- TLB 覆盖范围变小，TLB miss 可能增加；
- 管理开销更高。

现代处理器常支持 **multiple page sizes**，因为不同 workload 对 page size 的需求不同。大页可以减少 TLB miss；小页可以减少碎片。

## Virtual Memory and Protection

体系结构必须限制 user process 能访问的内容，同时允许 OS process 拥有更多权限。

课件列出的四个任务：

1. 提供至少两种模式：user mode 和 supervisor/kernel mode。
2. 提供一部分处理器状态，user process 可以读取或使用，但不能随意写。
3. 提供从 user mode 进入 supervisor mode 的机制，例如 system call，也要能返回 user mode。
4. 提供限制内存访问的机制，在不频繁 swap 的情况下保护进程内存状态。

虚拟内存和特权级共同提供 isolation。进程看到的是自己的虚拟地址空间，实际访问必须经过 page table 和权限检查。

## Virtual Machines

**Virtual Machine** 可以看作一种更强的保护和隔离抽象。它让一台物理机器运行多个 VM，每个 VM 可运行不同 OS。

**VMM**（Virtual Machine Monitor）也叫 hypervisor。

VMM 的三条基本特征：

1. 为程序提供与原机器基本相同的运行环境。
2. 在这个环境中的程序只表现出较小性能损失。
3. VMM 完全控制系统资源。

虚拟化的动机：

- 现代系统越来越重视 isolation 和 security；
- 传统 OS 代码规模大，漏洞和可靠性问题多；
- 单台计算机需要被多个用户或多个 OS 安全共享；
- 便于迁移、管理、测试和资源隔离。

### VMM Requirements

定性要求：

- guest software 在 VM 上应像在真实硬件上一样运行；
- guest software 不能直接改变真实系统资源；
- VMM 必须能捕获并模拟敏感操作。

特权要求：

- 至少需要 system/user 两种 processor modes；
- privileged instructions 只能在 system mode 执行；
- 当 guest OS 尝试执行敏感指令时，VMM 能接管控制。

## Address Decomposition Examples

做 cache/TLB 题时，通常按以下顺序拆地址。

### Cache Address Bits

给定：

- cache size $C$；
- block size $B$；
- associativity $A$；
- physical address bits $P$。

则：

$$
\text{block offset bits}=\log_2 B
$$

$$
\text{set index bits}=\log_2\frac{C}{B\times A}
$$

$$
\text{tag bits}=P-\text{set index bits}-\text{block offset bits}
$$

### Page and TLB Bits

给定：

- virtual address bits $V$；
- physical address bits $P$；
- page size $S$。

则：

$$
\text{page offset bits}=\log_2 S
$$

$$
\text{VPN bits}=V-\text{page offset bits}
$$

$$
\text{PPN bits}=P-\text{page offset bits}
$$

若 TLB 有 $E$ 个 entries，$A$-way set associative：

$$
\text{TLB set index bits}=\log_2\frac{E}{A}
$$

$$
\text{TLB tag bits}=\text{VPN bits}-\text{TLB set index bits}
$$

## Processor Examples

### ARM Cortex-A53

课件给出的 ARM Cortex-A53 特点：

- ARMv8-A ISA；
- 支持 32-bit / 64-bit mode；
- 最多 2 instructions per clock；
- clock rate 可达约 1.3 GHz；
- 作为 IP core 集成到 SoC 中；
- memory hierarchy 包含 two-level TLB、two-level cache 和 LRU 近似替换策略。

课件通过 instruction access path 和 data access path 展示了地址拆分：

- 64 KiB page：page offset 为 16 bits；
- 64 B block：block offset 为 6 bits；
- 32 KiB cache、2-way 时，set index 为 $\log_2(32\text{ KiB}/(64\text{ B}\times2))=8$ bits；
- 32 KiB cache、4-way 时，set index 为 7 bits。

这类题的关键不是背图，而是按容量、块大小和相联度拆 bit。

### Intel Core i7-6700

课件给出的 Intel Core i7-6700 特点：

- 64-bit x86-64 ISA；
- out-of-order 4-core processor；
- multiple issue、dynamically scheduled；
- 16-stage pipeline；
- 每核每周期最多 4 条指令；
- 最多 3 个并行 memory channel；
- DDR3-1066；
- peak memory bandwidth 超过 25 GB/s；
- 48-bit virtual address；
- 36-bit physical address，约 36 GiB physical memory；
- two-level TLB；
- three-level cache hierarchy；
- L1 是 virtually indexed and physically tagged；
- L2/L3 是 physically indexed。

课件中的性能观察：

- instruction cache miss rate 在 SPECInt2006 上接近 0 或低于 1%；
- data cache miss rate 随 benchmark 差异很大；
- prefetch 很激进，可能有大量 prefetch request；
- L2 miss 中很大比例可能来自 prefetch miss；
- L3 average miss rate 约 0.5%。

这说明：真实处理器的 memory performance 不能只看单一 miss rate，必须同时看 demand access、prefetch access、各级 local/global miss rate 和 miss penalty。

## Exam Notes

- AMAT 公式必须熟：$\text{AMAT}=\text{Hit time}+\text{Miss rate}\times\text{Miss penalty}$。
- 多级 cache 题中，先分清 **local miss rate** 和 **global miss rate**。
- Larger block size 主要减少 compulsory miss，但可能增加 miss penalty 和 conflict/capacity miss。
- Larger cache 主要减少 capacity miss，但可能增加 hit time、cost 和 power。
- Higher associativity 主要减少 conflict miss，但可能增加 hit time 和 power。
- Nonblocking cache 的关键词是 `hit under miss`、`miss under miss` 和 MSHR。
- VIPT cache 的核心条件是 page offset 能覆盖 set index 和 block offset。
- Page table 题先算 page offset，再算 VPN 数量和 PTE 数量。
- TLB 是地址翻译的 cache，不是普通数据 cache。
- 虚拟内存的写策略通常是 write-back，并用 dirty bit 判断替换时是否写回磁盘。
- 大页减少 TLB miss 和页表开销，小页减少内部碎片。
- 虚拟化依赖 user/supervisor mode、特权指令、异常/陷入机制和 VMM 对资源的控制。
