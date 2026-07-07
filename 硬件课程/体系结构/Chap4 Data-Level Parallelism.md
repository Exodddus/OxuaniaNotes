---
counter: true
---

# Chap4 Data-Level Parallelism

***"本章核心"***

- **数据级并行**：从大量数据上的同类操作中挖掘并行性，而不是从多个控制线程中挖掘并行性。
- **向量架构**：用一条向量指令描述许多独立元素运算，是本章最重要的模型。
- **RV64V**：通过向量寄存器、向量功能单元、向量加载/存储单元、动态寄存器类型、`vl` 和谓词寄存器等机制实现向量处理。
- **执行时间**：重点理解 `convoy`、`chime`、链式执行、结构冒险和启动开销。
- **优化手段**：多通道、向量长度寄存器、谓词寄存器、内存分区、步幅访问、聚集-分散。
- **Multimedia SIMD**：适合短而固定长度的向量，硬件成本低，但缺少向量架构的灵活性。
- **GPU**：通过 CUDA/SIMT、线程块、多线程 SIMD 处理器、PTX、谓词和分支同步栈等机制支持大规模 DLP。

## Data-Level Parallelism

**数据级并行**(data-level parallelism, DLP)指的是：在大量数据元素上同时执行相同或相近的操作。例如矩阵计算、图像和音频处理、机器学习中的张量运算，都天然包含大量同构数据操作。

它的特点是：

- 并行性来自数据集合，而不是来自多个独立控制流。
- 一条指令或一段程序结构可以触发许多数据操作。
- 相比经典 MIMD 并行编程，程序员通常仍能按接近串行的方式思考循环和数组。
- 能效较高，因为取指、译码、控制等开销可以被许多数据元素共享。

本章 PPT 讨论的三类 DLP 架构是：

- **Vector Architecture**：向量架构。
- **Multimedia SIMD**：多媒体 SIMD 指令扩展。
- **GPU**：图形处理器上的通用计算。

## Vector Architecture

向量架构的基本流程是：

1. 从内存中取出一组数据元素。
2. 将它们放入较大的顺序向量寄存器文件。
3. 在向量寄存器上执行加、乘、比较等操作。
4. 将结果写回内存。

一条向量指令会作用于一个向量中的多个元素，因此会在许多彼此独立的数据元素上产生多个寄存器到寄存器的运算。向量寄存器既能作为由编译器控制的缓冲区，也能隐藏内存时延、利用内存带宽。

向量架构适合功耗受限的处理器设计：不必像复杂乱序超标量处理器那样投入大量控制逻辑，也能通过规则的数据并行获得高性能。

### RV64V

![[Pasted image 20260619210524.png]]

RV64V 可以理解为 RISC-V 标量架构加上向量扩展。一个典型 RV64V 向量处理器包含：

- **32 个向量寄存器**：PPT 中设定每个向量寄存器宽 64 位并保存一个向量。向量寄存器堆至少需要 16 个读端口和 8 个写端口，并通过交叉开关连接到功能单元。
- **向量功能单元**：每个单元完全流水线化，每个时钟周期可以启动一个新操作。控制单元负责检测结构冒险和数据冒险。
- **向量加载/存储单元**：以流水线方式从内存加载向量或将向量写回内存，也可以处理标量加载和存储。
- **标量寄存器组**：RV64G 中的 31 个通用寄存器和 32 个浮点寄存器为向量功能单元提供标量输入，并计算向量加载/存储地址。

向量处理器和标量浮点流水线的不同点在于：向量指令本身包含许多元素操作，流水线启动后可以连续处理多个元素；标量流水线则每条指令只描述一个标量操作。

### Vector Instructions

RV64V 向量指令一般采用 R 格式，并用后缀区分操作数来源：

- `.vv`：两个操作数都是向量。
- `.vs`：一个向量操作数和一个标量操作数。
- `.sv`：一个标量操作数和一个向量操作数。

