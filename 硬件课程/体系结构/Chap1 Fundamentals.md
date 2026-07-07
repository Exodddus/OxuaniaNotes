---
counter: true
---

# Chap 1: Fundamentals of Computer Architecture

***"本章核心"***

- **计算机体系结构是什么**：它不是单纯的 ISA，也不是单纯的硬件实现，而是面向程序员可见抽象、组织结构和实现约束的整体设计。
- **设计任务**：在功能、性能、成本、功耗、可用性和实现复杂度之间做取舍，并用定量方法评估瓶颈。
- **计算机分类**：从市场看有 PMD、Desktop、Server、Cluster/WSC、Embedded；从并行模型看有 SISD、SIMD、MIMD 等。
- **应用驱动架构**：通用处理器也会围绕典型应用集合优化，架构设计必须理解 workload。
- **技术趋势**：摩尔定律、晶体管密度、DRAM/Flash/磁盘/网络带宽增长共同影响体系结构设计。
- **三堵墙**：ILP wall、memory wall、power wall 迫使体系结构从单核 ILP 转向 TLP、DLP、RLP 和专用化。
- **性能思维**：区分 latency/response time 与 bandwidth/throughput；真实系统常常是带宽增长快于延迟改善。

## What Is Computer Architecture?

Amdahl、Blaauw 和 Brooks 在 1964 年给出的经典定义强调：计算机体系结构是程序员所看到的系统属性，也就是概念结构和功能行为；它区别于数据流、控制逻辑组织和物理实现。

换句话说，体系结构关注“软件认为这台机器是什么样的”，而不仅是“芯片内部如何接线”。

从程序员视角看，系统层次大致为：

```text
Applications
Operating System / Compiler / Firmware
Instruction Set Architecture
Instruction Set Processor / I/O System
Datapath & Control
Digital Design
Circuit Design
Layout
```

其中 **ISA** 是软件与硬件之间最核心的接口。汇编程序员能直接看到的是用于编程的硬件抽象，而不是全部硬件组织或物理实现细节。

### Three Aspects of Architecture

课件把计算机体系结构至少分成三类内容：

| 层次 | 含义 | 例子 |
|---|---|---|
| **Instruction Set Architecture** | 程序员可见的机器接口 | 指令、寄存器、寻址、异常、内存模型 |
| **Microarchitecture / Organization** | ISA 的具体组织实现 | 流水线、cache、分支预测、总线、CPU 内部组织 |
| **System Design / Hardware Implementation** | 更低层硬件系统与实现 | 逻辑实现、电路实现、物理实现、I/O 与系统组件 |

所以，选择题中“Computer architecture intended to cover all three aspects: ISA, organization and hardware”是更完整的说法。

### Architecture as Science and Art

课件还给出一个更工程化的定义：

> Computer Architecture is the science and art of selecting and interconnecting hardware components to create computers that meet functional, performance, cost and power goals.

这句话的重点在 **tradeoff**。体系结构设计不是单指标优化，而是在功能、性能、成本、功耗、可靠性、可用性和上市时间之间做系统性取舍。

## ISA

不同机器可以有非常不同的 ISA，例如 PDP-11、IBM 360、VAX、x86、PowerPC、MIPS、ARM、RISC-V 等。ISA 决定程序如何表达计算，也决定编译器、操作系统和微体系结构之间的接口。

### Seven Dimensions of ISA

课件列出的 ISA 七个维度为：

| 维度 | 需要回答的问题 |
|---|---|
| Class of ISA | 是寄存器-寄存器、寄存器-内存、栈式、累加器式，还是其他模型 |
| Memory addressing | 内存如何编号、访存粒度如何、地址空间如何组织 |
| Addressing modes | 操作数地址如何计算，例如立即数、寄存器、偏移寻址等 |
| Types and sizes of operands | 支持哪些数据类型和宽度 |
| Operations | 支持哪些算术、逻辑、访存、控制和系统操作 |
| Control flow instructions | 分支、跳转、调用、返回和异常控制如何表达 |
| Encoding an ISA | 指令如何编码，定长还是变长，字段如何安排 |

ISA 设计会影响编译器复杂度、硬件实现难度、代码密度、性能潜力和向后兼容性。

## Task of Computer Design

