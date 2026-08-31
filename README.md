# REEVDF：Rate-aware Earliest Eligible Virtual Deadline First 调度算法

## 摘要

现代多核操作系统的通用调度器在公平性与延迟敏感任务响应之间长期存在张力。Linux 内核从 6.6 版本起采用 EEVDF（Earliest Eligible Virtual Deadline First）替代 CFS，通过合格性检查与虚拟截止时间双重机制改善了延迟边界[1][2]。然而，EEVDF 及其前身均以墙钟时间片为调度粒度，无法感知线程在单位时间内的实际指令推进速率差异——计算密集型线程与频繁陷入内核或等待内存的线程在相同时长内完成的有效工作量可能相差数倍。本文提出 REEVDF（Rate-aware EEVDF）调度算法，在 EEVDF 基础上引入基于指令指针（RIP）进度的速率感知反馈机制。REEVDF 通过在线采样每个线程的指令推进速率，与核内平均速率比较，经指数加权移动平均（EWMA）平滑后生成速率倍率；同时设计乘法通道与有符号修正通道并存的双通道动态量子调整架构，分别承担稳态塑形与快速双向响应。为保证系统稳定性，REEVDF 引入老化衰减机制，使长期未采样的线程参数向中性值收敛。在多核扩展方面，REEVDF 实现了带节流控制的批量工作窃取、权重差阈值驱动的主动推送、SIMD 兼容性感知的线程迁移以及饥饿阈值保护。本文基于 SkylineSystem 研究型内核的完整 C++ 实现[7][8]，给出 REEVDF 的算法形式化描述、数据结构设计、理论复杂度分析与实现细节。REEVDF 的核心贡献在于将调度粒度从墙钟时间推进到实际指令进度，为调度器适应异构计算行为提供了一种轻量级、无硬件依赖的在线反馈路径。

关键词：操作系统调度；EEVDF；速率感知；动态时间量子；多核负载均衡；指令进度

## 1 引言

CPU 调度器是操作系统最核心的子系统之一，其设计目标是在公平性、吞吐量、延迟和响应性之间取得平衡。过去二十年间，Linux 内核的通用调度器经历了从 O(1) 调度器到 CFS（Completely Fair Scheduler）再到 EEVDF 的演进[2][3]。CFS 自 Linux 2.6.23 起作为默认调度器，使用红黑树按虚拟运行时间（vruntime）排序，始终选择最左侧节点运行，以加权公平队列（WFQ）的近似实现 CPU 时间的比例共享[3]。CFS 的设计优雅且在大多数工作负载下表现良好，但其结构在表达延迟敏感任务的调度需求时存在固有困难——CFS 中任务的时间片长度由权重和运行队列负载共同决定，延迟敏感任务无法通过更短的时间片获得更频繁的调度机会[2]。

EEVDF 算法最早由 Stoica 和 Abdel-Wahab 于 1995 年提出[1]，其核心思想是在公平队列框架中引入"合格性"（eligibility）概念：一个任务只有在其滞后值（lag）大于等于零时才被视为合格，调度器从合格任务集合中选择虚拟截止时间最早的任务运行。由于虚拟截止时间等于合格时间加上时间片剩余长度，短时间片任务自然获得更早的截止时间，从而在保持公平性的同时提供更好的延迟边界[1][5]。Peter Zijlstra 于 2023 年将 EEVDF 引入 Linux 内核，并在 6.6 版本中作为默认调度器替代 CFS[2]。

尽管 EEVDF 在延迟敏感场景下相比 CFS 有显著改进，但其调度粒度仍然基于墙钟时间片。在实际系统中，不同线程在单位墙钟时间内的有效指令推进量可能存在巨大差异：计算密集型线程可能在一个时间片内执行数百万条指令，而频繁触发缺页异常、系统调用或缓存未命中的线程可能在相同时长内仅推进数千条指令。传统调度器将所有线程的时间片视为等价的"CPU 时间"，但从"完成有效工作量"的角度看，这种等价性并不成立。一个计算效率低下的线程占用与计算密集型线程相同的墙钟时间，却完成远少于后者的工作量，这在本质上构成了一种隐性的不公平。

基于上述观察，本文提出 REEVDF（Rate-aware EEVDF）调度算法。REEVDF 的核心洞察是：调度器可以通过在线监测线程的指令指针（RIP）推进速率，感知每个线程的实际计算效率，并据此动态调整时间量子，使调度决策从"墙钟时间公平"向"有效工作量公平"演进。REEVDF 的主要贡献包括：

第一，提出 RIP（Rate-aware Instruction Progress）反馈机制。该机制在每次调度上下文切换时记录线程的 RIP 起点，在调度 tick 到达时计算该时间片内的 RIP 推进量，进而得到观测指令速率。通过与核内平均速率的 EWMA 基线比较，生成每个线程的速率倍率估计。该机制完全基于软件，无需硬件性能计数器支持，开销仅为每次 tick 的若干次整数运算。

第二，设计双通道动态量子调整架构。乘法通道使用速率倍率对基础量子进行缩放，承担长期稳态塑形；修正通道使用有符号整数直接对量子进行加减，承担短期快速双向响应。两个通道并存使 REEVDF 既能平滑适应线程计算行为的长期变化，又能对瞬时速率波动做出快速修正。修正通道的有符号设计是关键——它允许量子不仅可以增大，也可以减小，从而实现对高速率线程的"奖励"和对低速率线程的"约束"。

第三，引入老化衰减机制保证系统稳定性。当一个线程超过 50 毫秒未被采样时，其速率倍率向 1.0 收缩、修正项向零收缩，防止历史采样结果在线程行为改变后持续产生误导性影响。该机制与 EWMA 平滑共同构成 REEVDF 的稳定性保障。

第四，实现完整的 SMP 多核扩展。包括带节流控制的批量工作窃取（每批最多 8 个线程）、权重差阈值驱动的主动推送（每 256 次 tick 尝试）、SIMD 兼容性感知的线程迁移（避免将使用 AVX-512 的线程迁移到不支持该指令集的核心）以及饥饿阈值保护（防止窃取过度滞后的线程）。

本文其余部分组织如下：第 2 节回顾相关工作；第 3 节详细描述 REEVDF 算法设计；第 4 节介绍 SMP 多核扩展；第 5 节给出实现细节；第 6 节进行性能评估与理论分析；第 7 节讨论；第 8 节总结。

## 2 相关工作

### 2.1 公平调度算法的演进

比例共享调度的理论基础可以追溯到 Demers 等人提出的加权公平队列（WFQ）算法，该算法通过模拟通用处理器共享（GPS）模型为每个流提供按权重比例的带宽分配。WFQ 需要维护每个流的虚拟完成时间，在分组交换网络中得到广泛应用，但其 O(log n) 的每包开销和对虚拟时间全局同步的依赖使其不直接适用于操作系统进程调度。

