---
tags:
  - Agent
  - MCP
  - CLI
  - 工具调用
  - 环境化Agent
source: https://chatgpt.com/share/6a97eaad-1a44-83e8-9d5e-5257de653d8e
topic: 领域 Agent 抛弃 MCP 转 CLI —— 从 Tool-centric 到 Environment-centric
---

# 领域 Agent 抛弃 MCP、转用 CLI 的观察与思考（AgentThought）

> 与 GPT 对话整理的要点。话题起点：实习中发现 AIOps 等**领域特化 Agent** 正逐渐抛弃 MCP 工具协议，把 MCP 工具换成对应 CLI 工具（认为更高效、更省 Token）。对话两轮：第一轮 GPT 系统分析现象；第二轮结合提问者团队真实做过「动态加载 MCP」的实践，修正与深化结论，并最终接回 Hermes 子代理编排。

## 一句话概述

> **不是「领域 Agent 抛弃 MCP」，而是「领域 Agent 抛弃 MCP 作为 LLM 的直接工具调用接口」，把 CLI / Shell 作为更靠近 Agent 的工具抽象层。** 背后是 Agent 架构的重要变化：从 **Tool-centric Agent 走向 Environment-centric Agent**。

---

## 1. 为什么 CLI 对领域 Agent 特别有吸引力

### 1.1 工具 schema 本身就是上下文负担（MCP 的真正痛点）

- AIOps 场景工具极多：kubectl / helm / argo / prometheus / grafana / aws / cloudwatch……全做成 MCP 会导致工具空间迅速膨胀（几十上百个 `k8s_get_pod`、`k8s_get_logs` 这类离散 tool）
- MCP 的问题不是 RPC 慢，而是**工具「描述」本身就是 Agent 上下文的一部分**：每个 tool 需要 name + description + inputSchema（JSON）
- 实测量级：**单个复杂 MCP tool 的 schema 可到约 1000 tokens；大量 MCP 工具累积可达数万甚至十万级 token**

### 1.2 CLI 如何绕过：粗粒度 + 环境原生

- Agent 不需要逐个声明 `k8s_get_pod_logs(namespace, pod, container...)`，只需一个粗粒度能力 `shell(command)`，然后直接写 `kubectl logs payment-api -n production`
- 链路由长变短：
  - MCP：Agent → Tool selection → tool schema → JSON arguments → MCP protocol → execution
  - CLI：Agent → shell → kubectl/aws/curl/jq/grep/awk → execution

> **真正省 Token 的核心不是 CLI 协议字节更少，而是 CLI 把「工具目录」和「工具 schema」从模型上下文中移走了。**

## 2. AIOps 天然是 CLI-native

- AIOps 工具本身（kubectl、docker、helm、terraform、aws、gcloud、curl、jq、promtool、istioctl……）已经是一套成熟的 **Infrastructure Control Plane**
- Kubernetes/AWS/Prometheus 已提供统一动词（`kubectl get/describe/logs/exec`、`aws ec2/cloudwatch/eks`、`promtool`）
- 再把它们包装成几十上百个 MCP tools，反而是一层额外的 **Agent abstraction tax**

## 3. CLI 的 composability（相对 MCP 最大优势）

- CLI 是**可组合的 primitive**（`kubectl get pods -A -o json | jq 'select(.status.phase=="CrashLoopBackOff")'`），MCP tool 是**已定义好的离散 function**，组合粒度更粗
- 接近 Unix 设计哲学：**Small tools + composability**；Agent 恰好擅长这种动态组合
- MCP 里 Agent 在离散 actions 间选择；CLI 里 Agent 自己 composition

## 4. 但 MCP 不会消失：两者解决不同层次的问题

```
              Agent
         CLI/Shell         MCP
    Local Environment  Remote Services
    kubectl aws curl   DB / SaaS / API
```

- **CLI 适合**：Agent 已经处于「有环境」的地方（Coding / AIOps / DevOps / Security / Terminal Agent）
- **MCP 适合**：Agent 需要连接外部 capability provider——GitHub、Slack、Notion、Jira、数据库、企业内部系统、第三方 SaaS（不天然有好用 CLI 的东西）
- 未来更可能是 **CLI + MCP 并存**，上层加一层 **Capability Router**（Domain Agent 下分 CLI→Infrastructure、MCP→SaaS/API）

## 5. MCP 自身也在演进（它也在解决 schema 太重的问题）

- 2026 年 7 月 MCP 新规范：引入**可缓存的 `tools/list` 结果、discovery、stateless core** 等，本质是降低 tool catalog 与 session/context 成本
- 问题不是「MCP 不好」，而是「**把几百个 MCP tools 永久常驻暴露给 Agent** 这个架构不好」
- 正确方向：`Tool discovery → 找到需要 tool → 加载 schema → 调用`（**Capability on demand**），而非「加载全部 schema 再从几百个里选」

## 6. 更深一层：Tool Calling → Capability Discovery

- 传统：Prompt → Tool List（常驻）→ LLM selects tool → Tool Call
- 下一代：Prompt → Capability Discovery → 找到相关能力 → 加载最小必要 interface → 执行 → 观察 → 继续 discovery/execution
- CLI 只是实现 Capability Discovery 的一种**极高效**方式