计算机设计的任务是：确定新机器的重要属性，在成本、功耗、可用性等约束下最大化性能。

课件把设计任务拆成几个层面：

- **Instruction set architecture design**：设计软件可见接口。
- **Functional organization**：高层组织结构，例如 memory system、bus architecture、internal CPU design。
- **Logic design**：把组织结构落实为硬件逻辑。
- **Implementation**：电路、工艺、封装、物理设计等。

### Design Engineering Life Cycle

课件中的工程流程可以理解为：

1. 分析用户需求和目标市场。
2. 根据应用构造 workload。
3. 用 benchmark 评估现有系统，找到 bottleneck。
4. 按 price-performance、power、availability 等指标评估方案。
5. 用定量原则指导取舍。
6. 结合技术趋势和上市时间预测最终实现。

设计者必须考虑：

- 功能需求和非功能需求；
- 实现复杂度，复杂设计需要更长开发周期，也必须提供更高收益才值得；
- 技术趋势，不只看今天能买到什么，还要预测系统交付时可用的工艺和器件；
- IC 功耗趋势、成本趋势和可用性目标；
- 对现有系统做瓶颈评估，而不是凭感觉优化。

## Applications and Workloads

体系结构设计必须理解应用行为。所谓“通用处理器”并不意味着对所有程序同等优化；现实中通常会围绕一组典型 workload 做设计。

课件列出的应用类型包括：

- **Scientific**：天气预报、碰撞分析、地震分析、医学成像、地球成像等。
- **Business**：数据库、数据挖掘、视频处理等。
- **General purpose**：文字处理、表格、日常办公应用等。
- **Real-time**：自动控制系统等。
- **Others**：游戏、移动应用、IoT、5G 等。

架构可以针对应用特征优化，例如高吞吐、低延迟、低功耗、高可靠性、强实时性或高内存带宽。

## Classes of Computers by Market

市场分类决定设计目标。不同市场的价格范围、功耗限制、性能指标和可靠性要求差异很大。

### Five Computing Markets

| 市场 | 系统价格 | 处理器价格 | 关键设计问题 |
|---|---:|---:|---|
| **PMD** | \$100-\$1000 | \$10-\$100 | 成本、能效、媒体性能、响应性 |
| **Desktop** | \$300-\$2500 | \$50-\$500 / processor | 性价比、能耗、图形性能 |
| **Server** | \$5000-\$10,000,000 | \$200-\$2000 / processor | 吞吐量、可用性、可扩展性、能耗 |
| **Clusters / WSC** | \$100,000-\$200,000,000 | \$50-\$250 / processor | 性价比、吞吐量、energy proportionality |
| **Embedded** | \$10-\$100,000 | \$0.01-\$100 / processor | 价格、能耗、应用特定性能 |

### Personal Mobile Devices

PMD 是带有无线连接和多媒体用户界面的个人移动设备。

特点：

- 典型应用是 web-based、media-oriented；
- 交互强调 responsiveness 和 predictability；
- 电池供电、无风扇，因此成本和能效非常关键；
- 需要实时性能；
- 尽量减少内存容量和能耗，常使用 flash memory。

### Desktop Computing

Desktop 曾经是按金额计最大的市场，笔记本占 desktop 类市场的重要部分。

关键目标：

- 优化 price-performance；
- 高性能、低成本微处理器往往先出现在 desktop 系统中；
- 新挑战来自 web-centric、interactive applications；
- 性能评价不能只看峰值算力，还要看用户响应和实际 workload。

### Servers

Server 提供大规模、更可靠的文件和计算服务。

关键目标：

- **Availability** 非常关键，停机成本可能极高；
- **Scalability** 是核心能力，memory、storage 和 I/O bandwidth 都要能扩展；
- 强调 efficient throughput。

课件用 downtime cost 说明 availability 的经济意义。例如金融交易、信用卡授权、航空订票等系统，即使 0.1% 的不可用时间也可能造成巨大损失。

### Clusters and Warehouse-Scale Computers

Cluster/WSC 常用于 SaaS，例如搜索、社交网络、视频分享、多人游戏、在线购物等。

特点：