Goyal 等人于 1996 年提出 Start-Time Fair Queueing（STFQ），使用起始时间标签而非完成时间标签进行调度决策，显著降低了算法复杂度，同时保持了与 WFQ 相当的公平性和延迟边界[4]。STFQ 的核心思想是：每个分组的起始标签等于 max(当前轮询虚拟时间, 该流上一分组的完成标签)，调度器始终选择起始标签最小的分组。STFQ 不需要维护全局虚拟完成时间，只需跟踪当前服务分组的起始时间，这一简化对后续操作系统调度器设计产生了深远影响。

Stoica 和 Abdel-Wahab 于 1995 年提出 EEVDF 算法[1]，在比例共享框架中引入合格性概念。EEVDF 为每个任务维护滞后值 lag = 应得服务量 - 实际获得服务量，当 lag >= 0 时任务被视为合格。调度器从合格任务中选择虚拟截止时间最早的任务，其中虚拟截止时间 = 合格时间 + 时间片长度。EEVDF 的关键性质是：当一个任务持续请求服务时，其获得的服务量始终在最大量子尺寸内与其应得量保持一致[1]。这一性质使 EEVDF 既能提供严格的公平性保证，又能通过短时间片为延迟敏感任务提供更早的截止时间。

### 2.2 Linux 内核调度器实践

Linux 2.6.23 引入的 CFS 由 Ingo Molnar 设计，其核心是一棵按 vruntime 排序的红黑树[3]。CFS 为每个任务维护 vruntime，任务运行时 vruntime 按实际运行时间 / 权重的速率增长。调度器始终选择 vruntime 最小的任务（红黑树最左节点）。CFS 还维护 min_vruntime 作为单调递增的系统进度基准，新唤醒的任务的 vruntime 被校准到 min_vruntime 附近以避免睡眠者获得过多补偿[3]。CFS 的设计在过去 16 年间被证明是健壮且高效的，但其"所有任务按 vruntime 排序"的单一维度无法表达延迟敏感任务对更短时间片的需求[2]。

Linux 6.6 引入的 EEVDF 由 Peter Zijlstra 实现，在 CFS 的红黑树和 vruntime 基础上增加了 deadline 和 lag 的概念[2][5]。EEVDF 的运行队列仍然按 deadline 排序，但选择下一个任务时首先过滤出合格任务（lag >= 0），再从中选择 deadline 最小的。时间片长度可以由任务的"延迟需求"属性控制——短时间片任务 deadline 更早，自然获得更频繁的调度。LWN.net 的分析指出，EEVDF 通过合格性检查避免了 CFS 中"睡眠者唤醒后立即抢占"导致的不公平，同时通过虚拟截止时间为交互任务提供了可证明的延迟边界[5]。

### 2.3 动态量子与反馈调度

动态调整时间量子的思想在调度研究中早有探索。传统 UNIX 调度器（如 4.4BSD 的多级反馈队列）通过根据线程的 CPU 消耗行为动态调整优先级，间接实现了对计算密集型和 I/O 密集型线程的区分。然而，这种基于优先级的调整粒度较粗，且优先级反转等问题使其难以提供严格的公平性保证。

在虚拟机管理程序领域，Credit 调度器和 Borrowed Virtual Time 算法也探索了动态调整调度参数的机制，但这些机制主要关注跨虚拟机的公平性，而非单线程内的指令速率感知。

与上述工作不同，REEVDF 的创新点在于：它不依赖线程的 I/O 行为或优先级分类，而是直接测量线程在时间片内的实际指令推进量，将调度反馈建立在"有效工作量"而非"墙钟时间"之上。这种直接测量的方式使 REEVDF 能够捕捉到更细粒度的计算效率差异——例如，两个都是计算密集型的线程，一个主要命中 L1 缓存，另一个频繁触发 L3 缓存未命中，它们的指令速率可能相差 5 倍以上，而传统调度器无法区分这种差异。

### 2.4 多核负载均衡

多核调度器的负载均衡主要有两种范式：推送（push）和窃取（steal）。Blumofe 和 Leiserson 提出的工作窃取算法是并行计算领域的经典成果[6]，每个处理器维护一个双端队列，本地任务从一端 LIFO 取出以利用缓存局部性，空闲处理器从另一端 FIFO 窃取任务。工作窃取被证明在空间和时间上都是最优的，其期望执行时间为 T1/P + O(T∞)，其中 T1 是串行执行时间，T∞ 是关键路径长度，P 是处理器数[6]。

在操作系统内核中，Linux 的 CFS 采用周期性负载均衡与空闲时负载均衡相结合的方式，在调度域（scheduling domain）层级内迁移任务。REEVDF 借鉴了工作窃取的思想，但针对操作系统内核的特点进行了适配：使用红黑树而非双端队列作为运行队列，窃取时从 deadline 最大的线程开始（即最"不紧急"的线程），并引入节流控制避免窃取操作过于频繁。

## 3 REEVDF 算法设计

### 3.1 系统模型与问题定义

考虑一个具有 P 个同质处理核心的多核系统，每个核心维护一个本地运行队列。系统中有 N 个可运行线程，每个线程 T_i 具有：

- 权重 w_i，表示该线程应获得的 CPU 时间比例（由优先级映射而来，范围 288 至 8192）；
- 虚拟运行时间 vruntime_i，表示该线程按权重归一化后的累计运行时间；
- 虚拟截止时间 deadline_i，表示该线程当前时间片的虚拟结束时间；
- 状态 state_i，取值为 RUNNING、BLOCKED、SLEEPING、TRANSFER 或 ZOMBIE。

每个核心 C_j 维护：

- 基础量子 base_quantum_j，范围 2 至 15，单位为 LAPIC tick；
- 平均虚拟运行时间 avg_vruntime_j，作为核心内所有线程的加权平均进度基准；
- 平均指令速率 rip_avg_rate_j，作为核心内线程指令推进速率的 EWMA 基线；
- 总权重 total_weight_j，为运行队列中所有线程权重之和。

传统 EEVDF 的调度决策可以形式化为：在每个调度点，从合格线程集合 E = {T_i | lag_i >= 0} 中选择 deadline_i 最小的线程。其中 lag_i = 应得服务量 - 实际服务量，deadline_i = eligible_time_i + slice_i，slice_i 为时间片长度。

REEVDF 在此基础上引入的核心问题是：如何根据线程 T_i 在单位墙钟时间内的实际指令推进量，动态调整其时间片长度 slice_i，使调度决策从"墙钟时间比例公平"演进到"有效指令工作量比例公平"。

### 3.2 EEVDF 基础与局限

REEVDF 保留了 EEVDF 的核心机制作为基础。运行队列使用红黑树按 deadline 排序，插入线程时通过 calibrate_and_set_deadline 函数计算其 deadline：