对交换律成立的操作，通常只需要 `.vv` 和 `.vs`，因为标量和向量的位置可以等价处理。

PPT 特别追问了“data type and size?”。RV64V 的回答是：不把所有数据类型和数据宽度都编码进每条指令，而是使用动态寄存器类型。

### Dynamic Register Typing

**动态寄存器类型**(dynamic register typing)把数据类型和数据宽度同向量寄存器关联，而不是同指令编码绑定。

执行向量指令之前，程序需要配置正在使用的向量寄存器，说明其中元素的数据类型和宽度：

- 整数：8、16、32、64 位。
- 浮点数：16、32、64 位。

这样做的好处是：

- **减少指令数量**：不用为所有类型和宽度组合都设计独立指令。
- **支持禁用未用寄存器**：如果向量存储空间固定，启用更少的向量寄存器，就能让每个寄存器获得更长的最大向量长度 `mvl`。例如 1024 B 向量内存只启用 4 个向量寄存器，则每个寄存器可分到 256 B。
- **支持隐式转换**：不同大小操作数的转换可以依赖寄存器配置，而不必总是插入显式转换指令。

### How Vector Processor Works?

向量处理器把循环中的多个独立迭代合并为向量操作。关键前提是：循环迭代之间没有会阻止并行执行的依赖，也就是没有不可消除的**循环携带依赖**(loop-carried dependence)。

当循环可以向量化时，标量处理器中反复出现的取指、译码、分支、地址更新等开销会被显著减少；数据相关造成的流水线停顿也通常只在每条向量指令的第一个元素处体现，而不是每个元素都停一次。

### DAXPY Example

DAXPY 表示：

```c
Y = a * X + Y
```

其中 `X` 和 `Y` 是双精度向量，`a` 是标量。PPT 例子中：

- `X` 和 `Y` 各有 32 个元素。
- `x5` 和 `x6` 分别保存 `X` 和 `Y` 的起始地址。
- 标量 RISC-V 代码需要循环处理每个元素。
- RV64V 代码可以用向量加载、向量乘法、向量加法、向量存储一次处理整个向量。

典型 RV64V 思路如下：

```asm
vld      v0, x5        # v0 <- X
vld      v1, x6        # v1 <- Y
vmul.vs  v2, v0, f0    # v2 <- a * X
vadd.vv  v3, v2, v1    # v3 <- a * X + Y
vst      v3, x6        # Y <- result
```

向量版的要点：

- 汇编器根据操作数自动选择 `.vs`、`.vv` 等版本。
- 如果迭代之间没有循环携带依赖，循环可以向量化。
- 每条向量指令只需为第一个元素承担流水线依赖停顿，后续元素会连续流过流水线。
- **Chaining**：依赖的向量操作可以按元素前递。例如 `vmul` 产生第 0 个元素后，`vadd` 不必等整个乘法向量结束，可以开始处理第 0 个元素。
- **Flexible chaining**：更灵活地允许一条向量指令同其他活跃向量指令链接，只要没有结构冒险。

单精度版本本质类似，只是每个元素更窄，同样的向量寄存器空间可以容纳更多元素。

### Vector Execution Time

向量序列的执行时间取决于：

- 操作数向量长度。
- 操作之间是否存在结构冒险。
- 操作之间是否存在数据冒险。

单条向量指令的时间还取决于：

- **向量长度**。
- **初始化速率**(initiation rate)：向量单元消耗新操作数并产生新结果的速率。

#### Convoy

**指令组**(convoy)是一组理论上可以一起执行的向量指令。

要求：

- 同一个 convoy 中不能有结构冒险。
- PPT 允许同一个 convoy 中存在 RAW 依赖，因为链式执行可以把前一个功能单元的元素结果前递给后一个功能单元。
- 不同 convoy 之间需要序列化执行。

#### Chime

**时钟间隔**(chime)是执行一个 convoy 所需的时间单位。

如果一个向量序列包含 `m` 个 convoy，向量长度为 `n`，在一通道且每周期处理一个元素的近似模型下：

