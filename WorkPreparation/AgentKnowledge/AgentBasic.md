---
tags:
  - Agent
  - 面试八股
source: https://xiaolinnote.com/ai/agent/1_whatisagent.html
topic: Agent 基础系列（持续追加）
---
# 1. 什么是 Agent？与大模型有什么本质不同？

## 一句话回答

Agent 是一个**能自主完成目标的 AI 系统**，核心在于「**自主性**」和「**能行动**」。传统 LLM 是一问一答的被动响应；Agent 拿到复杂目标后自己拆解任务、调工具、访问记忆、感知环境，一步步执行直到完成。**从生成文字，到执行任务。**

## 普通大模型的三大局限

| 局限 | 说明 |
|---|---|
| 知识冻结 | 训练数据有截止日期，无法获取实时信息 |
| 不能行动 | 本质是文本生成器，只能给建议，无法真正执行（发邮件、查库、跑代码） |
| 无持续状态 | 每次调用之间失忆，跨任务记不住偏好 |

三者环环相扣 → 普通 LLM 只能做「一问一答」，多步任务无能为力。

## Agent 的核心闭环：感知 → 规划 → 行动 → 再感知

给 Agent 一个目标，它不是直接输出文字，而是先拆解任务（搜什么关键词、访问哪些网站、怎么组织内容），一步步执行，**每步结果反馈回来再指导下一步**。

背后有三件核心支撑：

### 1. 工具调用（Tool Use）—— 从「说话」变「做事」
- 关键点：**不是模型执行，而是模型「决策」，你的代码「执行」**（决策与执行分离）
- 工具定义 = 一份说明书（名称/描述/参数），模型读后决定调哪个、参数填什么，以 JSON 输出决策
- 一举突破三大局限：接搜索引擎解决知识冻结，接 API/执行器解决不能行动
- 示例：配 `get_weather` + `send_email` 两个工具，Agent 会分两步真实执行（查天气 → 发邮件）

### 2. 记忆机制 —— 能「记事」
- **短期记忆**：当前任务的中间状态（存在上下文），保证不会做到一半忘了前面
- **长期记忆**：跨任务的偏好/历史记录，通常用向量数据库存储 + 语义检索

### 3. 多步推理与自我纠错 —— 最像「人」的部分
- 某步失败不会崩：感知失败 → 分析原因 → 换方式重试（换关键词、看报错调参数）
- 边做边反思：这步对不对？结果符合预期吗？要不要调整后续计划？
- 这是 Agent 区别于**简单自动化脚本**的关键

## 为什么 Agent 现在才爆发？（三个条件同时成熟）

1. **模型能力过门槛**：GPT-4 / Claude 3 一代起，推理和指令遵循能力质变，能做合理多步决策
2. **工具调用标准化**：OpenAI 2023 年推出 Function Calling（结构化 JSON 输出），各厂商跟进
3. **配套生态完善**：LangChain/LlamaIndex 降门槛、向量数据库解决记忆存储、API 服务丰富

## Agent 生态最新趋势（面试加分项）

- **MCP（Model Context Protocol）**：Anthropic 2024 底提出，Agent 工具世界的「USB-C 接口」
  - 解决 M×N 适配问题：工具方按标准暴露为 MCP Server，任何支持 MCP 的 Agent 直接调用
  - 三层架构：Host（AI 应用）→ Client（连接管理）→ Server（暴露工具）
  - 2025.12 捐给 Linux 基金会 AAIF（Anthropic/Block/OpenAI 创立，Google/Microsoft/AWS 支持）
- **A2A（Agent2Agent）**：Google 2025.4 推出，Agent 与 Agent 协作协议
  - Agent Card「名片」机制：每个 Agent 声明能做什么、需要什么输入
  - 2025.6 捐给 Linux 基金会（SAP/Salesforce/ServiceNow 接入）
- **两者互补**：MCP 管「Agent ↔ 工具」，A2A 管「Agent ↔ Agent」

## 面试总结（答题要点）

**三个雷**（反面案例）：
1. 把 Agent 等同于插件/工具调用 —— 工具调用只是能力的一部分
2. 停在「能调工具」层，没点出自主性 —— 关键是「自己决定用不用、什么时候用、用哪个」
3. 忽略执行闭环 —— 感知→规划→行动→再感知 才是核心机制

**答题三件事**：
1. 自主规划：复杂目标能自己拆解成多步
2. 能行动：通过工具调用与外部世界真实交互
3. 有闭环：每步结果反馈指导下一步，不是一次性生成完就结束

**易混点**：模型只是「大脑」，工具真正执行的是你的代码，模型只负责决策。

## 延伸理解（补充自学习讨论）

### 多步推理与自我纠错是怎么实现的？

核心：**LLM 本身无状态，"多步推理"是 Agent 框架搭的循环，LLM 只是循环里被反复调用的决策函数**（事件循环 + 状态机）。

#### 多步推理 = ReAct 循环 + 上下文拼接
```go
for {
    resp := llm.Call(history)              // LLM 读完整轨迹，决定下一步
    decision := parseJSON(resp)            // 结构化输出 {"thought","action","action_input"}
    if decision.action == "final_answer" { return decision.output }
    result := executeTool(decision)        // 你的代码真正执行
    history = append(history, observation(result))  // 关键：结果拼回上下文
}
```
- 模型每次看到的输入 = 从任务开始到现在的完整轨迹（思考→行动→观察…），即 **scratchpad（草稿纸）**
- 类比：事件溯源（event sourcing），每个 worker 不记状态，靠完整事件流决策

#### 自我纠错 = 把"失败"翻译成文本喂回循环
1. **隐式纠错**：ReAct 循环天然自带——上一步结果不理想，模型下一轮看到就会调整（换关键词等）
2. **异常反馈**（最实用）：框架 try-catch 捕获异常，把错误信息转成 observation 塞回上下文 → 模型看到报错自己调整思路
3. **显式反思（Reflection/Critic）**：插入专门反思步骤，让模型（或评审模型）审视已生成内容，输出改进意见再重做（Self-Refine、Reflexion）；开销大，关键步骤才用

### FC 与 MCP 的关系（分层模型）

**不是"概念 vs 方案"，也不是 PV/PVC 式"资源 vs 声明"，而是上下两层、配合分工**：

| 层 | 规范 | 谁定义 | 解决什么 |
|---|---|---|---|
| 模型表达层 | FC | 模型厂商（OpenAI/Anthropic/Google 各有差异） | 模型如何"说出"我要调这个工具、传什么参数 |
| 框架翻译层 | （你的代码） | Agent 框架 | 解析 FC 决策 + 对接工具调用 |
| 工具接入层 | MCP | 开放标准（Anthropic 提出，AAIF 维护） | 工具如何被统一接入，避免 M×N 适配爆炸 |

一次完整调用链路：框架通过 MCP 发现工具 → 把 schema 转成 FC 格式塞进 prompt → LLM 输出 FC 决策 JSON → 框架解析 → 框架通过 MCP 真正调用 → 结果作 observation 回上下文。

**K8s 类比（修正版）**：MCP ≈ CRI（统一 kubelet↔运行时接口，解决 M×N 适配）；FC ≈ 模型 API 自带的工具调用输出格式（函数签名规范）。

### FC 的"API 定义"与"真实输出"之间的距离