```
virtual_slice = base_quantum
max_lag = virtual_slice
target_vr = max(avg_vruntime - max_lag, 0)
if vruntime < target_vr:
    vruntime = target_vr
elif vruntime > avg_vruntime + max_lag * 2:
    vruntime = avg_vruntime + max_lag * 2
deadline = vruntime + virtual_slice
```

该校准机制确保新插入或唤醒的线程的 vruntime 不会过度偏离核心平均进度，从而防止睡眠者获得过多补偿或被过度惩罚。选择下一个线程时，Pick 函数从红黑树根节点开始，沿 min_vruntime_subtree 指针找到 vruntime <= avg_vruntime 的最左合格线程；若所有线程的 vruntime 均大于 avg_vruntime，则回退到选择第一个可运行线程。

EEVDF 的局限在于其时间片长度 slice_i 仅由权重和基础量子决定，不考虑线程的实际计算效率。在 REEVDF 的术语中，EEVDF 隐含假设所有线程的指令推进速率相同，即 obs_rate_i = rip_avg_rate_j 对所有 i 成立。这一假设在实际系统中并不成立，由此产生的调度偏差正是 REEVDF 试图修正的目标。

### 3.3 RIP 速率感知反馈机制

RIP（Rate-aware Instruction Progress）反馈机制是 REEVDF 的核心创新。其基本思想是：在每次将线程分派到 CPU 时记录其指令指针 RIP，在调度 tick 到达时计算该时间片内的 RIP 推进量，进而得到观测指令速率。

3.3.1 采样不变量。

RIP 反馈的采样路径遵循以下不变量：

```
progress = rip_now - dispatch_rip
obs_rate = progress / slice_ms
obs_mult = obs_rate / rip_avg_rate_j
```

其中：
- dispatch_rip 是线程上次被分派到 CPU 时的 RIP 值，在上下文切换时记录；
- rip_now 是当前调度 tick 时的 RIP 值，来自中断上下文；
- slice_ms 是该线程当前时间片的长度（毫秒），由 get_dynamic_quantum 计算；
- progress 是该时间片内的 RIP 推进量（字节地址差）；
- obs_rate 是观测指令速率（RIP 推进量 / 毫秒）；
- obs_mult 是观测倍率，即该线程速率与核心平均速率的比值，使用 10 位定点数表示（1024 = 1.0x）。

3.3.2 异常防御与噪声过滤。

RIP 采样包含多层防御机制以确保数据质量：

第一，环绕防御。若 progress > 2^44，则视为异常（可能由 RIP 回绕或上下文损坏导致），丢弃本次采样。该阈值对应约 17 TB 的地址空间推进，在正常执行中不可能达到。

第二，碎片段过滤。若 slice_ms < 1 毫秒，说明这是一个极短的碎片时间片（例如线程刚被分派就被更高优先级事件抢占），其采样噪声过大，仅执行老化更新而不进行速率采样。

第三，零速率过滤。若 obs_rate == 0，说明线程在该时间片内完全停滞（可能陷入死循环在同一地址、或被硬件断点阻塞），不进行采样。

第四，离群值丢弃。若 obs_mult > RIPRATE_MAX_MULT << 2（即超过 16x），视为离群值丢弃。该阈值远大于正常速率波动范围（4x 上限），用于过滤由异常事件导致的极端值。

3.3.3 EWMA 平滑与倍率更新。

通过质量过滤的采样用于更新两个 EWMA 基线：

核心平均速率基线：
```
rip_avg_rate_j += (obs_rate - rip_avg_rate_j) >> 3
```
使用 1/8 的 EWMA 系数（位移 3），对核心内所有线程的速率进行平滑，形成稳定的比较基线。首次采样时直接将 rip_avg_rate_j 设为 obs_rate。

线程速率倍率：
```
rip_rate_mult_i += (obs_mult - rip_rate_mult_i) >> RIPRATE_SHIFT
```
其中 RIPRATE_SHIFT = 3，同样使用 1/8 的 EWMA 系数。更新后进行钳位：
```
rip_rate_mult_i = clamp(rip_rate_mult_i, RIPRATE_MIN_MULT, RIPRATE_MAX_MULT)
```
其中 RIPRATE_MIN_MULT = 256（0.25x），RIPRATE_MAX_MULT = 4096（4x）。倍率的初始值为 RIPRATE_INIT = 1024（1.0x）。

EWMA 平滑的作用是防止单次采样的随机波动导致量子的剧烈振荡。1/8 的系数意味着新采样的权重为 12.5%，历史值的权重为 87.5%，大约需要 8 次连续同向采样才能使倍率接近新的观测值。

### 3.4 双通道动态量子调整

REEVDF 的时间量子计算采用双通道架构：乘法通道承担稳态塑形，修正通道承担快速双向响应。最终有效量子的计算公式为：

```
eff_quantum = (base_quantum_i × rip_rate_mult_i >> RIPRATE_FRAC_BITS) + rip_quantum_adj_i
```

其中 base_quantum_i 是线程 i 的基础量子（由核心 base_quantum 和线程权重计算，或由 custom_quantum 直接指定），rip_rate_mult_i 是乘法通道的速率倍率（10 位定点），RIPRATE_FRAC_BITS = 10，rip_quantum_adj_i 是修正通道的有符号修正项。

3.4.1 乘法通道：稳态塑形。

乘法通道的输出为 base_quantum_i × rip_rate_mult_i >> 10。由于 rip_rate_mult_i 的范围是 256 至 4096（即 0.25x 至 4x），乘法通道可以将基础量子缩放到 [0.25x, 4x] 的范围。

乘法通道的设计意图是捕捉线程计算行为的长期稳态特征。例如，一个持续以 2x 平均速率推进的计算密集型线程，其 rip_rate_mult_i 将逐渐收敛到约 2048（2.0x），其时间量子也将稳定为基础量子的 2 倍。这意味着该线程在每次调度中获得 2 倍的墙钟时间，但由于其指令速率也是 2 倍，它在每次调度中完成的有效指令工作量与平均速率线程相同——这正是"有效工作量公平"的体现。

反之，一个持续以 0.5x 平均速率推进的低效率线程（例如频繁触发缓存未命中），其 rip_rate_mult_i 将收敛到约 512（0.5x），时间量子缩短为基础量子的一半。这防止了低效率线程占用过多墙钟时间而完成极少有效工作量。

3.4.2 修正通道：快速双向响应。

修正通道是 REEVDF v2 版本新增的关键机制，其设计动机是：乘法通道的 EWMA 平滑虽然保证了稳定性，但响应速度较慢——当线程的计算行为突然改变时（例如从计算密集型切换到频繁系统调用），乘法通道需要约 8 次采样才能完成调整。在这 8 次采样期间，线程的量子可能严重偏离最优值。