$$
\text{cycles} \approx m \times n
$$

这个估算忽略了向量指令发射开销、向量启动时间等处理器相关开销，因此对长向量更准确。

#### Execution-Time Example

PPT 中的例子假设每类向量功能单元只有一个副本，因此：

- 两条 `vld` 不能在同一个 convoy 中，因为有加载单元结构冒险。
- `vld` 和 `vst` 也可能共享加载/存储资源，产生结构冒险。
- 有 RAW 依赖的 `vld -> vmul`、`vld -> vadd` 可以通过 chaining 放在同一个 convoy。

示例划分：

- convoy 1：`vld` 与依赖它的 `vmul`。
- convoy 2：另一条 `vld` 与依赖它的 `vadd`。
- convoy 3：`vst`。

所以共有 3 个 convoy，也就是 3 个 chime。若向量长度为 32：

- 约需 `3 * 32 = 96` 个周期。
- 每个元素包含乘法和加法，共 `32 * 2 = 64` 个 FLOP。
- 周期/FLOP 约为 `96 / 64 = 1.5`。

### How to Optimize Vector Arch?

PPT 中列出的向量架构优化包括：

- **Multiple Lanes**：增加通道数，每周期处理多个元素。
	![[Pasted image 20260619210443.png]]
- **Vector-Length Registers**：告诉处理器这一次向量指令实际处理多少个元素。是为了处理运行时才知道，超过 `mvl` 的向量长度。
- **Predicate Registers**：处理循环中的 IF。先算出哪些分支满足条件，再让满足条件的分支才能写回结果。
	- Predicate的意思是“断言”，不要和predict弄混
	- 把控制流转成了数据流，只要前面的数据准备好，程序就可以往前走
- **Memory Banks**：提供足够内存带宽。
	![[Pasted image 20260619214323.png]]
		192x7 的意思是，如果每个 CPU cycle 都有 192 个新请求进来，而每个 bank 要忙 7 个 cycles，那么需要准备 7 批 banks，一共是1344个。
- **Stride**：支持非连续元素访问。VLDS, VSTS分别是带步长的load和store
- **Gather-Scatter**：支持稀疏数据结构。类似指针的指针
- **Programming Vector Arch**：通过编译器提示和程序改写提升可向量化程度。

### Multiple Lanes

**多通道**(multiple lanes)用多个功能单元或多个流水线并行处理向量元素，从而提高吞吐率。

核心思想：

- 单通道每周期处理 1 个元素。
- `k` 通道理想情况下每周期处理 `k` 个元素。
- 向量寄存器内存会按通道划分，每个通道保存各向量寄存器中的一部分元素。
- RV64V 算术指令只让第 `i` 个元素和另一个向量的第 `i` 个元素相运算，因此通道之间通常不需要交换数据。

多通道能在不改变机器码的情况下提升性能，但要真正发挥作用，需要足够长的向量和足够的内存带宽。

### Vector-Length Registers

现实中，向量长度经常在编译时未知，或者不是最大向量长度 `mvl` 的整数倍。例如：

```c
for (i = 0; i < n; ++i)
    Y[i] += a * X[i];
```

`n` 可能运行时才知道。RV64V 用 **向量长度寄存器** `vl` 控制每条向量指令实际处理的元素数：

- `vl <= mvl`。
- 向量加载、存储、算术操作都受 `vl` 控制。

当 `n > mvl` 时，需要使用**条带挖掘**(strip mining)：

- 每次循环处理 `min(mvl, n_remaining)` 个元素。
- `setvl` 设置本轮 `vl`。
- 指针按本轮处理的元素数前移。

伪代码如下：

```c
while (n > 0) {
    vl = min(mvl, n);
    vector_compute(vl);
    x += vl;
    y += vl;
    n -= vl;
}
```

对双精度 DAXPY，元素大小为 64 位，即 8 字节，因此每轮结束后 `x5`、`x6` 这样的地址寄存器要增加 `8 * vl` 字节。