- **API 定义 = 契约/壳**：请求带 `tools` schema，响应按 `tool_calls` JSON 结构返回——只规定"长什么样"
- **真实输出 = 训练内化的能力**：模型本质是 token 预测器，"输出 FC JSON"只是一串 token，没有执行语义；厂商用海量「带工具调用的对话样本」做 SFT，让模型在"prompt 里有工具 schema + 用户请求"时学会生成结构化 token
- 这段"距离"的直接证据：输出可能是非法 JSON、可能编造不存在的工具名、可能一次输出多个 tool_calls → 框架必须做解析、校验、重试
- 类比：FC 的 API 定义 ≈ gRPC 的 proto（契约），模型的 FC 输出能力 ≈ 训练写进权重的服务端处理逻辑（实现）

### 不同模型的 FC 兼容性差异（为什么有的模型表现更优）

**成立**：同一框架接不同模型，FC 调用成功率有明显差异。

- 原因：模型训练时见的 FC 语料格式 = 模型"母语"；框架的 FC 约定是"外语"，匹配度决定稳定性
- 各家母语：OpenAI = `tool_calls` JSON；Anthropic Claude = XML 风格 `<tool_use>`；Gemini = `functionCall`；DeepSeek = 主动对齐 OpenAI 格式
- 两层缓解：① API 厂商后处理兜底（能兜住部分格式错误）；② 框架按模型选适配器（prompt 模板 + 响应解析器），前提是框架为这个模型写了适配
- 市场现象印证：OpenAI 格式成事实标准（开源模型主动对齐）、框架有"模型兼容矩阵"、工程选型把"工具调用稳不稳"当硬指标

---

*此文档为 Agent 基础系列，后续面试问题与理解持续追加。*

---

# 2. Agent 的基本架构由哪些核心组件构成？

## 一句话回答

四个核心组件：**LLM（大脑）、工具系统（手脚）、记忆系统（档案室）、规划模块（项目经理）**。

- **LLM**：理解任务 + 做决策
- **工具**：与外部世界交互（搜索、执行代码、调 API）
- **记忆**：任务执行中保持状态，不会「失忆」
- **规划**：把复杂目标拆解成可执行步骤

四个组合在一起，Agent 才具备自主完成任务的能力。

## 详细解析（公司类比）

| 组件 | 类比 | 职责 |
|---|---|---|
| LLM 核心 | 老板 | 所有决策经过它拍板：继续思考 / 调工具 / 给最终答案 |
| 工具系统 | 外包执行团队 | 真正干活（搜索、发邮件），LLM 说做什么它们做什么 |
| 记忆系统 | 公司档案室 | 信息存档和调档 |
| 规划模块 | 项目经理 | 大目标拆成可执行的任务单 |

### LLM 核心
- **System Prompt 易被忽略但极重要**：等于「岗位说明书」，开工前定义角色、行为边界、输出格式；工程里调优占开发时间很大比重，是最直接控制 Agent 行为的手段
- **选模型是工程关键**：
  - 推理能力：决定能否正确拆任务、选对工具（GPT-4o/Claude Sonnet 强于小模型）
  - 推理模型趋势：o1/o3、DeepSeek-R1 适合做 Agent 大脑，但延迟高、token 贵 → 常见做法：大模型做核心决策（规划、关键判断）+ 小模型做简单任务（意图分类、格式提取）
  - 工具调用稳定性：JSON 格式错乱导致失败重试、token 浪费
  - 上下文窗口：每步工具结果都塞进上下文，十几步就满；选型是「任务复杂度 × 延迟 × 成本」的权衡

### 工具系统
- 唯一与外部世界交互的入口；任何能用函数封装的能力都能变成工具
- 工具定义格式（OpenAI function calling）：只有「名字、描述、参数」三件事，无执行逻辑
- **决策与执行分离**：模型决定调什么、参数填什么（JSON 输出），你的代码真正执行
- **工具描述质量直接影响表现**：description 写含糊，模型会误用/漏用工具（"查询数据"太宽泛 vs "查询公司内部销售数据库，支持按日期、产品类别筛选"）→ 描述调优重要性不亚于 prompt 工程
- 工具多了要标准化 → **MCP**（Anthropic 2024 底提出）：基于 JSON-RPC，不止标准化 Tools，还有三类能力：Tools（改变世界，如发邮件）/ Resources（只读数据源）/ Prompts（预置提示词模板）；三层架构 Host → Client → Server；工具世界的「USB-C 接口」

### 记忆系统
- **短期记忆**：当前对话上下文（context window），存中间状态；像「工作记忆」，容量有限，任务结束清空
- **长期记忆**：向量数据库（embedding 存储 + 语义检索）；像「长期记忆」，容量大跨天保留，需主动「回忆」
  - 认知科学分类（便于理解的类比）：语义记忆（事实知识）/ 情景记忆（具体经历）/ 程序性记忆（做事方法论，沉淀标准流程下次直接套用）
- **工程挑战**：
  - 短期：上下文有限 → 摘要压缩 or 滑动窗口，都会丢信息；「记住够多」vs「不撑爆上下文」是核心设计问题
  - 长期：「什么该存」→ 存入前做重要性评估，只存有价值信息；**记忆衰减（Memory Decay）**：每条记忆加时间权重，相关性分数 = 语义相似度 × 时间衰减因子，过期信息自动排后面（客服衰减快、法律合规衰减慢）

### 规划模块
- 底层依赖 LLM 推理能力，提升手段：
  - **CoT（思维链）**：让模型把思考过程写出来（"Let's think step by step"）；token 逐步生成，中间步骤写出来 = 更多思考空间
  - **ToT（思维树）**：每个节点展开多个分支、评估后选最优；CoT 是一条路走到底，ToT 是岔路口先看几条路；更贵但适合创造性/复杂决策
- 两种主流模式：
  - **Plan-and-Execute（先规划后执行）**：先输出完整步骤列表再逐步执行；结构清晰可审核，但中间结果不符预期时计划要重调
  - **ReAct（边执行边规划）**：每步根据当前结果重新思考；灵活但容易走偏（局部最优忽略整体）；工程上常结合：先粗略计划定方向，再动态微调

## 核心运行循环（伪代码）

```python
def agent_run(user_goal: str):
    plan = llm.plan(user_goal)          # 规划模块：目标拆步骤
    memory = []                          # 短期记忆：存中间结果
    for step in plan:
        action = llm.decide(             # LLM 核心：这一步怎么做？
            step=step,
            history=memory,              # 短期记忆 → 知道之前做了什么
            long_term=vector_db.search(step)  # 长期记忆 → 捞出相关历史
        )
        if action.type == "tool_call":
            result = tools.execute(action.tool_name, action.args)  # 工具系统执行
            memory.append({"step": step, "result": result})        # 结果入短期记忆
        elif action.type == "final_answer":
            return action.content
```

核心节奏：**规划 → 决策 → 执行 → 结果存入记忆 → 再决策**，循环往复直到完成。LangChain/LlamaIndex/AutoGen 等框架本质都是围绕这四个组件设计。

## 面试总结（答题要点）

**三个雷**：
1. **漏掉组件**：只说 LLM 和工具，忘了记忆和规划——恰恰是跑复杂任务的关键
2. **记忆理解太浅**：「记忆就是上下文」不完整；正确说法：短期记忆（context window，存中间状态）+ 长期记忆（向量数据库，跨任务保存偏好/历史），机制用途完全不同
3. **工具分工理解偏差**：模型不执行工具，只输出「调哪个、传什么参数」的决策，执行的是你的代码（决策与执行分离，高频追问点）

**加分答法**：四个组件 + 类比（LLM 老板 / 工具外包团队 / 记忆档案室 / 规划项目经理）结合起来说。

## 延伸理解（补充自学习讨论）

### 记忆模块的本质：是 Harness 的代码吗？