修正通道使用有符号整数 rip_quantum_adj_i 直接对量子进行加减，其范围为 [RIPADJ_MIN, RIPADJ_MAX] = [-4, +4]。修正项的更新规则为：

```
dev = obs_mult - RIPRATE_ONE    // 定点偏差，有符号
step = dev >> 6                   // 修正步长，dev/64
if step == 0:
    step = (dev > 0) ? 1 : ((dev < 0) ? -1 : 0)
rip_quantum_adj_i = clamp(rip_quantum_adj_i + step, RIPADJ_MIN, RIPADJ_MAX)
```

修正通道的关键设计决策包括：

第一，有符号性。rip_quantum_adj_i 是 int32_t 类型，可以为正或为负。正值增大量子（奖励高速率线程），负值减小量子（约束低速率线程）。这实现了用户要求的"修正还能减"的双向响应能力。

第二，温和步长。step = dev >> 6 意味着每次修正最多移动偏差的 1/64。例如，若 obs_mult = 1536（1.5x），则 dev = 512，step = 8，但由于钳位上限为 +4，实际步长为 4。若 obs_mult = 1100（约 1.07x），则 dev = 76，step = 1。这种温和的步长防止修正通道本身产生振荡。

第三，小偏差保证。当 |dev| < 64 时，dev >> 6 = 0，此时强制至少 ±1 的修正步长（只要 dev != 0）。这确保了即使是微小的速率偏差也能被修正通道感知和响应，避免"永远修不动"的死区。

第四，钳位保护。修正项被钳位在 [-4, +4] 范围内，防止极端偏差导致量子的剧烈变化。该范围相对于典型基础量子（5-10）是合理的——最大修正幅度约为基础量子的 40%-80%。

3.4.3 双通道协同。

乘法通道与修正通道的协同关系可以概括为：乘法通道负责"方向"，修正通道负责"速度"。具体而言：

- 当线程的速率偏差持续存在时，乘法通道的 EWMA 逐渐调整倍率，提供大范围（0.25x-4x）的稳态调整；
- 修正通道对每次采样的瞬时偏差做出快速响应，提供小范围（±4）的即时修正；
- 两者叠加后，REEVDF 既能对长期行为变化做出大幅度调整，又能对瞬时波动做出快速响应。

在量子计算的最后，REEVDF 还执行两层钳位：下限为 1（确保至少一个 tick 的量子），上限为 base_quantum × 8（防止量子过大导致调度延迟过高）。

### 3.5 老化衰减与稳定性保证

RIP 反馈机制的一个潜在风险是：如果一个线程长时间未被调度（例如被阻塞等待 I/O），其历史采样结果可能在它被唤醒后仍然产生影响，而此时它的计算行为可能已经完全改变。为解决这一问题，REEVDF 引入老化衰减机制。

老化检查在每次采样之前执行，使用线程上次采样时间戳 rip_last_sample_ms_i：

```
if now_ms - rip_last_sample_ms_i > RIPRATE_AGING_MS (50ms):
    // 倍率向 1.0 收缩 1/16
    if rip_rate_mult_i > RIPRATE_ONE:
        rip_rate_mult_i -= (rip_rate_mult_i - RIPRATE_ONE) >> RIPRATE_DECAY_SHIFT
    elif rip_rate_mult_i < RIPRATE_ONE:
        rip_rate_mult_i += (RIPRATE_ONE - rip_rate_mult_i) >> RIPRATE_DECAY_SHIFT

    // 修正项向 0 收缩
    if rip_quantum_adj_i > 0:
        rip_quantum_adj_i -= (rip_quantum_adj_i + 7) >> RIPRATE_DECAY_SHIFT
    elif rip_quantum_adj_i < 0:
        rip_quantum_adj_i += ((-rip_quantum_adj_i) + 7) >> RIPRATE_DECAY_SHIFT
```

其中 RIPRATE_DECAY_SHIFT = 4，即每次老化收缩 1/16。(adj + 7) >> 4 的写法确保正值修正项至少减少 1（向上取整），避免小修正项永远衰减不到零。

老化机制的设计参数选择基于以下考虑：

- 50ms 的老化阈值大约相当于 5-10 个典型时间片（假设基础量子为 5-10ms）。如果一个线程在 50ms 内未被采样，说明它要么被阻塞，要么在运行队列中等待了较长时间——在这两种情况下，其历史速率数据的可信度都应降低。
- 1/16 的收缩步长是温和的——单次老化仅消除偏差的 6.25%，需要约 16 次老化（约 800ms）才能完全收敛到中性值。这防止了老化机制本身引入参数振荡。
- 倍率和修正项独立衰减，因为它们捕捉不同时间尺度的行为——倍率是长期稳态，修正项是短期偏差。修正项的衰减速度应快于倍率，但在当前实现中两者使用相同的 1/16 步长，这是一个可以进一步优化的参数。

老化机制与 EWMA 平滑共同构成 REEVDF 的稳定性保障：EWMA 防止单次采样导致参数剧变，老化防止历史参数在行为改变后持续误导。两者的结合使 REEVDF 的动态量子调整在响应性和稳定性之间取得了平衡。

### 3.6 动态基础量子调整

除了线程级别的动态量子调整，REEVDF 还在核心级别动态调整基础量子 base_quantum_j。该调整每 100ms 执行一次，基于核心的空闲比例和上下文切换频率：

```
idle_ratio = idle_tsc / total_tsc * 100
ctx_sw = context_switches - last_ctx_sw

if idle_ratio > 50:
    if base_quantum_j < 15: base_quantum_j++    // 空闲多，增大量子减少切换开销
elif idle_ratio < 10 and ctx_sw > 500:
    if base_quantum_j > 2: base_quantum_j--     // 繁忙且切换频繁，减小量子提高响应性
else:
    if base_quantum_j > 5: base_quantum_j--
    elif base_quantum_j < 5: base_quantum_j++   // 中等负载，向默认值 5 收敛
```

基础量子的范围被限制在 2 至 15。该机制的设计逻辑是：

- 当核心空闲比例超过 50% 时，说明系统负载较轻，此时增大量子可以减少上下文切换的开销，提高吞吐量；
- 当核心空闲比例低于 10% 且每 100ms 上下文切换超过 500 次时，说明系统高度繁忙且线程切换频繁，此时减小量子可以提高调度响应性，避免单个线程长时间占用 CPU；
- 中等负载时，基础量子向默认值 5 收敛。

TSC（Time Stamp Counter）用于精确测量空闲时间和总时间。dyn_adjust_ctx 结构为每个核心维护 last_tsc、idle_tsc、total_tsc、last_ctx_sw 和 last_adjust_ms 等状态。在每次调度 tick 时，计算自上次 tick 以来的 TSC 增量，如果当前线程是 idle_thread 则计入 idle_tsc，否则计入 total_tsc。

