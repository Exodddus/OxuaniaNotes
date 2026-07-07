---
counter: true
---

# Assignment 3 Answers

说明：题目要求只需自选五题用于评分，但这里按复习目的逐一回答全部 32 题。若需要提交评分，可从中挑选最熟悉的五题。

## 参考资料

- 本仓库笔记：[Chap1 Fundamentals](../Chap1%20Fundamentals.md)、[Chap2 Memory](../Chap2%20Memory.md)、[Chap3-1 ILP Static](../Chap3-1%20ILP%20Static.md)、[Chap3-2 ILP Dynamic](../Chap3-2%20ILP%20Dynamic.md)、[Chap4 Data-Level Parallelism](../Chap4%20Data-Level%20Parallelism.md)、[Chap5 Thread-Level Parallelism](../Chap5%20Thread-Level%20Parallelism.md)。
- 本仓库课件：`2026Arch_2_Ch1_1.pdf`、`2026Arch_3_Ch1_2.pdf`、`chapter2-memory-basics.pdf`、`chapter2-memory-advances.pdf`、`chapter3-ilp-static.pdf`、`chapter3-ilp-dynamic.pdf`、`chapter4-dlp-architecture.pdf`、`chapter5-tlp-coherence.pdf`。
- 网络资料：[Intel Moore's Law press kit](https://newsroom.intel.com/press-kit/moores-law)、[SPEC CPU 2017 run rules](https://www.spec.org/cpu2017/Docs/runrules.html)、[NVIDIA CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/index.html)、[RISC-V Vector Extension spec](https://github.com/riscvarchive/riscv-v-spec/blob/master/v-spec.adoc)、[Linux kernel memory barriers guide](https://docs.kernel.org/core-api/wrappers/memory-barriers.html)。

## 01. RISC 架构的两个关键性能技术

RISC 的两个典型性能技术是 **pipelining** 和 **compiler-assisted simple instruction execution**，也可以概括为“简单规整指令 + 流水线化执行”。

**流水线**把一条指令拆成 IF、ID、EX、MEM、WB 等阶段，让多条指令重叠执行。它不减少单条指令的延迟，但能提高吞吐率，理想情况下 $k$ 级流水线的最大加速接近 $k$。

**简单规整的 load-store ISA** 让绝大多数运算只发生在寄存器之间，访存只由 load/store 完成，指令格式、执行时间和控制逻辑更规则。这有利于：

- 缩短时钟周期；
- 简化译码和控制；
- 更容易做 forwarding、hazard detection 和 branch handling；
- 让编译器通过寄存器分配、指令调度、循环展开暴露 ILP。

所以 RISC 的性能优势并不是“单条指令更强”，而是让硬件和编译器更容易把大量简单指令高吞吐地执行。

## 02. Dennard Scaling 与 Moore's Law

**Moore's Law** 观察到：集成电路上的晶体管数量大约每两年翻倍，而单位成本没有同比上升。Intel 对 Moore's Law 的官方说明也强调它是经验性预测，不是自然定律。

**Dennard Scaling** 的关键观察是：当 MOSFET 尺寸按比例缩小时，电压和电流也可按比例降低，使功率密度近似保持不变。因此在经典时期，晶体管更小意味着：

- 晶体管更多；
- 电容更小；
- 电压更低；
- 频率可提高；
- 总功耗不必显著上升。

它们在高级 CPU 设计中逐渐失效，原因包括：

- 电压不能继续按比例降低，否则噪声容限和阈值电压受限；
- 漏电流上升，静态功耗占比增加；
- 频率继续提高会遇到 power wall 和散热限制；
- 线延迟、互连复杂度、验证成本和制造成本上升；
- 晶体管更多不等于所有晶体管都能同时开启，出现 dark silicon。

结论是：工艺仍能增加晶体管数量，但单线程频率和性能不能像过去那样免费增长，因此体系结构转向多核、DLP、TLP、加速器和 DSA。

## 03. Amdahl's Law 的结论与例子

Amdahl 定律说明：整体加速比受未优化部分限制。若程序中比例 $f$ 的部分被加速 $S$ 倍，则：

$$
\text{Speedup}=\frac{1}{(1-f)+\frac{f}{S}}
$$

若 $f$ 是可并行部分，使用 $N$ 个处理器：

$$
\text{Speedup}=\frac{1}{(1-f)+\frac{f}{N}}
$$

若 $s$ 是串行部分：

$$
\text{Speedup}=\frac{1}{s+\frac{1-s}{N}}
$$

例 1：程序 80% 可并行，使用 4 个处理器：

$$
\text{Speedup}=\frac{1}{0.2+\frac{0.8}{4}}=\frac{1}{0.4}=2.5
$$

例 2：某程序 30% 部分加速 10 倍，其余不变：

$$
\text{Speedup}=\frac{1}{0.7+\frac{0.3}{10}}=\frac{1}{0.73}=1.37
$$

例 3：两个部分分别占 40% 和 30%，分别加速 4 倍和 3 倍，剩下 30% 不变：

$$
\text{Speedup}=\frac{1}{0.3+\frac{0.4}{4}+\frac{0.3}{3}}=\frac{1}{0.5}=2
$$

核心结论：优化高占比部分才有意义；串行部分越大，并行硬件越多也越难提高整体性能。

## 04. 四类并行架构

按 Flynn taxonomy，四类并行架构是：

| 类型 | 含义 | 例子或技术 |
|---|---|---|
| **SISD** | 单指令流、单数据流 | 传统单核串行处理器；可用流水线提高吞吐，但不是 Flynn 意义上的并行机器 |
| **SIMD** | 单指令流、多数据流 | 向量处理器、Multimedia SIMD、GPU 的 SIMT 风格执行 |
| **MISD** | 多指令流、单数据流 | 很少见，可联想到容错冗余流水线或同一数据流经多种处理 |
| **MIMD** | 多指令流、多数据流 | 多核 CPU、SMP、NUMA、集群、服务器 |

从课程角度，常见硬件利用并行性的方式是：

- ILP：流水线、超标量、乱序执行、分支预测、推测执行；
- DLP：向量架构、SIMD、GPU；
- TLP：多核、多处理器、多线程；
- RLP：数据中心和服务器中的请求级并行。

## 05. 寄存器、立即数和位移寻址示例

常见 addressing modes：

```asm
# Register addressing: 操作数在寄存器中
add x5, x6, x7        # x5 = x6 + x7

# Immediate addressing: 一个操作数是立即数
addi x5, x6, 16       # x5 = x6 + 16

# Displacement/Base addressing: 地址 = base register + displacement
ld x5, 32(x6)         # x5 = Memory[x6 + 32]
sd x5, 40(x6)         # Memory[x6 + 40] = x5

# PC-relative addressing: 分支目标 = PC + offset
beq x1, x2, Loop
```

寄存器寻址速度快；立即数适合常量；位移寻址适合访问结构体字段、数组元素、栈帧局部变量。

## 06. 没有技术进步时仍影响计算机成本的三个因素

即使没有新工艺，计算机成本仍会受三类因素影响：

1. **产量和规模经济**：开发成本摊到 10,000 台和 10,000,000 台上，单位成本差别巨大。
2. **学习曲线和制造成熟度**：良率提高、流程优化、供应链成熟都会降低成本。
3. **系统复杂度和市场定位**：组件数量、服务支持、可靠性要求、利润率不同，会导致服务器、PC、嵌入式系统成本结构不同。

课件还强调：更小系统、更少组件、低服务成本和更高产量，都能降低价格。

## 07. Fault、Error、Failure 示例

三者关系：

```text
fault -> error -> failure
```

例子：内存芯片受到宇宙射线影响，某个 bit 从 0 翻转为 1。

- **Fault**：宇宙射线造成的瞬态扰动，或存储单元永久损坏。
- **Error**：内存中实际保存的 bit 值错了。
- **Failure**：程序读取该错误值后输出错误结果，或者系统崩溃，对用户可见。

并非每个 fault 都会变成 failure。例如错误 bit 从未被读取，或 ECC 纠正了错误，就不会产生用户可见 failure。

## 08. 如何由子系统 MTTF 计算系统 MTTF

若系统中任意一个子系统故障都会导致系统故障，且各子系统故障独立、服从恒定故障率，则先把 MTTF 转换为故障率：

$$
\lambda_i=\frac{1}{\text{MTTF}_i}
$$

系统总故障率：

$$
\lambda_{\text{system}}=\sum_i \lambda_i
$$

系统 MTTF：

$$
\text{MTTF}_{\text{system}}=\frac{1}{\lambda_{\text{system}}}
$$

若某类组件有 $n$ 个，每个 MTTF 相同：

$$
\lambda=n\times\frac{1}{\text{MTTF}}
$$

例：10 个磁盘，每个 MTTF 为 1,000,000 小时；控制器 500,000 小时；电源 200,000 小时；风扇 200,000 小时；线缆 1,000,000 小时：

$$
\lambda=10/10^6+1/5\times10^5+1/2\times10^5+1/2\times10^5+1/10^6=0.000023
$$

$$
\text{MTTF}=1/0.000023\approx 43478\text{ hours}\approx 5\text{ years}
$$

## 09. RAID 0 到 RAID 6 的设计原则

| 级别 | 关键思想 | 容错 | 性能与代价 |
|---|---|---|---|
| RAID 0 | Striping，无冗余 | 不能容忍磁盘故障 | 并行读写快，容量利用率高，但可靠性下降 |
| RAID 1 | Mirroring | 可容忍一个镜像副本故障 | 读性能好，写入需写多个副本，容量利用率约 50% |
| RAID 2 | bit-level striping + Hamming ECC | 可纠错 | 需要同步磁盘和专用 ECC，现代少用 |
| RAID 3 | byte-level striping + dedicated parity disk | 可容忍一个磁盘故障 | 大顺序访问好，parity disk 成为瓶颈 |
| RAID 4 | block-level striping + dedicated parity disk | 可容忍一个磁盘故障 | 随机读较好，随机写仍受 parity disk 限制 |
| RAID 5 | block-level striping + distributed parity | 可容忍一个磁盘故障 | 分散 parity，避免单 parity disk 瓶颈；小写有 read-modify-write 代价 |
| RAID 6 | block-level striping + dual distributed parity | 可容忍两个磁盘故障 | 可靠性更高，写入校验计算和容量代价更大 |

RAID 的核心权衡是性能、容量利用率、可靠性和恢复成本。

## 10. 处理器性能公式：CPU time、CPI、IPC、Clock Cycles

核心公式：

$$
\text{CPU Time}=\text{Instruction Count}\times\text{CPI}\times\text{Clock Cycle Time}
$$

又因为：

$$
\text{Clock Cycle Time}=\frac{1}{\text{Clock Rate}}
$$

所以：

$$
\text{CPU Time}=\frac{\text{Instruction Count}\times\text{CPI}}{\text{Clock Rate}}
$$

总周期数：

$$
\text{Clock Cycles}=\text{Instruction Count}\times\text{CPI}
$$

CPI 和 IPC 的关系：

$$
\text{IPC}=\frac{1}{\text{CPI}}
$$

对多发射处理器，IPC 可大于 1。

影响：

- Instruction count 由算法、编译器、ISA 共同影响；
- CPI 由流水线、cache miss、分支预测、ILP、结构冲突等影响；
- clock cycle time 由工艺、电路、流水级数、功耗和关键路径影响。

不能只看频率。一个高频但 CPI 很差的处理器可能比低频但 IPC 高的处理器更慢。

## 11. Direct mapped、fully associative、set associative cache 组织原则

**Direct mapped cache**：每个 memory block 只能映射到一个 cache line。

```text
address = tag | index | block offset
```

优点是硬件简单、hit time 短；缺点是 conflict miss 多。

**Fully associative cache**：任意 memory block 可放入任意 cache line。

优点是几乎消除 conflict miss；缺点是需要并行比较所有 tag，硬件复杂、功耗高。常用于 TLB 这类小而 miss penalty 高的结构。

**Set associative cache**：memory block 映射到一个 set，可放入该 set 中任意 way。

```text
address = tag | set index | block offset
```

它是 direct mapped 和 fully associative 的折中。相联度越高，conflict miss 越少，但 hit time、功耗和复杂度增加。

## 12. Cache 地址映射示例

设系统 byte-addressed，地址为 16 bit，cache 大小 64 B，block size 为 8 B。

### Direct mapped

cache line 数：

$$
64/8=8=2^3
$$

block offset 3 bit，index 3 bit，tag 10 bit。

地址 `0x0034`：

$$
0x0034=52
$$

block number：

$$
52/8=6
$$

index：

$$
6\bmod 8=6
$$

tag：

$$
\lfloor 6/8\rfloor=0
$$

地址 `0x0074=116`：

block number $=116/8=14$，index $=14\bmod8=6$，tag $=1$。因此它与 `0x0034` 冲突，都会映射到 line 6。

### Fully associative

同样 64 B cache、8 B block，共 8 lines。地址 `0x0034` 的 block number 是 6，可以放到任意 line。查找时比较所有 line 的 tag。没有 index，只有：

```text
tag = block number, block offset = 3 bits
```

### 2-way set associative

64 B cache、8 B block、2-way：

$$
\text{sets}=64/(8\times2)=4=2^2
$$

block offset 3 bit，set index 2 bit。

`0x0034`：block number 6，set index $6\bmod4=2$，tag $\lfloor6/4\rfloor=1$。

`0x0074`：block number 14，set index $14\bmod4=2$，tag 3。

二者映射到同一个 set 2，但若该 set 有两个 way，可以同时存在，不一定冲突。

## 13. 单级与多级 cache 的 AMAT 公式

单级 cache：

$$
\text{AMAT}
=
\text{Hit time}
+
\text{Miss rate}\times\text{Miss penalty}
$$

展开式：

$$
\text{AMAT}
=(1-\text{MR})\times\text{HT}
+
\text{MR}\times(\text{HT}+\text{MP})
$$

两级 cache：

$$
\text{AMAT}
=
\text{HT}_{L1}
+
\text{MR}_{L1}
\times
(\text{HT}_{L2}
+
\text{MR}_{L2}\times\text{MP}_{L2})
$$

多级递归：

$$
\text{AMAT}_{L_i}
=
\text{HT}_{L_i}
+
\text{MR}_{L_i}\times\text{AMAT}_{L_{i+1}}
$$

若用每条指令 stall cycles：

$$
\text{Memory stalls per instruction}
=
\text{Misses per instruction}_{L1}\times\text{HT}_{L2}
+
\text{Misses per instruction}_{L2}\times\text{MP}_{L2}
$$

注意 local miss rate 和 global miss rate：

$$
\text{Global MR}_{L2}=\text{MR}_{L1}\times\text{Local MR}_{L2}
$$

## 14. Cache 优化技术及影响

| 技术 | 主要改善 | 代价 |
|---|---|---|
| Larger block size | 减少 compulsory miss，利用空间局部性 | miss penalty 增大，可能增加 conflict/capacity miss |
| Larger cache | 减少 capacity miss | hit time、成本、功耗增加 |
| Higher associativity | 减少 conflict miss | hit time 和功耗增加 |
| Multilevel cache | 降低有效 miss penalty | 层次更复杂，需处理 local/global miss rate |
| Prioritize read miss over write | 降低读 miss 等待写缓冲的 penalty | 需检查 write buffer 冲突 |
| VIPT cache | TLB 与 cache index 并行，降低 L1 hit time | 受 page offset 限制，需处理 synonym/alias |
| Small simple L1 | 降低 hit time 和功耗 | miss rate 可能上升 |
| Way prediction | 兼顾相联度和快速访问 | 预测错有额外延迟 |
| Pipelined cache | 提高 cache bandwidth | 单次 load latency 增加 |
| Multibanked cache | 支持并发访问 | 可能 bank conflict |
| Nonblocking cache | hit under miss、miss under miss | 需要 MSHR，控制复杂 |
| Critical word first / early restart | 降低有效 miss penalty | 内存返回顺序控制更复杂 |
| Write merging | 减少写事务和缓冲压力 | 需要合并逻辑 |
| Compiler loop interchange/blocking | 减少 miss rate | 依赖程序结构 |
| Hardware/compiler prefetch | 隐藏 miss penalty 或减少 miss | 可能污染 cache、浪费带宽和功耗 |
| HBM as L4 | 增大高带宽缓存容量 | tag 开销和 miss 检测复杂 |

总体目标仍是优化：

$$
\text{AMAT}=\text{Hit time}+\text{Miss rate}\times\text{Miss penalty}
$$

## 15. 两级 cache + TLB 下的地址翻译过程

以 VIPT L1 + physically indexed L2 为例：

1. CPU 生成 virtual address：

```text
VA = VPN | page offset
```

2. TLB 用 VPN 查找 PPN，同时 L1 cache 用 page offset 中的 set index 先索引 cache set。

3. 若 TLB hit，得到：

```text
PA = PPN | page offset
```

4. L1 用 physical tag 与 cache tag 比较。若匹配且 valid，L1 hit，返回数据。

5. 若 L1 miss，用 physical address 查 L2。L2 通常 physically indexed and physically tagged。

6. 若 L2 hit，把数据返回给 L1 和 CPU；若 L2 miss，访问主存并填充 L2/L1。

7. 若 TLB miss，硬件或 OS page table walker 访问 page table。若 PTE valid，填入 TLB 后重试；若页面不在内存，触发 page fault，由 OS 从磁盘调页。

VIPT 的关键条件：

$$
\text{page offset bits}\ge \text{set index bits}+\text{block offset bits}
$$

这样 indexing 不必等待地址翻译。

## 16. 三类依赖

三类 dependences：

1. **Data dependence**：后一条指令需要前一条指令产生的值，也叫 true dependence，对应 RAW。
2. **Name dependence**：两条指令使用同一寄存器或内存名字，但没有真实数据流，包括 WAR 和 WAW。
3. **Control dependence**：某条指令是否执行取决于前面的分支或条件。

例子：

```asm
I1: add x1, x2, x3
I2: sub x4, x1, x5   # RAW data dependence on I1
I3: add x1, x6, x7   # WAW name dependence with I1
```

数据依赖不能通过重命名消除；名称依赖可以通过寄存器重命名消除；控制依赖可通过预测、推测、predication 等减轻。

## 17. 常见 hazard 及 dependence 与 hazard 的关系

常见 hazards：

| Hazard | 含义 | 对应依赖 |
|---|---|---|
| **Structural hazard** | 多条指令竞争同一硬件资源 | 资源冲突 |
| **RAW** | Read after Write，后读需要前写结果 | 真数据依赖 |
| **WAR** | Write after Read，后写覆盖前读需要的旧值 | 名称依赖 |
| **WAW** | Write after Write，后写/前写顺序被打乱 | 名称依赖 |
| **Control hazard** | 分支方向或目标未确定 | 控制依赖 |

Dependence 是程序语义关系；hazard 是在具体流水线或调度机制中可能导致错误或停顿的实现问题。

有依赖不一定有 hazard。例如两条 RAW 相关指令间隔很远，结果已经写回，就不会停顿。反过来，结构 hazard 不一定来自程序依赖，而来自资源不足。

## 18. 常用 forwarding 方案

Forwarding/bypassing 的思想是：结果已经产生但还没写回寄存器时，直接把结果送给消费者。

常见路径：

- **EX/MEM -> EX**：上一条 ALU 指令结果送到下一条 ALU 输入。
- **MEM/WB -> EX**：较早指令写回前，把结果送到 EX。
- **MEM -> EX**：load 数据从 memory stage 送给后续 EX。
- **EX/MEM 或 MEM/WB -> store data input**：store 要写入的数据可从前面指令转发。
- **Forward to branch comparator**：分支比较需要的新值可转发到 ID/EX 比较逻辑。

典型 load-use hazard 即使有 forwarding 也常需停顿 1 cycle，因为 load 数据到 MEM 末尾才可用，而紧随其后的 EX 在更早时刻需要操作数。

## 19. 分支预测器例子与性质

| 预测器 | 用到的信息 | 存储开销 | 特点 |
|---|---|---|---|
| Static not taken | 不看历史 | 几乎无 | 简单，循环尾部表现差 |
| Static taken / BTFNT | 分支方向或偏移 | 很小 | backward taken, forward not taken 对循环较好 |
| 1-bit predictor | 每个分支上次结果 | 每项 1 bit | 循环边界常错两次，交替模式很差 |
| 2-bit saturating counter | 近期方向偏置 | 每项 2 bit | 需连续两次反向才改变强预测，抗噪声 |
| Local predictor | 同一静态分支的局部历史 | local history + PHT | 能学习单个分支周期模式 |
| Global/two-level predictor | 最近多个动态分支历史 | GHR + PHT | 能捕获跨分支相关性 |
| Gshare | PC XOR GHR | PHT + GHR | 降低简单拼接导致的 aliasing |
| Tournament predictor | 多个预测器 + 选择器 | 较大 | 对每个分支动态选择 local 或 global |
| TAGE | 多个几何长度历史表 + tag | 较大 | 现代高精度预测器代表 |
| BTB | 分支 PC 到目标地址 | tag + target | 预测 taken 时给出目标地址 |
| RAS | 调用返回栈 | 小栈 | 预测函数 return 地址 |

一个分支自身模式适合 local predictor；多个分支之间有关联时适合 global/correlating predictor；没有单一预测器适合所有分支，所以 tournament/TAGE 用多个信息源组合。

## 20. Superscalar 及其对 IPC/CPI 的帮助

**Superscalar** 处理器每个周期可以取指、译码、发射、执行多条指令。若理想每周期提交 $W$ 条指令，则 IPC 最高接近 $W$，CPI 最低接近 $1/W$。

它提高性能的方式：

- 多个功能单元并行执行独立指令；
- 动态调度绕过等待长延迟操作的指令；
- 分支预测和推测执行保持指令窗口有足够候选；
- 寄存器重命名消除 WAR/WAW，使更多指令并行。

限制：

- 程序本身 ILP 有限；
- 分支预测错误会浪费大量工作；
- cache miss 和长延迟操作会拖慢窗口；
- issue queue、ROB、rename、bypass 网络复杂度和功耗快速上升。

## 21. Scoreboarding、Tomasulo、Tomasulo with speculation

### Scoreboarding

Scoreboard 是集中式控制。

1. **Issue**：顺序发射。检查功能单元是否空闲，以及是否有 WAW。如果有结构冲突或 WAW，停发。
2. **Read operands**：等待所有 RAW 生产者完成，源操作数可读后再读寄存器。RAW 在此处理。
3. **Execution**：功能单元执行。不同指令可乱序开始和完成。
4. **Write result**：写回前检查是否会造成 WAR。如果更早指令还没读同名源寄存器，延迟写回。

Scoreboard 允许乱序执行，但通过阻塞处理 WAR/WAW，没有真正消除名称依赖。

### Tomasulo

Tomasulo 使用保留站、tag 和 CDB。

1. **Issue**：顺序发射到 reservation station。若目的寄存器会被写，寄存器结果状态记录生产者 tag，实现动态寄存器重命名。
2. **Execute**：若源操作数已有值，直接执行；若没有，保留站监听 CDB 上对应 tag 的结果。
3. **Write result**：执行完成后通过 CDB 广播 tag/value，等待者捕获结果，寄存器结果状态匹配时更新。

Tomasulo 保留 RAW，但通过 tag 重命名消除 WAR/WAW。

### Tomasulo with speculation

加入 ROB 支持硬件推测和精确异常。

1. **Issue/dispatch**：为指令分配 ROB entry 和 reservation station，目的寄存器 rename 到 ROB entry。
2. **Execute**：操作数就绪后乱序执行；branch 可按预测路径继续取指。
3. **Write result**：结果写入 ROB 或通过 CDB 唤醒等待者，但不立即更新体系结构状态。
4. **Commit**：ROB 头部按程序顺序提交。普通寄存器指令更新 architectural register；store 到提交时才写 memory；branch 若预测错，清空年轻指令并从正确 PC 重启；异常也在提交点处理。

关键区别：可以乱序执行、乱序完成，但通常必须顺序提交，以保证 precise exception。

## 22. 动态调度时间表示例

指令序列：

```asm
I1: LD   F2, 0(R1)
I2: MUL  F4, F2, F6
I3: ADD  F8, F10, F12
I4: ADD  F6, F14, F16
I5: ADD  F18, F4, F8
```

依赖：

- I2 RAW 依赖 I1 的 F2；
- I5 RAW 依赖 I2 的 F4 和 I3 的 F8；
- I4 写 F6，与 I2 读 F6 形成 WAR 名称依赖。

假设：

- 单发射；
- load 执行 4 cycles，mul 执行 4 cycles，add 执行 2 cycles；
- 有 1 个 load unit、1 个 mul unit、2 个 add units；
- 一个 CDB/写回端口；
- 同周期广播的结果下一周期可用于执行；
- Scoreboard 不重命名，Tomasulo 重命名。

一种 Scoreboard 时间表：

| 指令 | Issue | Read | Execute | Write |
|---|---:|---:|---|---:|
| I1 | 1 | 2 | 3-6 | 7 |
| I2 | 2 | 8 | 9-12 | 13 |
| I3 | 3 | 4 | 5-6 | 8 |
| I4 | 4 | 5 | 6-7 | 9 |
| I5 | 5 | 14 | 15-16 | 17 |

这里 I4 的写回要等 I2 读完旧 F6，否则会造成 WAR。若写端口冲突，也要按约定顺延。

一种 Tomasulo 时间表：

| 指令 | Issue | Execute | Write result |
|---|---:|---|---:|
| I1 | 1 | 2-5 | 6 |
| I2 | 2 | 7-10 | 11 |
| I3 | 3 | 4-5 | 7 |
| I4 | 4 | 5-6 | 8 |
| I5 | 5 | 12-13 | 14 |

Tomasulo 中 I4 的 F6 被重命名为新版本，不会覆盖 I2 需要的旧 F6，因此 WAR 不阻塞写回。

若加 ROB/speculation，执行和写结果时间可类似 Tomasulo，但 commit 必须按 I1、I2、I3、I4、I5 顺序发生，例如：

| 指令 | Commit |
|---|---:|
| I1 | 7 |
| I2 | 12 |
| I3 | 13 |
| I4 | 14 |
| I5 | 15 |

具体周期会随“写回和发射能否同周期”“CDB 数量”“功能单元流水化程度”等假设变化。考试中最重要的是写清假设，并解释 RAW、WAR、WAW 分别在哪里处理。

## 23. 三类常用 DLP 架构

三类常用 DLP 架构：

1. **Vector Architecture**：一条向量指令作用于多个元素，例如 RV64V。
2. **Multimedia SIMD**：短固定宽度 SIMD 扩展，例如 SSE/AVX/NEON。
3. **GPU**：通过 SIMT、线程块、大量线程和多级内存层次利用大规模数据并行。NVIDIA CUDA 官方文档也把 CUDA 定义为利用 GPU 的并行计算平台和编程模型。

三者都利用 DLP，但抽象层次不同：向量架构偏长向量指令，Multimedia SIMD 偏短向量寄存器，GPU 偏显式线程和吞吐计算。

## 24. 向量架构工作原理

向量架构把循环中多个独立迭代合并为少数向量指令。

基本流程：

1. 向量 load 把一组数据从内存装入向量寄存器。
2. 向量功能单元对各元素流水执行加、乘、比较等操作。
3. 向量 store 把结果写回内存。

例如 DAXPY：

```c
Y = a * X + Y
```

向量版本：

```asm
vld      v0, x5
vld      v1, x6
vmul.vs  v2, v0, f0
vadd.vv  v3, v2, v1
vst      v3, x6
```

优点：

- 一条指令描述多个元素操作，减少取指/译码/分支开销；
- 功能单元可流水化，吞吐高；
- 向量寄存器隐藏内存延迟；
- chaining 允许按元素前递，而不必等整条向量完成。

## 25. Dynamic register typing 的关键思想

Dynamic register typing 把数据类型和元素宽度与向量寄存器配置相关联，而不是为每种类型设计不同指令。

程序先配置向量状态，例如元素宽度、向量长度、寄存器分组等，再执行相同形式的向量指令。RISC-V V spec 中，`vtype` 描述 vector data type，`vl` 描述 vector length，`vsetvli/vsetvl` 负责设置配置。

好处：

- 减少 ISA 编码组合；
- 同一指令可用于不同元素宽度；
- 支持运行时向量长度；
- 便于同一二进制在不同 VLEN 实现上运行。

## 26. Vector execution time、影响因素与 chaining

向量执行时间取决于：

- 向量长度 $n$；
- convoy 数；
- chime 数；
- 功能单元启动开销；
- 功能单元 latency 和 initiation rate；
- 内存 bank 冲突；
- 是否支持 chaining；
- lane 数量；
- load/store 带宽。

课件中简化模型：

$$
\text{cycles}\approx \text{number of chimes}\times \text{vector length}
$$

若有 $m$ 个 convoy，每个 convoy 约 1 chime，向量长度为 $n$：

$$
T\approx m\times n
$$

**Chaining** 允许依赖的向量指令按元素前递。例如 `vmul` 产生第 0 个元素后，`vadd` 可以立即处理第 0 个元素，不必等整个 `vmul` 向量完成。

因此 chaining 减少依赖向量指令之间的等待，把“整向量级等待”变成“元素流水级等待”。

## 27. Stride 对多 bank memory 访问延迟的影响示例

设有 8 个 memory banks，连续地址按 word 交错映射：

$$
\text{bank}=\text{address word index}\bmod 8
$$

若 stride = 1，访问元素：

```text
0,1,2,3,4,5,6,7,...
```

bank 序列：

```text
0,1,2,3,4,5,6,7,...
```

访问均匀分散，bank 并行度高。

若 stride = 2：

```text
0,2,4,6,8,10,...
```

bank 序列：

```text
0,2,4,6,0,2,...
```

只使用 4 个 bank，带宽约下降。

若 stride = 8：

```text
0,8,16,24,...
```

bank 序列全是：

```text
0,0,0,0,...
```

所有访问落到同一 bank，严重 bank conflict，访问几乎串行化。

一般地，有 $B$ 个 bank、stride 为 $s$，可用 bank 数大致与 $B/\gcd(B,s)$ 有关；$\gcd(B,s)$ 越大，冲突越严重。

## 28. Loop-carried dependence 示例与消除

有循环携带依赖的例子：

```c
for (int i = 1; i < n; i++) {
    A[i] = A[i-1] + B[i];
}
```

第 $i$ 次迭代需要第 $i-1$ 次迭代刚写出的 `A[i-1]`，所以不能直接向量化。

可消除的依赖例子：

```c
for (int i = 0; i < n; i++) {
    A[i] = A[i] + s;
}
```

每次迭代只读写自己的 `A[i]`，可改写成向量：

```asm
vld      v0, xA
vadd.vs  v1, v0, fs
vst      v1, xA
```

有些循环看似相关，其实是名称依赖，可通过临时数组或重命名消除：

```c
for (int i = 0; i < n; i++) {
    X[i] = Y[i] + 1;
    Y[i] = Z[i] + 2;
}
```

这里同一迭代中读旧 `Y[i]` 后写新 `Y[i]`，若改成：

```c
for (int i = 0; i < n; i++) {
    X[i] = Y[i] + 1;
    Y_new[i] = Z[i] + 2;
}
```

就更容易向量化，最后再让 `Y = Y_new`。

## 29. 多核通信成本如何影响性能

多核系统中，通信成本包括远程内存访问、cache miss、coherence 消息、同步等待、互连拥塞等。

若远程访问延迟高，即使比例很小，也会显著增加 CPI。课件例子：

- 远程访问延迟 100 ns；
- 频率 4 GHz，所以 100 ns = 400 cycles；
- 基础 CPI 为 0.5；
- 远程访问比例 0.2%。

附加 CPI：

$$
0.002\times400=0.8
$$

实际 CPI：

$$
0.5+0.8=1.3
$$

性能损失很大。因此多核程序要减少通信、提高局部性、减少共享写、避免 false sharing，并让线程粒度足够大。

## 30. Memory system coherent 的条件

一个 memory system 对同一地址 coherent，通常需要满足：

1. **Read own write**：处理器写入 X 后，后续自己读 X，若中间没有其他写，必须读到自己写的值。
2. **Write propagation**：一个处理器写入 X 后，其他处理器最终能看到这个写入。
3. **Write serialization**：所有处理器必须以同一顺序观察到同一地址的所有写入。

Coherence 主要约束单个地址。Consistency 则约束不同地址的读写操作之间的顺序。

## 31. Snooping 与 directory coherence 状态转换示例

设 3 个 core：C0、C1、C2。每个 cache 使用 write-back、write-invalidate MSI 协议。初始 X 不在任何 cache 中。

### Snooping-based MSI

| 步骤 | 请求 | 总线事务 | C0 | C1 | C2 | 说明 |
|---|---|---|---|---|---|---|
| 0 | 初始 | - | I | I | I | memory 有最新 X |
| 1 | C0 read X | BusRd | S | I | I | C0 从内存取 X |
| 2 | C1 read X | BusRd | S | S | I | C0/C1 共享 |
| 3 | C0 write X | BusUpgr | M | I | I | C1 被 invalidated |
| 4 | C2 read X | BusRd | S | I | S | C0 flush/downgrade，C2 获得数据 |
| 5 | C2 write X | BusUpgr | I | I | M | C0 被 invalidated，C2 独占修改 |

核心动作：

- read miss 发 BusRd；
- write miss 或 shared-to-modified 发 BusRdX/BusUpgr；
- 其他 cache snoop 到 invalidate 后置 I；
- M 状态被别人读时需提供/写回最新数据并降级。

### Directory-based MSI

目录位于 home node，记录状态和 sharers/owner。

初始：Directory(X) = Uncached，Sharers = empty。

| 步骤 | 请求 | Directory 状态变化 | 消息 |
|---|---|---|---|
| 1 | C0 read X | Uncached -> Shared, Sharers={C0} | home 向 C0 返回数据 |
| 2 | C1 read X | Shared, Sharers={C0,C1} | home 向 C1 返回数据 |
| 3 | C0 write X | Shared -> Modified, Owner=C0 | home 向 C1 发 invalidate，收到 ack 后授予 C0 |
| 4 | C2 read X | Modified -> Shared, Sharers={C0,C2} | home 向 owner C0 请求数据，C0 写回/转发并降级 |
| 5 | C2 write X | Shared -> Modified, Owner=C2 | home 向 C0 发 invalidate，授权 C2 |

Snooping 依赖广播，适合小规模共享总线；directory 用定向消息和 sharer 位向量，适合更大规模系统。

## 32. 不同 consistency policies 要求的 ordering

Memory consistency 定义不同地址的 load/store 在多核中何时可见。可用四种基本顺序描述：

```text
R -> R, R -> W, W -> R, W -> W
```

常见策略：

| Consistency policy | Ordering 要求 |
|---|---|
| **Strict consistency** | 读总是返回最近真实时间上的写；过强，实际多处理器难实现 |
| **Sequential consistency** | 所有处理器的 memory operations 形成一个全局顺序，并保持每个处理器的 program order；要求 R->R、R->W、W->R、W->W 都按程序顺序 |
| **Total Store Order, TSO** | 写对所有处理器保持同一顺序，但允许本处理器后续 load 绕过 store buffer 中较早 store，即放松 W->R |
| **Processor consistency** | 保持单个处理器写的顺序，但不同处理器写的可见顺序可能更宽松 |
| **Partial Store Order, PSO** | 进一步允许 W->W 重排，不同地址 store 可乱序可见 |
| **Weak consistency** | 普通读写可重排，但同步操作前后的内存访问必须被约束 |
| **Release consistency** | acquire 前的操作不能越过 acquire；release 后的操作不能越过 release；临界区内普通访问可更自由重排 |

Linux kernel memory barrier 文档也强调，barrier 的作用是给读写建立部分顺序，例如 read barrier 约束 load，write barrier 约束 store，full barrier 同时约束读写。

考试可这样记：

- coherence 解决“同一地址读到什么值”；
- consistency 解决“不同地址读写以什么顺序对其他核心可见”；
- 越强的模型越容易编程，但限制硬件优化；
- 越弱的模型性能空间更大，但需要 fence、lock、acquire/release 等同步原语。