**对，记忆模块属于 Agent Harness（LLM 之外的一切代码），但短期/长期的实现性质完全不同**：

- **短期记忆 ≈ 上下文组装策略（不落盘）**：harness 每次调用 LLM 时把 `[System Prompt] + [历史消息] + [最新工具结果]` 拼成一个 prompt 发出去。历史消息就存在 harness 内存里（`[]Message` 切片），"记忆"体现在"每次都把它拼进请求"，不是"存哪里"。超长处理才是真正的策略：滑动窗口（只留最近 N 轮，LRU 式丢弃）/ 摘要压缩（LLM 浓缩旧历史成摘要）。都会丢信息——context window 是硬约束。
- **长期记忆 ≈ 持久化存储组件**：独立组件，写入时内容 → embedding → 向量库（向量 + 原文 + metadata）；读取时问题 → embedding → 相似度检索 → 拼进 prompt。策略都是代码：重要性评估（什么值得存）、记忆衰减（时间权重）、去重、更新。
- **修正点：System Prompt 不是记忆**。它每次请求固定带上，是"配置/岗位说明书"（类似服务的 config.yaml），记忆存的是运行时产生的状态（用户偏好、历史对话、工具结果、总结的经验）。

### 上下文窗口机制（每次调用怎么算）

**context window 是模型侧硬上限：每次调用时，发出去的所有 token（system prompt + 历史消息 + 当轮问题 + 工具结果）加起来不能超过，不是"每条消息各一个窗口"。**

- **多轮对话就是全部拼接**：`[system prompt] + [第1轮] + [第2轮] + ... + [工具结果] + [当前问题]` 整个串起来发；LLM 没有"轮次"概念，靠拼接顺序理解先后
- **随轮次自然增长**：每轮 append 到消息数组 → 下次请求 token 更多 → 越聊越贵；增长是必然的，除非 harness 主动裁剪
- **超限处理**：发送前 harness 检测 token 数自行裁剪；直接发则 API 报错或模型截断最前面的内容（相当于"忘了最早的"）
- **易误解点**：输入侧 context window（能塞多少）≠ 输出侧 max_tokens（一次生成多少）；1M 窗口不代表能输出 1M token

### 长期记忆存储：不止向量数据库

按**检索方式**分层，实际系统往往混合使用：

| 存储方式 | 检索能力 | 适合场景 | 例子 |
|---|---|---|---|
| 向量库 | 语义相似 | 模糊语义匹配、开放问题 | Milvus、pgvector、FAISS |
| 知识图谱 | 关系/多跳推理 | 实体关系密集、推理 | Neo4j（文本→LLM 抽实体关系→建图） |
| 关键词/BM25 | 精确匹配 | 专有名词、代码标识符 | Elasticsearch |
| 结构化 KV/DB | 精确查询 | 用户偏好、元数据 | Redis、PostgreSQL |

- 向量库只懂"相似"不懂"关系"；图数据库回答"什么和什么**相关/相连**"，适合多跳推理，但构建维护成本高（抽取准确性、schema 设计）
- **面试说法**：向量管语义、图管关系、关键词管精确、结构化管属性，实际按需混合 + 多路召回融合（rerank）

### Workflow 的三种形态（推理范式 vs Workflow）

**先理清概念**：推理范式（ReAct/Plan-and-Execute/Reflection）= LLM 怎么思考的策略，是 harness 里预定义的选择；Workflow = 执行这个策略时实际走出来的流程编排。Workflow 按"流程由谁定"分三档：

1. **人工预定义（硬编码）**：流程完全写死（Go 函数链），LLM 不参与决策或只在节点内做局部处理；确定性 100%、可审计，但无智能
2. **框架编排 + LLM 节点（工程主流）**：拓扑（节点+边）预先定义，节点内 LLM 自主决策；典型体现 LangGraph 图定义；可控性高（骨架可控）+ 智能（节点内自由）
3. **纯 LLM 自主（纯 ReAct）**：无固定拓扑，每步"下一步做什么"由 LLM 实时决定；最灵活但路径不可预测、难审计，生产需护栏

### 纯 ReAct 是怎么实现的？（循环代码 + Prompt 约束）

**光有循环代码不够，规则藏在 system prompt 里**：

```go
// ① 代码：最简循环，不定义具体步骤
for {
    resp := llm.Call(history)             // 完整轨迹发给 LLM
    decision := parseJSON(resp)           // 解析 {"thought","action","action_input"}
    if decision.action == "final_answer" { return decision.output }
    result := executeTool(decision)
    history = append(history, observation(result))
}
```

```text
// ② system prompt（harness 首轮注入的"操作说明"）：
你是自主执行任务的 Agent，必须按以下格式响应：
Thought: <当前思考>
Action: <工具名，必须来自可用工具列表>
Action Input: <JSON 参数>
任务完成时输出：Final Answer: <最终结果>
规则：每轮只能输出一个 Thought + Action；Action 必须合法
```

- **纯 ReAct 的"纯"是相对的**：代码不定义步骤拓扑，但 prompt 定义了输出格式、工具范围、终止条件（Final Answer）——约束从"代码拓扑"转移到"prompt 规则"
- **真实框架**（如 LangChain `create_react_agent`）：把工具清单 + ReAct 格式说明拼进 system prompt，循环是通用骨架，换工具只需换 prompt 里的清单
- 所以 ReAct 能适配任意任务，但也因约束在 prompt 里（模型可能不遵守）而比硬编码 Workflow 难控制

---

# 3. Workflow，Agent，Tools 这三个的概念和区别介绍一下？

## 一句话回答

三者是**粒度从小到大、可相互嵌套的三层结构**，不是三选一关系。最核心的区分角度只有一个：**谁来做「下一步该干什么」的决策**。

| 维度 | Tools | Agent | Workflow |
|---|---|---|---|
| 决策能力 | 无（只执行，不决策） | 有（LLM 自主动态决策） | 无（开发者在代码里写死） |
| 执行方式 | 被动，等待被调用 | 主动，自主循环直到完成 | 按开发者定义的顺序执行 |
| 确定性 | 高 | 低（同输入可能走不同路径） | 高（行为完全可预测） |
| 灵活性 | 只做一件事 | 高（应对预料之外情况） | 低（流程提前写死） |
| 调试难度 | 容易 | 难（执行路径不确定） | 容易（链路清晰） |
| 适用场景 | 封装单一能力 | 路径未知的复杂任务 | 流程相对固定的业务系统 |

一句话：**Tools 不做决策只执行，Agent 自己做决策，Workflow 是开发者替所有节点把决策提前写好。**

## 第一层：Tools —— 最小的能力积木

- **本质**：按特定格式暴露给 LLM 的函数（普通函数给程序员调用，Tool 给 LLM 调用）
- 与普通函数唯一区别：额外配一份 LLM 看得懂的 schema（名字、描述、参数类型），否则 LLM 不知道它存在、也不知道怎么用
- **工具本身没有任何决策能力**，甚至不知道自己"应该"什么时候被用（瑞士军刀刀片，决定拿哪把的是手）
- 好的工具设计原则（影响 Agent 表现的关键，很多系统表现不好根源是工具设计问题）：
  1. **职责单一**：一个工具只做一件事（"查天气+发邮件"混一起，模型无法精确判断何时用）
  2. **描述精确**：模型完全靠 description 理解工具（"查询数据"太宽泛 vs "查询公司内部销售数据库，支持按日期和产品类别筛选"）
  3. **错误信息清晰**：返回 LLM 能看懂的（"参数 city 不能为空" > "Error code 400"），帮它自己修正重试
  4. **参数简洁**：能少传就不传，有默认值给默认值（参数越多出错概率越大）
