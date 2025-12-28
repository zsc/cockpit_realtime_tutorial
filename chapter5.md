# Chapter 5｜Agents SDK 编排与多代理设计（Orchestration）

## 5.1 开篇段落

在上一章中，我们通过 **OpenAI Realtime API** 建立了一条高性能的“语音-语音”交互管道，解决了“听得清”和“说得快”的问题。然而，真正的智能座舱不仅需要耳聪目明，更需要一个逻辑清晰、分工明确的“大脑”来处理复杂的车主需求。

本章将聚焦于 **OpenAI Agents SDK** 的应用。在车载环境中，试图用单一的 Prompt 和数百个工具（Tools）来解决所有问题是不可行的——这会导致上下文污染、高延迟和安全隐患。因此，我们需要构建一个 **多代理（Multi-Agent）系统**。

**本章学习目标**：
1.  **架构设计**：掌握基于 **Supervisor（路由器）+ Specialists（专家）** 的星型拓扑结构。
2.  **交接机制（Handoffs）**：深度理解代理间的控制权转移、上下文传递（Context Propagation）与回退逻辑。
3.  **状态管理**：区分 **会话级记忆**（Session Memory）与 **任务级状态**（Task State），并了解其生命周期。
4.  **双循环集成**：解决 Realtime API 的“实时事件流”与 Agents SDK 的“逻辑执行流”之间的同步与冲突问题。
5.  **安全编排**：通过代理设计实现车控权限的物理隔离与护栏（Guardrails）。

---

## 5.2 核心架构：Supervisor-Worker 拓扑

### 5.2.1 为什么选择星型拓扑？

在车载场景中，我们推荐使用 **星型（Star）** 或 **Hub-and-Spoke** 拓扑结构，而不是网状结构。

*   **单一入口**：Router Agent 是唯一的入口，负责初筛。
*   **职责单一**：专家 Agent 不需要知道彼此的存在，只需要知道如何完成自己的任务以及何时把控制权交还给 Router。
*   **低延迟启动**：Router 加载的 System Prompt 极短，工具极少（仅用于 Handoff），保证了首字延迟（TTFT）最低。

### 5.2.2 架构概览图

```ascii
                                    [ User Audio Stream ]
                                            |
                                            v
                                  +-------------------+
                                  | OpenAI Realtime   |
                                  |       API         |
                                  +---------+---------+
                                            | (Function Call Event)
                                            v
+---------------------------------------------------------------------------------+
|                            AGENTS SDK RUNTIME ENV                               |
|                                                                                 |
|  +------------------+         Handoff (Transfer)        +------------------+    |
|  |                  | --------------------------------> |  Vehicle Control |    |
|  |   Supervisor     |                                   |      Agent       |    |
|  |    (Router)      | <-------------------------------- | (High Safety)    |    |
|  |                  |         Result / Handoff Back     +------------------+    |
|  +--------+---------+                                                           |
|           |                                                                     |
|           | Handoff                 +------------------+                        |
|           +-----------------------> |    RAG Knowledge |                        |
|           |                         |       Agent      |                        |
|           | <---------------------- | (High Latency)   |                        |
|           |                         +------------------+                        |
|           |                                                                     |
|           | Handoff                 +------------------+                        |
|           +-----------------------> |    GUI Service   |                        |
|                                     |       Agent      |                        |
|                                     | (Stateful)       |                        |
|                                     +------------------+                        |
+---------------------------------------------------------------------------------+
```

### 5.2.3 角色定义

| 角色 | 职责 (Responsibilities) | 典型工具 (Tools) | 上下文策略 |
| :--- | :--- | :--- | :--- |
| **Router (Supervisor)** | 意图分类、全局状态维护、错误兜底 | `transfer_to_car_control`, `transfer_to_rag`, `transfer_to_gui` | 极简，仅保留最近几轮对话概要 |
| **Vehicle Control Agent** | 精确控制车辆硬件、参数校验 | `set_seat_heat(level)`, `open_window(pos)`, `get_tire_pressure()` | 严谨System Prompt 包含安全规范 |
| **RAG Agent** | 查询手册、故障解释、闲聊 | `search_knowledge_base(query)`, `format_citation()` | 宽泛，包含大量检索到的文档片段 |
| **GUI Agent** | 操作屏幕 App、多轮业务流程 | `click_element(id)`, `scroll(direction)`, `input_text(val)` | 状态机，保留当前页面 UI 树信息 |

---

## 5.3 路由与交接机制 (Routing & Handoffs)

这是 Agents SDK 的核心。路由不仅仅是“分类”，更是“带参数的跳转”。

### 5.3.1 基础交接 (Basic Handoff)

最简单的交接只是切换处理对话的 Agent 对象。