### How to Handle IF in Loops?

如果循环中有条件语句，直接向量化会遇到控制依赖。例如：

```c
for (i = 0; i < 64; ++i)
    if (X[i] != 0)
        X[i] -= Y[i];
```

目标是：只对满足 `X[i] != 0` 的元素执行减法，而不是让整个循环退回标量执行。

### Predicate Registers

**谓词寄存器**(predicate registers)用于保存向量掩码，支持每个元素级别的条件执行。这也叫**向量掩码控制**(vector-mask control)，编译器里常称为 **IF-conversion**。

规则：

- 谓词寄存器 `p0` 中每一位对应一个向量元素。
- 当 `p0[i] = 1` 时，第 `i` 个元素执行向量操作。
- 当 `p0[i] = 0` 时，第 `i` 个目标元素保持不变。
- 启用谓词寄存器时，通常初始化为全 1，表示默认对所有元素执行操作。

处理 IF 的典型流程：

```asm
vld      v0, x5          # load X
vld      v1, x6          # load Y
vcmp.ne  p0, v0, x0      # p0[i] = (X[i] != 0)
vsub.vv  v0, v0, v1, p0  # only active lanes subtract
vst      v0, x5
```

这样能消除显式分支，把控制依赖转成数据掩码。

### Memory Banks

向量处理器需要很高的内存带宽，尤其是多个加载/存储操作同时发生时。由于 SRAM/DRAM 分区周期通常比处理器周期长，必须把内存划分成多个可独立访问的 **memory banks**。

内存分区的作用：

- 支持单周期内多个加载或存储请求。
- 支持非连续地址访问。
- 支持多个处理器共享同一内存系统。

PPT 例子：Cray T90

- 32 个处理器。
- 每个处理器每周期 4 次加载、2 次存储。
- 处理器周期：2.167 ns。
- SRAM 周期：15 ns。

计算：

- 每处理器每周期内存引用数：`4 + 2 = 6`。
- 全系统每周期内存引用数：`6 * 32 = 192`。
- SRAM 周期约为：`15 / 2.167 ≈ 7` 个处理器周期。
- 最少 memory banks：`192 * 7 = 1344`。

### Stride

向量中逻辑相邻的元素，在内存里不一定物理相邻。矩阵按行主序存储时，访问一列元素就会跨过整行。

**步幅**(stride)是待收集到同一个向量寄存器中的相邻元素在内存中的距离。

- **Unit stride**：步幅为 1，访问相邻元素。
- **Nonunit stride**：步幅大于 1，访问间隔元素。

RV64V 提供：

- `vlds`：load vector with stride，用步幅加载向量。
- `vsts`：store vector with stride，用步幅存储向量。

PPT 例子：

- 8 个 memory banks。
- bank busy time 为 6 个周期。
- 总内存延迟为 12 个周期。
- 加载 64 元素向量。

若 stride = 1：

- bank 数 8 大于 busy time 6，不会因为同一 bank 忙而停顿。
- 总时间约为：`12 + 64 = 76` 个周期。

若 stride = 32：

- `32 mod 8 = 0`，每次访问都落到 bank 0。
- 从第 2 次访问开始反复发生 bank conflict。
- 总时间约为：`12 + 1 + 6 * 63 = 391` 个周期。

判断 bank conflict 的一个直观标准是：同一个 bank 再次被访问的间隔若小于 bank busy time，就会停顿。PPT 中用 LCM 关系表达这一点：

$$
\frac{\mathrm{LCM}(\text{stride}, \text{number of banks})}{\text{stride}} < \text{bank busy time}
$$

例如：

- stride = 1, banks = 8：再次访问同 bank 间隔为 8，不小于 6，无冲突。
- stride = 32, banks = 8：再次访问同 bank 间隔为 1，小于 6，严重冲突。
- stride = 6, banks = 16：访问序列为 0, 6, 12, 2, 8, 14, 4, 10, ...，再次访问同 bank 间隔为 8。