动态基础量子调整与线程级双通道调整是正交的：基础量子是所有线程的基准，线程级调整在此基准上根据每个线程的速率特征进行个性化缩放。两者的结合使 REEVDF 能够同时适应核心级负载变化和线程级计算行为差异。

## 4 SMP 多核扩展

REEVDF 的多核扩展围绕三个核心机制展开：工作窃取（StealThread）、主动推送（TryPush）和 SIMD 兼容性感知迁移。所有多核操作均遵循饥饿保护原则，防止过度滞后的线程被迁移。

### 4.1 工作窃取机制

当一个核心的运行队列为空时，它尝试从其他核心窃取线程。工作窃取的触发条件是：当前核心 Pick 返回空（没有可运行线程），此时调用 StealThread。

为避免窃取操作过于频繁导致锁竞争，REEVDF 引入节流控制：每个核心维护 per_cpu_steal_throttle[cpu_id].skip 计数器，每次调用 StealThread 时递增，只有当 skip >= SCHED_STEAL_THROTTLE（8）时才执行实际窃取，否则直接返回空。这意味着每个核心最多每 8 次调度尝试执行一次窃取，有效降低了跨核心锁竞争。

窃取的目标选择采用轮询起点随机化策略：

```
start_cpu = (sched_pid + PIT::TimeSinceBootMS() + per_cpu_steal_cursor[cpu_id].v) % ncpu
per_cpu_steal_cursor[cpu_id].v++
```

起点由全局 PID 计数器、系统启动时间（毫秒）和核心本地游标三者异或取模生成，确保不同核心在不同时间的窃取起点不同，避免多个核心同时涌向同一个受害者。

窃取执行两轮（pass = 0 和 pass = 1）：
- 第一轮（pass = 0）：仅考虑 SIMD 兼容性匹配的核心（cpu_simd_mask 相同），优先保持线程的 SIMD 状态局部性；
- 第二轮（pass = 1）：不限制 SIMD 兼容性，考虑所有核心，作为兜底确保负载均衡。

对每个候选受害者核心，首先使用原子加载检查其 has_surplus 标志（是否有超过 1 个可运行线程），若没有则跳过。然后尝试获取受害者的 sched_lock（自旋锁，最多重试 100 次，每次 pause）。获取锁后，从红黑树的最右节点（deadline 最大，即最不紧急的线程）开始，批量窃取最多 SCHED_STEAL_BATCH（8）个线程。

窃取的线程筛选条件：
- 不是受害者当前正在运行的线程；
- 不是 idle_thread；
- 状态为 THREAD_RUNNING；
- 没有关联定时器（timer_bucket == nullptr），避免迁移正在等待定时唤醒的线程；
- vruntime <= hunger_limit（avg_vruntime + SCHED_HUNGER_THRESHOLD，即 5ms 的虚拟时间），防止窃取过度饥饿的线程。

窃取成功后，线程状态先设为 THREAD_TRANSFER（迁移中），释放受害者锁，然后获取当前核心锁，将线程的 cpu_num 和 timer_cpu 更新为当前核心 ID，状态恢复为 THREAD_RUNNING，插入当前核心运行队列。最后调用 Pick 选择下一个运行线程。

### 4.2 主动推送机制

工作窃取是"拉"模式——空闲核心主动从繁忙核心拉取线程。REEVDF 还实现了"推"模式——繁忙核心主动将线程推送到空闲核心。主动推送由 TryPush 函数实现，每 256 次调度 tick（tick_count & 0xFF == 0）且运行队列线程数 > 2 时触发。

推送的目标选择基于权重差阈值：

```
my_weight = total_weight + current_thread.weight
for each other core:
    ow = other.total_weight + other.current_thread.weight
    if ow < my_weight and ow < target_weight:
        if SIMD compatible: target = other, target_weight = ow
        else if no fallback: fallback_target = other
```

优先选择 SIMD 兼容性匹配且权重最低的核心作为推送目标；若没有匹配的，则回退到任意权重更低的核心。

推送的执行需要一个关键的权重差检查：

```
if target_weight + (target_weight >> SCHED_PUSH_GAP_SHIFT) >= my_weight:
    return  // 权重差不足 1/4，不推送
```

SCHED_PUSH_GAP_SHIFT = 2，即只有当目标核心权重比当前核心低超过 25% 时才执行推送。该阈值防止了在权重差异不大时频繁迁移线程，因为线程迁移本身有成本（缓存失效、TLB 刷新等）。

推送时需要同时获取当前核心和目标核心的锁，按核心 ID 从小到大的顺序获取以避免死锁。然后从当前核心红黑树的最右节点开始，批量推送最多 8 个线程，每推送一个后重新检查权重差，一旦目标核心权重不低于当前核心则停止。

### 4.3 SIMD 兼容性感知迁移

现代 x86_64 处理器可能存在核心间 SIMD 能力不对称的情况（例如大小核架构中，大核支持 AVX-512 而小核不支持）。REEVDF 在所有线程迁移操作（窃取和推送）中都考虑了 SIMD 兼容性。

每个核心的 SIMD 能力由 cpu_simd_mask 函数计算，是一个 32 位掩码，每一位对应一种 SIMD 特性：
- bit 0: SupportSIMD（基本 SSE）
- bit 1: SupportXSAVE
- bit 2: SupportSSE4_2
- bit 3: SupportAVX
- bit 4: SupportAVX2
- bit 5: SupportAVX512
- bit 6: SupportXSAVEOPT

线程迁移时，只有源核心和目标核心的 cpu_simd_mask 完全相同才被视为兼容。不兼容的迁移可能导致线程在目标核心上执行不支持的 SIMD 指令而触发无效操作异常（#UD）。

在工作窃取中，第一轮仅考虑 SIMD 兼容的核心，第二轮才放宽限制。在主动推送中，同样优先选择 SIMD 兼容的目标。这种设计在大多数情况下避免了不兼容迁移，同时在系统极度不均衡时允许兜底迁移以保证整体吞吐量。

### 4.4 饥饿保护

REEVDF 的所有线程迁移操作都遵循饥饿保护原则：不迁移 vruntime 超过 hunger_limit 的线程。hunger_limit 定义为 avg_vruntime + SCHED_HUNGER_THRESHOLD，其中 SCHED_HUNGER_THRESHOLD = 5ms（虚拟时间）。

该保护的设计逻辑是：一个 vruntime 远超平均水平的线程是"饥饿"的——它已经很久没有获得 CPU 时间。将这样的线程迁移到另一个核心可能导致它在新核心的运行队列中再次等待，从而加剧饥饿。相反，饥饿线程应该在当前核心尽快获得调度机会。饥饿保护确保了迁移操作只影响"不紧急"的线程，不会使已经滞后的线程雪上加霜。

## 5 实现细节