> **Rule of Thumb**: Router Agent 不应该回答领域问题。如果用户问“胎压多少”，Router **必须** 切换到车控 Agent，而不是自己尝试回答（哪怕它偶尔能猜对）。

### 5.3.2 带载荷的交接 (Handoff with Payload)

为了避免用户重复说话，Handoff 函数必须定义参数 Schema。

*   **场景**：用户说“把空调开到 24 度”。
*   **错设计**：Router 切换到 ClimateAgent -> ClimateAgent 问“你要调到多少度？” -> 用户怒。
*   **正确设计**：
    *   Router 工具定义：`transfer_to_climate(initial_intent: str, temperature: int | null)`
    *   执行流：Router 提取 `temperature=24` -> 调用 Handoff -> ClimateAgent 初始化 -> 检测到参数 -> 直接调用 `set_temp(24)`。

### 5.3.3 反向交接与控制权回归 (The "Pop" Strategy)

专家 Agent 完成任务后，必须有明确的“出口”。

1.  **任务完成 (Task Complete)**：调用 `handoff_back_to_supervisor(result_summary="空调已调至24度")`。
2.  **能力不足 (Out of Scope)**：例如在点餐 Agent 中用户问“我的车能跑多快？”，点餐 Agent 必须识别这是“非点餐意图”，调用 `handoff_back_to_supervisor(reason="off_topic", user_query="...")`。

---

## 5.4 状态管理与记忆设计

在多代理系统中，记忆（Memory）需要分层管理。

### 5.4.1 全局会话状态 (Global Context)
存储 Agents SDK 的 `runner` 或顶层上下文中，所有 Agent 可读（部分可写）。
*   **用户画像**：称呼、偏好（如：喜欢冷风）。
*   **车辆实时快照**：当前车速、档位、地理位置（经纬度/城市）、车内人员分布。
    *   *设计要点*：这些信息通常作为 System Prompt 的动态变量注入，每轮对话更新。

### 5.4.2 局部/瞬时状态 (Agent-Local State)
仅在当前 Agent 活跃期间有效。
*   **GUI 状态**：当前页面截图的 Hash、可点击元素列表、购物车内容。
*   **RAG 缓存**：上一次检索到的 5 个文档块（用于支持追问）。

### 5.4.3 状态的生命周期
当从 Agent A 切换到 Agent B 时：
1.  **压栈**：如果 A 的任务未完成（如点餐中途切去开窗），应将 A 的关键状态（购物车）序列化保存到全局 `task_stack`。
2.  **销毁**：如果 A 的任务已完成，直接丢弃 A 的上下文，防止 Token 累积导致后续推理变慢。

---

## 5.5 Realtime API 与 Agents SDK 的集成

这是系统实现的难点：Realtime API 是流式的（WebSocket），而 Agents SDK 逻辑通常是同步或异步的函数调用。

### 5.5.1 事件驱动循环 (The Event Loop)

我们需要一个适配层（Adapter）来桥接两者。

1.  **监听**：适配器监听 Realtime API 的 `response.function_call_arguments.done` 事件。
2.  **解析**：提取 `call_id`、`name` (工具名) 和 `arguments` (JSON)。
3.  **代理执行**：
    *   将工具调用请求转发给 Agents SDK 的 `AgentRunner`。
    *   **如果是 Handoff 工具**：SDK 内部切换 `current_agent` 指针，**不**向 Realtime API 返回结果，而是让新 Agent 生成新的回复指令。
    *   **如果是普通工具**（如 `set_seat`）：执行 Python 函数，获取返回值。
4.  **响应**：适配器通过 `conversation.item.create` (function_call_output) 将结果传回 Realtime API。
5.  **触发生成**：发送 `response.create` 让 Realtime API 基于工具结果成语音。

### 5.5.2 处理打断 (Handling Interruptions)

在 Agent 执行耗时任务（如 RAG 搜索）时，用户可能会打断。

*   **Realtime 端**：收到新语音，发送 `input_audio_buffer.speech_started`。
*   **适配层**：
    1.  立即收到 `response.cancel` 事件。
    2.  **关键动作**：检查 Agents SDK 中是否有正在运行的 **副作用操作**（如正在支付、正在写数据库）。
    3.  如果是**只读操作**（如搜索），立即 `kill` 线程/协程。
    4.  如果是**写操作**，允许其完成（原子性），但丢弃其语音回复，转而处理用户的新指令。

---

## 5.6 常见模式与策略 (Patterns & Strategies)

### 5.6.1 模式：Middle-out 护栏 (The Guardrail Pattern)
不要完全信任 Agent 的输出。在 Agent 和车辆底层 API 之间插入一个 **确定性代码层**。

