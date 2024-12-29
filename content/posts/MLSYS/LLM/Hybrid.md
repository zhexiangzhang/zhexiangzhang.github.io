---
title: "Hybrid Heterogeneous Clusters Can Lower the Energy Consumption of LLM Inference Workloads"
date: 2024-12-29T13:58:55+08:00
type: post
# draft: true
tags: ["LLM", "Inference System", "Heterogeneous"]
showTableOfContents: true
---

---
# Hybrid Heterogeneous Clusters Can Lower the Energy Consumption of LLM Inference Workloads


> arXiv \
> Grant Wilkins, Richard Mortier \
> University of Cambridge

## Background

LLMs require significant computational resources, making energy efficiency a critical challenge for data centers. Traditional hardware setups struggle to balance energy consumption and performance, especially when handling diverse LLM workloads.

## Problem Formulation
The objective is to minimize the overall cost of running tasks across all systems in the data center, balancing energy consumption and runtime. The cost function is defined as:
U(m,n,s)=λE(m,n,s)+(1−λ)R(m,n,s)
where:

* 𝑚 and 𝑛 denote the number of input and output tokens, respectively
* 𝐸 (𝑚, 𝑛, 𝑠) : the energy consumed by system 𝑠 to process 𝑚 input tokens and generate 𝑛 output tokens
* 𝑅(𝑚, 𝑛, 𝑠) : time required to process these tokens on system 𝑠. 
* 𝜆 : a tunable parameter that determines the balance between energy consumption and runtime.


The optimization objective is to minimize the total cost across all tasks and systems:
<br><img src="../../../../images/LLM/Hybrid4.png" alt="" width="25%">


## Experiment
Hardware: M1-Pro CPU vs GPU

* Vary Input Tokens. input : 8-2048 tokens, output=32.
* Vary Output Tokens. output : 8-4096 tokens, input=32.

<br><img src="../../../../images/LLM/Hybrid1.png" alt="" width="95%">

## Energy-optimal solution
Intuition: energy expended per token for the M1 Pro < A100 up to a certain point in the number of input and output tokens (Figure 1,2). So there is an energy-optimal way to construct a hybrid datacenter with a combination of them.

#### Number of Input Tokens Analysis
Find a cutoff threshold, 𝑇_𝑖𝑛, for input token length.
Queries with 𝑛 ≤ 𝑇_𝑖𝑛 tokens are processed on M1 Pro (good energy efficiency) 
Queries with 𝑛 > 𝑇_𝑖𝑛 tokens are processed on A100 GPUs (greater energy-per-token advantages)

To find an optimal threshold 𝑇_𝑖𝑛 empirically, this paper analyze the token distribution in prompts from the Alpaca dataset and visualized the input and output tokens.
<br><img src="../../../../images/LLM/Hybrid2.png" alt="" width="50%">

The energy component of cost function E(m,n,s):
<br><br><img src="../../../../images/LLM/Hybrid5.png" alt="" width="50%">

Energy and runtime simulation results of performing inference for the input token sizes from the Alpaca dataset.
<br><img src="../../../../images/LLM/Hybrid3.png" alt="" width="40%">
<span style="color:DarkTurquoise;">Threshold of 32 tokens strikes an optimal balance for reducing energy consumption.</span>


#### Number of Output Tokens Analysis
The output threshold is found in a similar way
<br><img src="../../../../images/LLM/Hybrid6.png" alt="" width="40%">


