---
title: "[Note] Attention Optimization"
date: 2025-01-15T20:49:58+08:00
type: post
# draft: true
tags: ["Attention", "Note"]
showTableOfContents: true
---

## <a href="https://zhuanlan.zhihu.com/p/672698614" class="custom-link">FlashAttention</a>

由于Transformer的计算复杂度和空间复杂度随序列长度N呈二次方增长，其难以处理长token。

flashAtention通过利用更高速的上层存储计算单元，减少对低速更下层存储器HBM的访问次数，来提升模型的内存访问(I/0)，普通的attention其内存访问量是$O(N^2)$, 而FlashAttention是$O(N)$，注意，模型中GEMM计算FLOPS不变，Softmax计算FLOPS增加。

GPU的存储架构类似CPU，即内存越快，越昂贵，容量越小。以A100为例，其有40-80GB的高带宽内存(HBM，它是由多个DRAM堆叠出来的)，带宽为1.5-2.0 TB/s，而每108个流处理器(SM)有192KB的SRAM，带宽约19TB/s。

<img src="../../../../images/LLM/transformerop1.png" alt="" width="105%">

#### 传统attention的计算访存

- 首先，read(Q), read(K) <span style="color:#9d9e8d;"> n x d</span>，computer(S) <span style="color:#9d9e8d;"> n x n</span>，write(S)，需要进行$O(2nd + n^2)=O(n^2)$次HBM访问。
- read(S), <span style="color:#9d9e8d;"> n x n</span>，computer(P=softmax(x))，write(P)，需要进行$O(2n^2)=O(n^2)$次HBM访问。
- 最后，read(P), read(V)，<span style="color:#9d9e8d;"> n x n 和 n x d</span>, computer(O=PV)，write(O)，需要进行$O(n^2+2nd)$次HBM访问。

总的来说，传统Attention的总HBM访问次数为$O(n^2)$。当N比较大时，总的HBM访问次数可能比较昂贵。

<img src="../../../../images/LLM/transformerop2.png" alt="" width="85%">

#### 缺陷
标准Attention算法在GPU内存分级存储的架构下，存在以下缺陷：

1. 过多对HBM的访问，数据在存入HMB后又立即被访问，HBM带宽较低，导致算法性能受限 
2. S,P需要占用O(N^2)的存储空间，显存占用较高

#### 改进
1. kernel fusion减少部分的访问HBM的次数
2. 在计算过程中要尽量的利用SRAM进行计算，避免访问HBM操作

难点：虽然SRAM的带宽较大，但其计算可存储的数据量较小。如果我们采取“分治”的策略将数据进行Tilling处理，放进SRAM中进行计算，由于SRAM较小，当sequence length较大时，sequence会被截断，从而导致标准的SoftMax无法正常工作。

#### 关键技术
1. Tiling（在前向和后向传递中使用）- 简单讲就是将softmax/分数矩阵划分成适合存储在SRAM上的小块。
2. Recomputation重新计算（仅在后向传播） - 不存储梯度和每一层的正向传播的中间状态，而是在计算到反向某一层的时候再临时从头开始重算正向传播的中间状态，从而减小内存占用。

详细见:
- https://zhuanlan.zhihu.com/p/672698614
- https://fancyerii.github.io/2023/10/23/flashattention/
- https://llmsystem.github.io/llmsystem2024spring/assets/files/Group-FlashAttention-0b70d553037a7729dd2a9af5e23d8b3e.pdf

