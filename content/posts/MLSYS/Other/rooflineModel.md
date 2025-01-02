---
title: "[Note] Roofline Model"
date: 2024-12-31
type: post
# draft: true
tags: ["Roofline", "Hardware", "Note"]
showTableOfContents: true
---
<a href="https://zhuanlan.zhihu.com/p/34204282" class="custom-link">Roofline Model</a>

核心思想：Roofline模型通过分析一个任务的“操作强度”来判断其性能受限于内存带宽还是计算能力<br>
操作强度：计算任务中每访问1字节数据所能完成的浮点运算次数（FLOPs/Bytes）<br>
性能限制：
- 内存限制：如果数据访问速度（内存带宽）低于计算所需速度，性能受限于内存
- 计算限制：如果数据访问足够快，但处理器的计算能力有限，性能受限于计算能力

Use for measuring the theoretical performance upper bound 𝑃 of model x can achieve on a computing platform

- [Platform] Computility &pi; : maximum FLOPS per second.<br><p style="line-height: 0.12em;"></p>

- [Platform] Bandwidth &beta; : maximum memory access per second.<br><p style="line-height: 0.12em;"></p>

- [Platform] Computational intensity *I_max* = &pi; / &beta; <span style="color:#9d9e8d;"> 计算任务中每访问1字节数据所能完成的浮点运算次数<span><br><p style="line-height: 0.12em;"></p>
- [Model] Computational workload 𝐴 : <br>the number of floating-point operations (#FLOPs) that occur during a single forward pass when processing one input sample (for a CNN, this would be a single image).<br><p style="line-height: 0.12em;"></p>
- [Model] Memory access 𝐵 : <br>the total amount of memory (#Bytes) exchanged during a single forward pass when processing one input sample. In the ideal case, B = model's weight parameters + memory used for each layer's output.<br><p style="line-height: 0.12em;"></p>
- [Model] Computational intensity *I*=𝐴/𝐵 : <br> the number of floating-point operations performed per byte of memory exchanged during the computation (#FLOPs/Byte). <span style="color:#9d9e8d;">The higher the computational intensity, the more efficiently the model utilizes memory, as it performs more computations for each unit of memory accessed.</span> <br><p style="line-height: 0.12em;"></p>

- [Model] theoretical peak performance 𝑃 : <br>
theoretical maximum number of floating-point operations it can achieve per second (#FLOPs) on a given computing platform. 

<img src="../../../../images/LLM/roofline1.png" alt="" width="65%">
<img src="../../../../images/LLM/roofline2.png" alt="" width="45%">


Memory Roof: Intuitively, if the data needed for the computation is supplied slower than the computation itself, the processor will idly wait for data, making memory bandwidth the primary bottleneck. 

### Example
I(VGG16)= 25 FLOPs/Byte<br>
I(MobileNet)= 7 FLOPs/Byte
<br><p style="line-height: 0.12em;"></p>
Platform — 1080Ti : &pi;=11.3 TFLOP/s, &beta;=484GB/s
<br><p style="line-height: 0.12em;"></p>
<img src="../../../../images/LLM/roofline3.png" alt="" width="65%">