### Gather-Scatter

稀疏矩阵和稀疏数组中，非零元素通常通过索引数组间接访问。例如：

```c
for (i = 0; i < n; ++i)
    A[K[i]] += C[M[i]];
```

其中：

- `A` 和 `C` 是稀疏数组。
- `K` 和 `M` 是索引向量，指定 `A` 和 `C` 中的非零元素。
- `A` 和 `C` 的非零元素数量相同。

**Gather**：

- 根据“基地址 + 索引向量中的偏移”取出分散在内存中的元素。
- 结果放到一个向量寄存器中，形成稠密向量。

**Scatter**：

- 将向量寄存器中的稠密结果，根据索引向量写回内存中分散的位置。

RV64V 指令：

- `vldx`：indexed vector load，也就是 gather。
- `vstx`：indexed vector store，也就是 scatter。

注意：编译器未必能自动判断 `K[i]` 是否唯一。如果索引可能重复，就可能存在不同迭代写同一位置的依赖，因此常常需要程序员或编译器提示来保证向量化安全。

### How to Vectorize Program?

程序能否向量化，主要取决于循环中是否存在真实数据依赖。

程序员可以通过以下方式帮助编译器：

- 改写循环，消除不必要的依赖。
- 使用编译器提示说明别名关系或索引唯一性。
- 选择更适合向量化的数据布局。
- 避免复杂分支，或让分支可被谓词化。

### Programming Vector Arch

向量架构的一个优点是：编译器通常能告诉程序员哪些循环没有被向量化，以及原因是什么。领域专家可以据此修改代码或添加提示，让更多循环以向量模式运行。

但根本因素仍是程序本身：循环是否真的有跨迭代依赖，算法是否适合改写成规则的数据并行形式。

## Multimedia SIMD

**多媒体 SIMD 指令扩展**适合短而固定长度的向量。例如 256-bit 宽 SIMD 寄存器可以一次处理：

- 4 个 64-bit 双精度数。
- 8 个 32-bit 单精度数。
- 更多 16-bit 或 8-bit 媒体数据。

和向量架构相比，多媒体 SIMD 通常省略了：

- 向量长度寄存器。
- 步幅访问。
- gather/scatter 数据传输。
- 传统意义上的谓词/掩码寄存器，尽管现代 SIMD 正在补足这类能力。

这些省略让硬件更简单、处理器状态更少、上下文切换代价更低，但也让编译器自动生成高质量 SIMD 代码更困难。

### SIMD DAXPY Example

PPT 中的 RISC-V SIMD DAXPY 示例使用 256-bit SIMD 指令，`.4D` 表示一次对 4 个双精度操作数执行运算：

```asm
# 每条 SIMD 指令一次处理 4 个 double
ld.4D    v0, 0(x5)
ld.4D    v1, 0(x6)
mul.4D   v2, v0, f0
add.4D   v3, v2, v1
st.4D    v3, 0(x6)
```

实际循环仍需要按 SIMD 宽度分块，并处理尾部不足一个 SIMD 宽度的元素。

### Roofline Visual Performance Model

**屋顶线模型**(Roofline model)用二维图表示：

- 浮点性能。
- 内存性能。
- 算术强度。

**算术强度**(arithmetic intensity)定义为：

$$
\text{Arithmetic Intensity} =
\frac{\text{浮点操作次数}}{\text{访问内存字节数}}
$$

其含义是一段程序每读写 1 字节内存数据，能够完成多少次浮点运算。低算数强度可能是访存密集型程序，只有少量计算，大部分时间开销在读写内存上；高算数强度可能是计算密集型，可能数值较大，读入一小块数据后需要进行大量浮点运算，例如物理仿真、深度学习训练。
可达到的性能为：

$$
\text{Attainable GFLOP/s} =
\min(\text{Peak Memory BW} \times \text{Arithmetic Intensity},\ 
\text{Peak Floating-Point Perf.})
$$
如下图所示。

