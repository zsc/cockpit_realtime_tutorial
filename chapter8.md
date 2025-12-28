# Chapter 8｜车控 ToolCall 设计 (Vehicle Control Tools)

## 1. 开篇段落

本章聚焦于语音助手与物理车辆硬件交互的核心机制——**车控工具调用（Vehicle Control ToolCall）**。在 OpenAI Realtime API 和 Agents SDK 的架构下，大模型不仅仅是对话者，更是车辆的“执行中枢”。

车控系统的设计难度在于其连接了两个截然不同的世界：一个是**概率性、模糊语义**的 LLM 世界，另一个是**确定性、硬实时、人命关天**的嵌入式工程世界。

**学习目标**：
1.  **全景架构**：掌握从 LLM 意图到 CAN/SOA 信号的完整转换链路。
2.  **防御性设计**：学习如何通过“车控网关”构建鉴权（AuthZ）参数清洗（Sanitization）和安全门禁（Safety Gate）。
3.  **模糊语义映射**：如何处理“开一点窗”、“调得凉快点”等相对指令与硬件绝对参数之间的转换。
4.  **状态一致性**：理解乐观更新、最终一致性在车载交互中的应用，以及如何处理硬件故障导致的回滚。

---

## 2. 核心设计论述

### 2.1 车控工具的分层架构 (Layered Architecture)

为了解耦模型与硬件，我们采用经典的**三层漏斗模型**设计 ToolCall。

```ascii
[ User Voice ] "我有点热"
      |
      v
[ Layer 1: Intent & Semantic Layer ] (LLM / Agent)
   -> 识别意图: AdjustClimate
   -> 提取参数: {"direction": "cooler", "magnitude": "small"}
      |
      v
[ Layer 2: Vehicle Control Gateway (VCG) ] (Middleware)
   -> 语义映射: "cooler" + current(24°C) = target(23°C)
   -> 安全校验: 检查车辆电源状态、检查儿童锁
   -> 权限裁决: 副驾是否有权调整全车温度？
      |
      v
[ Layer 3: Vehicle Hardware Abstraction Layer (V-HAL) ] (Embedded)
   -> 信号转换: target(23°C) -> CAN Message ID 0x123, Data 0x17
   -> 硬件执行: 发送至空调控制器 (HVAC ECU)
```

### 2.2 工具目录设计与粒度控制 (Granularity & Taxonomy)

工具定义的粒度直接决定了 LLM 的理解准确率和令牌消耗。我们推荐**“面向意图”**而非**“面向寄存器”**的定义方式。

#### 2.2.1 推荐的工具定义模式

1.  **聚合型工具 (Aggregated Tools)**
    *   *Bad:* `set_fan_speed(level)`, `set_ac_mode(on/off)`, `set_temp(val)` 分散定义。
    *   *Good:* `adjust_climate(temperature, fan_speed, ac_mode, zone)`。
    *   *理由:* 用户常说“把空调开到24度，风大点”。聚合工具允许一次 ToolCall 完成多个寄存器修改，减少 Realtime API 的来回交互（Turn-taking）延迟。

2.  **场景化工具 (Scenario Tools)**
    *   定义一键直达的高级工具，对应车辆的“模式”。
    *   `enable_nap_mode()`: 同时执行（座椅躺平 + 车窗关闭 + 播放白噪音 + 闹钟设置）。
    *   `enable_baby_mode()`: 同时执行（音量限制 + 后排车窗锁止 + 空调柔风）。
    *   *Rule-of-Thumb:* 如果一个操作序列在车机屏幕上有一个专门的“一键启动”按钮，它就应该是一个独立的 Tool。

#### 2.2.2 风险分级矩阵 (Expanded)

| 等级 | 标识 | 定义 | 典型功能 | 交互/鉴权策略 |
| :--- | :--- | :--- | :--- | :--- |
| **L0** | `INFO` | 只读，无物理动作 | 查续航、胎压、说明书 | 直接返回数据，无鉴权 |
| **L1** | `COMFORT` | 可逆，低干扰 | 音乐、氛围灯、香氛 | 隐式确认（"好的"），全员可用 |
| **L2** | `PHYSICAL` | 物理动作，低风险 | 座椅加热、空调风向 | 需检查能源状态，建议区分座次权限 |
| **L3** | `CAUTION` | 物理动作，环境敏感 | 车窗、天窗、遮阳帘 | **环境校验**（雨天/高速），失败需解释 |
| **L4** | `CRITICAL` | 涉及行驶安全 | 后备、车门锁、大灯 | **强制二次确认** + **零车速门禁** + 仅主驾 |
| **L5** | `BLOCKED` | 法律/安全禁止 | 油门、转向、换挡 | 不暴露给 LLM，物理层隔离 |

### 2.3 车控网关 (VCG) 的核心逻辑

