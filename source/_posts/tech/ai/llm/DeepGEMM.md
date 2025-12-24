---
title: 模型启动加速篇——DeepGEMM预编译缓存
date: 2025-12-24 16:02:55
tags:
  - ML System
  - LLM Infer
  - SGLang
categories:
  - 技术
  - AI
  - LLM
---

# 概览
随着模型越来越大，模型启动耗时越来越久，在一些紧急扩容场景、推理部署调试场景，受制于模型启动耗时，影响效率甚至影响收入，所以模型启动加速也是一个重要的课题。

这一篇分享下如何通过DeepGEMM预编译缓存来实现启动加速。

DeepGEMM(General Matrix Multiplication)，是DeepSeek研发团队专为DeepSeek优化的高性能矩阵乘法库，特别针对FP8精度做了深度优化。

DeepGEMM kernel不是提前编译好的固定程序，会根据实际运行的模型尺寸（包括模型结构、并行策略、精度等）动态生成。

所以默认在模型启动时需要对DeepGEMM进行JIT，而我们对于同一个模型和启动参数，自然可以缓存下来DeepGEMM编译后的内容，用于下一次启动加速。

# 启动耗时
以DeepSeek-V3.1-Terminus为例，我们在一次启动耗时可能达到8min多。其中DeepGEMM JIT达到了6min多。

| 阶段 | 耗时 | 说明 |
|------|------|------|
| 服务器初始化 + 分布式通信 | ~12s | 配置加载、8 GPU worker 初始化 |
| 主模型权重加载 | ~53s | DeepseekV3ForCausalLM |
| CUDA Graph 捕获 + DeepGEMM JIT 编译 | **~6min 17s** | 主要耗时，包含7次 JIT 编译 |
| Draft 模型加载 (EAGLE) | ~2s | 投机解码模型 |
| Draft CUDA Graph 捕获 | ~10s | Draft decode + extend graph |
| Host Memory + Mooncake 初始化 | ~14s | |
| Warmup 首次推理 | ~35s | |
| **总计** | **~8min 32s** | - |

{% asset_img img-01.jpg "优化前" %}

# 优化方案
1. 首先进行一次配置了`export SGLANG_ENABLE_JIT_DEEPGEMM=1`的服务启动，启动之后可以在`~/.cahce/`下看到`deep_gemm`的目录，里面包含cache和tmp缓存文件。这就是DeepGEMM JIT生成的kernel缓存，这是我们优化的关键。
2. 推理服务配置`export SGLANG_DG_CACHE_DIR=/dev/shm/deep_gemm`，表示指定DeepGEMM使用该目录作为缓存目录，在JIT时发现里面有缓存便会直接使用了。
3. 以我们的方案为例，我们将打包的deep_gemm缓存放在对象存储或者其他文件系统，在启动前的预置脚本中将deep_gemm拷贝到我们的目标目录，以当前为例即/dev/shm下并解压即可。

之后就可以看到启动过程中对于DeepGEMM的JIT一闪而过，从8分多 优化至 10几秒。

{% asset_img img-02.jpg "优化后" %}

# 版权声明

> 本博客所有原创内容，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 协议，转载请注明出处。