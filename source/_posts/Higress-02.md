---
title: Higress(02)——基于文件实现Higress的AI路由配置
date: 2025-06-15 16:55:28
tags:
  - Cloud Native for AI
  - Higress
  - LLM Infer System
categories:
  - 技术
  - Gateway
---

# 前言
Higress更多是通过控制台进行路由规则配置，不过在项目开发过程中，为了完成推理服务的部署与上线的全自动化流程，我们需要通过远程调用的方式实现网关配置，经过管理台配置与k8s中ConfigMap、McpBridge、Wasm配置的比对，产生了这个实践经验。

我们是使用AI网关的能力，使用了AI服务路由、认证管理的能力，所以涉及的配置有：域名配置、AI服务提供者配置、AI消费者配置、AI路由规则配置。

模拟一个场景：
- 假设我们的LLM服务提供者的访问域名为https://lololo.com/v1，使用openai/v1的协议
- 假设我们希望配置消费者为labubu，使用key auth校验
- 假设我们希望对域名http://zimomo.com/v1的访问，都可以转发给https://lololo.com/v1这个服务提供商
- 假设我们期望创建的路由规则名字为：zimomo2lololo

OK，Let's GO!

# 域名配置
ConfigMap增加域名配置domain-zimomo.com，注意需要填写的内容metadata.name，namespace，data.domain，data.enableHttps
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: domain-zimomo.com
  namespace: higress-system
  labels:
    higress.io/config-map-type: domain
    higress.io/resource-definer: higress
data:
  domain: zimomo.com
  enableHttps: "off"
```

# AI服务提供者配置
1.配置networking.higress.io/v1/McpBridge，追加spec.registries
```yaml
apiVersion: networking.higress.io/v1
kind: McpBridge
metadata:
  name: default
  namespace: higress-system
spec:
  registries:
  - domain: lololo.com
    name: llm-lololo.internal
    port: 443
    protocol: https
    type: dns
```

2.配置ai-proxy.internal的extensions.higress.io/v1/alpha1/WasmPlugin，追加defaultConfig.providers，matchRules.config
```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: ai-proxy.internal
  namespace: higress-system
  labels:
    higress.io/internal: "true"
    higress.io/resource-definer: higress
    higress.io/wasm-plugin-built-in: "true"
    higress.io/wasm-plugin-category: ai
    higress.io/wasm-plugin-name: ai-proxy
    higress.io/wasm-plugin-version: 1.0.0
  annotations:
    higress.io/comment: PLEASE DO NOT EDIT DIRECTLY. This resource is managed by Higress.
    higress.io/wasm-plugin-description: Provide unified OpenAI API compatible interface to call different AI service providers.
    higress.io/wasm-plugin-icon: https://img.alicdn.com/imgextra/i1/O1CN018iKKih1iVx287RltL_!!6000000004419-2-tps-42-42.png
    higress.io/wasm-plugin-title: AI Proxy
spec:
  defaultConfig:
    providers:
    - failover:
        enabled: false
      id: lololo
      openaiCustomUrl: https://lololo.com/v1
      retryOnFailure:
        enabled: false
      type: openai
  defaultConfigDisable: false
  failStrategy: FAIL_OPEN
  imagePullPolicy: UNSPECIFIED_POLICY
  matchRules:
  - config:
      activeProviderId: lololo
    configDisable: false
    service:
    - llm-lololo.internal.dns
  phase: UNSPECIFIED_PHASE
  priority: 100
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/ai-proxy:1.0.0
```

# AI服务消费者配置
1.如果涉及认证，则需要配置AI服务消费者，还需要配置key-auth.internal的extensions.higress.io/v1/alpha1/WasmPlugin，这里追加defaultConfig.consumers，配置认证信息。这个配置中还存在matchRules，可以在后文看到。
```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: key-auth.internal
  namespace: higress-system
  labels:
    higress.io/internal: "true"
    higress.io/resource-definer: higress
    higress.io/wasm-plugin-built-in: "true"
    higress.io/wasm-plugin-category: auth
    higress.io/wasm-plugin-name: key-auth
    higress.io/wasm-plugin-version: 1.0.0
  annotations:
    higress.io/comment: PLEASE DO NOT EDIT DIRECTLY. This resource is managed by Higress.
    higress.io/wasm-plugin-description: Authentication based on API Key.
    higress.io/wasm-plugin-icon: https://img.alicdn.com/imgextra/i4/O1CN01BPFGlT1pGZ2VDLgaH_!!6000000005333-2-tps-42-42.png
    higress.io/wasm-plugin-title: Key Auth