- 工具多了要管理 → MCP（工具注册/描述/调用标准化协议，USB 接口）

## 第二层：Agent —— 拿着工具自己做决定的人

- **核心区别**：Tools 被动等待调用，Agent 主动做决策（要不要、用哪个、够不够、停不停全由内部 LLM 判断）
- 运行循环：**想清楚（Thought）→ 行动（Action）→ 看结果（Observation）→ 再想**，直到 LLM 判断任务完成（开发者不知道循环跑几次——Agent 与普通代码最大的不同）
- **停止条件（Stop Condition）**——工程必须，防失控无限循环：
  1. LLM 主动判断完成（最理想）
  2. 最大循环次数（如 15 轮强制停）
  3. 总 token 预算上限（防成本失控）
  4. 超时机制（如 60 秒终止）
  - 实际同时存在，哪个先触发用哪个
- **副作用：行为不确定**（LLM 是概率模型）→ 生产环境加详细执行日志（记录每步思考和工具调用结果），方便事后追溯复现

## 第三层：Workflow —— 把所有人组织起来的总指挥

- **定义**：把整个执行流程的"骨架"写在代码里，LLM/Agent/Tools 都只是流程里的"节点"，整体走哪条路、下一步去哪**全由开发者代码决定**
- 例子：客服系统 = LLM 做意图分类（节点）→ `if/elif` 决定分支（产品→知识库检索→LLM 生成回答；退款→查订单→审核）→ LLM 只是"两个工位"
- **Workflow vs Agent 核心区别**：谁做"下一步去哪"决策——Agent 是 LLM 自己决定，Workflow 是开发者在代码里写死
- **最大优点**：可预测、可控、好调试（代码看到什么做什么，出问题打断点逐步追）

## 三者怎么组合：Agentic Workflow 才是生产主流

- 纯 Agent 系统生产环境少（难控制、难排查、成本失控）；纯 Workflow 太脆（无法穷举所有情况）
- **Agentic Workflow**：用 Workflow 固定主流程骨架 + 需要灵活判断的节点嵌入 Agent + 固定节点直接用 LLM 或 Tools → 可控性和灵活性兼得
- Anthropic 总结的常见编排模式（面试加分）：
  1. **Prompt Chaining（提示链）**：大任务拆多步，前一步输出作为后一步输入，流水线串联
  2. **Routing（路由）**：LLM 先分类判断，按结果分发到不同分支（客服系统例子）
  3. **Parallelization（并行化）**：可同时进行的子任务并行执行，最后汇总（多维分析、多数据源检索）
  4. **Orchestrator-Workers（编排者-工人）**：中央编排者分配任务，多个 Worker 各自完成独立子任务
  5. **Evaluator-Optimizer（评估者-优化者）**：一个 LLM 生成 + 另一个 LLM 评估质量，不通过则反馈改进重试，直到通过或达最大次数（营销文案、法律条款、代码生成）；本质是"人类审稿-修改"自动化；**评估标准必须在代码里定义清楚**（打分函数），不能让评估者自由发挥
  - 各模式不互斥，实际混合使用
- **性能/成本角度**：纯 Agent 复杂任务 LLM 跑十几轮，每轮全量上下文 → token 线性增长、延迟累积；Workflow 流程固定 → 可精确控制每节点 token 预算、该并行就并行 → 延迟成本可控。**原型用纯 Agent 快速验证 → 生产重构为 Agentic Workflow** 是常见演进路径

## 面试总结（答题要点）

**最典型误区**：把 Workflow 理解成"多个 Agent 串联"——不对！Workflow 节点可以是任意 LLM 调用、Tools 或 Agent，关键不是节点类型，而是**控制流由谁掌握**（Workflow = 代码里写死的 if/else，Agent = LLM 动态决定）。

**答题核心角度**："谁做决策"：Tools 无决策只执行 / Agent 由 LLM 运行时动态决策（同输入可能走不同路径）/ Workflow 决策提前写死在代码里（行为完全可预测）。

**加分句**：三者可相互嵌套，生产环境主流不是纯 Agent 而是 **Agentic Workflow**（Workflow 固定主流程骨架 + 灵活节点嵌 Agent），兼顾可控性与灵活性。

## 延伸理解（补充自学习讨论）

### 推理范式 vs 编排模式：正交关系

**两个不同维度，不是包含关系**：

- **推理范式（ReAct / Plan-and-Execute / Reflection）**：微观层——**LLM 在节点内部怎么思考**（走一步看一步 / 先规划后执行 / 做完反思）
- **编排模式（5 种）**：宏观层——**节点之间怎么连接组织**（串联 / 路由 / 并行 / 编排者-工人 / 评估-优化）

**两者正交，自由组合**，不是"ReAct 由 5 种模式组合"：

| 组合 | 例子 |
|---|---|
| Routing + ReAct | 路由节点用 ReAct 风格判断该分到哪个分支 |
| Orchestrator-Workers + Plan-and-Execute | 编排者先规划子任务列表（Plan）再分发给 Workers（Execute）——Plan-and-Execute 的宏观版 |
| Evaluator-Optimizer + Reflection | 评估不通过让生成者反思重做——评估节点内部用 Reflection 范式 |

> K8s 类比：调度策略 ≈ 单个 Pod 内容器怎么跑（微观）；Deployment/StatefulSet ≈ Pod 们怎么组织（宏观）。两者正交。

### 三者的嵌套结构（不是并列）

**三个概念不是并列的，是不同抽象层级 + 不同决策权的东西**：

| 概念 | 是什么 | 决策权 | 粒度 |
|---|---|---|---|
| Tools | 能力单元（函数） | 无（只执行） | 最细 |
| Agent | 决策系统（LLM 循环） | LLM 运行时动态决策 | 中间 |
| Workflow | 编排骨架（节点拓扑） | 开发者代码写死 | 最粗 |

**递归嵌套关系**：

```
Workflow（开发者定决策）
  ├── 节点：LLM 调用 / Tools / Agent
  │         └── Agent（LLM 定决策）
  │               └── 调 Tools（无决策）
  └── Agent 还可以生成/调整 Workflow（递归，Plan-and-Execute 的落地形态）
```

- **Agent 生成 Workflow**：LLM/Agent 规划出节点序列 + 条件边，harness 物化成可执行图（Plan-and-Execute 的落地）
- **Workflow 节点是 Agent**：子 Agent 作为节点自己做内部循环（Orchestrator-Workers 的典型形态）

**为什么题面总把三者并列？** 因为面试题问的是"**谁做决策**"这个维度上的三种答案：Tools=没有人决策 / Agent=LLM 决策 / Workflow=开发者决策。它们在**一个问题上是并列的（决策权归属）**，但不是**结构上的并列（粒度/层级不同）**。就像"总裁/经理/员工谁说了算"——在一个问题上并列，结构上是上下级。

---

# 4. 了解哪些其他的 Agent 设计范式？Agent 和 Workflow 的区别是什么？

## 一句话回答

Agent 和 Workflow 最核心的区别是「**谁来决定下一步**」。Workflow 是提前把流程写死（确定性高、好控制）；Agent 是让 LLM 自己决定下一步（灵活但不可控）。常见设计范式：ReAct、Plan-and-Execute、Reflection。工程上用得最多的反而是两者混用——**固定流程用 Workflow，需要灵活决策的节点嵌入 Agent**。

## Workflow 和 Agent 的区别（代码结构对比）

