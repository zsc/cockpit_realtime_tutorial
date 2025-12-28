# Chapter 8｜车控 ToolCall 设计 (Vehicle Control Tools)

## 1. 开篇段落

车控（Vehicle Control）是车载语音助手最基础也最核心的功能。与聊天机器人不同，车控直接作用于物理世界，涉及座椅移动、温度变化、车窗升降等动作。这要求系统不仅要准确理解意图，还要具备极高的**安全性（Safety）**、**确定性（Determinism）**和**即时性（Real-time Responsiveness）**。

本章将详细设计车控子系统的架构。我们将探讨如何定义工具（Tools），设计一个能够处理权限校验、参数清洗和指令执行的“车控网关”，以及如何利用 OpenAI Realtime API 的 Function Calling 能力实现流畅的“边说边做”体验。我们将结合 Agents SDK 的模式，确保主驾、副驾及后排乘客在不同行驶状态下拥有恰当的控制权限。

**学习目标**：
1. 理解车控工具的分级与风险管理策略。
2. 掌握“车控网关”的中间层设计，实现从 LLM 意图到车辆信号的转换。
3. 学会设计基于身份（DMS/OMS）和车辆状态（行驶/静止）的动态权限矩阵。
4. 熟悉在 Realtime API 中处理工具调用延迟与语音反馈的 UX 模式。

---

## 2. 系统设计与论述

### 8.1 工具目录与分级 (Tool Catalog & Classification)

在 OpenAI Realtime API 中，所有的车控能力都通过 `tools` 定义暴露给模型。为了安全和管理的便利，我们需要对工具进行严格的分级。

**Rule of Thumb**：不要创建一个万能的 `set_car_state` 函数。应保持工具的**原子性（Atomicity）**和**语义清晰性**，有助于模型准确选择。

#### 工具分级金字塔
1.  **L1 绿色（信息/娱乐）**：无物理动作风险极低。
    *   *例子*：`play_music`, `volume_up`, `switch_ambient_light_color`.
    *   *策略*：无需二次确认，响应最快。
2.  **L2 黄色（舒适性/座舱物理）**：改变座舱环境，通常无行车安全风险。
    *   *例子*：`set_ac_temperature`, `turn_on_seat_heating`, `open_sunroof`.
    *   *策略*：需要参数校验，建议在执行前或执行中给予语音反馈。
3.  **L3 红色（高风险/行车相关）**：涉及车辆行驶部件或对外物理边界。
    *   *例子*：`unlock_doors`, `open_trunk`, `set_drive_mode` (如切换到运动模式).
    *   *策略*：**必须**进行二次确认（Human-in-the-loop），且在行驶中通常被禁用。

### 8.2 车控网关架构 (Vehicle Control Gateway)

LLM 输出的仅仅是 JSON 参数，不仅不可靠，而且无法直接驱动车辆 CAN/SOA 总线。我们需要一个“车控网关”作为防腐层。

#### 架构数据流图 (ASCII)

```ascii
+---------------------+      +----------------------+
|  OpenAI Realtime    |      |  Agents SDK Runtime  |
|  (Session/Voice)    |<---->|  (Orchestrator)      |
+----------+----------+      +-----------+----------+
           ^                             |
           | (Tool Call: Name, Args)     | (1. Route & Validate)
           v                             v
+---------------------------------------------------+
|             Vehicle Control Gateway               |
+---------------------------------------------------+
| [Security Layer]                                  |
|  - AuthZ (Who are you?)                           |
|  - Safety Check (Is car moving?)                  |
+---------------------------------------------------+
| [Normalization Layer]                             |
|  - "hotter" -> +2 degrees                         |
|  - "max" -> 32.0 (Model specific)                 |
+---------------------------------------------------+
| [Execution Layer]                                 |
|  - Debounce/Rate Limit (Anti-spam)                |
|  - Signal Dispatch (SOA/CAN)                      |
+--------------------------+------------------------+
                           |
                           v
                 +-------------------+
                 | Vehicle Hardware  |
                 | (ECU / Domain)    |
                 +-------------------+
```

**关键组件职责**：
1.  **Security Layer**：拦截非法请求。例如，后排乘客（通过 OMS 识别）试图控制主驾座椅，或者车辆时速 > 0 时试图打开后备箱。
2.  **Normalization Layer**：处理模糊语义。如果模型输出 `temp="warm"`, 网关应依据当前温度将其转换为具体数值（如 `current + 2`）。
3.  **Execution Layer**：防抖动（Debounce）。防止用户连续喊“调高调高调高”导致模型在一秒内触发 5 次 API，网关应合并或平滑这些指令。

### 8.3 动态权限矩阵 (Dynamic Authorization Matrix)

利用感知系统（DMS/OMS）的信息，我们不再把所有语音指令视为等同。Agents SDK 在执行工具前，必须通过 `PermissionGuard`。