VCG 是本系统的核心守门员，它运行在车机的高算力芯片（如 Qualcomm 8155/8295）的 Android/QNX 层。

#### 2.3.1 影子状态 (Shadow State) 与参数补全
LLM 说明的指令往往是不完整的（Partial）。VCG 必须维护一份车辆状态的**缓存副本（Shadow State）**。

*   **场景**: 用户说“把副驾窗户也关上”。
*   **Context**: 上一轮对话刚关了主驾窗户。
*   **逻辑**:
    1.  LLM 输出: `control_window(seat="passenger", action="close")`。
    2.  VCG 读取 Shadow State: 发现副驾窗户当前位置 `pos = 50%`。
    3.  VCG 发现指令缺失目标位置，默认 `action="close"` 意味着 `target_pos = 0%`。
    4.  如果 Shadow State 显示副驾窗户已经是 `0%`，VCG 直接拦截并返“已关闭”，不发送 CAN 信号（节省总线带宽）。

#### 2.3.2 模糊语义映射 (Fuzzy Semantic Mapper)
LLM 擅长处理自然语言，但不擅长处理车企特定的物理量化标准。VCG 需负责“翻译”。

*   **相对量处理**:
    *   指令: `adjust_temp(delta="hotter")`
    *   映射: `Target = Current_Temp + Step_Size (e.g., 1.0°C)`
    *   边界: `Target = min(Target, Max_Limit)`
*   **程度词处理**:
    *   指令: `open_window(extent="a little bit")`
    *   映射: `Target = Current_Pos + 10%` (配置项: `define_little_bit = 10%`)
    *   指令: `open_window(extent="halfway")`
    *   映射: `Target = 50%`

### 2.4 安全策略与权限矩阵 (Safety & Authorization)

安全是车载系统的底线。我们采用**拦截器模式 (Interceptor Pattern)**。

#### 2.4.1 动态权限检查伪逻辑
```text
function authorize_action(user_role, vehicle_state, tool_name, params):
    # 1. 全局互斥锁 (e.g. 正在OTA升级中，正在紧急呼叫中)
    if GlobalLock.is_locked(): return DENY("系统占用中")

    # 2. 身份鉴权
    if user_role == "GUEST" and tool_name in ["open_trunk", "user_profile"]:
        return DENY("访客模式无权操作")

    # 3. 驾驶状态门禁
    if vehicle_state.speed > 0:
        if tool_name in ["open_trunk", "video_playback", "pairing_bluetooth"]:
            return DENY("行驶中禁止操作")
        if tool_name == "open_sunroof" and vehicle_state.speed > 100:
            return DENY("车速过快，禁止开天窗")

    # 4. 环境感知互斥
    if tool_name == "open_window" and vehicle_state.rain_sensor == ON:
        return CONFIRM_REQUIRED("正在下雨，确定要开窗吗？")

    return ALLOW
```

#### 2.4.2 声区定向与资源绑定
利用麦克风阵列的**声源定位 (DOA)**，将语音指令绑定到特定座位。
*   用户（后左位置）：“打开座椅加热”。
*   Agent 接收到的 ToolCall 参数应自动注入：`seat_id="rear_left"`。
*   *Gotcha:* 如果 LLM 错误地输出了 `seat_id="driver"`，VCG 应基于 DOA 信号进行二次校验或纠正。

### 2.5 反馈机制与延迟管理

车控动作的执行时间差异巨大（毫秒级到秒级）。

#### 2.5.1 响应策略分类
1.  **即发即弃 (Fire-and-Forget)**: 用于无副作用、极快的操作。
    *   *例:* “切歌”。
    *   *表现:* 语音回复“好的”与此同时发送信号。
2.  **乐观响应 (Optimistic Response)**: 假设大概率成功。
    *   *例:* “调高温度”。
    *   *表现:* 立即回复“已调高”，如果 2秒后 ECU 报错，再异步播报“抱歉，空调刚才响应超时”。
3.  **事务性响应 (Transactional Wait)**: 必须等待物理确认。
    *   *例:* “查询胎压”。
    *   *表现:* 
        *   LLM: 调用工具 `get_tire_pressure()`。
        *   (Silence / Filler sound "Hmm...") -> 等待 500ms 信号返回。
        *   LLM: 拿到结果后生成回复。

#### 2.5.2 处理长延迟 (Long-tail Latency)
对于如“打开敞篷这种耗时 10秒+ 的操作：
1.  **阶段 1**: 立即回复“正在打开敞篷...”。
2.  **阶段 2**: 保持静默，或播放机械音效。
3.  **阶段 3**: (通过 Agents SDK 的异步事件) 动作完成后，主动推送一条“敞篷已打开”。

---

## 3. 本章小结