| 维度 | Workflow | Agent |
|---|---|---|
| 决策者 | 开发者（硬编码流程） | LLM（动态决策） |
| 确定性 | 高，行为完全可预测 | 低，同输入可能走不同路径 |
| 灵活性 | 低，流程固定 | 高，能处理预料之外的情况 |
| 调试难度 | 容易，链路清晰 | 困难，行为不确定 |
| 适用场景 | 流程相对固定的业务 | 需要灵活判断的复杂任务 |

代码结构完全不同：
- **Workflow**：普通函数链（`vector_db.search → rerank → llm.generate`），控制流由代码决定，LLM 只是节点工具（"开发者在驾驶"）
- **Agent**：`while True { llm.decide() }` 循环，所有路径 LLM 运行时动态选（"LLM 在驾驶，开发者副驾驶设安全限制"）

## Agent 三种设计范式

### ReAct（Reasoning + Acting）—— 最常见
- 每轮循环三步：**Thought（写出推理过程）→ Action（决定调哪个工具/参数）→ Observation（工具结果反馈）**，直到 LLM 判断信息够了给出最终答案
- 为什么显式分三步：直接输出行动容易"冲动决策"；Thought 可见，出问题能定位是哪步想歪了
- **短板**：走一步看一步，局部最优，复杂任务（十几步）容易迷失目标或反复打转

### Plan-and-Execute —— 针对 ReAct 的短板
- **规划和执行解耦**：先一个 LLM 专门输出完整步骤列表，再由另一个 LLM（或同模型不同角色）逐步执行
- **关键机制：动态重规划**——每执行完一步把结果反馈给规划器，判断结果是否符合预期、计划是否需要调整/插入新步骤（"计划是活的"）
- 优点：全局视野清晰，执行前可人工审核计划；缺点：多了规划和重规划的 LLM 调用，延迟成本增加；初始规划方向错了后面难挽回

### Reflection（反思）—— 前两种之上的质量保障
- 完成一步或整个任务后，让 LLM（或专门评估模型）判断做得好不好，不通过就重试/换策略
- **Reflexion 变体**：不只"重做"，而是生成"反思总结"（失败原因 + 改进建议）作为额外上下文传给下次尝试（"错题本"）
- 数据佐证：HumanEval 上 GPT-4 直接做约 80%，加 Reflexion 后 91%
- 代价：每轮反思多一次/多次 LLM 调用，token 和延迟增加（质量 vs 成本取舍）

### 三种范式怎么选（面试回答）
看**任务复杂度**和**质量要求**两个维度：
- 步骤不多、每步独立 → **ReAct**（简单直接）
- 复杂、步骤有依赖需全局统筹 → **Plan-and-Execute**
- 质量要求特别高、允许多花时间成本 → 叠加 **Reflection**
- 不互斥可混合：Plan-and-Execute 做整体规划 + 每步内部用 ReAct + 关键步骤加 Reflection

## 为什么纯 Agent 不是生产首选：Agentic Workflow

- 纯 Agent 生产用很少：行为不确定、难调试、成本失控（LLM 跑太多轮）
- **Agentic Workflow**：整体用 Workflow 框住主流程 + 需要灵活处理的节点嵌入 Agent。例：客服系统主链路（意图识别→知识检索→回答生成）固定为 Workflow，"知识检索"节点内部用 Agent 动态决定检索几轮、用哪些工具
- **Anthropic 原则**：能用 Workflow 解决的问题就不要用 Agent（可控性 > 灵活性）。从最简单 Workflow 开始，只有发现某节点确实需要灵活决策、写死逻辑无法覆盖时才升级为 Agent（**"从简单到复杂、按需升级"是面试加分点**）

## 面试总结（答题要点）

**三个雷**：
1. **设计范式不熟**：ReAct 之外，Plan-and-Execute（规划执行解耦）和 Reflection（执行后自我评估）必须说出来
2. **把 Reflection 当调试手段**：它是正式运行时机制，内嵌执行流程，代价是 token 消耗和延迟增加（取舍常被追问）
3. **以为 Agent 是生产首选**：纯 Agent 生产用很少（行为不确定、难调试、成本失控）；真正的工程答案是 Agentic Workflow

**拿高分关键**：能主动说出"为什么纯 Agent 在生产里有局限"+ Agentic Workflow 的混合方案。

## 延伸理解（补充自学习讨论）

### 推理范式 vs Workflow 的实现差别（代码级对比）

**核心：差别不在"用了什么技术"，而在"控制流（谁决定下一步去哪）写在代码里，还是交给 LLM 运行时决定"。**

**① Workflow —— 控制流 = 代码 if/else（静态）**
```go
func WorkflowAnswer(userQuery string) string {
    docs := vectorDB.Search(userQuery, 5)      // 第1步：固定检索
    reranked := reranker.Rank(userQuery, docs) // 第2步：固定 rerank
    return llm.Generate(userQuery, reranked)   // 第3步：LLM 只生成回答
}  // 代码 = 普通函数链，LLM 是乘客不是司机
```

**② ReAct —— 控制流 = LLM 每轮现想（动态）**
```go
for {
    resp := llm.Call(history)                  // 每轮 LLM 看完整历史决策
    act := parseJSON(resp)                     // {"thought","action","action_input"}
    if act.action == "final_answer" { return act.output }
    result := tools.Execute(act.action, act.args)   // 执行 LLM 选的工具
    history = append(history, Observation(result))
}  // 代码里没有 if/else 业务分支，只有 for{llm.decide()}，LLM 是司机
```

**③ Plan-and-Execute —— 先让 LLM 生成"计划数据"再执行**
```go
plan := llm.Plan(goal)          // 阶段一：一次 LLM 调用生成步骤列表（数据）
for i, step := range plan {     // 阶段二：按计划循环执行
    if shouldReplan(step, results) { plan = llm.Plan(goal, results) }  // 动态重规划
    results = append(results, executeStep(step))
}
// 与 ReAct 区别：ReAct 每轮现想；P&E 先想完再动手，但计划是活的
```

**④ Agentic Workflow —— 外层静态 + 节点内动态**
```go
func CustomerServiceFlow(query string) string {   // 外层骨架：代码写死
    switch llm.Classify(query) {
    case "product": return productHandler(query)  // 节点内是完整 Agent（动态）
    case "refund":  return refundHandler(query)
    }
}
func productHandler(query string) string {
    for { /* ReAct 循环：Agent 自己决定检索几轮、用哪些工具 */ }
}
```

| | Workflow | ReAct | Plan-and-Execute | Agentic Workflow |
|---|---|---|---|---|
| 控制流在哪 | 代码 if/else | LLM 每轮现想 | LLM 先出计划再执行 | 代码定骨架+节点内 LLM |
| 代码形态 | 函数链 | `for{llm.decide()}` | `plan→for step` | 骨架函数+内嵌 Agent 循环 |
| 有"计划"概念？ | 无 | 无 | 有（可重规划） | 骨架即计划 |
| 确定性 | 最高 | 低 | 中 | 高（骨架可控） |
| 适用 | 流程固定 | 步骤少独立 | 复杂有依赖 | 生产主流 |

**一句话**：Workflow = 开发者在驾驶；ReAct/Plan = LLM 在驾驶（边走边想 vs 先想后走）；Agentic Workflow = 开发者定路线图，LLM 在路线图内自由开。

### Workflow 三分法（自学习总结的核心洞察）

**所有范式本质都是"静态模板（骨架）+ 运行时实例化（轨迹）"**——区别不在"有没有骨架"，而在**骨架的业务化程度**：