*   **流程**：Agent 输出 `open_trunk()` -> **Guardrail Interceptor** 拦截 -> 检查 `speed > 5km/h` -> 拒绝执行并抛出 `SecurityError` -> Agent 捕获错误并向用户解释“行驶中无法打开后备箱”。

### 5.6.2 模式：人机协同确认 (Human-in-the-loop)
对于高风险操作（如购买、恢复出厂设置），Agent 不直接调用执行工具。

1.  用户：“帮我下单。”
2.  GUI Agent：调用 `stage_order(items)`（暂存，不提交）。
3.  GUI Agent 回复：“好的，一共 50 元，请确认支付。”
4.  用户：“确认。”
5.  GUI Agent：调用 `execute_payment()`。
> **注意**：必须在 System Prompt 中强制要求 Agent 遵循此“两步走”策略。

---

## 5.7 本章小结

*   **架构决定上限**：Supervisor 模式通过解耦降低了复杂度，是车载多模态交互的最佳实践。
*   **Handoff 是核心**：优秀的 Handoff 设计包含意图、参数（Payload）和上下文，能实现无缝的“无感切换”。
*   **Realtime 适配**：必须处理好 WebSocket 事件流与 Agent 逻辑流的同步，特别是打断（Cancellation）逻辑。
*   **状态隔离**：通过区分全局与局部状态，既保证了体验的连贯性，又防止了 Token 爆炸和隐私泄露。

---

## 5.8 练习题

### 基础题 (50%)

**Q1: 为什么 Supervisor Agent 的 System Prompt 应该尽可能短，且不包含具体业务逻辑？**
<details>
<summary>点击查看提示与参考答案</summary>

*   **提示**：考虑延迟和 Token 成本。
*   **参考答案**：
    1.  **降低延迟 (TTFT)**：Supervisor 是每轮对话的必经之路。Prompt 越短，首字生成速度越快，用户感知到的响应越迅速。
    2.  **减少干扰**：过多的业务细节会干扰 LLM 的路由判断能力，使其倾向于自己回答而不是转交给更专业的专家代理。
    3.  **节省成本**：作为入口，它的调用频率最高，保持轻量级可以显著降低 Token 消耗。
</details>

**Q2: 请设计一个 `transfer_to_music_agent` 的工具定义（JSON Schema），要求能承接用户的模糊意图（如“我想听点放松的）。**
<details>
<summary>点击查看提示与参考答案</summary>

*   **提示**：参数不能只是布尔值，需要传递语义。
*   **参考答案**：
    ```json
    {
      "name": "transfer_to_music_agent",
      "description": "Transfer control to the music specialist when user wants to play media.",
      "parameters": {
        "type": "object",
        "properties": {
          "intent_summary": {
            "type": "string",
            "description": "Summary of user's music request, e.g., 'play relaxing music' or 'play Jay Chou'."
          },
          "mood": {
            "type": "string",
            "enum": ["happy", "relaxing", "energetic", "focus", "unknown"],
            "description": "Detected mood from user utterance."
          }
        },
        "required": ["intent_summary"]
      }
    }
    ```
    *解析*：通过 `intent_summary` 和 `mood`，Music Agent 启动时就可以直接调用推荐算法，而不需要再次问用户“你想听什么样的放松音？”。
</details>

**Q3: 当 RAG Agent 发现用户的问题（如“轮胎气压怎么看”）在手册里找不到答案时，它应该怎么做？**
<details>
<summary>点击查看提示与参考答案</summary>

*   **提示**：不要瞎编，也不要直接结束会话。
*   **参考答案**：
    1.  **尝试回退**：首先判断问题是否可能属于其他 Agent（例如是否可以通过 GUI 查看胎压？）。
    2.  **坦诚回答并交权**：如果确定无法回答，应生成回复“抱歉，我在手册中没找到相关信息”，然后调用 `handoff_back_to_supervisor`，结束当前代理的控制权。
    3.  **切忌**：编造虚假信息（幻觉）或长时间占用控制权不释放。
</details>

### 挑战题 (50%)

**Q4: 复杂场景设计：用户在与 GUI Agent 进行多轮对话（如筛选餐厅）时，Realtime API 突然断线重连。请设计一套恢复机制，确保用户不需要重新开始筛选。**
<details>
<summary>点击查看提示与参考案</summary>

