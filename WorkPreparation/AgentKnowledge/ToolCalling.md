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

早期 MCP 使用 HTTP + SSE：POST 端点负责发送请求，GET 端点建立 SSE 长连接接收 Server 推送。2025 年 3 月规范更新后，推荐使用 Streamable HTTP；旧方案被标记为 deprecated，但通常仍需考虑兼容性。

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

来源：[小林面试笔记：MCP 由哪几部分组成？](https://xiaolinnote.com/ai/tools/5_mcp_components.html)