| 请求者身份 (Who) | 车辆状态 (State) | 操作目标 (What) | 权限动作 (Action) |
| :--- | :--- | :--- | :--- |
| **主驾 (Driver)** | 行驶中 (Driving) | 播放音乐 | ✅ 允许 |
| **主驾 (Driver)** | 行驶中 (Driving) | 看视频/输入法 | ❌ 拒绝 (安全法规) |
| **副驾 (Passenger)** | 任意 | 主驾座椅调节 | ❌ 拒绝 (干扰驾驶) |
| **后排 (Rear)** | 任意 | 自己区域空调 | ✅ 允许 |
| **后排 (Rear)** | 任意 | 全车车窗 | ⚠️ 降级 (仅允许开自己侧) |
| **儿童 (OMS检测)** | 任意 | 车门/车窗 | ❌ 拒绝 (开启童锁逻辑) |

**Rule of Thumb**：**“谁主张，谁受益，不干扰他人。”** 除非是明确的全局指令（如“打开全车空调”），否则默认将作用域限制在发出语音的乘客区域（Zone-based Control）。

### 8.4 工具调用与 Realtime 交互模式

在 Realtime API 中，工具调用会产生事件流。为了保证体验流畅，需要精心设计交互时序。

**模式 A：先说后做 (Speak-then-Act)**
*   **适用**：复杂指令或需要确认的操作。
*   *流程*：用户：“帮我把副驾座椅加热打开。” -> Agent：“好的，正在为您开启副驾座椅加热。” -> (调用工具) -> (工具返回成功) -> Agent：“已开启。”
*   *优点*：用户有确定感。

**模式 B：边做边说/静默执行 (Act-and-Acknowledge)**
*   **适用**：低延迟、高频操作（L1 类）。
*   *流程*：用户：“声音大点。” -> (立即调用工具) -> (音量物理提升) -> Agent (可选简短回复)：“嗯” / “好了”。
*   *设计要点*：在 Realtime API 中，收到 `response.function_call_arguments.done` 事件时，即可并行下发指令，而不必等待整个音频生成的文本结束。

### 8.5 异常处理与“反悔”机制 (Undo/Cancel)

车控涉及物理改变，必须提供“后悔药”。

1.  **Undo 栈**：网关层维护一个短期的作历史栈。
    *   用户：“太热了。” (系统执行：温度 24 -> 20)
    *   用户：“不对，太冷了，撤销。”
    *   系统：执行 `pop()`，恢复到 24 度。
2.  **Stop 信号**：
    *   Realtime API 支持**打断（Barge-in）**。当监测到用户在工具执行后的语音流中喊出“停！”、“别打开！”时，需立即触发紧急停止工具（Emergency Stop Tool）。

---

## 3. 本章小结

*   **分级管理**：将车控工具按风险分为 L1/L2/L3，高风险操作必须有人类确认。
*   **网关模式**：不要让 LLM 直连车辆总线，**车控网关**负责所有权限校验、模糊参数映射和防抖。
*   **动态权限**：结合 DMS/OMS 的身份识别和车辆行驶状态，构建动态权限矩阵，确保安全。
*   **交互时序**：根据任务类型选择“先说后做”或“静默执行”，利用 Undo 机制提升容错率。

---

## 4. 练习题

### 基础题

**Q1. 工具定义**
请为“调整空调温”设计一个 Function Schema（描述性定义），包含必要的参数和约束。
<details>
<summary>点击查看参考答案</summary>

```json
{
  "name": "set_ac_temperature",
  "description": "Adjust the air conditioner temperature for a specific zone within the car. Valid range is 16 to 32 degrees Celsius.",
  "parameters": {
    "type": "object",
    "properties": {
      "temperature": {
        "type": "number",
        "description": "Target temperature in Celsius.",
        "minimum": 16,
        "maximum": 32
      },
      "zone": {
        "type": "string",
        "enum": ["driver", "passenger", "rear_left", "rear_right", "all"],
        "description": "The specific zone to adjust. Defaults to the zone where the user is sitting if known.",
        "default": "all"
      },
      "mode": {
          "type": "string",
          "enum": ["absolute", "relative"],
          "description": "Whether to set to a specific value or adjust relative to current."
      }
    },
    "required": ["temperature"]
  }
}
```
**提示**：注意 `zone`（分区）的重要性，以及数值的 `min/max` 约束。
</details>

**Q2. 权限判断**
车辆正在高速公路上以 100km/h 行驶。后排座位上的 6 岁儿童喊道：“打开天窗！”。根据本章的权限策略，系统应如何反应？
<details>
<summary>点击查看参考答案</summary>

**系统行为**：拒绝执行。
**逻辑链路**：
1. **身份识别**：OMS 识别声源位置为后排，且特征为儿童（Child）。
2. **状态检查**：车辆状态为 `Driving` (High Speed)。
3. **策略匹配**：
   - 规则 A：儿童通常无权控制车窗/天窗（童锁逻辑）。
   - 规则 B：高速行驶中打开天窗可能产生高风噪或危险。
4. **输出**：语音助手回复“为了安全，现在不能打开天窗哦”，并未调用 `open_sunroof` 工具。
</details>

**Q3. Realtime API 事件流**
在 Realtime API 中，当模型决定调用工具时，客户端会收到哪些关键事件？请按顺序列出。
<details>
<summary>点击查看参考答案</summary>

