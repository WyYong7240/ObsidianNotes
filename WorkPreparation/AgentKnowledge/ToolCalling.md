---
tags:
  - Tools
  - 面试八股
source: https://xiaolinnote.com/ai/tools/tools_info.html
topic: LLM 工具调用面试题系列（持续追加）
---
# LLM 工具调用面试题系列

> 本文件与 `AgentBasic.md` 并列，存放「03｜ LLM 工具调用面试题」系列的问题（FC 原理、训练机制、MCP、Skill、A2A 等），格式与 Agent 基础系列一致，持续追加。

---

# 1. LLM 是如何学会调用外部工具的？Function Call 是怎么训练出来的？（SFT + RLHF）

> 本条目合并「LLM 工具调用面试题」系列第 2 题（如何学会调工具）与第 3 题（FC 能力怎么训练出来）——两题都是讲工具调用训练，第 3 题是第 2 题的深化（训练数据场景覆盖、数据来源、PPO、RLAIF 细节），合并沉淀。

## 一句话回答

工具调用能力不是涌现能力，是专门「教」出来的，分两个训练阶段：**SFT（监督微调）教会「怎么调」**——喂大量工具调用示范对话，让模型学会识别工具定义、判断要不要调、输出结构化 JSON 请求；**RLHF（人类反馈强化学习）教会「什么时候调」**——用人类打分训练奖励模型，再用强化学习塑造「能直接回答就不调工具」的边界感。运行时靠 **Function Calling** 机制落地：模型只负责输出决策 JSON（`tool_calls`），真正执行工具的是宿主代码。

## 详细解析

### 为什么原始 LLM 不会调工具

- 预训练只在文本空间学「预测下一个 token」，从未见过工具调用这种输出模式
- 没训练过的模型只会输出自然语言描述（"我需要调用天气 API"），不会输出**可被程序解析的结构化 JSON**——JSON 格式在预训练语料里不存在，参数量大也不会自动涌现
- 结论：工具调用能力 = 后天教出来的（SFT 教怎么调，RLHF 教什么时候调）

### 第一阶段：SFT（监督微调），让模型「见过」工具调用

- 一条完整训练样本的结构：
  1. **System**：工具说明书（有哪些工具、每个工具的名称/功能/参数）——模型从这里「认识」工具
  2. **User**：用户提问（"北京今天天气怎么样？"）
  3. **Assistant 调用请求（关键）**：结构化 JSON `{"tool_calls": [{"name": "get_weather", "arguments": {"city": "北京"}}]}`——答案是 JSON 而非自然语言，因为格式固定、机器好解析
  4. **Tool**：模拟工具返回（"晴，15°C，东北风3级"）
  5. **Assistant 最终回答**：基于工具结果组织自然语言答案
- 几十万~上百万条样本反复训练，学会整套流程；数据来源：人工标注（种子数据，成本高质量好）+ 强模型（如 GPT-4）批量生成（主流，成本低量大）

#### 训练数据需要覆盖哪些场景（多样性决定能力上限，不能只有「正常调一个工具」）

1. **单工具调用**：基础入门款，一个问题对应一个工具
2. **多工具并行调用**：如「查北京和上海的天气」应一次性输出两个调用请求，而不是傻乎乎一个个来——没见过这种样本就不知道可以并行
3. **工具调用失败后的处理**（容易忽略但关键）：API 超时、参数格式不对、权限不足等错误，模型要能识别错误信息并换方式处理，而不是崩掉或傻傻重复同样调用
4. **不需要调工具、直接回答**（很多人想不到但非常重要）："1+1 等于几""帮我总结这段话"完全不需要工具；全正例会形成「遇到问题就调工具」的惯性
5. **多轮对话中的工具调用**：上下文已有工具结果时，要能正确理解和引用之前的结果，而不是无视历史重新调用

缺哪个场景，模型就会在哪个场景下翻车——覆盖程度直接决定实际表现。

#### 训练数据从哪来（两种方式）

- **人工标注**：雇标注员写出正确调用示例——质量好（人写准确度有保障），但成本极高，通常只用于核心种子数据，无法大规模扩展
- **模型自动生成**（Self-Instruct / Distillation 蒸馏）：用已具备 FC 能力的强模型（GPT-4）批量生成样本 + 人工抽查——业界主流，成本低、量大；**隐患：模型蒸馏的幻觉传递**——上游模型生成错误样本，下游模型会一起学进去，所以抽查不能省

### SFT 的短板：会了，但不知道「该不该调」

- 训练样本里「该调」占绝大多数（教的就是调工具）→ 过拟合「积极调用」倾向
- 模型对"1+1 等于几"也去调计算器；遇到工具调用失败也不知道怎么处理——**行为边界感弱**
- 训练信号只告诉"这是正确答案"，没告诉"不调也是正确答案"

### 第二阶段：RLHF，用反馈建立边界感（四步）

1. **生成多样回答**：同一问题生成多种处理方式（调了/直接答/参数填错），故意覆盖各种情况
2. **人类打分**：标注员判断哪种更合理（"1+1"直接答最好，"北京天气"调工具才对），记录人类偏好
3. **训练奖励模型**：用打分数据单独训练一个小模型当「裁判」，只打分不回答；**关键点**：奖励模型的判断力是从人类标注偏好里"蒸馏"出来的——标注员标准不稳定，裁判就学歪，主模型再被优化方向也歪
4. **强化学习优化主模型**：用奖励模型分数反复调整主模型参数，趋向产出"高分回答"（边界感更准的工具调用）

- 效果：能直接回答就直接回答，需要实时数据/执行操作才调

#### 为什么偏偏用 PPO（近端策略优化）

- 强化学习算法很多，选 PPO 两个务实理由：
  1. **训练稳定**：传统策略梯度算法单步更新太大容易把模型直接调废；PPO 相对不容易崩
  2. **内置 KL 散度约束**：强迫新模型和旧模型的输出分布不要差太远，防止「为了讨好奖励模型把自己训成只会重复几句套话的怪胎」的退化
- 本质：RLHF 是让模型在「追求高奖励」和「保持语言能力」之间走钢丝，PPO 是目前公认好用的平衡工具

#### RLAIF：用 AI 代替人工打分（RLHF 的改进版）

- 背景：RLHF 最大痛点是**人工标注成本极高**（需专业背景、打分慢、价格贵，难大规模扩展）
- 做法：用更强的 AI 模型（如 GPT-4）代替人类标注员打分——成本能低 **10~100 倍**，速度快得多
- 代价：**「AI 的偏见会传递」**——打分 AI 对某些场景判断有偏差或盲区，这些偏差会被模型学进去（例：AI 评委觉得"遇到数学题都该调计算器"，模型就学到这倾向，哪怕简单算术不用调）→ 打分 AI 的质量和评分标准设计很关键
- 业界实践：**混用**——关键数据用人工保质量，量大的地方用 AI 提效率，两者互补

### 两个阶段各司其职（缺一不可）

- **只有 SFT 没有 RLHF**：模型可能遇到什么问题都冲动地调工具
- **只有 RLHF 没有 SFT**：模型连工具调用的格式都输不出来，奖励信号根本没地方发力
- 两个阶段配合，才能训练出「知道怎么调、也知道什么时候该调」的工具使用能力

### 运行时：训练好之后怎么用（Function Calling）

1. 应用把工具 schema（JSON 说明书）连同用户问题发给模型
2. 模型判断需要工具 → 输出 `{"tool_calls": [{"name": ..., "arguments": {...}}]}` 后**停止**
3. **代码**解析 JSON → 找到函数 → 真正调用天气 API
4. 把结果塞回对话历史 → 再次调用模型 → 模型组织最终回答

### 关键认知：模型只「决策」，不「执行」

- 模型全程只做一件事：判断调哪个工具、参数填什么，用 JSON 输出决策
- 执行（跑函数、访问网络、查数据库）是宿主程序代码——可做权限控制、参数校验、执行沙箱
- 分工合理：LLM 擅长意图推理，但不应直接拥有操作系统资源权限；这也是主流工具调用框架的核心设计原则

## 延伸理解（补充自学习讨论）

### 【重点】「想说 vs 想做」：靠什么判断？（自学习确认）

用户疑问「模型输出不是 JSON 就直接输出，能 JSON 解析就走工具逻辑？」——**方向对，但主流 API 不是靠「内容能否被 JSON.parse」判断，而是靠响应对象里的专用结构化字段**：

| API | 「想做」（调工具）的信号 | 「想说」（直接回答）的信号 |
|---|---|---|
| OpenAI 格式 | `message.tool_calls` 非空 | `tool_calls` 为空/不存在 |
| Claude 格式 | `stop_reason == "tool_use"`，content 有 `<tool_use>` block | `stop_reason == "end_turn"` |
| 手搓/非结构化模型 | 约定的文本格式（纯文本 JSON / `Action: xxx`），此时才靠「尝试解析」 | 解析失败视为普通回答（兜底启发式） |