- 上万台服务器作为一个整体工作；
- 关注 price-performance、power、availability 和 scalability；
- 数据中心功耗非常敏感，课件指出节省 10% power 可能带来数百万美元级别收益；
- 与传统 server 相比，WSC 更强调整体系统作为一台“计算机”来设计。

### Embedded Computers

嵌入式系统是增长最快、处理能力和成本跨度最大的市场之一。

关键目标：

- 软实时或硬实时性能；
- 严格资源约束；
- 有限内存、低功耗、低成本；
- 常把通用 processor core 与 application-specific circuitry 结合，例如 DSP、移动计算、手机、数字电视等。

## Other Classifications

除了市场分类，课件还列出若干体系结构分类角度：

- Quantum computer vs Chemical computer；
- Scalar processor vs Vector processor；
- NUMA vs UMA；
- Register machine vs Stack machine vs Accumulator machine；
- Harvard architecture vs Von Neumann architecture vs Non-Von Neumann architecture；
- RISC vs CISC；
- Cellular architecture。

这些分类不是彼此互斥的。例如一台现代多核服务器通常既是 MIMD，又可能包含 SIMD/vector 扩展，还可能采用 NUMA 内存组织。

## Parallelism and Parallel Architecture

应用中的并行性主要有两类：

- **Data-Level Parallelism**（DLP）：对大量数据元素执行相同或相近操作。
- **Task-Level Parallelism**（TLP）：不同任务或线程可以并行执行。

硬件利用并行性的主要方式有：

- **Instruction-Level Parallelism**（ILP）：流水线、乱序执行、分支预测、超标量等。
- **Vector Architecture and GPUs**：利用 DLP。
- **Thread-Level Parallelism**：多核、多处理器、多线程。
- **Request-Level Parallelism**：服务器和 WSC 中的大量独立请求。

## Flynn's Taxonomy

Flynn 分类根据 instruction stream 和 data stream 的数量划分计算机。

| 类型 | Instruction streams | Data streams | 含义与例子 |
|---|---:|---:|---|
| **SISD** | Single | Single | 串行非并行机器；传统单 CPU 计算机 |
| **SIMD** | Single | Multiple | 所有处理单元同一时刻执行同一条指令，但处理不同数据；适合图像处理等规则问题 |
| **MISD** | Multiple | Single | 多条指令作用于同一数据流；现实中少见 |
| **MIMD** | Multiple | Multiple | 多个处理器执行不同指令流、处理不同数据流；现代多核、SMP、集群和多数超算属于此类 |

### SISD

SISD 是串行计算模型：

- 任一时钟周期只有一个指令流被 CPU 执行；
- 任一时钟周期只有一个数据流作为输入；
- 执行通常是 deterministic；
- 传统 PC、单 CPU workstation 和 mainframe 都可以看作 SISD。

### SIMD

SIMD 是单指令多数据：

- 所有处理单元在同一时刻执行同一条指令；
- 每个处理单元处理不同数据元素；
- 通常需要 instruction dispatcher、高带宽内部网络和大量小处理单元；
- 同步 lockstep 执行，通常 deterministic；
- 适合高度规则的问题，例如图像处理。

SIMD 的两类典型形式：

- **Processor arrays**：例如 Connection Machine CM-2、MasPar。
- **Vector pipelines**：例如 Cray C90、NEC SX 等。

### MIMD

MIMD 是当前最常见的并行计算模型：

- 每个处理器可以执行不同指令流；
- 每个处理器可以处理不同数据流；
- 执行可以同步或异步，也可以 deterministic 或 non-deterministic；
- 现代超算、网络并行计算、多处理器 SMP 和许多多核机器都属于 MIMD。

## Technology Trends

体系结构长期受底层技术推动。课件强调：技术不是背景知识，而是 CPU 性能提升、成本下降和架构转向的关键原因。

### Moore's Law

Gordon Moore 在 1965 年预测芯片上可集成元件数量每年翻倍；1975 年更新为大约每两年翻倍。摩尔定律长期成为半导体工业提高性能、降低成本的指导原则。

需要注意：

- 摩尔定律描述的是集成密度趋势，不等价于单线程性能必然同速增长；
- 密度提升会降低单位晶体管成本，但也带来功耗、散热、线延迟和设计复杂度问题；
- 体系结构设计常常要面向“下一代技术”做决策。