*   **网关模式 (Gateway Pattern)**: VCG 是连接 AI 概率世界与工程确定性世界的桥梁，负责清洗、映射和鉴权。
*   **安全分级**: 必须建立严格的 L0-L5 风险分级，不同等级对应不同的门禁策略（隐式/显式/禁止）。
*   **状态驱动**: 工具参数应尽量使用“目标状态”（Set Target）而非“动作差值”（Delta），以确保幂等性。
*   **环境感知**: 车控不仅仅是听懂指令，还要结合车速、雨量、座椅占用等传感器数据做决策。

---

## 4. 练习题

### 基础题

**Q1. 什么是“幽灵操作” (Ghost Operation)？在车控场景中如何通过设计避免它？**
<details>
<summary>点击展开答案提</summary>
<strong>Hint</strong>: 考虑 VAD（语音活动检测）误触发或模型幻觉。
<br>
<strong>Answer</strong>:
<strong>幽灵操作</strong>指用户未发指令或仅是在闲聊，系统却误识别为指令并执行了物理动作。
<br>
<strong>避免设计</strong>:
1. <strong>唤醒词依赖</strong>: 高风险操作（L3+）必须在带有唤醒词的对话中，ec 降低免唤醒的误触发率。
2. <strong>二次确认</strong>: 对于敏感操作（开窗/开后备箱），如果置信度低于 0.9，强制反问“您是想打开后备箱吗？”。
3. <strong>可视化反馈</strong>: 执行前在中控屏弹窗倒计时（Toast），允许用户点击“撤销”。
</details>

**Q2. 为什么建议使用 <code>adjust_climate(...)</code> 这样一个大工具，而不是拆分成 <code>set_temp</code>, <code>set_fan</code> 等多个小工具？**
<details>
<summary>点击展开答案提示</summary>
<strong>Hint</strong>: 关注 LLM 的推理成本和 Realtime API 的往返迟。
<br>
<strong>Answer</strong>:
1. <strong>减少延迟</strong>: 用户常在以一句话中包含多个指令（“太热了，风大点”）。如果拆分，模型可能需要生成两个独立的 ToolCall，甚至分两轮对话完成，增加等待时间。
2. <strong>关联逻辑</strong>: 空调各参数是联动的。例如“最大制冷模式”可能意味着同时调整温度最低、风量最大、内循环开启。一个聚合工具更容易在代码层处理这种组合逻辑。
3. <strong>Token 节省</strong>: 减少工具列表定义的长度（Schema Size），降低 Input Token 消耗。
</details>

**Q3. 在设计“座椅加热”工具时，如何处理“主驾”和“副驾”的区分？**
<details>
<summary>点击展开答案提示</summary>
<strong>Hint</strong>: 显式参数 vs. 隐式上下文。
<br>
<strong>Answer</strong>:
工具定义中应包含 <code>seat_id</code> 参数 (enum: <code>driver, passenger, rear_left, rear_right</code>)。
1. <strong>显式指定</strong>: 用户说“打开副驾座椅加热”，LLM 提取 <code>seat_id="passenger"</code>。
2. <strong>隐式推断</strong>: 用户说“我有点冷”，LLM 输出 <code>seat_id="current_speaker"</code> (或留空)。
3. <strong>网关解析</strong>: VCG 接收到 "current_speaker" 后，查询音频模块的声源定位 (DOA) 结果，将其替换为实际的物理座位 ID。
</details>

### 挑战题

**Q4. (架构设计) 设计一个“自动泊车”的语音交互流程。考虑到这是一个高风险、长耗时的过程，请详细描述从用户发出指令到泊车结束的每一个步骤，包括屏幕交互和异常处理。**
<details>
<summary>点击展开答案提示</summary>
<strong>Hint</strong>: 涉及 ToolCall、GUI 联动、持续监控、用户接管。
<br>
<strong>Answer</strong>:
1. <strong>唤醒与意图</strong>: 用户“帮我泊车”。
2. <strong>环境检查 (Pre-check)</strong>: VCG 检查车速 < 10km/h，挡位 D/R，且感知系统已识别到车位。
   - <em>异常分支</em>: 未识别车位 -> 回复“未检测到可用车位，请继续向前行驶”。
3. <strong>GUI 确认 (Selection)</strong>: 屏幕高亮显示识别到的车位 (A, B, C)。
   - Bot: “找到 3 个车位，要停在左边这个吗？”
4. <strong>工具调用</strong>: 用户“对，就这个”。LLM 调用 <code>start_auto_parking(slot_id="A")</code>。
5. <strong>安全握手 (Handshake)</strong>:
   - 系统不直接行动，而是弹窗并语音提示：“请松开刹车，双手离开方向盘，随时准备接管。”
   - 用户确认（可能是语音“好了”，或点击屏幕“开始”）。