- **为什么不是「JSON 可解析」**：普通回答也可能是合法 JSON（如让模型输出 JSON 配置），按"能否解析"会误判；真正区分的是**字段而非内容**——模型被训练成两种输出约定：想说话填 `content`，想调工具填 `tool_calls`，在 API 返回结构层就分开了
- **代码判断示例**（KubernetesAgent 项目 `route_after_ops`）：`hasattr(last_message, 'tool_calls') and len(last_message.tool_calls) > 0` → 走工具执行；否则视为已得出结论
- **JSON 解析出现在两个更内层的位置**：① 走 tool_calls 分支后，`arguments` 是 JSON 字符串，代码要二次 `json.loads(arguments)` 解析参数；② 非结构化模型/兜底场景才用「解析成功=调用意图」的启发式
- **兜底必要性**：模型输出的 arguments 可能是非法 JSON、可能编造不存在的工具名 → 框架必须做解析校验、重试、纠错（呼应第 1 题「FC 的 API 定义与真实输出之间的距离」）

## 面试总结（答题要点）

- **三个雷**：① 把「涌现能力」当工具调用能力——工具调用要输出结构化 JSON，预训练学不到，必须专项训练；② 只知道 SFT 忽略 RLHF——SFT 解决会不会调，RLHF 解决该不该调，缺一不可；③ 以为训练数据只覆盖单工具调用就够了——多工具并行、失败重试、不需要工具直接回答、多轮对话调用，缺哪个就在哪个场景翻车
- **核心讲清两阶段作用**：SFT 通过「system 工具定义 + user 问题 + assistant JSON 调用 + tool 执行结果 + assistant 最终回答」的完整对话样本（反向传播学习），让模型学会整套流程；RLHF 通过「人类对多种回答偏好排序 → 训练奖励模型 → PPO 强化学习调整主模型」，建立「能直接回答就不调」的边界感
- **训练数据来源**：人工标注（质量高但成本高，用于种子数据）+ 模型自动生成（Self-Instruct / Distillation，成本低量大，注意幻觉传递风险）
- **加分点**：① 补充 RLAIF 作为 RLHF 的低成本替代（AI 打分代替人类标注，成本低 10~100 倍，注意 AI 偏见传递）；② 能说出为什么用 PPO（训练稳定 + KL 散度约束防退化）
- **运行时**：Function Calling 机制——模型决策输出 JSON，代码执行，分工是关键认知

---

# 2. 什么是 MCP（模型上下文协议）？讲讲它的核心内容？

## 一句话回答

MCP 是 **Anthropic 2024 年底推出的开放协议**（非框架、非 Anthropic 专属），解决「AI 接工具太碎片化」的问题：工具提供方按协议实现一个 Server，任何支持 MCP 的客户端（Claude Desktop、Cursor、各种 Agent 框架）直接接入，**一次实现、到处复用**。采用 **Client-Server 架构**（一个 Client 可连多个 Server），暴露三类能力：**Tools**（有副作用的操作）、**Resources**（只读数据）、**Prompts**（提示词模板）；底层通信用 **JSON-RPC 2.0**，传输支持 **stdio**（本地子进程）和 **Streamable HTTP**（远程，早期 HTTP+SSE 双端点已 deprecated）。

## 详细解析

### 没有 MCP 之前，接工具有多麻烦

- 给 Claude 接 GitHub：手写 API 调用、处理认证（OAuth token）、处理返回格式、转成模型能理解的格式
- 模型升版接口变化 → 对接代码要改；接了十个工具 → 十套各自为政的代码；换客户端（Claude → Cursor）→ 全部重写
- 真实状态：**碎片化、难复用、强绑定**——每个工具、每个模型都是一座孤岛，接一个新工具就要重新搭一座桥

### MCP 的核心思路：定一套行业标准接口（USB 类比）

- USB 之前：鼠标、键盘、打印机各用各的接口，换电脑就愁兼容；USB 之后：外设统一接口，厂商做一次适配，全球都能用
- MCP 同理：工具方按规范实现 MCP Server，任何支持 MCP 的客户端自动发现工具并使用，无需定制对接代码

### Client-Server 架构

- **Server** = 工具实现方（GitHub 官方维护 GitHub MCP Server，封装「列出 PR」「创建 Issue」「搜索仓库」等操作）
- **Client** = AI 应用侧（Claude Desktop、Cursor），连上 Server 自动获得工具能力
- 一个 Client 同时连多个 Server（文件系统 + GitHub + PostgreSQL），配置文件加几行 JSON、重启即可，零代码

### 三类核心能力：Tools、Resources、Prompts

| 能力 | 本质 | 例子 | 授权策略 |
|---|---|---|---|
| **Tools** | **有副作用的操作**（执行后改变外部世界状态、往往不可逆） | 创建文件、提交代码、发 Slack、调第三方 API | 通常需用户授权确认 |
| **Resources** | **只读数据**（无副作用，把数据提供给模型看） | 读日志、查数据库记录、获取文档内容 | 可宽松暴露，不需谨慎授权 |
| **Prompts** | **提示词模板**（带参数占位符，复用优质 prompt） | 团队代码审查标准 prompt（参数：编程语言+代码内容） | —— |

### 底层通信：JSON-RPC 2.0

- 轻量级远程函数调用协议：Client 发 JSON 请求（调哪个方法、参数、请求 ID）→ Server 执行 → 返回 JSON 响应（结果或错误）
- 用 JSON 而非二进制：易读、易调试、语言无关；2.0 版比 1.0 增加批量请求、通知消息

### 传输层两种方式 + 演进

- **stdio**：Server 作为本地子进程，Client 通过管道通信（stdin 读消息、stdout 写结果）——适合本地工具，启动快延迟低（Claude Desktop 接本地 Server 用这个）
- **Streamable HTTP**：Server 作为 HTTP 服务远程部署——适合远程工具/多 Client 共享一个 Server
- **演进**：早期（2024-11-05 规范）是「HTTP + SSE」双端点（一个 GET 开 SSE 长连接收推送 + 一个 POST 发请求）；2025 年 3 月更新为单端点 **Streamable HTTP**（老的 HTTP+SSE 标记 deprecated 但保留兼容）——不是抛弃 SSE，而是把两个端点合并成一个 `/mcp`：Client 用 POST 发请求，短请求直接回普通 JSON，长请求把响应升级为 SSE 流持续推送中间结果。架构更简洁、部署更友好（一个端点、serverless 也能跑）

### MCP 生态发展快的原因

1. **极低的实现门槛**：Anthropic 开源协议规范 + 多语言 SDK（Python/TypeScript），写一个最简单 Server 不到 30 行
2. **头部工具第一时间跟进**：GitHub、Slack、PostgreSQL、Puppeteer、Google Maps 等都有官方/社区 Server，配置几行 JSON 零代码接入 → 工具多了开发者更愿意采用 → 正向循环

## 延伸理解（补充自学习讨论）

### 【重点】MCP 为什么要提供 Prompts？和 Skills 像吗？（自学习辨析）

- **为什么需要 Prompts**：MCP 三类能力覆盖「模型与外部世界交互」的三种需求——Tools 是"让模型做事"（执行），Resources 是"给模型资料"（读取），Prompts 是"**教模型怎么用这些的引导模板**"（复用提示词工程）。它解决的不仅是个人省事，而是把**组织沉淀的优质 prompt 资产通过协议标准化**：Server 声明"我有这些模板、带这些参数"，任何 Client 都能发现并调用展开——让 prompt 像工具一样可发现、可复用、可统一标准（团队协作场景价值大）
- **和 Skills 确实很像，但层级不同**：
  - **MCP Prompts** = 协议层（transport-level）的**纯提示词模板**暴露机制：一段模板 + 参数占位符，调用后展开成文本
  - **Agent Skill**（如 Claude Code Skills / Anthropic Agent Skills）= 客户端/产品层的**能力包**：prompt 指令 + 可选的脚本/资源/工作流 + 元数据，由客户端加载注入
  - 关系：Skills 是比 MCP Prompts 更丰富的封装（模板 + 配套能力），Prompts 是其中最纯粹的形式（纯模板）；两者都属"预设提示词复用"，**用户的直觉（"像"）成立，区别在粒度与载体**——Skills 可看作"prompt + 资源"的打包，甚至可以通过 MCP 暴露（详见第 9/10/11 题）
- **⏳ 待追问（已标记）**：用户保留此疑问——后续看到第 9/10/11 题（Skill 是什么 / MCP vs Skill / FC-Skill-MCP 区别）时，回到这里对比「MCP Prompts vs Agent Skill 到底差在哪」

### 【重点】HTTP+SSE 双端点（一个 GET 一个 POST）具体怎么做

- 背景：MCP 底层是 JSON-RPC 2.0 消息，但**HTTP 是请求-响应模式**，client 主动发请求没问题，**server 要主动推送**（资源更新通知、工具执行进度、server 向 client 请求权限确认）时单向请求-响应做不到 → 用 SSE 长连接补一条 server→client 的通道
- 双端点分工：
  - **POST 端点**（如 `/mcp`）：Client 把 JSON-RPC 请求（method/params/id）POST 过去，Server 处理并返回（普通 HTTP JSON 响应）——负责 client → server 方向
  - **GET 端点**（如 `/sse`）：Client 先发起 GET 建立 SSE 长连接（保持打开），Server 通过这个流推送 server → client 的消息（异步结果、通知、server 主动发起的事件）