### Integrated Circuit Logic

课件给出的经验数据：

| 指标 | 趋势 |
|---|---|
| Transistor density | 每年增加约 35%，约 4 年增长 4 倍 |
| Die size | 每年增加约 10%-20% |
| Transistor count per chip | 每年增加约 40%-55%，约 10-24 个月翻倍 |

特征尺寸缩小会提高晶体管性能，课件给出的经验是 transistor performance roughly improves linearly with decreasing feature size。

但芯片密度提升同时也是挑战：线延迟近似与线的电阻和电容乘积相关，wire delay 成为重要设计限制。

### Memory and Storage Trends

| 技术 | 课件中的趋势 |
|---|---|
| Semiconductor DRAM capacity | 每年约 25%-40%，约 2-3 年翻倍 |
| DRAM speed | 每年约 10% |
| Flash capacity | 每年约 50%-60%，约 2 年翻倍 |
| Flash cost | 通常比 DRAM 便宜 15-20 倍 |
| Magnetic disk density | 不同时期增长率不同，1996-2004 年约每年 100%，之后约每年 30% |
| Network bandwidth | 从 10Mb 到 100Mb 再到 1Gb，升级周期约 10 年、5 年量级 |

一条重要经验：

$$
\text{Cost decrease rate} \approx \text{density increase rate}
$$

技术进步往往是连续发生的，但对体系结构的影响可能表现为离散跃迁。例如容量达到某个阈值后，新的 cache 层次、存储模型或数据中心设计才变得可行。

### Bandwidth over Latency

课件强调要区分：

- **Bandwidth / throughput**：单位时间完成的总工作量。
- **Latency / response time**：事件开始到完成之间的时间。

经验上，很多技术的 bandwidth 提升远快于 latency 改善：

- Bandwidth 可达到 $10,000\times$ 到 $25,000\times$ 级别增长；
- Latency 可能只有 $30\times$ 到 $80\times$ 级别改善；
- 课件给出的经验关系是：bandwidth growth rate 大约接近 latency improvement 的平方。

这解释了为什么现代系统越来越强调并行传输、批处理、prefetch、cache blocking 和吞吐优化：单次访问变快有限，但一次做更多事的能力提升很大。

## CPU Performance Improvement

课件用 VAX、RISC 和 x86 的历史性能趋势说明体系结构的重要性：

| 时期 | 年性能提升 | 主要原因 |
|---|---:|---|
| 1978-1986 VAX | 约 25%/year | 技术改进较稳定，例如 feature size、clock speed |
| 1986-2002 RISC + x86 | 约 52%/year | RISC 兴起后，体系结构创新与技术进步共同发挥作用 |
| 2002 之后 RISC + x86 | 约 22%/year | 单处理器性能提升放缓，功耗和 ILP 收益受限 |

1986-2002 年的高速提升来自体系结构创新与工艺进步的叠加，例如：

- pipeline；
- dynamic scheduling；
- out-of-order execution；
- branch prediction；
- speculation；
- superscalar；
- VLIW；
- predication 等。

## Four Decades of Microprocessors

课件按年代总结微处理器发展：

| 年代 | 关键词 | 代表特征 |
|---|---|---|
| 1970s | Microprocessors | programmable controller、single-chip microprocessor、PC |
| 1980s | Quantitative Architecture | instruction pipelining、fast cache memories、compiler considerations、workstations |
| 1990s | Instruction-Level Parallelism | superscalar、speculative microarchitectures、aggressive code scheduling |
| 2000s | Thread/Data-Level Parallelism | multicore、TLP、DLP |

这条线索说明：体系结构的主战场从“把单条指令流跑得更快”，逐渐转向“显式利用数据、线程和请求中的并行性”。

## Three Walls

课件把现代体系结构面临的核心挑战概括为三堵墙。

### ILP Wall

通过硬件自动发现更多 ILP 的收益递减。复杂乱序执行、超标量和推测执行可以提高单线程性能，但硬件窗口、依赖、分支不确定性和功耗都会限制继续扩大。

结论：必须利用更显式的并行性，例如 TLP 和 DLP。

### Memory Wall

CPU 与片外内存速度差距持续扩大。内存延迟可能成为系统性能瓶颈，即使处理器本身很快，也会被等待数据拖住。