6. <strong>执行与监视 (Execution)</strong>:
   - 车辆开始运动。
   - 语音系统进入“实时播报模式”：“正在倒车... 正在避让行人...”。
7. <strong>完成</strong>: 动作结束，挂 P 挡。Bot: “泊车已完成。”
</details>

**Q5. (模糊逻辑) 用户说“把窗户开个缝”。请编写一段伪代码，说明 VCG 如何将其转化为具体的 CAN 信号。假设车窗全关是 0%，全开是 100%。**
<details>
<summary>点击展开答案提示</summary>
<strong>Hint</strong>: 需要读取当前状态，并处理边界情况。
<br>
<strong>Answer</strong>:
<pre><code>
function handle_open_crack(seat_id):
    # 1. 定义 "缝" 的大小
    const CRACK_SIZE = 15% 
    
    # 2. 获取当前位置 (Shadow State)
    current_pos = vehicle.windows[seat_id].position
    
    # 3. 计算目标位置
    # 如果已经在开着，就在当前基础上再开一点？还是保持最小通风位置？
    # 策略：如果当前几乎全关 (<5%)，设为 15%。
    # 如果当前已经开很大了 (>20%)，"开个缝"可能意味着"关小到只剩个缝"。
    
    if current_pos < 5:
        target_pos = 15
    elif current_pos > 20:
        target_pos = 15  # 意图理解为：关小到缝
    else:
        # 已经在缝的状态，可能用户没看清，保持不变或微调
        target_pos = 15 
        return Response("窗户已经开了一条缝了")

    # 4. 下发指令
    vehicle.hardware.set_window(seat_id, target_pos)
    return Response("已打开通风模式")
</code></pre>
</details>

**Q6. (回滚策略) LLM 调用了 <code>set_seat_heat(level=3)</code>，但底层硬件返回 `ERROR_OVERHEAT_PROTECTION` (过热保护中)。此时 Realtime Session 应当如何处理？**
<details>
<summary>点击展开答案提示</summary>
<strong>Hint</strong>: Tool Output 需要包含错误信息，Agent 需要理解错误并转述。
<br>
<strong>Answer</strong>:
1. <strong>VCG 层</strong>: 捕获硬件错误码，不抛出异常导致程序崩溃，而是封装为结构化结果：
   <code>{"status": "failed", "error_code": "E_OVERHEAT", "user_msg": "座椅温度过高，已触发保护"}</code>
2. <strong>Tool Output 注入</strong>: 将上述 JSON 推送回 Realtime API 的会话上下文。
3. <strong>Agent 推理</strong>: 模型看到 status=failed，不再回复“好的，已开启”，而是根据 `user_msg` 生回复。
4. <strong>TTS 输出</strong>: “抱歉，检测到座椅温度过高，系统已触发过热保护，暂时无法开启加热。”
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 权限穿透 (Privilege Escalation)
*   **错误**: 简单地将车控 API 暴露给 Agent，没有检查**谁**在说话。
*   **后果**: 后排顽皮的儿童喊“打开后备箱”，车在高速上行驶时后备箱弹开。
*   **对策**: 所有的 Critical 工具必须校验 `Speaker_Role` 和 `Vehicle_Speed`。

### 5.2 状态震荡 (State Flapping)
*   **错误**: 用户说“温度调高”，Gateway 读取当前是 20度，设为 21度。用户紧接着（1秒内）又说“再高点”。
*   **问题**: 由于 CAN 总线延迟，第二次读取时“当前温度”可能还没更新成 21度，依然读到 20度，结果还是设为 21度。用户感觉系统没反应。
*   **对策**: **乐观锁定**或**本地预测**。Gateway 在发送指令后，应立即更新本地的 Shadow State 为 21度（即使硬件还没反馈），后续基于 21度计算。

### 5.3 忽略多模态冲突
*   **错误**: 用户在看电影，突然想调空调。语音助手回复“好的！已为您调整！”声音巨大，打断了电影氛围。
*   **对策**: 识别到 `Media_Status == Playing` 时，车控类指令应**静默执行**（不播放 TTS），或者仅以极短的提示音（"Ding"）作为反馈，并在屏幕上显示 Toast。

### 5.4 过于自信的“已执行”
*   **错误**: 只要发送了信号就回复“已打开”。
*   **后果**: 假如电机故障，窗户没动，用户会觉得系统在撒谎。
*   **对策**: 对于容易感知失败的动作（如天窗），话术应留有余地，或者使用 Transactional 模式确认硬件状态变化后再回复。

### 5.5 误触发“全车控制”
*   **错误**: 用户只想开自己的窗户，说“打开窗户”，结果四个窗户全开了。
*   **对策**: 默认作用域（Scope）应最小化。如果未指定位置，优先操作当前说话人的位置（DOA），而不是全车。必须显式说“打开**所有**窗户”才执行全车操作。