- 典型流程：① Client GET /sse 建立连接 → ② SSE 流里 Server 发一个 `endpoint` 事件，告知"请求发到这个 URL"（通常就是 POST 端点）→ ③ Client 用 POST 发 JSON-RPC 请求 → ④ 短响应直接作为 POST 的 HTTP 响应返回；异步/推送消息走已建立的 SSE 连接 → ⑤ 用会话 ID（header）把 GET 的 SSE 连接和 POST 请求关联起来
- 本质：**一个常驻 SSE（server→client）+ 一个普通 POST（client→server）**，两个端点绑在一起工作

### 【重点】Streamable HTTP 合二为一后怎么做

- 只有一个端点（`/mcp`），Client 用 **POST** 发请求；Server 按响应类型灵活处理：
  - **短请求**：直接返回普通 JSON（`Content-Type: application/json`）
  - **长/流式请求**：把 HTTP 响应升级为 **SSE 流**（`Content-Type: text/event-stream`），持续推送中间结果/增量
- 为什么能合并：HTTP 本身是双向的（请求 client→server，响应 server→client）。早期双端点是为了"server 任意时刻主动推送"；Streamable HTTP 的洞察是——**大部分场景 client 只需"发一次请求、收完整响应（或流式响应）"，不需要常驻推送通道**，于是把"先建 SSE 再 POST"的两步变成"一次 POST，响应按需流式"（**响应式 SSE 取代常驻 SSE**）
- 会话维持：POST 响应里 Server 返回 `Mcp-Session-Id`，Client 后续请求带上关联；如果确实需要接收 server 主动推送（如资源订阅通知），仍可额外开一个挂起的 GET 作为 SSE 流，用同一 session id 关联
- 好处：一个端点、部署简单、serverless 友好；**本质还是 HTTP + SSE，只是用法从"双端点绑定"变成"单端点按需流式"**
- **规范细节（官方 2025-03-26 规范确认）**：
  - **响应格式的选择权在 Server，不是 Client**——规范原文："If the input contains any number of JSON-RPC requests, the server **MUST** either return `Content-Type: text/event-stream` ... or `Content-Type: application/json`"。Server 根据「能否立即完成 + 是否需要推送中间消息」自行决定
  - **Client 不传 stream 参数**——POST body 是 JSON-RPC 消息本身（request/notification/response 或批量数组），没有流式开关；Client 通过 **`Accept: application/json, text/event-stream`** 头声明「两种响应都能接受」（两个类型必须都列），具体拿到哪种由 Server 决定
  - **SSE 流的规则**：流内**每个 request 必有一个对应 response**（可批处理，最后一个事件就是最终结果）；Server 可在 response **之前**发 requests/notifications（如执行进度、Server 反向向 Client 请求权限确认）；**全部 response 发完才关闭流**
  - **GET 挂起流**：只用于 Server 主动推送（notifications/requests，与 POST 请求无关）；该流上 **MUST NOT** 发 response（除非是恢复断连流的重放）；Server 不支持时返回 405
  - **与 LLM API 的 stream 参数对比**：OpenAI 的 `stream=true` 是 **Client 主动要求**流式（生成式输出，Client 要边收边用）；MCP 是 **RPC 调用**，只有 Server 才知道该请求是否需要流式推送，所以是 **Server 决定 + Client 用 Accept 头声明兼容**——两种设计刚好相反
- **【重点】传输演进 + Client 自适应（结论）**：
  - Server 端：早期「HTTP+SSE」双端点（GET 常驻 SSE + POST 请求）→ 现在 **Streamable HTTP 单端点、双方法（POST/GET）**：POST 发请求、响应二选一（JSON 或 SSE）；GET 是可选的挂起 SSE 流（server 主动推送，不支持返 405）
  - Client 端**必须**自适应（规范 MUST support both）：① 内容层——按响应头 `Content-Type` 分流（`application/json` 直接解析；`text/event-stream` 按 SSE 逐个收事件、为每个 request 收集对应 response）；② 协议层——新旧 transport 探测切换（先 POST initialize，4xx 则改 GET 等 `endpoint` 事件确认旧协议）；③ 连接层——POST 用于请求 + 可选 GET 流用于接收主动推送

## 面试总结（答题要点）

- **最大的雷**：把 MCP 和 Function Calling 搞混——FC 解决「模型怎么输出结构化的工具调用请求」，MCP 解决「工具怎么标准化接入、一次实现到处复用」，不同层面；另一个雷：以为 MCP 是 Anthropic 专属——它是开放协议，任何支持 MCP 的客户端都能接入
- **先说解决的核心问题**：工具接入碎片化（每接一个新工具单独写对接代码，换客户端又重写）
- **架构**：Client-Server，Server 是工具实现方、Client 是 AI 应用侧，一个 Client 连多个 Server
- **三类能力要能区分**：Tools 有副作用（需授权）、Resources 只读（无副作用）、Prompts 可复用提示词模板
- **底层**：JSON-RPC 2.0；传输 stdio（本地）和 Streamable HTTP（远程）；早期 HTTP+SSE 双端点已 deprecated，2025 年 3 月起推荐单端点 Streamable HTTP
- **加分点**：MCP 生态发展快的原因——实现门槛极低（开源规范 + SDK、30 行写一个 Server）+ 头部工具第一时间跟进（正向循环）

---

# 3. MCP 由哪几部分组成？（Host / Client / Server + Tools / Resources / Prompts）

> 本条目重点回答「MCP 的组成」这道面试题。可以把 MCP 拆成三层：**角色架构层**说明谁负责什么，**能力层**说明 Server 能提供什么，**协议层**说明消息长什么样以及如何传输。

## 一句话回答

MCP 不是简单的「Client + Server」二元结构，而是由三类角色、三类能力和一套通信协议组成：**Host** 是 Claude Desktop、Cursor 等 AI 宿主应用；**Client** 是 Host 内部负责连接某个 Server 的通信模块；**Server** 是工具提供方实现的独立进程。Server 可以暴露 **Tools（有副作用的操作）**、**Resources（只读数据）** 和 **Prompts（提示词模板）**。底层消息统一采用 **JSON-RPC 2.0**，本地通常使用 **stdio**，远程使用 **Streamable HTTP**。

## 详细解析

### 第一层：角色架构——Host / Client / Server

#### Host：AI 应用本身

- Host 是整个 MCP 系统的宿主，例如 Claude Desktop、Cursor、Windsurf 或自研 Agent 应用
- 负责管理 MCP Client、决定连接哪些 Server、维护连接生命周期，并把 Server 能力提供给模型使用
- Host 不直接承担每个 Server 的协议细节，而是通过内部 Client 统一管理连接

#### Client：Host 内部的连接模块

- 一个 Client 通常对应一个 Server 连接
- 负责初始化连接、能力发现（查询 Server 提供哪些 Tools / Resources / Prompts）
- 负责把模型或 Host 的请求转发给 Server，再把结果带回 Host

#### Server：能力提供方

- Server 是独立运行的程序，对外暴露工具、资源和提示词模板
- Server 不需要关心上层究竟是 Claude Desktop、Cursor 还是其他 Host，只需按 MCP 协议响应 Client
- 一个 Host 可以连接多个 Server；每个 Server 通常由一个对应的 Client 负责通信

> **面试辨析**：Host 不是 Client 的别名。Host 是完整的 AI 应用，Client 是 Host 内部负责与某一个 Server 通信的模块。可以把 Host 理解为公司，Client 是公司派出的联络员，Server 是外部服务提供方。

### 第二层：能力类型——Tools / Resources / Prompts

| 能力 | 本质 | 典型例子 | 主要特点 |
|---|---|---|---|
| **Tools** | 可执行的操作 | 创建文件、提交代码、发送消息、调用 API | 通常有副作用，可能改变外部状态，通常需要授权确认 |
| **Resources** | 可读取的数据 | 日志、文档、数据库记录 | 只读、无副作用，主要为模型提供上下文 |
| **Prompts** | 可复用的提示模板 | 代码审查模板、数据分析模板 | 带参数占位符，调用时展开为完整提示词 |

可以用一句话记忆：**Tools 改变世界，Resources 观察世界，Prompts 结构化表达。**

其中 Tools 与 Function Calling 中的函数比较接近，但 MCP 不止提供 Tools；Resources 和 Prompts 也是协议标准化的一部分。把 Server 暴露的所有能力都笼统称为「工具」，会丢失 MCP 的设计意图。

### 第三层：协议与传输——JSON-RPC 2.0 + stdio / Streamable HTTP

#### JSON-RPC 2.0：规定消息格式

JSON-RPC 2.0 规定了 Client 和 Server 之间消息的结构。请求中包含方法名、参数和请求 ID，响应中携带结果或错误，并通过 ID 对应请求与响应：

```json
// Client 查询工具列表
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}

// Client 调用工具
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"read_file","arguments":{"path":"/tmp/log.txt"}}}
```

JSON-RPC 与传输层是解耦的：同一套消息格式既可以通过本地管道传输，也可以通过 HTTP 传输。

#### stdio：本地进程通信