常见应对方向：

- cache hierarchy；
- prefetching；
- memory-level parallelism；
- 3D stacked memory；
- near-memory / in-memory processing；
- 更高带宽互连。

### Power Wall

提高频率会带来功耗快速上升。课件强调：运行频率每翻倍，功耗也可能显著增加。2003 年后单处理器性能提升放缓，很大程度上与 power wall 有关。

因此现代设计更加重视：

- power-aware design；
- energy efficiency；
- multicore/manycore；
- accelerator；
- domain-specific architecture。

## Architecture Trends

课件给出的趋势可以概括为：

```text
ILP -> TLP -> DLP -> RLP
Performance-only -> Power-aware design
2D memory -> 3D stacked memory
General-purpose cores -> accelerators / DSA / PIM
```

### New Performance Models

单处理器性能提升在 2003 年后明显放缓，体系结构转向新的性能来源：

- **Data-Level Parallelism**：向量架构、SIMD、GPU。
- **Thread-Level Parallelism**：多核、多线程、多处理器。
- **Request-Level Parallelism**：数据中心和互联网服务的大量独立请求。

这些模型通常要求应用显式重构。课件特别提醒：程序员必须理解硬件中的并行性，并在算法与编程中主动暴露并行性。

### PIM and Memory as Accelerator

**Processor in Memory**（PIM）指把处理器与 RAM 集成到同一芯片或近距离系统中。目标是减少数据搬移，把部分计算推向内存附近。

课件中还提到 memory as an accelerator：

- 在内存附近放置 mini-CPU、GPU core、video core、imaging core 等；
- 支持 in-DRAM copy、zero、and、or、not 等 bitwise operation；
- 用 specialized compute capability in memory 减少数据移动成本。

这与 memory wall 直接相关：当搬数据比算数据更贵时，把计算靠近数据会变得有吸引力。

### Recent Research Fields

课件列出的近期方向包括：

- power-aware computer architecture；
- reconfigurable computer architecture；
- multicore、manycore；
- HPS；
- PIM；
- AI processor，例如 GPU 和 FPGA；
- DSA（Domain Specific Architecture）；
- hardware/software codesign。

## Cost, Power, Availability

### Cost

成本受技术密度、产量、系统复杂度和市场定位共同影响。

课件中解释计算机 60 年变化的原因时提到：

- CMOS VLSI 在成本和性能上取代 TTL、ECL 等旧技术；
- RISC、superscalar、OOO、speculation、VLIW、RAID 等体系结构进展提升低端系统能力；
- 更小系统、更少组件降低开发和制造复杂度；
- 同样开发成本摊到 10,000 台和 10,000,000 台上，单位成本差异巨大；
- 不同计算机类别服务和利润率不同，价格结构也不同。

### Power

功耗从约束条件变成核心设计目标。对 PMD 来说，功耗决定电池寿命和是否需要风扇；对 WSC 来说，功耗直接影响运营成本；对高性能处理器来说，功耗限制频率、并行度和散热。

所以现代体系结构不能只问“能不能更快”，还要问：

- 每焦耳完成多少工作；
- idle 时能否降低功耗；
- 性能是否随功耗线性或近似线性增长；
- 是否需要用专用硬件换取能效。

### Availability

服务器和数据中心设计特别重视 availability。可用性不只是技术指标，也是经济指标。

常用表达是：

$$
\text{Availability}=\frac{\text{MTTF}}{\text{MTTF}+\text{MTTR}}
$$

其中：

- **MTTF**（mean time to failure）：平均无故障时间；
- **MTTR**（mean time to repair）：平均修复时间。

提高 availability 的思路包括提高 MTTF、降低 MTTR、使用冗余、快速故障恢复、热插拔、容错软件和可扩展监控。

## Performance Concepts

虽然这两份课件主要展开到技术趋势，但第一章主题列表中明确包含 performance measuring/reporting/summarizing 和 quantitative principles。与后续章节衔接时，需要牢记以下基本概念。

### Response Time and Throughput

| 指标 | 含义 | 常见关注场景 |
|---|---|---|
| **Response time / latency** | 单个任务从开始到完成的时间 | 交互、实时、用户体验 |
| **Throughput / bandwidth** | 单位时间完成的任务总量 | server、WSC、批处理、并行系统 |

