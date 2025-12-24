---
title: Higress(01)——使用Higress作为LLM推理的接入层网关
date: 2025-06-15 16:21:18
tags:
  - Cloud Native for AI
  - Higress
  - LLM Infer System
categories:
  - 技术
  - Gateway
---

# 前言


# 踩坑记录
## 1. 长文本压测，部分请求处于等待队列，3min后中断
检查higress-gateway日志，发现报错信息`"response_code_detail": "stream_idle_timeout"`，官方文档没找到相关说明，翻了下各个config，在higress-config配置文件中找到相关配置`data.higress.downstream.idleTimeout=180`，决定了下游的闲置超时时间，修改该配置后问题解决

# 版权声明

> 本博客所有原创内容，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 协议，转载请注明出处。