- Server 作为本地子进程启动
- Client 通过标准输入输出与 Server 通信：stdin 接收请求，stdout 返回结果
- 不需要监听端口，延迟低，适合个人电脑上的文件系统、Git 等本地工具

#### Streamable HTTP：远程服务通信

- Server 作为独立 HTTP 服务部署，多个 Client 可以共享同一个远程 Server
- Client 通过 HTTP POST 发送 JSON-RPC 请求
- Server 可以直接返回 JSON，也可以返回 SSE 流来传递长时间运行任务的中间结果
- 相比旧的 HTTP + SSE 双端点方案，单端点部署更简单，也更适合负载均衡和 serverless 环境

**早期 MCP 使用 HTTP + SSE：POST 端点负责发送请求，GET 端点建立 SSE 长连接接收 Server 推送。**2025 年 3 月规范更新后，推荐使用 Streamable HTTP；旧方案被标记为 deprecated，但通常仍需考虑兼容性。

> StreamableHTTP并不是抛弃SSE，而是把双端点合并成一个/mcp。Client用 POST发请求，Server根据情况灵活返回：短请求直接回普通JSON，长请求则把HTTP响应升级为SSE流持续推送中间结果。这样一个端点就能干完所有事，对负载均衡器和serverless环境都更友好。

## 整体关系

```text
Host（AI 应用）
  ├── Client 1 ── stdio / Streamable HTTP ── Server 1
  ├── Client 2 ── stdio / Streamable HTTP ── Server 2
  └── Client 3 ── stdio / Streamable HTTP ── Server 3
                                             ├── Tools
                                             ├── Resources
                                             └── Prompts
```

这里要区分两组概念：

- **Host / Client / Server** 是角色和连接关系
- **Tools / Resources / Prompts** 是 Server 暴露的能力类型
- **JSON-RPC / stdio / Streamable HTTP** 是通信协议和传输方式

## 面试总结（答题要点）

- 先给出三层结构：**角色层、能力层、协议层**
- 角色层：Host 是 AI 应用，Client 是 Host 内部的连接模块，Server 是独立的能力提供方；一个 Host 可以连接多个 Server
- 能力层：Tools 有副作用、Resources 只读、Prompts 是可复用的提示词模板
- 协议层：JSON-RPC 2.0 规定消息格式，stdio 和 Streamable HTTP 规定消息如何传输
- 关键辨析：Host 不等于 Client；Server 不只提供 Tools；MCP 的协议消息格式和传输方式彼此解耦
- 进阶加分：说明 HTTP + SSE 是旧的双端点方案，当前推荐 Streamable HTTP；后者仍可使用 SSE 流式返回，只是把请求与响应统一到一个端点

### MCP Prompts 和 Agent Skills 的区别（易混点）

MCP Prompts 和 Skills 都可以包含提示词，但解决的问题不同：

| 对比项 | MCP Prompts | Agent Skills |
|---|---|---|
| 定位 | 可被客户端发现和调用的提示词模板 | Agent 完成一类任务的能力包 |
| 主要维护位置 | 通常在 MCP Server，尤其适合远程集中维护 | 通常随 Skill 安装在 Agent 本地 |
| 内容 | 一组可参数化的 messages / Prompt 模板 | 指令、工作流、脚本、参考资料和模板 |
| 触发方式 | 用户或客户端调用 `prompts/get` | Agent 根据任务判断后加载和执行 |
| 运行能力 | 本身主要生成提示消息，不负责执行脚本 | 可以指导 Agent 调工具、运行脚本和组织完整流程 |

#### MCP Prompt 的参数化

MCP Prompt 可以预先定义模板，再由调用方传入运行时参数。例如：

```text
review_code(language, repository, code)
```

调用时传入当前语言、仓库和代码，Server 再生成完整 Prompt：

```text
请按照 payment-service 团队当前的 Go 代码规范审查以下代码……
```

因此，MCP Prompt 不一定只是简单的字符串替换。Server 还可以根据参数、团队、仓库或远程配置动态生成不同的消息内容。

#### 为什么有些场景更适合 MCP Prompt

例如企业统一维护代码审查规范：

```text
Claude Desktop ─┐
Cursor          ├── 远程 MCP Server ── 最新代码审查 Prompt
内部 Agent      ┘
```

只要不同的 Host / Client 连接这个 Server，就可以发现并调用同一套模板。规范更新时只需更新 Server，不需要给每个 Agent 单独重新安装 Skill。

#### 为什么 Skill 不能简单等价替代

Skill 也可以保存一个本地代码审查模板，但每个 Agent 通常需要安装和更新对应 Skill。如果 Skill 想从远程获取最新模板，还需要额外实现远程发现、连接、参数传递和版本管理逻辑。

因此可以这样理解：

```text
MCP Prompt = 远程、可发现、可参数化的 Prompt 服务
Skill      = 本地加载的 Agent 任务执行方法包
```

不过这不是绝对的技术限制。Skill 也可以调用远程服务，甚至封装一层 MCP Client；只是此时 Skill 负责“如何发现和调用”，远程服务负责维护 Prompt，整体上仍然是在使用远程 Prompt 能力。

#### 两者可以组合使用

复杂任务中，Skill 和 MCP Prompt 可以各司其职：

```text
Skill
  ├── 加载任务执行规则
  ├── 调用 MCP Prompt，填入当前上下文
  ├── 调用 MCP Tools 执行操作
  └── 根据结果生成最终报告
```

最容易记忆的一句话是：**MCP Prompt 解决“如何共享和调用一段结构化提示词”；Skill 解决“Agent 如何组织指令、工具、脚本和上下文完成一类任务”。**