### 5.1 数据结构

REEVDF 的核心数据结构定义在 sched.h 和 smp.h 中[8]。

thread_t 结构中与 REEVDF 相关的字段包括：

| 字段 | 类型 | 含义 |
|---|---|---|
| vruntime | uint64_t | 虚拟运行时间，按权重归一化的累计运行时间 |
| deadline | uint64_t | 虚拟截止时间，vruntime + 时间片长度 |
| weight | uint32_t | 线程权重，由优先级映射，范围 288-8192 |
| rb_node | rb_node_t | 红黑树节点，挂入核心运行队列 |
| on_rq | bool | 是否在运行队列中 |
| vruntime_rem | uint64_t | vruntime 计算的余数，用于精确累加 |
| min_vruntime_subtree | uint64_t | 子树最小 vruntime，用于加速 Pick |
| rip_rate_mult | uint64_t | RIP 速率倍率，10 位定点，1024 = 1.0x |
| dispatch_rip | uint64_t | 上次分派时的 RIP 值，用于计算进度 |
| rip_quantum_adj | uint64_t | RIP 量子修正项，有符号，范围 [-4, +4] |
| rip_last_sample_ms | uint64_t | 上次 RIP 采样的毫秒时间戳，用于老化检查 |
| custom_quantum | uint64_t | 自定义基础量子，若 > 0 则覆盖权重计算 |

cpu_t 结构中与 REEVDF 相关的字段包括：

| 字段 | 类型 | 含义 |
|---|---|---|
| runqueue_root | rb_root_t | 运行队列红黑树根节点 |
| base_quantum | uint64_t | 核心基础量子，范围 2 至 15 |
| avg_vruntime | uint64_t | 核心平均虚拟运行时间 |
| avg_vruntime_rem | uint64_t | avg_vruntime 计算余数 |
| total_weight | uint64_t | 运行队列总权重 |
| rip_avg_rate | uint64_t | 核心平均指令速率（EWMA） |
| thread_count | volatile uint64_t | 运行队列线程数（不含当前运行线程） |
| has_surplus | volatile bool | 是否有超过 1 个可运行线程 |
| sched_stats | sched_stats_t | 调度统计（上下文切换、窃取次数等） |

优先级到权重的映射使用 sched_prio_to_weight 数组（16 个元素），优先级 0（最高）对应权重 8192，优先级 15（最低）对应权重 288。该映射近似遵循 1.25 的几何级数，与 Linux CFS 的 nice 到 weight 映射类似。

### 5.2 调度主循环

调度主循环由 Schedule::Internal::Switch 函数实现，在每次 LAPIC 定时器中断（SCHED_VEC = 48）时触发。主循环的执行流程如下：

第一步，早期 SMP 启动窗口处理。如果核心尚未创建 idle_thread 或 current_thread 为空（AP 核心在 smp_cpu_init 后、Schedule::Install 前的窗口期），直接重新武装定时器并返回，避免空指针解引用。

第二步，账本结算（锁外执行）。包括：
- dynamic_adjust_quantum：更新 TSC 统计，每 100ms 调整基础量子；
- vruntime 更新：当前线程的 vruntime 按 delta * 1024 / weight 增长，核心 avg_vruntime 按 delta * 1024 / active_weight 增长；
- RIP 反馈采样：调用 riprate_update，传入当前 RIP、动态量子和当前时间。

第三步，免锁快速路径判定。如果当前线程状态为 RUNNING、不是 idle_thread、没有 yield 请求、且运行队列中没有其他等待线程（thread_count == 0），则跳过锁直接继续运行当前线程。该快速路径避免了单线程场景下不必要的锁获取和红黑树操作。

第四步，慢路径（持锁）。获取核心 sched_lock，处理僵尸线程回收（zombie_count >= 8 时批量回收最多 16 个），将当前线程重新插入运行队列（若状态为 RUNNING），调用 Pick 选择下一个线程。

第五步，多核负载均衡。每 256 次 tick 且线程数 > 2 时调用 TryPush 主动推送。若 Pick 返回空，调用 StealThread 工作窃取；若仍为空，使用 idle_thread。

第六步，上下文切换。如果选择的下一个线程与当前线程不同，执行实际的上下文切换：保存当前线程的上下文（通用寄存器、SIMD 状态、FS 基址），恢复下一个线程的上下文，切换页表（如果不同进程），记录 dispatch_rip，武装 LAPIC 定时器。

### 5.3 抢占机制

REEVDF 的抢占由 TriggerPreempt 函数实现，在线程被唤醒（例如 I/O 完成、定时器到期）时调用。抢占的判定条件为：

```
if woked_thread->vruntime <= cpu->avg_vruntime
   and woked_thread->deadline < curr->deadline:
    w_prod = curr->weight * woked->weight
    remaining_vr = curr->deadline - curr->vruntime
    if w_prod != 0 and remaining_vr < PREEMPT_THRESHOLD / w_prod:
        return  // 剩余时间片太短，不抢占
    // 触发抢占
```

PREEMPT_THRESHOLD = 1024 * 1024（约 1ms 虚拟时间）。该条件的含义是：只有当唤醒线程的 vruntime 不超过平均水平（合格）、其 deadline 早于当前线程（更紧急）、且当前线程的剩余时间片足够长（超过 PREEMPT_THRESHOLD / (w1*w2)）时，才触发抢占。剩余时间片检查避免了在当前线程即将结束时间片时进行不必要的抢占——因为很快就会在正常的调度 tick 中切换。

抢占的执行方式取决于唤醒发生在哪个核心：如果是远程核心，发送 IPI（处理器间中断）触发调度；如果是当前核心，设置 need_resched_flag，在下次 CheckPreempt 时触发软中断进入调度。

### 5.4 无锁快速路径与 yield 处理

REEVDF 的无锁快速路径是一个关键的性能优化。当运行队列中只有当前线程一个可运行线程时（thread_count == 0），调度器不需要获取锁、不需要操作红黑树，直接继续运行当前线程并重新武装定时器。这在单线程负载（例如计算密集型基准测试）下可以显著降低调度开销。

然而，无锁快速路径有一个重要的例外：自愿 yield（Schedule::Yield）。当线程主动调用 Yield 时，即使运行队列中只有它和另一个线程，也必须执行实际的调度切换——否则调用 Yield 的线程会继续运行，而唯一的其他线程永远得不到调度机会。REEVDF 通过 yield_request_flags  per-CPU 标志来处理这种情况：Yield 函数设置该标志，Switch 函数在判定无锁快速路径时检查该标志，若为真则强制走慢路径。

## 6 性能评估与理论分析

### 6.1 评估方法论