| 类型 | 骨架形态 | 骨架里有什么 | 动态范围 | 需要推理模式？ |
|---|---|---|---|---|
| 纯静态 Workflow | 业务图（最"大"骨架） | 具体业务步骤（检索→rerank→生成） | 几乎没有（LLM 只是节点内生成文本） | 不需要 |
| Agentic Workflow | 业务图 + 节点内嵌 Agent | 主流程固定 + 某节点是完整循环 | 骨架内 Agent 节点 | 只在节点内 |
| Dynamic Workflow（= 纯 Agent） | 通用循环（最"小"骨架） | 只有 Thought→Action→Observation，无业务步骤 | 整个任务（业务步骤都由 LLM 现场生成） | 整个流程都是 |

- **骨架是否"认识业务"是区分关键**：Agentic 的骨架写着业务词（意图/检索/回答）；Dynamic 的骨架只认识思考/行动/观察
- ReAct、Plan-and-Execute、Reflection、Reflexion 都是**通用逻辑、无业务化** → 都属于 **Dynamic Workflow 家族**
- 执行轨迹（trace）= 引擎跑一次实际走出的路径；静态 Workflow 蓝图=实例，Dynamic Workflow 蓝图≠实例（引擎=轨迹生成器）

**Anthropic 术语对应**：Anthropic 说的 Agent 本质就是 Dynamic Workflow（"agents = dynamic workflows"，路径运行时动态生成）。Agentic Workflow 是业界对"静态骨架+动态节点"混合架构的称呼，不是同一概念，而是动态程度谱系上的不同位置：静态 → Agentic → Dynamic（纯 Agent）。

### 静态 Workflow 的 5 种组织模式（Anthropic 总结）

> 注意：这些是**静态骨架的组织方式**（路径由代码决定，LLM 不决定路径）。与 Dynamic Workflow 的推理范式（ReAct/P&E/Reflection）**正交**——前者是宏观层"节点怎么连"，后者是微观层"节点内 LLM 怎么想"。

**① Prompt Chaining（提示链）—— 串行流水线**
前一步输出作为后一步输入，严格顺序，每步职责明确：
```go
func PromptChain(goal string) string {
    outline := llm.Call("写大纲", goal)          // 步骤1
    draft := llm.Call("按大纲写初稿", outline)    // 步骤2（吃步骤1输出）
    return llm.Call("检查并润色", draft)          // 步骤3（吃步骤2输出）
}  // 组织：线性 one→one，第 N 步只依赖第 N-1 步
```
适用：任务可明确拆成多个固定顺序的独立阶段（写文档、数据处理流水线）。

**② Routing（路由）—— 分类后分发**
先分类，再按类走不同分支（各分支独立）：
```go
func Routing(query string) string {
    intent := llm.Classify(query)   // 1. 分类节点（LLM 只做分类）
    switch intent {                  // 2. 分发由代码 if/else 决定（路径静态）
    case "product": return productPipeline(query)
    case "refund":  return refundPipeline(query)
    default:        return human(query)
    }
}
```
适用：输入类型差异大、不同类型走完全不同的处理链（客服分流、多场景路由）。与 Orchestrator 区别：Routing 是"分类选一条路"，Orchestrator 是"拆多份给多人再合并"。

**③ Parallelization（并行化）—— 同时跑再汇总**
多个独立子任务并行执行后融合，两种变体：
- **Sectioning（分块）**：大任务切成独立块并行做（分章节写报告）
- **Voting（投票）**：同一问题多个答案，投票选最优
```go
func Parallel(query string) string {
    r1 := goTask("分析竞品A", query)    // 任务1（并行）
    r2 := goTask("分析竞品B", query)    // 任务2（并行）
    r3 := goTask("分析市场报告", query)  // 任务3（并行）
    return merge(r1, r2, r3)           // 汇总（fan-out → fan-in）
}
```
适用：子任务互不依赖、可同时进行、多维度分析。

**④ Orchestrator-Workers（编排者-工人）—— 中央分配**
一个编排者分解任务+分发，多个 Worker 各自干活，结果回收：
```go
func Orchestrator(goal string) string {
    tasks := orchestrator.Decompose(goal)   // 1. 编排者拆任务（LLM 生成任务列表）
    results := []string{}
    for _, t := range tasks {               // 2. 分发（静态循环）
        results = append(results, workers[t.Kind].Execute(t))  // 3. 各 Worker 干活
    }
    return orchestrator.Synthesize(results) // 4. 编排者汇总
    // 组织：中心化 1:N，编排者是唯一入口/出口，Workers 互相不通信
}
```
适用：任务可分解且子任务相互独立、需要动态确定子任务列表。

**⑤ Evaluator-Optimizer（评估者-优化者）—— 生成-评估循环**
生成者产出 → 评估者打分 → 不合格带反馈重做，直到达标或达最大轮数：
```go
func EvaluatorOptimizer(input string) string {
    output := generator.Call(input)           // 1. 生成者产出
    for i := 0; i < maxRounds; i++ {
        score, feedback := evaluator.Eval(input, output)  // 2. 评估者打分+反馈
        if score >= threshold { return output }           // 3. 达标即停
        output = generator.Call(input, feedback)          // 4. 带反馈重做
    }
    return output  // 组织：闭环循环，质量门槛由代码控制
}
```
**易踩坑**：它看起来"动态"（生成→评估→改进循环），但**骨架固定**（路径不靠 LLM 决定），所以是静态 Workflow 模式，不是 Dynamic。适用：质量要求高、一次做对难（营销文案、法律条款、代码生成）。

**各模式不互斥**，实际项目按需组合。

### Agentic vs Dynamic 的判定实例（Supervisor 多 Agent 场景）

**问题背景**：Supervisor（监督者/编排者）"选择调用哪个子 Agent"这个流程，算 Agentic 还是 Dynamic？不同情况归类不同，关键是看**路由决策权在代码还是 LLM**：

| 情况 | 谁决定调哪个子 Agent | 决策权 | 正确归类 |
|---|---|---|---|
| A. 正则/关键词/传统ML 分类器硬编码路由 | 代码/规则 | 代码 | **静态 Workflow（Routing 模式）**；节点是子 Agent → 也可叫 Agentic Workflow 的骨架版 |
| B. Supervisor 把问题+历史上下文+各 Agent 能力描述丢给 LLM，LLM 决定调用谁/顺序/是否重试 | LLM | LLM | **Dynamic Workflow（= 纯 Agent）**，即 Orchestrator-Workers 的 Agent 形态 |
| C. 没有 Supervisor，单个 Agent 全自主 | LLM | LLM | 纯 Agent = Dynamic Workflow |

**易错点（朋友的图踩的坑）**：把"运行时根据输入走不同分支"当成 Dynamic 的特征——错！这是 **Routing（路由）**，属于静态 Workflow 的组织模式。"Dynamic"的正确含义是"**路径由 LLM 运行时生成**"，不是"运行时走哪个分支"。硬编码路由和 Dynamic 是反义词（路径是代码预定义的=静态）。

**同一个图两种归类**：编排者（Supervisor）是代码 if/else → 静态骨架 → Agentic；编排者是 LLM 自主决策 → 动态骨架 → Dynamic。**区别只在编排者本身是代码还是 LLM。**

### 决策占比连续谱：Agentic 与 Dynamic 之间没有硬线

**本质**：判断不是"有没有骨架"，而是"**有多少控制流决策是代码预定义 vs LLM 运行时决定**"——一个连续谱：

```
代码预定义控制流 ───────────────────────→ LLM 运行时控制流
[0% LLM]           [局部 LLM]            [100% LLM]
静态 Workflow      Agentic Workflow      Dynamic Workflow
  全 if/else        骨架静态+节点动态       全自主
```

