---
title: "CUDA 编程指南——GPU 存储器模型"
date: 2024-03-04
draft: false
author: "Iorest"
description: "GPU 存储器模型"
summary: "列出 GPU 中各类存储器的比较"
topics: ["CUDA"]
tags: ["CUDA", "GPU", "HPC", "AI"]
---

GPU 执行的是一种内存的加载/存储模型（load-storage
model），即所有的操作都要在指令载入寄存器之后才能执行，因此掌握 GPU 的存储器模型对于 GPU 代码性能的优化具有重要意义。

下面简单列出 GPU 中各类存储器比较：

| 存储器 | 位置 | 拥有缓存 | 访问权限 | 变量生命周期 |
| --- | --- | --- | --- | --- |
| register | GPU 片内 | N/A | device 可读/写 | 与 thread 相同 |
| local_memory | 板载显存 | L1/L2 | device 可读/写 | 与 thread 相同 |
| shared_memory | CPU 片内 | N/A | device 可读/写 | 与 block 相同 |
| global_memory | 板载显存 | L1/L2 | device 可读/写，host 可读/写 | 可在程序中保持 |
| constant_memory | 板载显存 | constant cache | device 可读，host 可读/写 | 可在程序中保持 |
| texture_memory | 板载显存 | texture cache | device 可读，host 可读/写 | 可在程序中保持 |
| host_memory | host 内存 | 无 | host 可读/写 | 可在程序中保持 |
| pinned_memory | host 内存 | 无 | host 可读/写 | 可在程序中保持 |

**设备内存**
- Register Files (RF): 寄存器是线程私有的。出于吞吐量的考虑，寄存器被划分成 banks。相比于 CPU 寄存器，GPU 寄存器的 latency 更大，而且有更多潜在的 bank conflicts。如果线程使用超过硬件限制的寄存器，则使用 local memory 来代替多占用的寄存器。

- Local Memory (LM): 线程私有的。local memory 不是物理空间，而是 global memory 的一部分，所以延时较大。它是线程私有的，主要用来临时的 spilling，比如寄存器溢出，或者数组在 Kernel 里声明了，但编译器无法获得他准确的 indexing。Local memory 在 Fremi 和 Kepler 中可以被 L1 和 L2 cached，但是在 Maxwell 和 Pascal 中只能被 L2 cached。寄存器溢出到 local memory 会造成巨大的性能下降（引起了更多的指令和内存拥挤）尤其是 cache miss 时。

- Shared Memory (HM，SMEM): 线程块中所有的线程共有，持续线程块的整个生命周期。又称为 scratchpad memory，是片上的存储，SM 里的所有单元共享。它可以用来作为同一个 thread blocks 里不同线程之间快速数据交换的通讯接口。由于是片上存储，带宽很大，访问延迟很小。因此，把 global/local memory 的访问迁移到 SMEM 上的优化是被编程手册推荐的。为了获得更高的带宽，SMEM 被划分成 banks，这样就可以并行的访问（寄存器文件和 L2 cache 也类似）。但是如果在一次内存请求中，访问的两个地址落在了同一个 bank，就造成了 bank conflict，此时请求需要串行化，从而降低了 SMEM 的性能。

- Global Memory (GM): 所有线程都可访问。又称 device memory，GPU 片下内存，GPU 主存。它是 GPU 性能的主要瓶颈。GM 可达到的吞吐量取决于：(1)GM 物理带宽限制，即 pin 数、线长度和 DRAM 的物理特性。因此在 kelper 以前增长缓慢，但是 Pascal 使用了 3D 栈内存技术，性能获得了巨大提升。(2) 访存请求合并。LSU 一开始会计算每个 warp 的目标地址。在 memory fetch 之前，有一个特殊的地址合并硬件来检查同一个 warp 里地址是否是连续分布的。

- Constant Memory (CM) / Constant Cache: 所有线程可访问且只读。常量内存用来存储在 Kernel 执行期间没有改变的数据。在所有 GPU 上都是 64KB 的 off-chip。和 local memory 类似，都是 Global memory 的一部分。不被 L1 或 L2 cached，有专门的 constant cache。每个 SM 上 8/10KB 的常量 cache 被专门设计，以便单个内存地址的数据可以在同一时间广播给整个经线的所有线程。线程束中的线程每读一次，都会广播给线程束中其它线程。

- Texture Memory (TM) / Texture Cache: 所有线程可访问且只读。又称 surface memory 也在全局内存上。被 texture cache cached。Texture cache 针对 2D 的空间局部性进行了优化。

每个线程都有独立的寄存器和 Local Memory，同一个 block 的所有线程共享一个 Shared Memory；Global Memory、Constantm Memory 和 Texture Memory 是所有线程都可以访问的。Global Memory、Constantm Memory 和 Texture Memory 对程序优化有特殊作用。

**设备缓存**

- L1 Data-Cache：在 Fermi 上首次被提出来。SM 的私有 L1 Cache 和 SMEM 共享片上存储，他们的大小是可以配置的，Fermi 上 16/48 or 48/16，Kepler 上 32/32 or 48/16。L1 cache line 是 128B。L1 data cache 缓存 global memory 读和 local memory 的读写，并且是 non-corherent。local memory 主要用来寄存器溢出、函数调用和自动变量。当 L1 cache 被用来缓存 global memory 时是只读的，当被用来缓存 local memory 时也是可写的。从 Maxwell 开始，传统的 L1 cache 被统一到了 Texture cache 里。

- L2 Cache： 它被用来缓存各种类型的 memory access

- Interconnection Network (NC): crossbar network. 它允许多个 SM 和 L2 banks 之间同时通信。crossbar NC 包含一条地址线和两条数据线。地址线是单向的，从 SM 到 L2 banks，而两条数据线则是双向的，存储 SM 和 L2 banks。因此，通讯是点到点的。一个内存请求队列（memory-request queue, MRQ）和一个 bank 加载队列（ bank load queue, BLQ）分别对应了一个 SM 和 L2 bank。当 load 请求从 SM 中的 LSU 产生时，它会首先缓存在 MRQ 里，然后通过 NC 被分发到目的 BLQ 中。在 BLQ 中等待一会，然后这个请求将被 L2 banks 处理。crossbar network 在同时多个连接时有很高的切换代价，特别是当访问请求随机且数据量大。

**说明**

（1） `Register` 和 `shared_memory` 都是 GPU 片上的高速存储器；

（2）通过 `mapped memory` 实现的 zero
copy 功能，某些功能 GPU 可以直接在 kernel 中访问 page-locked memory。