由于 REEVDF 是 SkylineSystem 研究型内核的一部分[7]，完整的跨平台 benchmark 对比（与 Linux CFS/EEVDF、FreeBSD ULE 等）需要在裸机或 QEMU/KVM 环境中运行标准基准测试套件（如 hackbench、perf bench、Schbench、Lmbench）。本文在此给出理论复杂度分析、算法特性定性分析和实现中的统计监控机制设计，完整的实证数据留待后续工作。

评估 REEVDF 性能应关注以下维度：

- 公平性：线程获得的 CPU 时间与其权重的比例偏差，可使用 Jain 公平指数或最大滞后值衡量；
- 延迟：调度延迟（从线程唤醒到获得 CPU 的时间）和尾延迟（P99）；
- 吞吐量：系统总指令完成率，尤其在混合计算密集型和 I/O 密集型负载下；
- 调度开销：每次调度 tick 的指令数和锁竞争率；
- 多核均衡：核心间负载偏差和线程迁移率。

### 6.2 理论复杂度分析

REEVDF 的核心操作复杂度如下：

| 操作 | 数据结构 | 时间复杂度 | 说明 |
|---|---|---|---|
| 插入运行队列 | 红黑树 | O(log n) | rb_insert + 向上更新 min_vruntime_subtree |
| 从运行队列移除 | 红黑树 | O(log n) | rb_erase + 向上更新 |
| 选择下一个线程 | 红黑树 | O(log n) | 沿 min_vruntime_subtree 指针下降 |
| RIP 采样更新 | 算术运算 | O(1) | EWMA + 钳位，无数据结构操作 |
| 动态量子计算 | 算术运算 | O(1) | 乘法 + 移位 + 有符号加减 |
| 工作窃取 | 红黑树 + 锁 | O(b log n) | b = 批量大小（最大 8），含两次锁获取 |
| 主动推送 | 红黑树 + 锁 | O(b log n + P) | P = 核心数，扫描所有核心找目标 |

其中 n 是单个核心运行队列中的线程数。红黑树的 O(log n) 复杂度与 Linux CFS/EEVDF 相同。RIP 反馈机制的所有操作（采样、EWMA 更新、量子计算）均为 O(1)，仅增加常数级开销。

在空间复杂度方面，每个线程增加 4 个 REEVDF 专用字段（rip_rate_mult、dispatch_rip、rip_quantum_adj、rip_last_sample_ms），共 32 字节；每个核心增加 1 个字段（rip_avg_rate），8 字节。相对于 thread_t 的总大小（约 500 字节），空间开销约为 6%。

### 6.3 算法特性定性分析

6.3.1 公平性。

REEVDF 在 EEVDF 的公平性保证基础上，将公平性的度量从"墙钟时间"扩展到"有效指令工作量"。在传统 EEVDF 中，两个权重相同的线程获得相同的墙钟时间，但如果它们的指令速率不同，则完成的有效工作量不同。REEVDF 通过动态量子调整，使高速率线程获得更长的时间片、低速率线程获得更短的时间片，从而使两者在每次调度中完成的有效工作量趋于相等。

需要指出的是，REEVDF 的"有效工作量公平"是一种近似而非严格保证。RIP 推进量（指令地址差）并不精确等于执行的指令数——分支跳转、循环、函数调用都会导致 RIP 推进量与指令数的偏差。此外，RIP 推进量无法衡量指令的"质量"（一条 SIMD 指令可能完成相当于 16 条标量指令的工作量）。因此，REEVDF 的公平性是基于"指令地址推进速率"的代理指标，而非严格的"有效工作量"度量。尽管如此，在大多数通用计算负载中，RIP 推进速率与实际指令吞吐量之间存在足够强的相关性，使 REEVDF 的动态调整能够产生有意义的公平性改善。

6.3.2 响应性。

REEVDF 的响应性由多层机制共同保障：

- EEVDF 的虚拟截止时间机制为延迟敏感任务提供了可证明的延迟边界[1]；
- 动态基础量子调整在高负载时减小基础量子，提高调度频率；
- 修正通道的快速双向响应使量子能够在行为变化后的 1-2 次采样内开始调整；
- 抢占机制在唤醒线程更紧急时触发即时抢占，而不必等待当前时间片结束。

6.3.3 稳定性。

REEVDF 的稳定性由以下机制保障：

- EWMA 平滑（1/8 系数）防止单次采样导致参数剧变；
- 老化衰减（50ms 阈值，1/16 步长）防止历史参数在行为改变后持续误导；
- 倍率钳位（[0.25x, 4x]）和修正项钳位（[-4, +4]）限制单次调整的幅度；
- 离群值丢弃（>16x）过滤异常采样；
- 量子下限（1）和上限（base_quantum × 8）防止极端量子值。

这些机制的叠加使 REEVDF 的动态参数调整具有良好的收敛性和抗噪性。

### 6.4 统计监控机制

REEVDF 在 sched_stats_t 结构中维护以下统计指标，可用于运行时性能监控和后续 benchmark 分析：

| 指标 | 含义 |
|---|---|
| context_switches | 上下文切换总次数 |
| thread_steals | 成功窃取的线程总数 |
| steal_attempts | 窃取尝试总次数（含失败） |
| zombie_reclaims | 僵尸线程回收次数 |
| try_pushes | 主动推送尝试次数 |
| push_success | 成功推送的线程总数 |

通过这些指标，可以计算窃取成功率（thread_steals / steal_attempts）、推送成功率（push_success / try_pushes）、每次调度的平均迁移率等。此外，rip_rate_mult 和 rip_quantum_adj 的运行时值可以通过内核调试接口读取，用于分析 RIP 反馈机制的实际行为。

## 7 讨论

### 7.1 RIP 作为计算效率代理的局限性

REEVDF 使用 RIP 推进量作为线程计算效率的代理指标，这一选择有其固有的局限性。如 6.3.1 节所述，RIP 推进量与实际指令吞吐量之间存在偏差，主要来源包括：

第一，控制流偏差。分支跳转（尤其是无条件跳转和函数调用返回）使 RIP 推进量不连续。一个包含大量函数调用的线程，其 RIP 推进量可能大于实际执行的指令数（因为调用和返回涉及非连续的地址跳转）。反之，一个在紧密循环中执行的线程，其 RIP 推进量可能远小于实际执行的指令数（因为循环体在相同的地址范围内重复执行）。

第二，指令密度偏差。x86_64 是变长指令集，不同指令的长度从 1 字节到 15 字节不等。一条 15 字节的 AVX-512 指令和一条 1 字节的 NOP 指令在 RIP 推进量上相差 15 倍，但它们的"有效工作量"可能相差更大（AVX-512 指令可能并行处理 16 个 32 位浮点数）。