**边界情况**：静态骨架 + 分支节点本身由 LLM 决策（如 LLM 路由分叉）→ 该节点已是 Dynamic 成分，但下游结构代码预定义 → 落在 **Agentic 区间**。它"像 Routing 又像 Dynamic"，因为它是 **Routing 拓扑 + LLM 路由决策** 的混合体。

**判定修正（面试加分）：数"决策"不数"节点"**。不要用 LLM 决策节点占比简单划分：

- **坑 1：节点权重不同**——1 个关键路由分叉节点（决定整个下游走向）可能抵过 10 个边角 LLM 节点
- **坑 2：看"关键路径分叉点"由谁决策**，而不是数节点比例：

| 维度 | 偏静态 | 偏 Dynamic |
|---|---|---|
| 路径分叉点 | 代码 if/else 预定义 | LLM 运行时决定 |
| 节点的下游边界 | 代码固定 | LLM 可自由扩展/跳过 |
| 分支数量 | 固定 | 不固定（LLM 想拆几个拆几个） |
| 终止条件 | 代码定义 | LLM 判断（Final Answer） |

**修正后的判断逻辑**：
1. 有固定拓扑 → 不是纯 Dynamic
2. 看关键路径分叉点由谁决策：大部分分叉是代码 → 偏静态；大部分分叉是 LLM → 偏 Dynamic；混合 → Agentic（中间）
3. 50% 混合 DAG → Agentic 区间，但具体靠哪边，看那部分 LLM 决策是否集中在关键分叉上

**一句话**：数"决策"不数"节点"——1 个 LLM 路由分叉可能比 10 个边角 LLM 节点更能把系统推向 Dynamic。

---

# 5. Agent 推理模式有哪些？ReAct 是啥？具体是怎么实现的？

## 一句话回答

推理模式从基础到进阶：直接输出 → **CoT**（先把推理过程写出来再给答案）→ **ReAct**（在 CoT 基础上加"行动"，思考↔工具调用交替循环）。ReAct 是目前 Agent 用得最广的模式：**推理过程可见 + 能动态利用外部工具**。

## 各章节重点

### 推理模式存在的根本原因
- LLM 本质是逐个 token 预测；多步推理时若"一口气"给答案，中间推导是隐式的，误差会悄悄累积（心算 vs 笔算）
- 推理模式 = 用不同方式让 LLM **把隐式思考显式化**，减少多步推理的累积误差

### CoT（Chain of Thought，2022 Wei 等）
- 核心：prompt 加"让我们一步步思考"，LLM 先写推理步骤再给答案
- 有效原因：LLM 输出顺序生成，先写的推理进入上下文，成为后续生成的依据（纸上的每一行在帮你记住上一步）
- 两种触发：
  - **Zero-shot CoT**：直接加"一步步思考"一句话，零成本即插即用，但格式/深度不稳定
  - **Few-shot CoT**：给几个带完整推理过程的例子让 LLM 模仿，效果稳定，适合固定格式，代价是准备示例 + 占 token
- **根本局限**：纯文字推理，无法与外部世界交互（拿不到实时数据、不能执行计算/访问数据库），接工具需要外部胶水

### ReAct（Reasoning + Acting，2022 Yao 等）
- 核心：在推理链里插入真实"行动"，按 **Thought → Action → Observation** 循环推进，直到 LLM 判断完成
- 对比：
  - 纯 CoT：只在脑子里推理，拿不到真实数据，易幻觉（无外部事实校准）
  - 纯 Act-only：直接输出工具调用序列无思考，动作序列脆弱（一步搜歪后面全跑偏，HotpotQA 上准确率低于 CoT/ReAct）
  - ReAct：推理为行动提供方向，行动为推理提供事实，闭环互补
- **实现原理（面试核心）**：循环**不是模型自己在转，是代码驱动的**！模型每次只输出 Thought + Action；你的代码负责检测输出（有无 Final Answer）→ 解析 Action → 执行工具 → 把 Observation 填回历史 → 再次调用 LLM，一轮轮转。典型 prompt 用文本格式约束（`Thought:/Action:/Action Input:/Observation:/Final Answer:`）
- **现代实现**：GPT-4/Claude 3 后原生支持 Function Calling / Tool Use，模型直接输出结构化 JSON 工具调用，不再解析文本格式——本质循环没变，只是"行动"从解析文本变成解析 JSON
- **两个实战坑（面试主动说出加分）**：
  1. **循环漂移**：每步局部决策、无全局计划约束，步骤越多历史越长越容易被无关信息带偏（查苹果营收，被搜索结果带去搜三星）
  2. **错误传播**：每步决策基于前面所有结果，早期一步错了后面全被带跑；ReAct 无内置"回头检查"机制
  - 根源：ReAct 是纯粹**前向推理**，无全局规划也无反思

### Plan-and-Execute（先规划再执行）
- 类比：ReAct = 边走边问路（可能被小吃摊带偏），P&E = 先看地图再出发（封路也能绕路不迷路）
- 两阶段：
  - **Planner（规划）**：LLM 只做规划不执行工具，输出结构化步骤列表（含依赖关系）
  - **Executor（执行）**：按计划逐步执行；每步内部仍可用 ReAct 模式跑，但执行器知道自己处于整体计划哪一步
- **关键：动态重规划**——每执行完一步检查结果是否与预期一致，不一致就把已有结果+剩余步骤重新交给 Planner 调整（导航遇到封路自动重规划）
- 选型：
  - ReAct：任务边界不明确、需探索性获取信息（开放问答、信息搜索）；代价是漂移 + 每步全量历史 token 线性增长
  - P&E：目标明确、多步骤协作的复杂任务（深度研究、长文写作、多工具数据分析）；代价是初始规划多一次调用，简单任务反而多余
- **工程常见混合**：P&E 做全局规划（大模型 GPT-4/Claude Opus 保证质量）+ ReAct 做每步执行（小模型控制成本）→ 可降 70%-90% LLM 调用成本

## 面试总结（答题要点）

- **最大误区**：以为模型自己在"循环"——模型每次只输出 Thought + Action，循环由代码框架驱动
- **必说两点**：① ReAct 本质 = 思考→行动→观察循环，显式推理 + 动态调工具，解决 CoT 纯文字局限；② 循环由代码驱动（解析、执行工具、填回 Observation、再调模型）
- **加分**：主动提 ReAct 两个局限（循环漂移、错误传播）+ P&E 如何解决漂移 + 实际工程混合（规划大模型/执行小模型）

---

# 6. ReAct、Plan-and-Execute、Reflection 三种范式有什么核心区别？实际项目中该如何选型？

## 一句话回答

三者核心区别在「决策和执行的关系」，**解决的问题层次不同**：ReAct 边想边干（单步灵活，长任务易跑偏）；Plan-and-Execute 先想全再干（长流程不跑偏）；**Reflection 不是独立流程，而是给前两者加的"检查修正 buff"（提升输出质量）**。选型看三个维度：任务复杂度、流程确定性、输出质量要求。

## 各章节重点

### 设计范式 vs 推理模式（贯穿概念）
- **设计范式** = 搭 Agent 的顶层做事流程框架（整个系统按什么大逻辑跑）
- **推理模式** = Agent 每步干活时脑子里具体怎么思考（底层思考逻辑）
- 关系：设计范式是管理制度，推理模式是员工干活方法，一一对应

### 一、ReAct 单步迭代范式（基础款）
- 本质：思考→行动→观察→再思考 循环，走一步看一步，无提前定死的完整计划
- 与 P&E 区别：没有"提前做完整规划"环节，规划和执行混在一起
- 与 Reflection 区别：循环里没有专门自我检查环节，只有行动后结果观察，不会停下来复盘
- 优势：实现简单、灵活度高、逻辑透明、新手入门零门槛
- 短板：长流程多步骤任务容易跑偏、陷入无效循环
- 适用：流程不固定、复杂度适中（信息搜索、问答助手、客服机器人）

