# klsys-api 项目简述

## 一、物理部署分布

集群由 3 台 NUC 组成，对外统一通过 `klsysapi` 暴露服务。

| 节点 | 型号 | 部署组件 | 对外角色 |
| --- | --- | --- | --- |
| 主节点 | NUC11TNKi7 | llama.cpp-hub + new-api（+ ccswitch） | 统一网关出口，承载 `klsysapi` 流量 |
| 从节点 1 | NUC | 仅 llama.cpp-hub | 推理算力节点 |
| 从节点 2 | NUC | 仅 llama.cpp-hub | 推理算力节点 |

- 两台从节点的 llama.cpp-hub 作为后端渠道注册到主节点的 new-api。
- 外部互联网用户（用户 1）只通过主节点的 `klsysapi` 入口访问，不直连任何 llama 节点。
- 组件路径：
  - llama.cpp-hub：`/home/lab/Large_Language_Model/llama.cpp-hub-linux`（Web 端口 `8080`，多协议 API 兼容 OpenAI/Anthropic/Ollama/LM Studio）
  - new-api：`/home/lab/klsys/new-api`（服务端口 `3000`）

## 二、请求链路

互联网用户发起的请求按以下顺序流转：

```
Internet User
     │
     ▼
klsysapi（主 NUC 对外入口）
     │
     ▼
new-api（主 NUC：鉴权 / 配额 / 路由 / 日志）
     │
     ├──► llama.cpp-hub（主 NUC 本机） ──► llama.cpp（本地模型推理）
     ├──► llama.cpp-hub（从 NUC-1）   ──► llama.cpp
     └──► llama.cpp-hub（从 NUC-2）   ──► llama.cpp
     │
     ▼
ccswitch（上游分发 / 协议切换）
     │
     ├──► ChatGPT（OpenAI）
     └──► Claude（Anthropic）
```

说明：
- **llama → llama.cpp-hub**：每台 NUC 本地运行 llama.cpp，由 llama.cpp-hub 做模型加载、参数管理与多协议 API 封装。
- **llama.cpp-hub → new-api**：三台 hub 作为后端渠道注册到 new-api，由 new-api 统一调度到任意一台。
- **new-api → ccswitch**：需要走外部商业模型时，new-api 将请求转发给 ccswitch，由其根据渠道策略切换到 ChatGPT 或 Claude。
- **ccswitch → ChatGPT / Claude**：ccswitch 完成 OpenAI/Anthropic 协议的最终对接与密钥管理。

## 三、组件职责

- **llama.cpp-hub**：本地模型（GGUF）的加载、卸载、参数预设与多协议 API 暴露，为集群贡献自建推理能力。
- **new-api**：AI 网关与资产管理（用户、Token、渠道、配额、用量统计、日志），对外统一入口。
- **ccswitch**：面向外部商业模型的转发与协议切换层，负责上游账号/Key 的分发与容灾。
- **klsysapi**：对外发布的公网入口标识，流量全部终结于主 NUC 的 new-api。