## 7. AIOps 的工作流本身契合 CLI

- AIOps 是**探索式工作流**：Observe → Hypothesize → Execute → Observe → Narrow down → Execute
- 例：CPU 高 → `kubectl top pod` → `describe`（OOM/restart）→ `logs`（DB timeout）→ curl/prometheus 定位 DB
- 把它切成 `get_pod_cpu() / get_pod_description() / get_logs()` 反而**把运维世界强行切成一堆函数**；CLI 保留环境原生表达能力

## 8. CLI 的软肋：安全边界（MCP 难替代）

- Expressiveness ↑ ⇒ Risk ↑（`rm -rf /`、`kubectl delete namespace`、`aws s3 rm --recursive`、`terraform destroy`）
- MCP tool 可把权限拆到很细（get→read、restart→write、delete→destructive），并有 tool annotations（`readOnly`/`destructive`/`idempotent`——注意：规范明确是 **hint，不是安全保证**）
- 企业级 Agent 大概率是 CLI 前面套 **Policy / Guardrail**：Command Allowlist、RBAC、Sandbox、Approval、Audit、Policy Engine

---

## 9. 第二轮校准（结合提问者团队「动态加载 MCP」的实践）

提问者反馈：MCP schema 大量占用上下文确是缺陷；但团队做过**动态加载 MCP**（精简加载所有 MCP 的功能描述，Agent 选中后再全量加载该 MCP 的 Schema）——这样做比 CLI 在 Token 效率上孰优？并质疑：CLI 能访问外部服务吗（实习中通过 CLI + 沙箱访问过其他团队服务）？CLI 的安全限制是否可在开发时做好？

### 9.1 结论修正

| 比较对象 | Token 结论 |
|---|---|
| 全量 MCP Schema vs CLI | CLI 优势通常非常明显 |
| **Dynamic MCP**（轻量描述→选中→加载 schema）vs CLI | **差距大幅缩小，甚至某些场景 MCP 更优** |

- 团队最终仍选 CLI，决定性因素很可能**不是 Token**，而是：**CLI 提供通用、可组合、可探索的 action space；MCP 提供的是离散的 function space**

### 9.2 CLI 到底怎么被 Agent 理解？（三个层次）

- **Level 1 纯 Shell**：只给 `shell(command)`，Agent 自己 `--help` 探索——自由，但 Token 和步骤数不漂亮
- **Level 2 提供 CLI 文档 / Skill**：环境里放 `/skills/kubernetes/SKILL.md`，轻量说明常用操作，需要时再 `kubectl logs --help`——**这已非常接近 Dynamic MCP**
- **Level 3 CLI 结构化发现**：CLI 提供 `agentctl capabilities`（返回命令 JSON 清单）+ `agentctl describe <cmd>`（按需取详细参数）——**本质就是 MCP Dynamic Tool Discovery 的 CLI 版本**，架构思想上无本质区别

### 9.3 Dynamic MCP 的 Token 数学

- 全量：100 个 MCP × 500 tokens/schema = **50,000**
- 动态：100 × 30-token 描述 = 3,000 → 选中 #17 → +500 schema = **3,500**
- 即从 `O(N × Schema)` 降为 `O(N × Description + Selected Schema)`

### 9.4 Dynamic MCP vs CLI 谁更省 Token（拆解公式）

`Token Cost = Capability Discovery + Action Specification + Arguments + Observation + Iterative Reasoning`

- Dynamic MCP 一次调用 ≈ 3,000(摘要) + 500(schema) + 50(args) + 300(result) = **~3,850**
- CLI + 反复 `--help` ≈ 1,000 + 800 + 300 = **2,100**；Agent 已知 CLI 时直接执行 ≈ **300**
- **真正拉开差距的是：Agent 是否「已经知道」CLI**
  - 通用 Agent 拿到 kubectl/helm/aws 会疯狂 `--help`，反而产生大量 observation，CLI 未必省
  - 领域 Agent 的 system prompt / training / skill 已内置 kubectl、PromQL、AWS CLI → 一个命令就是 primitive → **CLI 在领域 Agent 上特别有优势**

### 9.5 收回「CLI 本地 / MCP 远程」的绝对化

- CLI 完全可以访问远程：Agent → CLI → RPC/HTTP/gRPC → Remote Service（如 `companyctl incident search` 背后是 HTTPS → Incident Service → DB）
- aws / kubectl / gh / companyctl 都可以是 remote API 的 CLI wrapper——**「远程能力」不是 MCP 的专属优势**

### 9.6 MCP 与 CLI 的真正区别（重新定义）

- **MCP**：`Machine-readable typed capability interface`——「你可以调用这个函数，参数必须长这样」（函数签名）
- **CLI**：`Environment-native executable interface`——「这是一个你可以操作的环境」
- 组合能力：**MCP 是 function composition；CLI 更接近 language composition**（管道、xargs 链接任意命令）

### 9.7 安全：CLI 完全可以开发时做限制