### 二、Plan-and-Execute 规划执行范式（复杂任务款）
- 核心：把"规划推理"和"执行推理"完全解耦——专门一个 LLM 做规划（拆执行清单），另一个 LLM/模块按清单执行
- **强模型规划、弱模型执行**（实用技巧）：规划对推理要求高用 GPT-4/Claude，执行任务已具体用便宜小模型 → 总成本降 70%-90%（规划只调一次、执行多次但都便宜）
- 与 ReAct 区别：先完整计划再分步执行，全程不偏离目标；ReAct 无法做差异化模型分配
- 优势：结构清晰、链路可控、复杂长任务不易跑偏、无依赖步骤可并行、完成准确率通常高于 ReAct
- 代价：灵活度不如 ReAct（计划外情况易卡住）、实现复杂、token 增加
- 适用：流程长复杂度高（竞品分析报告、全流程项目开发、多维度行业调研）

### 三、Reflection 反思迭代范式（质量增强款）
- **关键定位（面试重点）：不是独立完整流程，而是给 ReAct/P&E 加的 buff**——不改变做事流程，只加一层"自我检查、自我修正"
- 核心闭环：生成 → 评估 → 改进（专门检查输出达不达标，不达标重试/调整策略）
- 与前两者本质区别：前两者核心是"把事做完"，Reflection 是"把事做好"
- 优势：输出质量明显提升（幻觉、逻辑错误、细节遗漏减少）
- 代价：至少多一次 LLM 调用，token 和延迟线性增加；无轮次限制易陷入"为改而改"死循环 → 一般限制最多反思 2-3 轮
- 适用：质量要求极高不能出错（生产代码、商业报告、法律文书）

### 进阶一：动态 Replan
- 解决 P&E 痛点：计划定死了中途遇意外怎么办（如执行到第三步发现竞品已被收购）
- 做法：每步执行完，把"当前结果 + 剩余计划"交给规划模块判断是否需要调整，需要就生成新剩余计划替换
- 保留 P&E 结构优势 + 不因计划僵硬翻车；代价是每步多一次评估计划的 LLM 调用

### 进阶二：Reflexion
- 比 Reflection 更深：不只"检查不对就重做"，还**把每次失败原因总结成经验教训存进记忆**，下次遇到类似任务作为上下文传给 LLM（错题本机制）
- 数据佐证：HumanEval 代码生成，GPT-4 pass@1 从 80% → 91%（+10 个百分点以上）
- 原理：代码可运行可测试，执行结果是最直接的反馈信号；**verbal reinforcement learning（语言强化学习）**——不需要梯度更新，从错误中学习，推理阶段提升质量

### token 消耗对比（5 步任务、每步约 2000 token 示例）
- **ReAct**：每轮带完整历史，输入 2000+4000+6000+8000+10000 = 30000 token，线性增长
- **P&E**：规划 3000 + 执行 5×1500（只带当前指令+前面结果摘要）= 7500 + 汇总 4000 ≈ 14500 token，比 ReAct 低一半多；配合强规划弱执行再降 70% 以上实际花费
- **Reflection**：每个反思节点至少多一次 LLM 调用，token 在基础范式上再增 30%-100%（反思两轮才通过则该步翻三倍）→ 设置上限
- 建议：先用 ReAct 快速验证跑通，再根据实际 token/延迟数据决定是否升级，不要一上来选最复杂方案

### 选型指南
- 任务不复杂、流程不固定、需实时调整 → **ReAct**
- 任务长、易跑偏、需整体结构清晰 → **Plan-and-Execute**（经常遇意外再 +动态 Replan）
- 输出要求高、不能出错 → 叠加 **Reflection**
- 需跨任务积累经验、避免重复犯错 → **Reflexion**
- 常见三层嵌套混合：P&E 定全局计划 + 每步用 ReAct 循环执行（单步可能多轮工具调用）+ 整体输出做一次 Reflection 检查（LangGraph 实现自然）
- **坑**：所有范式全堆一起 → 又复杂又慢又出 bug；工程"够用就好"，先把 ReAct 玩明白再按需加，别过度工程化

## 面试总结（答题要点）

- **第一步先说清 Reflection 定位**：不是独立流程，是可叠加在 ReAct/P&E 上的质量增强 buff（很多人搞错）
- 再按维度对比：ReAct 边想边干灵活高但长任务易跑偏；P&E 先规划再执行结构清晰但灵活不足；Reflection 解决输出质量，代价是 token 和延迟
- 追问展开：动态 Replan 解决"计划太僵硬"、Reflexion 通过错题本实现跨任务经验积累、三种范式 token 消耗差异
- 选型口诀：任务简单用 ReAct，流程长复杂用 P&E，输出要求高加 Reflection，**"别过度工程化、够用就好"**（显实际经验）

## 延伸理解（补充自学习讨论）

### Reflection / Reflexion 与 ReAct、P&E 的关系（自学习确认）
- **Reflection 和 Reflexion 都是叠加在 ReAct 或 P&E 之上的增强机制**，程度不同：
  - Reflection：检查任务完成得对不对（生成→评估→改进闭环）
  - Reflexion：不只检查对错，还总结失败原因、把经验教训沉淀为记忆（下次作上下文）
- **ReAct 叠加反思加在哪**：ReAct 循环本身无自我检查环节，只有行动后结果观察；叠加 Reflection 后检查节点插在 **Observation 之后**（评估这步结果）或整个循环结束后（评估整体输出）
- **P&E 叠加反思加在哪**：每步执行后可评估，或整个任务执行完汇总输出后评估最终结果
- **动态 Replan vs Reflection 是正交机制，不是谁基于谁**：

| | 动态 Replan | Reflection / Reflexion |
|---|---|---|
| 改的是什么 | 计划本身（剩余步骤） | 输出质量（生成的内容） |
| 针对问题 | 计划太僵硬、遇意外 | 输出有错误/逻辑漏洞 |
| 属于哪个范式 | 只属于 P&E | 通用 buff，可叠加 ReAct 或 P&E |
| 触发时机 | 每步执行后评估计划 | 产出后评估内容 |

一句话：**Replan 改"路"（计划），Reflection 改"货"（输出）**，两者互不依赖、可自由组合。

**补充洞察（共用反思机制）**：Replan 和 Reflection 确实**共用同一个底层机制**——"审视现状 → LLM 判断 → 调整"的反思动作（代码结构几乎一样：把当前状态喂给 LLM，LLM 检查后产出调整）。区别只在**检查对象和目的**：

| | 动态 Replan | Reflection |
|---|---|---|
| 检查对象 | 计划（剩余步骤，未来要做什么） | 结果/输出（已生成的内容） |
| 目的 | 路线还对吗？要不要改道？ | 货合格吗？要不要返工？ |
| 时间指向 | 面向未来（调整接下来的步骤） | 面向当下（修正已产出的内容） |
| 产出 | 新的步骤列表（继续往前走） | 反馈/重做（改进当前输出） |

- 类比：Replan 像导航（检查当前位置偏离计划就重规划新路线），Reflection 像质检员（检查产品合格就放行、不合格退回重做）
- 术语澄清：正因为审视动作相同，部分文献把 Replan 也归入广义"反思类方法"；但工程/面试中仍分开说——**Replan 属 P&E 的机制，Reflection 是通用质量增强**