改进一个指标不一定改进另一个指标。例如增加更多服务器通常提高吞吐量，但单个请求的延迟可能受排队、网络或存储影响。

### Execution Time and Performance

性能通常与执行时间成反比：

$$
\text{Performance}_X=\frac{1}{\text{Execution Time}_X}
$$

若机器 $X$ 比机器 $Y$ 快 $n$ 倍：

$$
\frac{\text{Performance}_X}{\text{Performance}_Y}
=
\frac{\text{Execution Time}_Y}{\text{Execution Time}_X}
=n
$$

### CPU Time Equation

经典 CPU 性能公式为：

$$
\text{CPU Time}
=
\text{Instruction Count}
\times
\text{CPI}
\times
\text{Clock Cycle Time}
$$

也可写成：

$$
\text{CPU Time}
=
\frac{\text{Instruction Count}\times\text{CPI}}{\text{Clock Rate}}
$$

三个因素分别受不同层次影响：

- **Instruction Count**：算法、编译器、ISA。
- **CPI**：微体系结构、cache、流水线、分支预测、并行性。
- **Clock Cycle Time / Clock Rate**：电路、工艺、流水线深度、功耗和物理设计。

### Benchmark and Workload

评价体系结构不能只看单个峰值指标。更可靠的做法是：

- 选取代表目标市场的 workload；
- 使用 benchmark 评估现有系统；
- 找到瓶颈；
- 比较 price-performance、energy、availability 和 scalability；
- 避免只优化一个容易展示但不代表真实应用的指标。

## Quantitative Principles

### Make the Common Case Fast

定量设计的基本原则是：优化最常见、最耗时或最有收益的情况。

原因是总执行时间由各部分贡献加权决定。如果某个部分很少发生，即使把它优化到极致，对总性能贡献也有限。

### Amdahl's Law

Amdahl 定律描述局部优化对整体加速比的限制。

若可优化部分占原执行时间比例为 $f$，该部分加速 $S$ 倍，则整体加速比为：

$$
\text{Speedup}
=
\frac{1}{(1-f)+\frac{f}{S}}
$$

如果有比例 $f$ 的部分完全无法并行，使用 $N$ 个处理器时：

$$
\text{Speedup}
=
\frac{1}{f+\frac{1-f}{N}}
$$

这解释了为什么并行系统必须压低串行部分。即使处理器数量很多，串行部分也会成为上限。

### Locality

存储层次结构依赖程序局部性：

- **Temporal locality**：最近访问过的数据或指令可能很快再次访问。
- **Spatial locality**：访问某个地址后，附近地址也可能被访问。

cache、prefetch、blocking 和 memory hierarchy 都是在利用局部性缓解 memory wall。

### Parallelism

并行性是贯穿后续章节的核心：

- Chapter 3 讨论 ILP：流水线、静态调度、动态调度、推测执行。
- Chapter 4 讨论 DLP：向量架构、SIMD、GPU。
- Chapter 5 讨论 TLP：多处理器、cache coherence、目录协议。

第一章的作用是把这些内容放到更大的背景下：单靠工艺和单线程 ILP 已不足以继续支撑性能增长，体系结构必须系统利用多层次并行性。

## Exam Notes

- 从体系结构视角看，汇编程序员可见的机器属性是 **hardware abstraction used when programming**。
- “计算机体系结构只等于 ISA”是不完整的；更完整的说法包括 **ISA、organization/microarchitecture、hardware implementation**。
- PMD 重点是成本、能效、媒体性能和响应；server 重点是 availability、scalability 和 throughput；embedded 重点是实时性、资源约束和应用特定性能。
- SIMD 适合规则的数据并行；MIMD 是现代多核、SMP、集群和超算的主流模型。
- Bandwidth 和 latency 不是同一个指标；很多技术中 bandwidth 提升远快于 latency 改善。
- 三堵墙分别是 ILP wall、memory wall、power wall；它们共同推动 TLP、DLP、RLP、PIM、accelerator 和 DSA。
- 体系结构设计的基本姿势是：用 workload 和 benchmark 找瓶颈，用定量原则做取舍，而不是只看峰值频率或峰值 FLOPS。