spec:
  defaultConfig:
    consumers:
    - credentials:
      - Bearer 123456-xxx
      in_header: true
      in_query: false
      keys:
      - Authorization
      name: labubu
    global_auth: false
    keys:
    - x-higress-dummy-key
  defaultConfigDisable: false
  failStrategy: FAIL_OPEN
  imagePullPolicy: UNSPECIFIED_POLICY
  matchRules:
  - config:
      allow:
      - labubu
    configDisable: false
    ingress:
    - ai-route-zimomo2lololo.internal
  phase: AUTHN
  priority: 310
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/key-auth:1.0.0
```

# AI路由规则配置
1.创建Ingress规则
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-route-zimomo2lololo.internal
  namespace: higress-system
  labels:
    higress.io/domain_lololo.com: "true" # 注意这里
    higress.io/internal: "true"
    higress.io/resource-definer: higress
  annotations:
    higress.io/comment: PLEASE DO NOT EDIT DIRECTLY. This resource is managed by Higress.
    higress.io/destination: llm-lololo-provider.internal.dns:30000  # 注意这里
spec:
  ingressClassName: higress
  rules:
  - host: lololo.com
    http:
      paths:
      - backend:
          resource:
            apiGroup: networking.higress.io
            kind: McpBridge
            name: default
        path: /v1
        pathType: Prefix
```

2.ConfigMap增加详细规则配置，ai-route-lololo
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-route-lololo
  namespace: higress-system
  labels:
    higress.io/config-map-type: ai-route
    higress.io/resource-definer: higress
data:
  data: '{"name":"lololo","version":"14260326171","domains":["lololo.com"],"pathPredicate":{"matchType":"PRE","matchValue":"/v1"},"headerPredicates":[],"urlParamPredicates":[],"upstreams":[{"provider":"lololo-provider","weight":100,"modelMapping":{}}],"modelPredicates":[],"authConfig":{"enabled":true,"allowedConsumers":["labubu"]},"fallbackConfig":{"enabled":false}}'
```

3.如果涉及认证，还需要配置key-auth.internal的extensions.higress.io/v1/alpha1/WasmPlugin，这里追加matchRules，声明规则支持的消费者
```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: key-auth.internal
  namespace: higress-system
  labels:
    higress.io/internal: "true"
    higress.io/resource-definer: higress
    higress.io/wasm-plugin-built-in: "true"
    higress.io/wasm-plugin-category: auth
    higress.io/wasm-plugin-name: key-auth
    higress.io/wasm-plugin-version: 1.0.0
  annotations:
    higress.io/comment: PLEASE DO NOT EDIT DIRECTLY. This resource is managed by Higress.
    higress.io/wasm-plugin-description: Authentication based on API Key.
    higress.io/wasm-plugin-icon: https://img.alicdn.com/imgextra/i4/O1CN01BPFGlT1pGZ2VDLgaH_!!6000000005333-2-tps-42-42.png
    higress.io/wasm-plugin-title: Key Auth
spec:
  defaultConfig:
    consumers:
    - credentials:
      - Bearer 123456-xxx
      in_header: true
      in_query: false
      keys:
      - Authorization
      name: labubu
    global_auth: false
    keys:
    - x-higress-dummy-key
  defaultConfigDisable: false
  failStrategy: FAIL_OPEN
  imagePullPolicy: UNSPECIFIED_POLICY
  matchRules:
  - config:
      allow:
      - labubu
    configDisable: false
    ingress:
    - ai-route-zimomo2lololo.internal
  phase: AUTHN
  priority: 310
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/key-auth:1.0.0
```

# 版权声明

> 本博客所有原创内容，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 协议，转载请注明出处。