1. `response.item_created`: 确认生成了一个 item。
2. `response.function_call_arguments.delta`: 流式传输工具参数（JSON 片段）。
3. `response.function_call_arguments.done`: 参数传输完成，客户端收到完整 JSON。
4. **(客户端此时执行本地车控逻辑)**
5. `conversation.item.create`: 客户端将工具执行结果（Tool Output）回传给服务端。
6. `response.create`: 触发服务端根据执行结果生成后续语音回应。
</details>

---

### 挑战题

**Q4. 模糊指令处理**
用户说：“我觉得有点闷。” 这句话没有明确调用任何工具。请设计一个 Agent 策略，使其能够通过多步推理最终改善车内环境。
<details>
<summary>点击查看参考答案</summary>

**设计思路**：
1. **意图理解**：“闷”通常意味着空气不流通或温度过高。
2. **状态查询**：Agent 首先调用 `get_car_status()` 查看当前车窗、空调、外循状态。
3. **决策逻辑**：
   - 如果空调关着 -> 建议打开空调。
   - 如果空调开着但也是内循环 -> 建议切换外循环或微开车窗。
   - 如果 PM2.5 高 -> 建议打开空气净化。
4. **交互脚本**：
   - Agent（思考后）：“现在车内二氧化碳浓度稍高，需要我帮您打开外循环，或者把车窗开条缝吗？”
   - 用户：“开个缝吧。”
   - Agent：调用 `control_window(action="vent")`。
</details>

**Q5. 竞态条件 (Race Condition)**
主驾喊“打开空调”，几乎同时副驾喊“关闭空调”。在 Realtime 系统中如何处理这种并发冲突？
<details>
<summary>点击查看参考答案</summary>

**策略**：**最后写入胜出（Last Write Wins）** 或 **主驾优先（Driver Priority）**，取决于产品定义。
**推荐方案（主驾优先）**：
1. 网关层设置一个极短的时间窗口（例如 500ms）。
2. 如果窗口内收到相互冲突的指令（开 vs 关），检查声源身份。
3. 优先执行主驾指令，丢弃副驾指令。
4. **语音反馈**：明确告知结果。“已为您（主驾）打开空调。”（以此暗示副驾的指令被覆盖）。
</details>

**Q6. 容错与回退**
Realtime API 偶尔会产生幻觉，输出一个不存在的工具参数（例如 `set_temp(val="very_cold")`，但 schema 要求是数字）。网关层应该如何处理这种情况以避免崩溃？
<details>
<summary>点击查看参考答案</summary>

1. **Schema Validation**：在网关入口处使用 JSON Schema Validator（如 Pydantic）强校验。
2. **Error Catching**：捕获验证错误。
3. **Self-Correction (自愈)**：
   - **方案 A（简单）**：向 Realtime API 回传错误信息 `ToolError: "very_cold" is not a valid number`，让模型重试。
   - **方案 B（启发式）**：网关层做一个简单的映射表（heuristic map），将 `very_cold` 映射为 `18` 度，`hot` 映射为 `28` 度。
   - **方案 C（询问）**：不执行，直接让 TTS 播报“您想调到多少度？”
**推荐**：对于实时语音，方案 C 或 A 较好，避免自作聪明导致误操作。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

1.  **无限递归调用**：
    *   *现象*：模型调用 `get_seat_status`，得到结果后，又觉得信息不够，再次调用同一工具，陷入死循环。
    *   *调试*：检查 System Prompt，确保明确告知模型拿到结果后应该向用户汇报。在 Agents SDK 中设置 `max_steps` 限制。

2.  **默认全车控制的灾难**：
    *   *现象*：后排乘客说“座椅往后调点”，结果主驾座椅动了，导致驾驶危险。
    *   *调试*：工具参数 `zone` 必须是 Required，且在 Prompt 中强制要求模型结合声源定位（Source Localization）填充此参数，如果不知道声源，必须反问。

3.  **网络延迟与 CAN 总线延迟不匹配**：
    *   *现象*：用户说“打开车窗”，Agent 立即回答“好的已打开”，但实际上车窗电机 2 秒后才动。用户会以为系统没听懂，又喊了一遍，结果车窗开了又关。
    *   *调试*：区分“指令已下发”和“动作已完成”。对于慢速动作，话术应为“正在打开...”，或者使用“乐观更新”策略但保持静默，直到动作开始有物理反馈。

4.  **音量调节的“爆音”风险**：
    *   *现象*：模型将 `set_volume(level)` 中的 level 错误理解为百分比（0-100），而车机底层 API 范围是 0-30。模型传入 50，导致最大音量。
    *   *调试*：网关层必须做**硬限幅（Clamping）**。无论模型传多少，不能超过安全阈值。

5.  **隐私模式下的冲突**：
    *   *现象*：车主开启了“代客泊车模式”或“隐私模式”，语音助手仍然允许通过 RAG 搜索或工具调用访问行车记录仪数据。
    *   *调试*：车控网关必须是所有工具调用的**最终守门员**，不仅校验参数，还要校验当前车辆的全局安全模式状态