*   **提示**：Session ID 持久化、状态快照。
*   **参考答案**：
    1.  **状态持久化**：每次 Agent 状态变更（如筛选条件更新），后端应将 `current_agent_id` 和 `agent_state` (JSON) 写入 Redis，Key 为 `session_id`。
    2.  **重连握手**：客户端重连时，发送带有 `session_id` 的握手包。
    3.  **状态恢复**：
        *   后端检测到存在的 Session。
        *   Router 此时不接管，而是直接实例化之前的 GUI Agent。
        *   将 `agent_state` 注入 GUI Agent 的 Prompt 或 Memory。
    4.  **主动引导**：恢复连接后，GUI Agent 主动发送一条文本（TTS）："刚才网络有点波动，我们正在筛选距离最近的川菜馆，对吗？"
</details>

**Q5: 竞态条件（Race Condition）：用户语速很快，说“打开空调”（指令A），紧接着又说“算了别开了”（指令B）。Realtime API 可能先后触发两次 Function Call。如何确保空调最后是关闭的？**
<details>
<summary>点击查看提示与参考答案</summary>

*   **提示**：基于时间戳的请求丢弃，或通过 Agent 自身的推理能力。
*   **参考答案**：
    1.  **方案一：Realtime 上下文聚合**。Realtime API 的 VAD 如果设置得当，会将这两句话合并为一个 Turn。模型接收到文本 "打开空调... 算了别开了"，Agent 自身的逻辑会判断出最终意图是“无操作”。
    2.  **方案二：命令队列与防抖**。如果被切分为两个 Turn，后端车控服务应维护一个指令队列。收到指令 A，延迟 500ms 执行。如果在 500ms 内收到指令 B（取消 A），则撤销指令 A。
    3.  **方案三：版本号机制**。Agent 的状态维护一个 `version_id`。指令 B 的处理必须基于指令 A 之后的最新状态。
</details>

**Q6: 如何设计一个“主动关心”功能？例如监测到 DMS 疲劳信号时，Router 应该如何介入当前正在进行的对话（可能是音乐，也可能是航）？**
<details>
<summary>点击查看提示与参考答案</summary>

*   **提示**：Server-side Event 触发、优先级仲裁。
*   **参考答案**：
    1.  **信号注入**：DMS 信号作为 `server_notification` 事件发送给 Agents Runtime。
    2.  **强行打断**：无论当前是哪个 Agent 在运行，Runtime 暂停其执行。
    3.  **切入 Safety Agent**：Runtime 临时将控制权交给 Safety Agent。
    4.  **执行动作**：Safety Agent 调用 TTS 输出“检测到您有些疲劳，是否需要为您播放提神的音乐或导航到最近的服务区？”
    5.  **恢复或分支**：
        *   用户拒绝 -> 恢复之前的 Agent（栈弹出）。
        *   用户接受 -> Handoff 到 Music 或 Nav Agent。
</details>

---

## 5.9 常见陷阱与错误 (Gotchas)

### 5.9.1 陷阱：Handoff 乒乓 (Ping-Pong Loop)
*   **现象**：Router 把任务转给 Agent A，Agent A 觉得自己处理不了又转回 Router，Router 觉得这事儿还得 A 办，又转给 A。系统死循环，Token 耗尽。
*   **原因**：Prompt 定义边界模糊，或者 Handoff 的 System Prompt 没有包含“不要把刚刚收到的任务原封不动退回”的指令。
*   **调试技巧**：
    *   在 Handoff 参数中增加 `history_routing` 字段，记录 `[Router, AgentA]`。
    *   Router 再次想转给 Agent A 时，检查历史，发现已经试过了，强制走 **Fallback 策略**（如道歉并转人工/结束）。

### 5.9.2 陷阱：Agent 的性格分裂
*   **现象**：车控 Agent 说话很机械，切到闲聊 Agent 突然变得很活泼，用户体验割裂感强。
*   **原因**：不同 Agent 的 System Prompt 中的 `Tone & Voice` 定义不一致。
*   **Rule of Thumb**：定义一个 **Global Persona Prompt**（全局人设模版），作为所有 Agent Prompt 的前缀（Prefix）。

### 5.9.3 陷阱：工具参数的幻觉
*   **现象**：用户说“打开那个...呃...副驾的窗户”。Agent 可能会调用 `open_window(seat="driver")`，因模型有偏见。
*   **调试技巧**：
    *   使用 `enum` 严格限制参数范围。
    *   在 System Prompt 中强调：**“如果用户没有明确指定参数，必须先发起询问，严禁猜测。”**

### 5.9.4 陷阱：Realtime 音频截断
*   **现象**：Agent 刚开始说话就被截断。
*   **原因**：Agent 执行 Handoff 后，新 Agent 立即输出了文本，但旧 Agent 的工具调用返回值（Function Output）还没完全传给 Realtime API，导致时序错乱。
*   **调试技巧**：确保 Agent 切换是原子操作，且在发送新文本前，必须先完成上一个 Function Call 的闭环（发送 output frame）。