![[Pasted image 20260620133440.png]]

理解方式：

- 左侧斜线区域受内存带宽限制。
- 上方水平线区域受峰值计算能力限制。
- 斜线和水平线的交点越靠左，越多程序容易达到峰值计算性能。
- Stream benchmark 常用于测量峰值内存带宽。

向量处理器通常有较强内存带宽，因此在 roofline 上更容易把交点推向左侧。

## GPU

GPU 支持多种并行：

- 多线程。
- MIMD。
- SIMD/SIMT。
- 指令级并行 ILP。

GPU 编程的挑战不仅是让 GPU 算得快，还包括：

- 协调 CPU 与 GPU 的任务调度。
- 管理系统内存和 GPU 内存之间的数据传输。
- 组织足够多的线程来隐藏内存时延。

### NVIDIA CUDA

CUDA 是 NVIDIA 的 **Compute Unified Device Architecture**。

CUDA 编程模型包含：

- 用 C/C++ 编写 host 端代码，即系统处理器上运行的代码。
- 用 CUDA C/C++ 方言编写 device 端代码，即 GPU 上运行的代码。
- 用 **CUDA Threads** 表达大规模并行。
- 用 **SIMT**(Single Instruction, Multiple Thread)描述一组线程执行同一指令流的模型。
- 用 **Thread Block** 把线程分组。
- 用 **Multithreaded SIMD Processor** 执行线程块。

### CUDA Program

CUDA 函数和变量常见修饰符：

- `__host__`：host 端函数。
- `__device__`：device 端函数或 GPU 内存变量。
- `__global__`：从 host 调用、在 device 上执行的 kernel。

GPU kernel 调用格式：

```c
name<<<dimGrid, dimBlock>>>(parameter_list);
```

其中：

- `dimGrid`：网格维度，决定线程块数量。
- `dimBlock`：线程块维度，决定每块线程数。
- `blockIdx`：当前线程块编号。
- `threadIdx`：当前线程在线程块内的编号。
- `blockDim`：每个线程块的线程数。

DAXPY 的 CUDA 版本：

```c
__global__
void daxpy(int n, double a, double *x, double *y) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n)
        y[i] += a * x[i];
}

int nblocks = (n + 255) / 256;
daxpy<<<nblocks, 256>>>(n, 2.0, x, y);
```

GPU 硬件负责并行执行和线程管理。CUDA 要求线程块彼此独立、可按任意顺序执行，因此不同线程块不能直接通信，只能通过全局内存等机制间接协调。

### Micro-Architectural Features

GPU 微架构把程序中的线程层次映射到硬件执行资源：

- Grid 对应一组 Thread Blocks。
- Thread Block 包含多个 Threads。
- 多线程 SIMD 处理器执行线程块。
- SIMD lanes 执行同一 SIMD 指令中的各个线程通道。

PPT 示例：

- 向量乘法 `A = B * C`。
- 共 8192 个元素。
- 每个线程处理 32 个元素。
- 每个线程块 16 个线程。
- 共 16 个线程块。

### Grid, Thread Blocks, Threads

三者关系：

- **Grid**：一次 kernel 启动生成的线程块集合。
- **Thread Block**：线程的调度和协作单位，同块线程可以共享局部资源。
- **Thread**：CUDA 暴露给程序员的最小并行单位。

为了可扩展，不同 GPU 上线程块可以被分配给不同数量的多线程 SIMD 处理器；程序不能依赖线程块执行顺序。

### Multithreaded SIMD Processor

多线程 SIMD 处理器类似“带多条 SIMD lanes 的向量处理器”，但 GPU 更强调多线程：

- 每个处理器有若干 SIMD lanes。PPT 中简化示例为 16 lanes。
- Pascal P100 GPU 示例：56 个多线程 SIMD 处理器，每个 16 lanes。
- 大量线程用于隐藏 DRAM 访问延迟。
- 硬件动态管理线程状态和寄存器资源。

### SIMD Thread Scheduler