第三，内存级并行偏差。RIP 推进量衡量的是指令提交（retire）的速率，而非指令执行的速率。一个频繁触发 L3 缓存未命中的线程，其指令可能在乱序执行引擎中被大量缓存未命中阻塞，RIP 推进速率很低，但 CPU 实际上在为该线程服务（等待内存）。REEVDF 会缩短这种线程的量子，这可能导致它需要更多次调度才能完成相同的工作量——这在某种意义上是"公平"的（它确实占用了 CPU 时间但完成的指令少），但也可能增加其总延迟。

尽管存在这些局限性，RIP 推进量作为代理指标仍然有其价值：它是唯一不需要硬件性能计数器（PMC）支持、可以在任何 x86_64 系统上以零额外硬件开销获取的计算效率指标。对于研究型内核和教学场景，这种零依赖的特性尤为重要。在未来的工作中，可以探索将 RIP 与硬件 PMC（如 INST_RETIRED、CPU_CLK_UNHALTED）结合，以获得更精确的计算效率度量。

### 7.2 与硬件感知调度的关系

现代处理器的性能监控单元（PMU）可以提供精确的指令退休数、缓存未命中数、分支预测失败数等硬件事件。基于 PMC 的调度器（如 Google 的 Energy-Aware Scheduling、Linux 的 schedutil 频率调控器）可以利用这些信息做出更精细的调度决策。

REEVDF 的 RIP 机制可以视为 PMC 调度的"软件退化版"——在没有 PMC 支持或 PMC 被其他子系统占用时，RIP 提供了一种轻量级的替代方案。两者不是互斥的：在支持 PMC 的系统上，REEVDF 可以将 obs_rate 的计算从 RIP 推进量替换为指令退休数（通过读取 PMC 计数器），从而获得更精确的速率估计。这种替换只需要修改 riprate_update 中的 progress 计算一行代码，不影响双通道调整、老化、SMP 扩展等其他机制。

### 7.3 参数可调性

REEVDF 包含多个可调参数（RIPRATE_SHIFT、RIPRATE_AGING_MS、RIPRATE_DECAY_SHIFT、RIPADJ_MAX/MIN、SCHED_STEAL_BATCH 等），当前实现中这些参数是编译期常量。在生产系统中，这些参数可以通过 sysctl 或 debugfs 接口暴露为运行时可调参数，以便根据工作负载特征进行调优。

例如，在延迟敏感的交互式系统中，可以减小 RIPRATE_SHIFT（提高 EWMA 响应速度）、减小 RIPRATE_AGING_MS（加快老化）；在吞吐量优先的批处理系统中，可以增大 RIPRATE_SHIFT（提高稳定性）、增大 SCHED_STEAL_BATCH（提高窃取效率）。当前的默认参数（1/8 EWMA、50ms 老化、±4 修正、8 批量）是在通用场景下的合理折中。

### 7.4 与实时调度的兼容性

REEVDF 设计用于通用调度（SCHED_NORMAL 类），不直接处理硬实时任务。在 SkylineSystem 中，实时任务（如果实现）应使用独立的调度类（如 SCHED_FIFO 或 SCHED_RR），优先级高于 REEVDF。REEVDF 的动态量子调整仅适用于通用任务，不应影响实时任务的确定性延迟保证。

## 8 结论

本文提出了 REEVDF（Rate-aware Earliest Eligible Virtual Deadline First）调度算法，在 EEVDF 的公平性和延迟边界基础上，引入了基于指令指针进度的速率感知反馈机制。REEVDF 的核心贡献包括：RIP 在线采样机制，通过测量时间片内的 RIP 推进量估计线程的实际计算效率；双通道动态量子调整架构，乘法通道承担稳态塑形、有符号修正通道承担快速双向响应；老化衰减机制保证系统稳定性；以及完整的 SMP 多核扩展，包括批量工作窃取、主动推送、SIMD 兼容性感知迁移和饥饿保护。

REEVDF 的设计哲学是将调度粒度从墙钟时间推进到实际指令进度，使调度器能够感知并适应不同线程间的计算效率差异。这种推进不需要任何硬件支持，仅通过软件在调度 tick 中增加常数级开销即可实现。在 SkylineSystem 研究型内核中的完整实现[7][8]验证了 REEVDF 的可行性——所有核心机制（RIP 采样、双通道调整、老化、SMP 扩展）均已在 C++ 中实现并集成到调度主循环中。

未来的工作方向包括：第一，在裸机和 QEMU 环境中运行标准基准测试套件，获取 REEVDF 与 Linux EEVDF/CFS 的实证对比数据；第二，探索将 RIP 与硬件 PMC 结合以获得更精确的计算效率度量；第三，在异构多核（大小核）系统中评估 REEVDF 的表现，特别是 SIMD 兼容性感知迁移的有效性；第四，将 REEVDF 的参数暴露为运行时可调接口，并开发自动调优机制。

REEVDF 展示了一种可能性：操作系统调度器可以超越"墙钟时间公平"的传统范式，向"有效工作量公平"演进。随着处理器架构日益复杂（异构计算、推测执行、内存级并行），感知线程实际计算行为的调度器将变得越来越重要。REEVDF 是这一方向上的一次具体探索。

## 参考文献

[1] Stoica I, Abdel-Wahab H. Earliest Eligible Virtual Deadline First: A Flexible and Accurate Mechanism for Proportional Share Resource Allocation[R]. Norfolk: Old Dominion University, 1995.

[2] Linux Kernel Documentation. EEVDF Scheduler[EB/OL]. (2024-2026)[2026-08-31]. https://docs.kernel.org/6.15/_sources/scheduler/sched-eevdf.rst.txt.

[3] Linux Kernel Documentation. Completely Fair Scheduler (CFS)[EB/OL]. (2026)[2026-08-31]. https://www.kernel.org/doc/html/v7.2-rc1/_sources/scheduler/sched-design-CFS.rst.txt.

[4] Goyal P, Vin H M, Chen H. Start-time Fair Queuing: A Scheduling Algorithm for Integrated Services Packet Switching Networks[R]. Austin: University of Texas at Austin, 1996.

[5] LWN.net. Completing the EEVDF scheduler[EB/OL]. (2024-04-11)[2026-08-31]. https://prodcs.lwn.net/Articles/969062/.

[6] Blumofe R D, Leiserson C E. Scheduling multithreaded computations by work stealing[J]. Journal of the ACM, 1999, 46(5): 720-748.

[7] Yo-yo-ooo. SkylineSystem: sched.cpp - Rate-aware EEVDF (REEVDF) Schedule ALGO IMPL[CP/OL]. (2026)[2026-08-31]. https://github.com/Yo-yo-ooo/SkylineSystem/blob/main/kernel/src/arch/x86_64/schedule/sched.cpp.

[8] Yo-yo-ooo. SkylineSystem: sched.h / smp.h[CP/OL]. (2026)[2026-08-31]. https://github.com/Yo-yo-ooo/SkylineSystem/blob/main/kernel/include/arch/x86_64/schedule/sched.h.
