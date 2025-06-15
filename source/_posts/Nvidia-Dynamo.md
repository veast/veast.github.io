---
title: Nvidia Dynamo, 高效的LLM分布式推理框架
date: 2025-06-02 19:41:20
tags:
  - ML System
  - LLM Infer
categories:
  - 技术
  - AI
  - LLM
---

【本文持续更新中..】内容目前基于2025.05及之前的内容整理总结，请注意时效性。

# 概览

先了解一下dynamo是什么东西，到官网看一下官方介绍，可以看到三个dynamo相关的概念，我们逐个看下介绍：


1. NVIDIA Dynamo Platform：关键字已经标出来了，是一个"推理平台"，支持"任何框架、架构、部署规模"的模型。
{% asset_img img-01.png "Dynamo Platform" %}


2. NVIDIA Dynamo：继续看关键字，"推理框架"，"分布式环境"场景支持，支持"所有主流推理后端"，支持分离部署。  
{% asset_img img-02.png "Dynamo Platform" %}


3. NVIDIA Dynamo-Triton：Dynamo-Triton看起来暂时没有什么亮点，似乎只是直接把NVIDIA之前的Triton改了个名字。 
{% asset_img img-03.png "Dynamo Platform" %}


然后官网也给了一张图，可以直观的看到上述三个产品的定位与关系：
{% asset_img img-04.png "Dynamo Platform" %}


题外话…对比dynamo在nvidia大陆和官方的文档，发现目前在大陆的介绍页，只有dynamo框架，没有dynamo平台介绍。
https://developer.nvidia.com/dynamo，https://developer.nvidia.cn/dynamo

再到github看下又说了什么，先看个大概，主要是NVIDIA Dynamo框架的内容：关键信息是这个"推理引擎无关"。
{% asset_img img-05.png "Dynamo Platform" %}
 

然后接下来是列举了Dynamo的几个主要特性：包含支持PD分离、LLM感知请求路由等，关键特性后文我们逐一展开。
{% asset_img img-06.png "Dynamo Platform" %}
 

## 为什么需要Dynamo

Nsight cannot serve or orchestrate models (coordination and mgmt of AI models, systems, and integrations).

Triton is for general AI inference serving - it's not optimized for distributed LLM workloads.

TensorRT-LLM, vLLM - For "Engine-level LLM inference acceleration" (basically optimizing core software/hardware/architecture-like using faster kernels, better memory handling, parallelization in the inference engine itself). So it's not a fully distributed serving/orchestration.

Dynamo on the other hand, is for Distributed, multi-node, LLM-optimized inference. It's designed for new scale and efficiency demands.

## Dynamo提供了什么能力

{% asset_img img-07.png "Dynamo Platform" %}

GPU Resource Planner: A planning and scheduling engine that monitors capacity and prefill activity in multi-node deployments to adjust GPU resources and allocate them across prefill and decode.

Smart Router: A KV-cache-aware routing engine that efficiently directs incoming traffic across large GPU fleets in multi-node deployments to minimize costly re-computations.

Low Latency Communication Library: State-of-the-art inference data transfer library that accelerates the transfer of KV cache between GPUs and across heterogeneous memory and storage types.

KV Cache Manager: A cost-aware KV cache offloading engine designed to transfer KV cache across various memory hierarchies, freeing up valuable GPU memory while maintaining user experience.

# Dynamo详解

https://github.com/ai-dynamo/dynamo/tree/main/docs/architecture

架构文档

## 关键要素


前端（Frontend）：采用 OpenAI 类似 http 请求服务，响应用户请求

路由（Router）：根据一定的策略，把前端请求分配到不同 worker

处理单元（Workers）： 有 prefill/decode 两种 worker 处理 LLM 计算逻辑


# 实践

https://developer.nvidia.com/blog/nvidia-dynamo-adds-gpu-autoscaling-kubernetes-automation-and-networking-optimizations/

Deploying Dynamo Inference Graphs to Kubernetes using Helm：https://github.com/ai-dynamo/dynamo/blob/release/0.2.0/docs/guides/dynamo_deploy/manual_helm_deployment.md

# 参考链接

https://developer.nvidia.com/dynamo

https://github.com/ai-dynamo/dynamo#

https://github.com/ramanahm1/dynamo-learnings

https://cloud.tencent.com/developer/article/2515122?policyId=20240001&frompage=seopage

https://mp.weixin.qq.com/s/t9rm_rG2NwXaZLe_SF5_hg


# 版权声明

> 本博客所有原创内容，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 协议，转载请注明出处。