SIMD 线程调度器负责：

- 判断哪些 SIMD 指令线程已经准备好运行。
- 将准备好的线程发送到 dispatch unit。
- 在多线程 SIMD 处理器上调度执行。

GPU 有两级调度：

- **Thread Block Scheduler**：把线程块分配给多线程 SIMD 处理器。
- **SIMD Thread Scheduler**：在某个 SIMD 处理器内部决定哪个 SIMD 指令线程运行。

调度器通常配合 scoreboard 追踪指令依赖、缓存/TLB 未命中等造成的可变时延。

### NVIDIA GPU ISA

NVIDIA 编译器面向的抽象 ISA 是 **PTX**(Parallel Thread Execution)。

PTX 的作用：

- 为编译器提供稳定目标。
- 在不同 GPU 代际之间提供兼容性。
- 在加载或运行时被翻译成更接近具体 GPU 的内部形式。

PTX 指令格式：

```asm
opcode.type d, a, b, c;
```

其中：

- `d`：目的操作数。
- `a, b, c`：源操作数。
- 源操作数可以是 32/64 位寄存器或常量。
- 非存储指令的目的操作数通常是寄存器；存储指令会使用内存地址。

PTX 的特点：

- 使用虚拟寄存器，编译器再映射到真实硬件寄存器。
- 所有指令都可通过 1 位谓词寄存器进行条件执行。
- 控制流指令包括 `call`、`return`、`exit`、`branch`、`bar.sync` 等。

### PTX Instructions

PTX 中 DAXPY 的关键动作是：

1. 每个线程处理一个元素。
2. 通过 `blockIdx`、`blockDim`、`threadIdx` 计算线程对应的全局元素编号。
3. 双精度元素大小为 64-bit，也就是 8 字节。
4. 用元素编号乘 8 得到地址偏移。
5. 加载 `x[i]`、`y[i]`，计算 `a*x[i]+y[i]`，再写回。

伪 PTX 流程：

```asm
# i = blockIdx.x * blockDim.x + threadIdx.x
mul.lo.u32   r1, blockIdx.x, blockDim.x
add.u32      r1, r1, threadIdx.x

# byte_offset = i * 8
mul.wide.u32 rd1, r1, 8

# load, multiply-add, store
ld.global.f64 fd1, [x + rd1]
ld.global.f64 fd2, [y + rd1]
mad.rn.f64    fd3, fd1, a, fd2
st.global.f64 [y + rd1], fd3
```

GPU 没有传统向量架构里的专门顺序、步幅、gather/scatter 指令；从抽象上看，每个线程都有自己的地址。为了让连续地址访问高效，GPU 用 **address coalescing** 把相邻线程访问的相邻地址合并成较少的内存块请求。

### Conditional Branch

GPU 处理条件分支比向量架构更依赖硬件机制。PPT 列出的 GPU branch hardware 包括：

- 谓词寄存器。
- 内部掩码。
- 分支同步栈。
- 指令标记器。

目标是管理 **branch divergence** 和 **branch convergence**：

- divergence：同一个 SIMD 指令线程中的不同 lane 走向不同分支。
- convergence：不同分支路径执行后重新汇合。

### Predicate Register

GPU 谓词寄存器：

- 在 PTX 汇编层由程序员或编译器指定。
- 支持每个 thread lane 独立断言一条指令是否执行。
- 通常是 1-bit predicate register。
- `setp` 指令用于设置谓词。

IF-THEN-ELSE 的执行特点：

- THEN 部分的 SIMD 指令会广播到所有 lanes。
- 谓词为 1 的 lanes 执行操作或写结果。
- 谓词为 0 的 lanes 不执行。
- ELSE 部分通常使用互补谓词。

性能影响：

- 若 THEN 和 ELSE 路径长度相等，效率最多约 50%。
- 两层等长嵌套 IF 可能降到约 25%。

### Branch Sync Stack

分支同步栈用于管理更复杂的控制流。

每个 SIMD thread 在硬件指令层有自己的栈。栈项包含：