- 可行手段：命令 **Allowlist**、RBAC、Filesystem/Network sandbox、Credential scope、Seccomp、容器隔离、Approval workflow、Audit log
- 可做到 **Command AST → Policy evaluation → Allow/Deny**（结构化解析而非简单字符串匹配：`kubectl delete` deny、`kubectl get` allow）
- 修正结论：**「CLI 不安全、MCP 安全」不成立**；准确说法是 **MCP 的 capability boundary 天然更结构化；CLI 的 capability boundary 需要额外工程化**

### 9.8 团队为何最终抛弃 MCP（推测的真实原因）

即便 Dynamic MCP 已解决 Token 问题，仍选 CLI，真正看中的是：
1. **Composability**（A | B | C 管道）
2. **Universal primitive**——只需一个 `shell`，不需要 tool1/tool2/tool3……
3. **Existing ecosystem**——kubectl/aws/git/curl/ssh/docker/helm 已存在几十年
4. **自主探索**（`--help` 逐级下钻）
5. **CLI 可作为统一 remote interface**——背后是 HTTP/gRPC/Kafka/DB，Agent 不需要知道
6. **Sandbox 可以解决安全问题**

---

## 10. 重新定义：Structured Tool Calling → Agent Operating Environment

- 团队做的不是「MCP → CLI」，而是从 **Structured Tool Calling** 走向 **Agent Operating Environment**
- 传统：LLM → Tool Registry → Tool Schema → Function Call → Result
- 新型 Domain Agent：LLM → **Sandboxed Environment**（CLI / Filesystem / Network / Services）→ Observation → Action → Observation → Action……——像一个「**小型操作系统**」
- Agent 不再是「调用工具的东西」，而是「**在一个受约束的环境里行动的东西**」

## 11. 与 Hermes / Delegate Task 编排的连接

- 若每个 Sub-Agent 拿一堆 MCP：编排系统要管 tool registry、schema、permissions、context、tool routing
- 若改为 **主 Agent 分发给子 Agent 一个受限环境**（Sandbox + CLI / filesystem / credentials / MCP namespace）：
  ```
  Main Agent
    ├── Sandbox A └── CLI
    ├── Sandbox B └── CLI
    └── Sandbox C └── CLI
  ```
- 此时 **Agent 的能力边界直接由 Environment 决定**：「给 Agent 什么工具」变成「给 Agent 什么环境」
  - AIOps Agent → kubectl + prometheus-cli；DB Agent → psql + mysql；Cloud Agent → aws + terraform；Coding Agent → git + npm + python
- 架构演化：`Agent = LLM + Tools` → `Agent = LLM + Environment + Capabilities + Policy`

## 12. 最终判断（分维度对比）

| 维度 | Dynamic MCP | CLI + Sandbox |
|---|---|---|
| Token 效率 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 参数约束（schema 校验） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 工具组合 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 自主探索 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 生态复用 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Remote Service | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 安全边界 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 工程自由度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 领域特化 Agent | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 通用 Agent | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

- **AIOps 这类领域特化 Agent → 倾向 CLI + Sandbox**
- Dynamic MCP 不是错误路线，而是**漂亮的优化方案**；只是当 CLI 天然具备 discovery / composition / remote access，且 Sandbox 解决安全问题后，**继续维护一层 MCP abstraction 的收益越来越小**

## 13. 下一步值得研究的问题

> **「CLI 对 Agent 来说，究竟是不是一种隐式的 Tool Protocol？」**

若答案是「是」，则 `--help`、man page、SKILL.md、shell completion、exit code、stdout/stderr、JSON output、命令约束……可被重新理解为一套**面向 Agent 的 CLI Protocol**。这条线再往下走，就接近 **Agent Skills / Shell Agent / Computer-use Environment / Hermes 这类 Agent Runtime** 的设计哲学。

---

## 复习要点（记忆锚点）

- **核心论断**：领域 Agent 弃 MCP 是「弃 MCP 作 LLM 直接工具接口」，不是弃 MCP；方向是从 Tool-centric 走向 **Environment-centric**
- **省 Token 的本质**：CLI 把 tool catalog + schema 移出模型上下文（而非协议字节更少）
- **Dynamic MCP** 可把 Token 从 `O(N×Schema)` 降到 `O(N×描述 + 选中 Schema)`，与 CLI 差距大幅缩小——最终弃 MCP 的真实原因是 **action space（可组合可探索）而非 Token**
- **CLI 如何被理解**：纯 Shell(--help) → Skill/文档（≈Dynamic MCP）→ 结构化 capabilities/describe（=MCP Discovery 的 CLI 版）；领域 Agent「已知道 CLI」是关键
- **安全**：「CLI 不安全」不成立——CLI 靠 Command AST + Policy + Allowlist + Sandbox 工程化；MCP 的优势是 boundary 天然结构化（非安全保证，annotations 只是 hint）
- **分层结论**：CLI 管本地/环境/基础设施，MCP 管外部能力提供方；远程能力非 MCP 专属；未来是 Capability Router + 按需 discovery
- **与编排的关系**：从「分发给子 Agent 工具集合」→「分发给子 Agent 受限环境」；Agent = LLM + Environment + Capabilities + Policy