来源：[小林面试笔记：MCP 由哪几部分组成？](https://xiaolinnote.com/ai/tools/5_mcp_components.html)

---

# 4. Function Calling 和 MCP 分别适合什么场景？

## 一句话回答

Function Calling 更像是**把工具直接写在当前应用里**：适合快速原型、工具数量少、只服务于一个应用、不需要复用的场景。MCP 更像是**把工具独立封装成标准 Server**：适合跨项目或跨团队复用、工具较多或复杂、已有现成 MCP Server，或者正在构建正式 Agent 系统的场景。

两者不是互相排斥的技术。MCP 负责工具的标准化封装和复用，模型在 MCP 场景下仍然可以通过 Function Calling 的方式产生工具调用决策。

## 核心区别：内嵌 vs 独立

| 对比项 | Function Calling | MCP |
|---|---|---|
| 工具位置 | 集成在应用代码中 | 独立运行在 MCP Server 中 |
| 接入方式 | 应用自行定义 schema、解析参数和执行函数 | Client 连接 Server，并通过协议发现能力 |
| 复用能力 | 通常需要为不同应用重复接入 | 一次实现，多个支持 MCP 的客户端复用 |
| 控制粒度 | 应用可以直接控制完整执行链路 | 工具逻辑与 Agent 应用解耦，独立维护 |
| 典型优势 | 简单、直接、定制方便 | 标准化、模块化、易管理、易复用 |

## Function Calling 的适用场景

以下情况通常直接使用 Function Calling 更合适：

1. **快速原型或 Demo**：只需接一两个工具，直接在应用代码中定义 schema 和调用函数，开发路径最短。
2. **工具只服务于一个应用**：例如一个内部应用专用的私有数据库查询接口，不会被其他项目复用，没有必要额外维护 MCP Server。
3. **需要精细控制执行逻辑**：权限校验、参数二次处理、特殊重试、链路追踪等逻辑都可以直接嵌入应用代码。
4. **部署环境受限**：如果云函数或 Serverless 环境不允许启动子进程，MCP 的 stdio Server 不方便部署，直接在主进程中执行函数更稳妥。

## MCP 的适用场景

以下情况更值得考虑 MCP：

1. **跨项目或跨团队复用**：同一套 GitHub、Slack、数据库或文件操作能力，需要被多个 Agent 或客户端使用。
2. **社区已有现成 MCP Server**：例如已有成熟的 GitHub、数据库或浏览器 MCP Server 时，直接配置复用，避免重复手写 API 对接代码。
3. **工具数量或复杂度逐渐增加**：工具 schema 和调用逻辑如果散落在应用各处，新增、修改和排错都会越来越困难；MCP 可以将工具集中管理并支持自动发现。
4. **正式的 Agent 系统**：Agent 往往需要同时接入文件系统、数据库、代码执行和外部 API 等多种能力，MCP 能把工具来源模块化，降低 Agent 核心逻辑与具体工具的耦合。

## 不要只看工具数量

「工具少用 Function Calling，工具多用 MCP」只能作为粗略经验，不能作为绝对规则。应综合考虑：

- 是否需要跨项目、跨团队复用
- 是否已经有可直接使用的 MCP Server
- 工具的复杂度和调用链是否变长
- 团队规模、接口变更频率和维护成本
- 部署环境是否允许运行独立进程
- 当前是在做 Demo，还是在建设长期运行的 Agent 系统

例如：两个工具如果会被十个项目共用，也值得封装为 MCP Server；反过来，五个极其简单且只服务于一个应用的函数，也未必需要引入 MCP。

## 一个实用的选型顺序

```text
是否已有现成 MCP Server？
  ├── 有：优先直接复用
  └── 没有
       ↓
是否需要跨项目或跨团队复用？
  ├── 是：考虑实现 MCP Server
  └── 否
       ↓
是否是正式 Agent 系统，或工具维护已变复杂？
  ├── 是：优先考虑 MCP
  └── 否：Function Calling 通常更简单
       ↓
检查部署环境是否支持独立进程和目标传输方式
```

## 面试总结（答题要点）

- 不要简单回答「小项目用 Function Calling，大项目用 MCP」，项目规模不是唯一判断标准
- Function Calling 适合轻量、临时、单应用内部使用，以及需要直接控制执行逻辑的场景
- MCP 适合复用、模块化管理、正式 Agent 系统，以及社区已有现成 Server 的场景
- Function Calling 的 schema 和执行代码通常内嵌在应用中；MCP 将工具独立成 Server，通过标准协议接入
- 两者不是竞争关系：MCP 解决工具的标准化接入和复用，Function Calling 解决模型输出结构化调用意图
- 最容易记忆的一句话：**只给自己用、只用一次、不需要复用，优先 Function Calling；需要共享、管理和长期维护，优先 MCP。**

来源：[小林面试笔记：Function Calling 与 MCP 的适用场景](https://xiaolinnote.com/ai/tools/7_fc_vs_mcp_usage.html)

---

# 5. 推理模型、传统 Function Calling 与 Interleaved Thinking

## 一句话回答

推理模型通常会先生成一段较长的 thinking，再给出结果；而工具调用要求模型先输出调用请求、暂停等待工具执行、拿到结果后再继续生成。两者的冲突不在于工具协议本身，而在于**连续推理生成与中途暂停之间的生成范式冲突**。

后续方案大致有两类：一类是让工具调用发生在一个完整 thinking 阶段之后，保证这一段推理不被中途打断；另一类是 **Interleaved Thinking**，允许模型在多个 thinking 片段之间穿插工具调用，并在工具结果返回后继续推理。

## 传统 Function Calling 的真实含义

传统 FC 的典型流程是：

```text
第 1 轮：模型生成 tool_call
    ↓
宿主程序执行工具
    ↓
第 2 轮：模型读取工具结果并继续生成
```

这里的“可以随时调用工具”，更准确地说是：模型可以在任意一轮输出工具调用请求，而不是在生成任意一个 token 的过程中无缝插入工具。

当模型输出 `tool_call` 后，当前这一轮生成就结束；工具结果回来后，宿主程序再发起下一轮模型调用。即使连续调用多个工具，形式上也通常是：

```text
模型 → 工具 A → 模型 → 工具 B → 模型 → 最终回答
```

传统 FC 的各轮并不是完全没有联系。下一轮通常可以看到完整对话历史和工具结果，因此在**上下文层面是连续的**；但它不一定保留模型上一轮尚未完成的隐藏推理状态，所以在**连续生成状态层面不一定连贯**。

## “思考阶段结束后调用工具”

一种折中方案是：

```text
完整 thinking 片段 A
    ↓
tool_call
    ↓
tool_result
    ↓
后续生成或新的 thinking 片段 B
```

它的重点是让 thinking 片段 A 在工具调用前完整结束，不在 thinking 中间强行暂停。这样可以保护这一段推理的完整性，避免模型在思考到一半时突然切换成工具调用格式。

但这里的“完整”主要是**一个生成阶段形式上的完整**，不代表整个问题已经在逻辑上解决。模型可能只是完成了一个子问题，工具结果回来后仍然需要继续推理。

这种方案的缺点是：模型在前面的 thinking 阶段看不到工具结果。如果任务必须“先查数据，再基于数据进行复杂推理”，初始 thinking 就无法利用外部信息。

## Interleaved Thinking 是什么

Interleaved Thinking 允许推理与工具调用交错发生：

```text
thinking A
  ↓
tool_call A
  ↓
tool_result A
  ↓
thinking B
  ↓
tool_call B
  ↓
tool_result B
  ↓
thinking C
  ↓
最终回答
```

例如旅行规划任务：

```text
thinking：先查询航班时间
调用航班工具
结果：只有晚上航班价格合适

继续 thinking：第一天上午不能安排活动，再查询酒店
调用酒店工具
结果：目标区域酒店已满

继续 thinking：调整到另一个区域，生成最终行程
```

它的关键不是消息表面上出现了“思考 → 工具 → 思考”，而是模型和 API 能够把工具结果当作**同一任务推理过程中的中间信息**，而不是一个完全陌生的新请求。

## 三者的区别

| 方案 | 工具调用位置 | 工具结果回来后 | 连续性特点 |
|---|---|---|---|
| 传统 FC | 一轮模型生成结束时 | 开启下一轮生成 | 共享对话上下文，但不保证共享原生推理状态 |
| 思考结束后调用工具 | 一个完整 thinking 阶段之后 | 继续处理工具结果 | 保护当前 thinking 阶段不被中途打断 |
| Interleaved Thinking | 多个 thinking 片段之间 | 沿着任务推理继续前进 | 支持工具结果参与同一条连续推理流程 |

因此，传统 FC 也可能出现多轮“思考 + 工具”，但不能仅凭外部流程判断它就是 Interleaved Thinking。真正的区别在于模型、训练方式和 API 是否支持推理过程的延续。

## “推理状态”由谁负责？

这不是单纯由 Agent Harness 决定的，而是模型、模型 API / 推理运行时和 Harness 共同完成：

| 层次 | 主要职责 |
|---|---|
| 模型 | 学会何时调用工具，以及工具结果回来后如何继续推理 |
| 模型 API / 推理运行时 | 保存、恢复或传递 reasoning item、推理上下文等状态 |
| Agent Harness | 执行工具、保存历史、组织上下文、控制重试和调用循环 |

Harness 可以保存并重新注入：

- 对话历史和工具结果
- 任务计划与中间摘要
- 当前步骤和下一步目标
- 模型公开输出的思考内容（如果 API 允许）

但 Harness 无法凭空制造模型原生的隐藏推理状态。如果模型或 API 不支持连续推理，Harness 最多通过计划、摘要和工作记忆让模型重新理解任务，这属于**基于外部上下文的重新推理**，不等同于恢复上一轮内部 thinking。

另外，完整隐藏思维链通常不会直接暴露给 Harness；即使把一段文字摘要重新放回上下文，也不等于恢复原来的 KV Cache 或内部隐式表示。

## 最容易混淆的三个“连续”

1. **对话连续**：下一轮能看到之前的消息和工具结果。
2. **语义连续**：下一轮理解之前的计划，并能沿着计划继续。
3. **内部状态连续**：模型能够恢复或延续上一轮隐藏推理状态。

传统 FC 通常能做到第 1 层，也可能做到第 2 层，但不保证第 3 层。真正支持 Interleaved Thinking 的模型和 API，才会针对第 2、3 层提供更强的保证。

## 面试总结（答题要点）

- 工具调用的暂停与推理模型的连续 thinking 之间存在生成范式冲突
- 传统 FC 的“随时调用”是指模型可以在任意一轮输出工具请求，不是 token 级别的无缝插入
- 传统 FC 共享对话上下文，但每次 tool_call 通常结束当前生成，不保证恢复上一轮隐藏推理状态
- “思考结束后调用工具”保护的是一个 thinking 生成阶段的形式完整性，不代表整个任务逻辑已经完成
- Interleaved Thinking 允许 thinking → tool → thinking，并让工具结果真正参与后续同一任务的推理
- Harness 可以保存外部信息和重新组织上下文，但不能让不支持原生连续推理的模型凭空获得相同能力
- 最准确的一句话：**传统 FC 是分轮次重新生成；Interleaved Thinking 是在工具结果介入后继续同一条任务推理流程。**

来源：[小林面试笔记：为什么有些特定的推理模型不支持 MCP 协议？](https://xiaolinnote.com/ai/tools/8_reasoning_no_mcp.html)

---

# 6. Slash Command 和 Skill 的关系

## 一句话回答

Slash Command 是一种**可手动触发、可以携带参数的任务入口**，通常对应一段可复用的 Prompt。Skill 则是 Agent 执行一类任务所需的**完整能力包**，可以包含 `SKILL.md`、脚本、参考资料、模板和资源。实际使用中，Slash Command 可以作为 Skill 的入口，但不能完全等价替代 Skill。

## Slash Command 是什么

用户通过命令显式触发某项任务：

```text
/code-review src/main.go
```

命令背后可能对应这样的 Prompt：

```text
请审查文件：src/main.go。
重点关注安全性、错误处理和性能问题，并按照固定格式输出报告。
```

因此 Slash Command 通常负责三件事：

1. 提供一个容易记忆的命令入口
2. 携带用户在运行时传入的参数
3. 将固定 Prompt 和参数交给 Agent 执行

参数可以通过字符串替换、特殊变量（例如 `$ARGUMENTS`、`$1`）或平台定义的上下文机制传递。它不一定是简单的文本替换，也可能由 Agent 结合任务语义理解参数。

## Skill 是什么

Skill 通常是一个目录形式的能力包：

```text
code-review/
├── SKILL.md
├── scripts/
│   └── check_security.py
├── references/
│   └── review_standards.md
└── assets/
    └── report_template.md
```

其中：

- `SKILL.md`：能力说明、适用场景、执行指令和工作流程
- `scripts/`：可执行脚本
- `references/`：详细规范和参考文档
- `assets/`：报告模板、图片或其他资源

Skill 不只是保存一段 Prompt，还可以规定 Agent 应该先做什么、如何调用工具、何时运行脚本、如何处理错误以及最终如何输出结果。

## 两者的核心区别

| 对比项 | Slash Command | Skill |
|---|---|---|
| 主要定位 | 快捷命令和任务入口 | 完整的 Agent 能力包 |
| 触发方式 | 用户输入 `/xxx` 手动触发 | Agent 可根据任务自动发现和加载，也可被手动触发 |
| 内容形式 | 通常是一份命令定义或 Prompt | `SKILL.md` 加脚本、参考资料、模板等资源 |
| 参数作用 | 将用户输入传入 Prompt 或任务上下文 | 作为完整工作流的运行时输入 |
| 执行能力 | 取决于平台，通常较轻量 | 可组织脚本、工具调用和多步骤流程 |
| 自动发现 | 通常依赖用户知道命令名 | 可以根据名称和描述匹配用户任务 |
| 加载方式 | 一般直接执行命令对应内容 | 通常支持按需加载指令和辅助资源 |

可以用一个简单的比喻理解：

```text
Slash Command = 快捷按钮
Skill          = 操作手册 + SOP + 工具箱
```

## Skill 可以被 Slash Command 触发

实际中完全可以把 Skill 包装在 Slash Command 后面：

```text
/code-review src/main.go
        ↓
加载 code-review Skill
        ↓
传入文件路径和审查要求
        ↓
读取规范、运行脚本、生成报告
```

这种情况下：

- Slash Command 负责“从哪里进入”和“如何传参”
- Skill 负责“进入之后如何完成任务”

例如命令定义可能只需要表达：

```text
请使用 code-review Skill 审查用户指定的文件：$ARGUMENTS。
按照 Skill 中的完整流程执行。
```

此时 Slash Command 是入口，真正的任务逻辑仍然来自 Skill。

## 为什么不能把两者完全等同

一个简单的 Slash Command 可能只是：

```text
/summarize file.md
```

然后展开成：

```text
请总结 file.md 的主要内容。
```

但一个完整的总结 Skill 还可以规定：

1. 先读取文件并识别文档类型
2. 按主题提取关键信息
3. 读取指定的写作规范
4. 使用模板组织结果
5. 检查是否遗漏重要内容

如果只把 Skill 的文字复制成 Slash Command，就可能失去脚本、参考资料、渐进式加载和完整流程管理能力。

## 参数替换不等于直接执行文件

执行：

```text
/code-review src/main.go
```

并不意味着 Slash Command 自己直接读取或修改 `src/main.go`。更准确的流程是：

```text
命令参数：src/main.go
        ↓
传给 Agent / Skill
        ↓
Agent 根据指令读取文件
        ↓
调用工具或脚本执行操作
```

Slash Command 主要是把“任务意图”和“运行时参数”交给 Agent；具体是否读取文件、调用工具或修改内容，取决于命令定义、Skill 指令和 Agent 的执行能力。

## 最终理解

```text
Slash Command：用户主动触发的一段可参数化任务指令

Skill：Agent 可以自动发现或被命令触发的完整任务能力包
```

二者可以组合，但职责不同：

```text
Slash Command = 进入任务 + 传入参数
Skill          = 理解任务 + 组织流程 + 使用工具和资源 + 产出结果
```

## 面试总结（答题要点）

- Slash Command 不只是“保存 Prompt”，它还是一个可手动触发、可携带参数的命令入口
- Skill 不只是 Prompt，而是由 `SKILL.md`、脚本、参考资料和资源组成的可复用能力包
- Slash Command 通常由用户显式触发；Skill 可以由 Agent 根据任务自动发现和加载
- Slash Command 的参数通常会进入 Prompt 或任务上下文，但不一定是简单字符串替换
- Slash Command 可以包装或触发 Skill，但它本身不必然拥有 Skill 的完整目录结构和执行能力
- 最容易记忆的一句话：**Slash Command 负责“怎么进入和传参”，Skill 负责“进入之后如何完成任务”。**

---

# 7. A2A 与主 Agent—子 Agent 的任务编排

## 一句话回答

A2A 是 Agent 之间交换任务、状态和结果的协议，不要求通信双方在组织结构上处于同一层级。主 Agent 和子 Agent 之间也可以使用 A2A，但“任务拆分、子任务路由、调度、依赖管理和结果汇总”通常属于主 Agent 或 Orchestrator 的编排逻辑。实际项目中，很多系统使用内部 Session、进程、RPC 或插件协议完成这些工作，功能上类似 A2A，但不一定实现了标准 A2A 协议。

## A2A 不等于“同等级 Agent 通信”

更准确的理解是：A2A 通信的双方都是独立的 Agent 端点，而不是要求双方拥有相同的权限、地位或组织层级。

```text
研究 Agent ── A2A ── 数据分析 Agent       # 同级协作
主 Agent ── A2A 委派任务 ── 研究 Agent    # 主从编排
```

主 Agent—子 Agent 是否使用 A2A，取决于子 Agent 是否作为独立 Agent 服务存在，以及是否需要跨服务、跨平台或跨团队通信，而不是取决于它们是否“同级”。

## 主 Agent—子 Agent 的基本编排流程

```text
用户任务
   ↓
主 Agent / Orchestrator
   ├── 理解任务
   ├── 拆分子任务
   ├── 建立依赖关系
   ├── 匹配子 Agent 能力
   ├── 派发任务
   ├── 跟踪状态和失败
   └── 汇总结果
        ├── A2A → 研究 Agent
        ├── A2A → 代码 Agent
        └── MCP → 数据库、搜索、文件系统等工具
```

例如用户要求：

```text
分析这个 Kubernetes 项目的性能问题，并给出修复方案。
```

主 Agent 可以拆成：

```text
任务 A：分析 Kubernetes 配置
任务 B：检查 Go 代码性能
任务 C：分析监控和日志
任务 D：综合分析并生成修复方案
```

其中 A、B、C 可以并行执行，D 等待前三个任务完成：

```text
任务 A ─┐
任务 B ─┼──> 任务 D：综合分析
任务 C ─┘
```

## 子任务如何路由到子 Agent

### 1. 静态规则路由

```text
task_type = "k8s_config"  →  k8s-config-agent
task_type = "code_review" →  code-review-agent
```

优点是稳定、可控、容易审计；缺点是任务类型多以后规则会变得复杂。

### 2. 基于能力描述路由

每个 Agent 注册自己的能力：

```json
{
  "agent": "observability-agent",
  "capabilities": ["日志分析", "Prometheus 指标分析", "调用链分析"],
  "input": "日志、指标或 Trace",
  "output": "诊断报告"
}
```

主 Agent 将子任务与能力描述匹配，再选择合适的 Agent。这种方式比硬编码更灵活。

### 3. LLM 动态路由

主 Agent 先分析任务：

```text
需要 Kubernetes 配置分析、Go 性能分析和监控数据分析
```

再动态选择：

```text
k8s-agent
code-review-agent
observability-agent
```

灵活性高，但需要限制任务拆分深度，避免错误路由、重复执行、循环委派或将敏感任务交给无权限 Agent。

### 4. 混合路由

生产环境通常将几种方式结合：

```text
权限规则过滤候选 Agent
        ↓
能力描述进行匹配
        ↓
LLM 负责复杂任务拆分和最终选择
        ↓
调度器负责并行、依赖和重试
```

## 任务交付消息

主 Agent 可以向子 Agent 传递结构化任务：

```json
{
  "task_id": "task-1024",
  "parent_task_id": "task-root",
  "to_agent": "code-review-agent",
  "objective": "分析 payment-service 中的 Go 性能问题",
  "inputs": {
    "repository": "payment-service",
    "files": ["internal/order/*.go"]
  },
  "constraints": {
    "read_only": true,
    "deadline_seconds": 120
  },
  "expected_output": {
    "type": "performance_report",
    "fields": ["problem", "evidence", "severity", "suggestion"]
  }
}
```

子 Agent 返回的内容可以包括：

```json
{
  "task_id": "task-1024",
  "status": "completed",
  "result": {
    "problems": [
      {
        "location": "internal/order/service.go:88",
        "problem": "循环中重复查询数据库",
        "severity": "high",
        "suggestion": "改为批量查询"
      }
    ]
  },
  "artifacts": ["performance-report.md"]
}
```

## 任务状态和调度

复杂系统通常需要维护任务状态：

```text
pending → dispatched → running → completed
                         ├──────> failed
                         ├──────> blocked
                         └──────> cancelled
```

编排器还需要处理并行任务、前置依赖、超时、重试、失败降级、取消、重复任务去重、子 Agent 接管和结果合并。

## A2A、MCP 与内部 Harness 编排

### 使用 A2A 委派 Agent

```text
主 Agent
  ↓ A2A：请完成 Kubernetes 性能诊断
专业 Agent
  ├── 自己规划步骤
  ├── 自己调用 MCP Tools
  └── 返回进度和最终报告
```

主 Agent 主要关心任务和结果，不需要了解专业 Agent 的内部执行细节。

### 将 Agent 封装为 MCP Tool

```text
主 Agent
  ↓ MCP tools/call：analyze_kubernetes_performance(...)
MCP Server
  ↓ 内部运行固定逻辑或 Agent
  ↓ 返回结构化结果
```

主 Agent 看到的是一个工具，需要自己决定什么时候调用、传入什么参数以及如何处理结果。

### 使用内部 Harness 协议

```text
主 Agent
  ↓ 内部函数 / Session / 子进程 / RPC
子 Agent
```

这种方式不一定使用网络 A2A，但可能已经实现了任务 ID、上下文传递、状态跟踪、结果回收、取消和重试等 A2A 类能力。

## 几种典型 Agent 架构的共同模式

公开实现中，不同项目的名称和接口不同，但通常都包含类似机制：

| 架构 | 常见编排特点 |
|---|---|
| Hermes Agent | 委派工具、独立子 Agent、并行执行、生命周期、角色和工具权限管理 |
| Pi Agent | 独立子进程、Agent 配置、单个 / 并行 / 链式工作流、隔离上下文 |
| OpenCode | `task` 工具、`subagent_type`、子 Session、后台任务、权限和深度限制 |
| DeepSeek Harness AgentTeams | Captain、持久化成员、任务 DAG、依赖调度、直接消息、重试和恢复 |

这些实现说明：**多 Agent 编排是一个独立的工程问题**。它通常需要自己实现 Agent 发现、路由、任务生命周期、并行调度、权限隔离和结果聚合；是否使用标准 A2A，只是其中的通信协议选择。

## 选型判断

| 场景 | 更适合 |
|---|---|
| 本地创建一个短生命周期子任务 | 内部 Harness、Session 或子进程 |
| 主 Agent 调用固定能力 | MCP Tool |
| 委派复杂任务给独立专业 Agent | A2A 或内部 Agent Task Protocol |
| Agent 跨服务、跨团队、跨平台复用 | A2A |
| 子 Agent 自己规划并使用多个工具 | A2A |
| 参数稳定、返回结构固定的能力 | MCP |

最容易记忆的一句话是：

```text
MCP：请执行这个能力
A2A：请完成这项任务
Harness：我来决定任务如何拆分、路由和调度
```

## 面试总结（答题要点）

- A2A 不要求双方处于同一组织层级；主 Agent 和子 Agent 也可以使用 A2A
- A2A 解决 Agent 之间的任务、状态、消息和结果交互
- 主 Agent / Orchestrator 负责任务拆分、能力匹配、路由、调度和结果汇总
- 路由可以采用静态规则、能力描述匹配、LLM 动态路由或混合方式
- 复杂系统通常需要任务 ID、父子任务关系、状态机、依赖 DAG、超时、重试和取消机制
- 很多 Agent 项目有自己的内部子 Agent 编排协议，功能上类似 A2A，但不一定兼容标准 A2A
- 将 Agent 封装为 MCP Tool，主 Agent 看到的是一个能力；通过 A2A 委派，主 Agent 看到的是另一个可以自主完成任务的 Agent
- 最准确的一句话：**A2A 规定 Agent 如何通信，Harness 决定任务如何拆分和调度，MCP 为 Agent 提供具体工具和外部能力。**

参考：[小林面试笔记：什么是 A2A 协议？它和 MCP 协议的区别是什么？](https://xiaolinnote.com/ai/tools/12_a2a_protocol.html)

---

# 8. MCP 的通信方式：stdio 与 Streamable HTTP

## 一句话回答

MCP 的消息格式统一使用 **JSON-RPC 2.0**，但传输方式根据部署场景不同而不同：本地工具通常使用 **stdio**，远程服务使用 **Streamable HTTP**。stdio 通过操作系统的标准输入输出和进程管道通信，不经过网络；Streamable HTTP 则通过 HTTP 连接远程 Server，并允许 Server 按需返回普通 JSON 或 SSE 流。

## 消息格式与传输方式是两层概念

```text
JSON-RPC 2.0：规定消息长什么样
stdio / Streamable HTTP：规定消息怎么传
```

例如调用 MCP 工具时，消息本身可以是：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "take_screenshot",
    "arguments": {"url": "https://example.com"}
  }
}
```

这条 JSON-RPC 消息既可以写入本地子进程的 stdin，也可以放入 HTTP POST 请求体。更换传输方式不会改变上层工具调用逻辑。

## stdio 到底是什么

stdio 是 standard input / output 的缩写，即标准输入和标准输出。MCP Client 会把 MCP Server 当作一个本地子进程启动：

```text
MCP Client
  ├── 向 Server 的 stdin 写入 JSON-RPC 请求
  └── 从 Server 的 stdout 读取 JSON-RPC 响应

操作系统管道
  └── 连接两个本地进程
```

它不是 HTTP，也不是访问 `localhost` 的网络请求。两个进程在同一台机器上运行，通过操作系统提供的管道交换数据，通常不经过网卡、TCP/IP 协议栈或监听端口。

### stdio 的启动方式

Client 配置的不是一个 URL，而是启动 Server 所需的命令：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": {}
    }
  }
}
```

典型流程是：

1. Client 根据配置启动 Server 子进程
2. Client 通过 stdin 发送 JSON-RPC 消息
3. Server 从 stdin 读取消息并执行操作
4. Server 将 JSON-RPC 响应写入 stdout
5. Client 读取响应并交给 Host / Agent

### stdio 的优点

- **不需要网络**：适合本地文件、Git、代码分析等工具
- **不需要端口**：没有监听端口暴露带来的网络攻击面
- **延迟较低**：数据通过本地进程管道传递
- **生命周期简单**：通常随 Client 启动和退出，不需要手动管理独立服务
- **适合本地权限隔离**：Server 可以作为受控的子进程运行

### stdio 的限制

- Client 和 Server 通常需要在同一台机器上
- 不适合多个远程 Client 共享同一个 Server
- 需要运行子进程的部署环境
- Server 崩溃、stdout 混入日志或进程生命周期异常时，需要由 Client 负责处理

## Streamable HTTP：远程 MCP 的当前方式

远程场景下，MCP Server 作为独立 HTTP 服务部署，多个 Client 可以通过网络访问它：

```text
Client A ─┐
Client B ─┼── HTTP ──> MCP Server
Client C ─┘
```

Streamable HTTP 通常使用一个 MCP 端点，例如 `/mcp`：

1. Client 通过 POST 发送 JSON-RPC 消息
2. Server 根据任务是否需要流式返回，选择响应方式
3. 简单任务返回 `application/json`
4. 长时间或需要增量结果的任务返回 `text/event-stream`

```text
POST /mcp
  ├── 普通任务 → application/json
  └── 流式任务 → text/event-stream（SSE）
```

它仍然可以使用 SSE 传递流式数据，但不再要求 Client 预先建立一个独立的 SSE 接收端点。

## 为什么远程方案从 HTTP + SSE 演进为 Streamable HTTP

### 早期的 HTTP + SSE 双端点

早期方案将两个方向拆开：

```text
POST 端点：Client → Server，发送 JSON-RPC 请求
GET / SSE：Server → Client，建立长连接并推送消息
```

一次 MCP 会话需要维护两条关联通道：一条负责请求，一条负责推送。Client 还要处理会话 ID、连接建立顺序和两条连接之间的对应关系。

### 双端点方案的问题

#### 1. 状态管理复杂

如果 Client POST 请求后网络断开，Client 可能无法立即判断：

- Server 是否已经收到请求
- Server 是否已经执行请求
- 响应是否正在 SSE 连接中返回
- 是否应该重试
- 重试是否会导致重复执行

请求通道和响应通道分离，使故障排查和幂等处理更复杂。

#### 2. 连接管理复杂

Client 需要：

- 先建立或维护 SSE 长连接
- 处理 POST 与 SSE 的会话关联
- 处理 SSE 断线和重连
- 区分普通响应、异步响应和 Server 主动推送

对简单的请求—响应任务来说，这种复杂度往往是不必要的。

#### 3. 部署和基础设施兼容性较差

双端点和长连接会增加对以下基础设施的要求：

- 反向代理
- 负载均衡器
- 会话保持
- 超时设置
- Serverless 平台

请求可能被转发到不同实例，而 SSE 长连接又需要和对应会话保持关联，部署与排错成本较高。

#### 4. 简单任务被迫使用长连接

很多 MCP 调用很快就能完成，例如读取一个配置或查询一个简单数据。如果每次都需要额外维护 SSE 长连接，就会增加连接和资源开销。

## Streamable HTTP 的改进

Streamable HTTP 将请求与响应统一到一个端点：

```text
Client POST /mcp
        ↓
Server 处理 JSON-RPC
        ├── 立即完成：返回普通 JSON
        └── 需要流式返回：返回 SSE 流
```

它带来的好处是：

- 请求和响应在同一次 HTTP 交互中关联，状态更容易管理
- 简单调用不需要预先建立 SSE 连接
- 长任务仍然可以通过 SSE 流式返回进度或增量结果
- 端点更少，配置、代理和负载均衡更简单
- 更适合云端部署、Serverless 和多 Client 共享
- 仍然可以兼容需要 Server 主动推送的场景：必要时额外建立 GET 流

因此，“抛弃 HTTP + SSE”并不是完全抛弃 SSE，而是：

> 从“POST 请求通道 + 独立 SSE 推送通道”，演进为“单端点按需返回普通 JSON 或 SSE 流”。

## stdio 与 Streamable HTTP 对比

| 对比项 | stdio | Streamable HTTP |
|---|---|---|
| 部署位置 | 本地子进程 | 远程或独立 HTTP 服务 |
| 通信方式 | stdin / stdout 管道 | HTTP POST，必要时 SSE 响应 |
| 是否经过网络 | 否 | 是 |
| 是否需要端口 | 否 | 是 |
| 延迟 | 通常较低 | 有网络开销 |
| 多 Client 共享 | 不适合 | 适合 |
| 生命周期 | 通常由 Client 管理 | Server 独立运行 |
| 认证与重连 | 相对简单 | 需要处理认证、断线和重连 |
| 典型场景 | 本地文件、Git、代码工具 | 云端数据库、团队共享服务、远程 API |

## 面试总结（答题要点）

- MCP 的底层消息格式是 JSON-RPC 2.0，传输方式是 stdio 或 Streamable HTTP
- stdio 是 Client 启动本地 Server 子进程，通过 stdin 发送请求、stdout 接收响应，不是 localhost HTTP
- stdio 的优点是无网络、无端口、延迟低、生命周期容易管理
- Streamable HTTP 适合远程部署和多个 Client 共享同一个 Server
- 早期 HTTP + SSE 使用 POST 请求端点和独立 SSE 推送端点，带来会话关联、断线重连、状态判断和部署方面的复杂度
- Streamable HTTP 将两者统一到一个端点，Server 可以按需返回普通 JSON 或 SSE 流
- Streamable HTTP 不是完全放弃 SSE，而是把 SSE 从“常驻独立通道”变成“按需的流式响应”
- 最容易记忆的一句话：**stdio 适合本地进程间通信，Streamable HTTP 适合远程服务通信；JSON-RPC 规定消息格式，传输层只规定消息如何到达。**

来源：[小林面试笔记：MCP 协议通常采用什么通信方式？](https://xiaolinnote.com/ai/tools/13_mcp_transport.html)

---

# 9. AI 场景下如何选择 HTTP + SSE 与 WebSocket？

## 结论先行

如果只是低频地允许用户中途停止模型输出，**HTTP + SSE 完全可以满足需求**，不需要为了“支持停止”专门引入 WebSocket。

```text
低频停止：SSE + 关闭流 / cancel API
高频双向控制：WebSocket
实时语音：WebSocket 或 WebRTC
```

真正需要 WebSocket 的，不是“能不能停止”这一点，而是是否需要持续、高频、低延迟的双向交互。

## 普通文本对话中的 SSE 流程

```text
客户端 POST /chat
        ↓
服务端建立 SSE 流
        ↓
持续返回模型 token
        ↓
用户点击“停止”
        ↓
客户端关闭 SSE 连接
        ↓
服务端取消模型生成
```

如果用户随后输入新问题：

```text
关闭旧 SSE
        ↓
POST 新消息
        ↓
建立新的 SSE 流
```

用户看到的效果仍然是：

```text
模型正在输出 → 用户点击停止 → 模型停止 → 用户发送新问题
```

只是底层经历了“关闭旧流 + 新建请求”，而不是在同一条连接里发送取消指令。

## SSE 下实现停止的两种方式

### 方式一：直接关闭 SSE

客户端通过 `AbortController` 或其他方式关闭当前 SSE 请求。服务端检测到连接断开后，取消对应的模型任务。

优点是简单，适合大多数文本聊天场景。

```text
客户端断开 SSE
        ↓
服务端检测 disconnect
        ↓
取消模型推理
        ↓
释放 GPU / CPU / 请求资源
```

### 方式二：额外发送取消请求

客户端保留 SSE，同时通过另一个 HTTP 请求显式通知服务端：

```http
POST /cancel
```

```json
{
  "generation_id": "gen-123"
}
```

服务端根据 `generation_id` 找到并取消对应的生成任务。这种方式适合任务管理较复杂的系统，但会增加一个接口和状态协调过程。

## 服务端必须正确处理取消

关闭 SSE 只代表客户端不再接收数据，不一定自动停止服务端的模型推理。服务端需要建立：

```text
SSE 连接断开
        ↓
检测客户端 disconnect
        ↓
取消模型推理任务
        ↓
释放资源
```

还应为每次生成分配唯一的 `generation_id`：

```text
旧任务：gen-123
新任务：gen-124
```

如果旧任务取消不及时，服务端收到旧任务 token 时应检查其状态：

```text
token 属于已取消的 generation_id
        ↓
丢弃，不再发送给客户端
```

否则可能出现旧回答和新回答交错的问题。

## WebSocket 下的停止流程

WebSocket 可以在同一条长连接中发送模型输出和控制指令：

```text
建立 WebSocket
        ↓
客户端发送 user_message
        ↓
服务端持续发送 token
        ↓
客户端发送 cancel
        ↓
服务端停止生成
        ↓
客户端继续发送新的 user_message
```

例如：

```json
{
  "type": "cancel",
  "generation_id": "gen-123"
}
```

然后继续使用同一条连接发送新消息：

```json
{
  "type": "user_message",
  "content": "帮我总结项目目录"
}
```

它的优势是没有“关闭旧流 → 新建请求”的连接切换过程，适合需要持续双向控制的场景。

但 WebSocket 收到 `cancel` 并不等于模型一定立即停止。真正的停止链路仍然是：

```text
WebSocket 收到 cancel
        ↓
找到对应模型任务
        ↓
将取消信号传递给推理服务
        ↓
模型任务停止
```

如果推理服务不支持取消，服务端可能只能停止转发结果，而模型本身仍然在后台生成。

## 为什么低频停止不需要 WebSocket

普通文本对话通常是：

```text
用户发送一条消息
        ↓
模型持续输出
        ↓
用户偶尔点击一次停止
```

这种交互中，停止动作发生频率低，而且关闭 SSE、重新发送 POST 的成本很小。引入 WebSocket 反而会增加：

- 长连接生命周期管理
- 连接断开与重连
- 连接状态和会话绑定
- 横向扩展和粘性会话
- 代理、防火墙和负载均衡配置

因此，判断标准不是“是否需要停止”，而是：

```text
是否需要在模型输出期间持续、高频地从客户端发送控制消息？
```

如果答案是否定的，SSE 通常更简单。

## SSE 与 WebSocket 的场景选择

| 场景 | 推荐方式 | 原因 |
|---|---|---|
| 普通 LLM 流式文字输出 | SSE | 服务端单向推送已经足够 |
| 低频点击“停止生成” | SSE | 关闭流或调用 cancel API 即可 |
| 用户发送新问题 | SSE + POST | 新问题走新的 HTTP 请求 |
| 高频发送暂停、恢复、切换指令 | WebSocket | 同一连接支持双向消息 |
| 实时语音对话和用户抢话 | WebSocket / WebRTC | 客户端持续发送音频并随时打断 |
| 多人协同编辑 | WebSocket | 双方持续交换实时状态 |
| 实时游戏或状态同步 | WebSocket | 高频双向事件交互 |

## 与连接数限制的关系

在 HTTP/1.1 下，一个 SSE 流会长期占用一条连接。用户点击停止后关闭 SSE，可以释放这条连接；之后的新 POST 可以复用或重新建立连接。

```text
SSE 持续输出：占用连接 1
用户点击停止：关闭连接 1
发送新消息：复用或新建连接
```

如果用户在旧 SSE 仍然运行时直接发送新 POST，那么 POST 可能需要占用另一条连接。HTTP/2 则可以在一条 TCP 连接上复用多个逻辑 Stream，从而缓解 HTTP/1.1 的连接数限制。

## 最终理解

```text
SSE：模型输出期间，客户端偶尔取消
     关闭当前流，再发送新的 HTTP 请求

WebSocket：模型输出期间，客户端持续发送控制消息
           在同一条双向连接中完成交互
```

所以：

> “支持中途停止”不是使用 WebSocket 的充分理由；只有当应用需要高频、持续、低延迟的双向通信时，WebSocket 的优势才真正明显。

## 面试总结（答题要点）

- 低频停止模型输出时，HTTP + SSE 已经足够
- SSE 下可以通过关闭流，或者额外调用取消接口实现停止
- 服务端必须将客户端断开或取消请求传递给模型推理任务，否则模型可能仍在后台生成
- `generation_id` 可以避免旧任务的残余 token 污染新任务
- WebSocket 的优势是同一条连接中可以随时发送 token、cancel、pause 和新消息
- 实时语音、抢话、多用户协同和高频控制更适合 WebSocket 或 WebRTC
- 最容易记忆的一句话：**低频控制用 SSE 更简单，持续双向交互才需要 WebSocket。**

来源：[小林面试笔记：说说 WebSocket 和 SSE 通信的区别及局限性？](https://xiaolinnote.com/ai/tools/14_sse_vs_websocket.html)