- identifier token。
- target instruction address。
- target thread-active mask。

硬件通过特殊指令：

- 压入分支路径和活跃 mask。
- 在路径结束时弹出栈项。
- 或回退到指定栈项并跳转到目标地址。

简单外层 IF-THEN-ELSE 可以优化为纯谓词指令；复杂控制流则需要谓词和分支同步栈混合处理。

### GPU Memory

NVIDIA GPU 内存结构中常见层次：

- **Private memory**：每个线程或 lane 私有，其他线程和 host 不可直接访问。常用于栈帧、溢出寄存器和私有变量。
- **Local/shared memory**：PPT 强调它由同一 thread block 的线程共享，对其他线程块和 host 私有。它通常位于芯片上，低延迟、高带宽，适合同块线程复用数据。
- **GPU memory/global memory**：芯片外内存，由整个 GPU 和所有线程块共享，host 可以读写。

GPU 不主要依赖大缓存来隐藏内存时延，而是依赖大量并行 SIMD 指令线程。当一些线程等待内存时，调度器切换到其他 ready 线程。

### Pascal GPU

PPT 中 Pascal GPU 的例子强调：

- 每周期双 SIMD thread scheduler 可以调度两条指令。
- Pascal P100 级别 GPU 有许多多线程 SIMD 处理器。
- 通过大量 lanes、线程和调度器并行来提高吞吐并隐藏时延。

### Compare GPU with Vector

GPU 与向量架构的对应关系：

- 向量架构的向量指令，类似 GPU 中一组 SIMD 指令线程的同步执行。
- 向量寄存器中的元素，类似 GPU thread lanes 中的线程数据。
- 向量长度 `vl`，类似 GPU 中活跃线程数或线程 mask。
- 向量处理器更显式地暴露向量长度、步幅、gather/scatter。
- GPU 更显式地暴露线程、线程块、网格和内存层次。

重要差异：

- 向量架构通常让编译器生成向量指令，程序员看到的是较高层的数组/循环。
- GPU 程序员通常需要组织线程块、管理 GPU 内存、考虑 coalescing 和同步。
- GPU 用多线程隐藏时延；向量处理器更强调长向量流水线和内存带宽。

### Compare GPU, Multimedia SIMD with Vector

三者可以放在同一条谱系上理解：

- **Vector Architecture**：最完整的向量模型，有 `vl`、stride、gather/scatter、predicate，适合长向量和科学计算。
- **Multimedia SIMD**：短、固定长度、硬件状态少，适合媒体和通用 CPU 中的轻量数据并行，但灵活性较弱。
- **GPU**：通过 SIMT 和大量线程提供极高吞吐，适合大规模规则并行，但需要显式管理线程组织和内存访问。

### Summary

本章总结：

- DLP 的核心是让一条指令或一个程序结构同时作用于大量数据。
- 向量架构用向量寄存器和向量功能单元表达规则数据并行。
- RV64V 通过动态寄存器类型减少指令数量，通过 `vl`、谓词、stride、gather/scatter 扩展适用范围。
- 执行时间分析重点看 convoy 和 chime；结构冒险决定 convoy 划分，链式执行允许 RAW 依赖在同一 convoy 中被流水化。
- 多媒体 SIMD 是较短、固定宽度的向量扩展。
- GPU 用 CUDA/SIMT、线程块、多线程 SIMD 处理器和 PTX 支撑大规模数据级并行。

### Review

复习时按这三条主线回看：

- **Vector Architecture**：RV64V 结构、动态寄存器类型、DAXPY、convoy/chime、多通道、`vl`、predicate、memory banks、stride、gather-scatter。
- **Multimedia SIMD**：短固定向量、缺少 `vl`/stride/gather-scatter/mask 的传统限制、roofline 模型。
- **GPU**：CUDA 编程模型、grid/block/thread、SIMT、多线程 SIMD 处理器、PTX、谓词、分支同步栈、GPU memory。

