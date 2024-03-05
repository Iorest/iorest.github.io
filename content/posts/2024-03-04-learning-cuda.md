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
| local_memory | 板载显存 | 无 | device 可读/写 | 与 thread 相同 |
| shared_memory | CPU 片内 | N/A | device 可读/写 | 与 block 相同 |
| constant_memory | 板载显存 | 有 | device 可读，host 可读/写 | 可在程序中保持 |
| texture_memory | 板载显存 | 有 | device 可读，host 可读/写 | 可在程序中保持 |
| global_memory | 板载显存 | 无 | device 可读/写，host 可读/写 | 可在程序中保持 |
| host_memory | host 内存 | 无 | host 可读/写 | 可在程序中保持 |
| pinned_memory | host 内存 | 无 | host 可读/写 | 可在程序中保持 |

**说明**

（1） `Register` 和 `shared_memory` 都是 GPU 片上的高速存储器；

（2）通过 `mapped memory` 实现的 zero
copy 功能，某些功能 GPU 可以直接在 kernel 中访问 page-locked memory。