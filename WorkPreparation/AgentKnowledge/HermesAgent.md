---
tags:
  - Agent
  - Hermes
  - 多Agent编排
source: https://chatgpt.com/share/6a97df61-cb4c-83e8-bc66-ebe85feaef3d
topic: Hermes Agent 子代理编排机制（Delegate Task vs Kanban）
---

# Hermes Agent 的两种子代理编排方式

> 与 GPT 对话整理的要点（对话在概念层面展开，未进入 Hermes 具体实现）。Hermes 的子代理编排主要有两种范式：**Delegate Task（任务委派）** 与 **Kanban（共享任务板）**。二者本质区别一句话：**Delegate Task 是「控制流驱动」的编排，Kanban 是「状态 / 任务流驱动」的编排。**

## 1. Delegate Task：主 Agent 委派

- 典型链路：**Main Agent → Delegate → Sub-Agent → Result → Main Agent**
- 核心特点：
  - **主 Agent 掌握任务分解权**，显式指定「哪个 Agent 做哪个子任务」
  - 子 Agent 通常是「一次性执行一个任务」，完成后把结果返回父 Agent
  - 父 Agent 决定下一步做什么
- 类比：**函数调用 / RPC / task delegation**
- 适合：**明确、独立、边界清晰**的子任务

## 2. Kanban：共享任务板 + 自主领取

- 不是「主 Agent 指定 Agent A 做任务 X」，而是把任务放入共享 Task Board / 任务池
- Agent 自行观察任务板 → **claim（认领）→ 执行 → 更新为 DONE**
- 任务执行完可能**产生新的子任务**放回 Kanban，由其他 Agent 领取
- 核心不是「谁委派给谁」，而是：**多个 Agent 围绕一个共享的任务状态空间协作**
- 类比：项目团队 / 生产线
- 适合：**大量、动态、可并行**的任务

## 3. 两者核心对比

| 维度 | Delegate Task | Kanban |
|---|---|---|
| 协作方式 | 主 Agent 委派 | Agent 从任务池领取 |
| 控制中心 | Parent Agent | Shared Task Board |
| Agent 关系 | 父子关系明显 | 相对松耦合 |
| 任务生命周期 | 调用 → 返回 | TODO → Doing → Done |
| 动态性 | 相对较低 | 很高 |
| 适合场景 | 明确的子任务 | 大量、动态、可并行任务 |
| 类比 | 函数调用 / RPC | 项目团队 / 生产线 |

编排层面一句话抽象：

- **Delegate**：Agent 决定「谁做什么」
- **Kanban**：任务池决定「现在有哪些工作」，Agent 决定「我做哪个工作」

## 4. 三个关键校准点（对话核心）

### 4.1 Delegate ≠ 天然串行，完全可以并行

- Delegate Task 描述的是**任务委派关系**，不是执行顺序
- 若多个子任务相互独立，主 Agent 可**并行委派**给多个子 Agent，完成后汇总
- 若存在依赖，则是 **DAG 式 Delegate**（如 Task A、Task B 都完成后才能执行 Task C）
- 更准确说法：Delegate = **由主 Agent 显式控制任务依赖和子 Agent 的调用关系**

### 4.2 Kanban 必须有「全局完成判定」，否则没有 End

- Kanban 天然允许「任务执行完 → 产生新任务 → 再放回 → 再执行」的链条，**本身无法定义任务何时结束**
- 若设计成简单的 `while TODO != empty: agent pick task`，会陷入无限循环
- 必须区分两个概念：
  - **Task completion**：某个子任务完成（Task A → DONE）
  - **Goal completion**：整个用户目标完成（Goal → COMPLETE）
- 全局完成条件可由以下角色承担：主 Agent / Orchestrator / Supervisor Agent / 专门的 evaluator·judge / 规则·状态机

### 4.3 主 Agent 在 Kanban 中不一定消失，可只做规划与收尾

典型结构：Main Agent 负责 **Task Decomposition（初始拆解）** + **Goal Control（完成判定）**，Worker Agents 负责执行层（领取任务、执行、更新状态、必要时产出新子任务）。

- **Main Agent**：理解用户目标 → 初始拆解 → 放入 Kanban → 观察整体进度 → 必要时调整任务 → 判断是否达成最终目标 → 汇总回复用户
- **Worker Agents**：从 Kanban 获取任务 → 执行 → 更新状态 → 产生结果 → 必要时产生新子任务

即主 Agent **不一定参与每个子任务的具体执行**，更像 **Supervisor / Planner / Evaluator**，Worker 更像 **Execution Layer**。

### 4.4 主 Agent 介入程度存在光谱（Kanban ≠ 拆完撒手不管）

- **强监督**：Main Agent 持续观察 / 调整（Workers 全程在其监控下）
- **弱监督**：Workers 自主协作，完成后通知 Main Agent
- **近似去中心化**：Goal → Kanban ↔ Agents 互相协作 → Goal Complete（接近 multi-agent swarm / decentralized orchestration，主 Agent 只在特定事件介入或只负责 Initial Goal 与 Final Answer）

## 5. 遗留待追问问题（对话结束时抛出的三个关键问题）

要把 Hermes 画成真实 orchestration architecture，需搞清楚：

1. **Hermes 的 Kanban 中，谁拥有 Task Queue / Kanban 的最终控制权？**
2. **子 Agent 能否自主创建新的 Task？创建出的 Task 是否必须经过 Main Agent approval？**
3. **整个 Kanban 的终止条件到底由谁判断？**

## 面试 / 复习要点

- **雷点**：把 Delegate Task 当成天然串行；以为 Kanban「没有 Orchestrator」——正确理解是它把 Orchestrator 从「中央决策者」变成「任务状态机 / 共享任务状态 + 全局完成判定」
- 两种模式本质区别讲「**控制流驱动 vs 状态/任务流驱动**」，而非表面「委派 vs 领取」
- 加分点：主动说出 Delegate 可并行 / 可 DAG；Kanban 必须区分 Task completion 与 Goal completion；主 Agent 介入程度有强监督 → 弱监督 → 近似去中心化三档
