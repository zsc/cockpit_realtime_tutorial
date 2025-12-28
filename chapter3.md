# Chapter 3｜系统总体架构 (System Architecture)

## 1. 开篇段落

本章是整个系统设计的蓝图。与传统的基于 HTTP 请求/响应的语音助手不同，基于 **OpenAI Realtime API** 的座舱系统是一个**全双工（Full-Duplex）、事件驱动（Event-Driven）且端云深度协同**的复杂分布式系统。

在本章中，我们将系统解构为三个物理与逻辑平面：**车端感知执行平面**（Vehicle Edge）、**实时连接平面**（Connectivity Layer）和**云端认知编排平面**（Cloud Cognitive Layer）。不仅要解决“如何对话”的问题，更要解决在 4G/5G 网络抖动下如何保持连接、如何处理多模态数据的高吞吐量、以及如何确保云端的大模型不会因为“幻觉”而威胁行车安全。

**学习目标**：
1.  **全局观**：掌握从麦克风采集到车轮转动的完整数据链路。
2.  **连接技术**：理解为何以及如何在车端实现 WebRTC/WebSocket 双通道连接。
3.  **编排逻辑**：深入理解 OpenAI Realtime API（负责拟人交互）与 Agents SDK（负责任务执行）的“双脑协作”模式。
4.  **安全架构**：掌握“不可信云端”与“安全车端”之间的隔离网关设计。

---

## 2. 文字论述

### 3.1 架构设计理念与拓扑

我们遵循以下核心设计原则：
*   **云脑端手 (Cloud Brain, Edge Body)**：复杂的推理、知识检索、多轮对话在云端；信号采集、指令执行、即时反馈在车端。
*   **悲观安全策略 (Pessimistic Safety)**：假设云端指令可能出错、被劫持或超时，车端网关拥有最终否决权。
*   **流式优先 (Streaming First)**：所有数据（音频、文本、工具参数）尽可能流式传输，拒绝批处理等待，以压榨每一毫秒的延迟。

#### 系统逻辑拓扑图 (ASCII)

```ascii
[ User / Cabin Environment ]
       ^      |      |
(Audio)|      |(Img) |(Touch)
       v      v      v
+-----------------------------------------------------------------------------------+
|  Vehicle Edge (Client Zone) - Trust Level: High (ASIL-B/QM)                       |
+-----------------------------------------------------------------------------------+
|  1. Perception Layer (Sensors)                                                    |
|  +-------------+   +------------------+   +-------------------+                   |
|  | Mic Array   |   | Cameras (DMS/7V) |   | Screen/IVI System |                   |
|  +-[DSP/AEC]---+   +--[FrameSampler]--+   +-[AccessService]---+                   |
|         |                   |                       |                             |
|         v                   v                       v                             |
|  2. Realtime Client Adapter (Mediator)                                            |
|  +-----------------------------------------------------------------------+        |
|  |  Session Manager (Auth, Reconnect, State Sync)                        |        |
|  |  +-----------------------+   +-------------------------------------+  |        |
|  |  | Audio Stream (WebRTC) |   | Control/Event Channel (WebSocket)   |  |        |
|  |  +-----------------------+   +-------------------------------------+  |        |
|  +-----------------------------------------------------------------------+        |
|                                     ^                                             |
|                                     | (Local Fallback & Feedback)                 |
|  3. Execution Layer (Actuators)     v                                             |
|  +---------------------+      +---------------------+                             |
|  | Vehicle Control GW  | <--> | Local TTS/Player    |                             |
|  | (Safety Check/CAN)  |      | (Buffer Mgmt)       |                             |
|  +---------------------+      +---------------------+                             |
+-------------+-----------------------+---------------------------------------------+
              | (RTP Media)           | (JSON Events / Tool Calls)
              |                       |
      [ Secure Network Tunnel (TLS 1.3 / mTLS) ]
              |                       |
+-------------+-----------------------+---------------------------------------------+
|  Cloud / Edge (Server Zone) - Trust Level: Zero Trust                             |
+-----------------------------------------------------------------------------------+
|  4. Conversation Layer (OpenAI Realtime API)                                      |
|  +-------------------------------------------------------------+                  |
|  |  Model Inference (GPT-4o class)                             |                  |
|  |  [VAD] -> [STT] -> [Reasoning] -> [TTS]                     |                  |
|  |  ^            | (Function Call Intent)                      |                  |
|  |  | (Context)  v                                             |                  |
|  +--+----------------------------------------------------------+                  |
|     | Events                                                                      |
|     v                                                                             |
|  5. Orchestration Layer (Agents SDK Runtime)                                      |
|  +-------------------------------------------------------------+                  |
|  |  Main Orchestrator (Router)                                 |                  |
|  |  +-------------+  +-------------+  +-------------+          |                  |
|  |  | CarCtrl Agt |  | KnowledgeAg |  | GUI Auto Agt|          |                  |
|  |  +-------------+  +-------------+  +-------------+          |                  |
|  +--------+----------------+-------------------+---------------+                  |
|           |                |                   |                                  |
|        [Vehicle]         [Vector DB]       [App API/Map]                          |
|        [State DB]        (RAG)             (Services)                             |
+-----------------------------------------------------------------------------------+
```

### 3.2 详细组件职责分解

#### 3.2.1 车端：感知与连接适配器 (Vehicle Client Adapter)

这是驻留在车机（IVI, Android/QNX）上的核心服务。

1.  **Audio Engine (音频引擎)**
    *   **AEC Loopback**：必须将系统音频输出（TTS + 音乐）作为参考信号送回 AEC 模块，确保模型不会听到“自己的声音”或背景音乐，这是实现“随时打断（Barge-in）”的基础。
    *   **Opus 编码**：将 PCM 音频实时编码为 Opus 格式（通常 24kHz 或 48kHz），通过 WebRTC DataChannel 或 RTP 发送。

2.  **Multi-modal Sampler (多模态采样器)**
    *   **DMS/OMS**：不上传视频流。仅当检测到特定事件（如“注视屏幕、“手势指向”）或 Realtime API 主动请求时，才截取关键帧（Keyframe），压缩为 Base64 发送。
    *   **Screen Reader**：监听 View 树变化。为了节省 Token，通常将 UI 树简化为 JSON 描述（文本、坐标、可点击性）上传，仅在 UI 解析失败时上传截图。

3.  **Connection Manager (连接管理器)**
    *   **Ephemeral Token 换取**：车端不存储 OpenAI API Key。每次启动会话前，通过车厂后端服务（Backend-for-Frontend）用设备证书换取一个有时效性（如 10 分钟）的 Ephemeral Token。
    *   **心跳与重连**：监测网络质量。当 RTT > 500ms 时，自动介入提示用户网络不佳；断网时切换至本地离线指令引擎。

#### 3.2.2 云端：Realtime API (会话层)

OpenAI Realtime API 是系统的“交互界面”。

*   **Session Config**：在连接建立初期下发 System Prompt（设定人设：“你是一个专业的车载助手...”）和 Available Tools（工具定义）。
*   **VAD Settings**：配置服务端 VAD 的灵敏度。在车内嘈杂环境，通常需要调高 `threshold`，避免风噪误触发打断。
*   **Function Calling 触发**：模型不执行代码，只生成 JSON 意图。它负责判断“用户想开窗”，并暂停 TTS，输出 `tool_calls` 事件。

#### 3.2.3 云端：Agents SDK (编排层)

这是业务逻辑的大脑，通常部署在车厂的私有云或 Serverless 环境中。

*   **Agent 拓扑结构**：
    *   **Router/Supervisor**：接收 Realtime API 的所有 `tool_calls`。根据函数名分发给下游 Agent。
    *   **RAG Agent**：负责查阅手册。包含 Query Rewrite（重写查询）、Embedding、Vector Search、Rerank（重排序）四个步骤。
    *   **Vehicle Control Agent**：负责状态校验。例如用户说“打开雨刮”，Agent 先查询车辆状态数据库（Vehicle Shadow），确认当前不是“洗车模式”，再生成指令。
    *   **GUI Agent**：负责复杂 APP 操作。维护一个有状态机（FSM），处理点击、滚动、输入文本、确认弹窗。

*   **Handoff（交接）**：Agents SDK 允许不同 Agent 之间移交控制权。例如，GUI Agent 在订餐时发现用户问“这餐厅有停车位吗？”，可以将上下文暂时移交给 RAG/Map Agent 回答，再切回来。

#### 3.2.4 车端：安全网关 (Vehicle Safety Gateway)

这是最后一道防线。

*   **Signal Firewall (信号防火墙)**：
    *   **黑名单**：行驶中禁止操作引擎盖、后备箱、座椅大幅度调节。
    *   **速率限制**：防止云端 Bug 导致 1 秒内发送 100 次“开关车窗”指令烧毁电机。
*   **User Confirmation (用户确认)**：
    *   对于高风险操作（如关闭大灯），网关会拦截指令，强制要求 Realtime API 发起二次语音确认（“检测到正在夜间行驶，确定要关闭大灯吗？”），收到明确肯定后才放行。

### 3.3 数据对象与接口契约

清晰的数据定义是解耦的关键。

#### 3.3.1 会话上下文 (Session Context)
除了对话历史，Context 还包含**车辆环境快照**，每次对话轮次更新：
```json
{
  "session_id": "sess_abc123",
  "vehicle_context": {
    "speed": 85.0,
    "gear": "D",
    "passengers": ["driver", "rear_right"],
    "location": {"lat": 31.2, "lng": 121.4},
    "screen_activity": "com.tesla.nav"
  }
}
```
*Rule of Thumb*：不要把所有车辆信号都塞进 Context，只放跟交互决策强相关的（如速度、乘客、当前APP）。

#### 3.3.2 工具调用契约 (Tool Contract)
Realtime API 发出的结构 vs. Agents SDK 返回的结构：

**Request (From Realtime API):**
```json
{
  "type": "function",
  "name": "adjust_climate",
  "arguments": "{ \"temperature\": 22, \"zone\": \"driver\" }"
}
```

**Response (From Agents SDK):**
```json
{
  "tool_call_id": "call_xyz",
  "status": "success",
  "output": "驾驶席温度已设定为22度。当前外部温度较低，建议开启座椅加热。",
  "visual_feedback": { "widget_id": "climate_toast", "duration": 3000 }
}
```
*注意*：返回不仅包含文本结果，还可能包含 `visual_feedback` 用于在车机屏幕上弹窗。

### 3.4 关键链路时序图 (Sequence Diagrams)

#### 场景：打断对话并执行车控 (Barge-in & Control)

```ascii
User      Mic/VAD    RealtimeAPI    AgentsSDK    VehicleGW
 |           |            |             |            |
 |(TTS playing "The weather is...")     |            |
 |           |            |             |            |
 |"Too loud!"|            |             |            |
 |---------->| [Detect]   |             |            |
 |           |--Truncate->|             |            |
 |           |--AudioOpus>|             |            |
 |           |            | [CancelTTS] |            |
 |           |            | [Infer]     |            |
 |           |            |<ToolCall>   |            |
 |           |            |--set_vol--->|            |
 |           |            |             | [Check]    |
 |           |            |             |--Cmd------>|
 |           |            |             |            |[Set Vol]
 |           |            |             |<--Ack------|
 |           |            |<--Result----|            |
 |           |            | [Gen Resp]  |            |
 |<--Audio ("OK")---------|             |            |
```

### 3.5 部署架构 (Deployment View)

为了满足高可用性：

1.  **车端 (Edge)**：
    *   Android Service / Linux Daemon。
    *   通过 mTLS (双向认证) 连接接入层。
2.  **接入层 (Gateway Cluster)**：
    *   负责 SSL 卸载、负载均衡。
    *   部署在离用户最近的 POP 点（边缘节点），通过专线回源到 OpenAI 和 Agents 服务。
3.  **计算层 (Compute)**：
    *   **Agents Runtime**：无状态容器（K8s），可水平扩展。
    *   **Redis**：存储会话的中间状态（State Store）。
    *   **Vector DB**：存储 RAG 知识库。

---

## 3. 本章小结

本章构建了一个深度融合的**端云混合架构**。

*   **Realtime API** 被定位为“感知与交互中枢”，负责让交互变得自然、实时、多模态。
*   **Agents SDK** 被定位为“逻辑与执行中枢”，负责处理复杂的业务规则、工具调用和状态管理。
*   **车端网关** 是不可逾越的“安全底线”，确保任何来自 AI 的指令都在物理安全范围内。

这种架构将不确定的 AI 生成能力（Probabilistic）与确定的车辆控制要求（Deterministic）通过明确的接口契约结合在了一起。

---

## 4. 练习题

### 基础题

<details>
<summary>1. 为什么在 Realtime API 架构中，推荐使用 WebRTC 而不是纯 WebSocket 传输音频？(点击展开)</summary>

**Hint**: 考虑协议底层基于 UDP 还是 TCP，以及对丢包的敏感度。

**参考答案**：
*   **延迟与抗抖动**：WebRTC 基于 UDP（RTP），在网络发生丢包时，它倾向于丢弃旧帧以保持实时性（低延迟），而不是像 WebSocket (TCP) 那样进行重传导致队头阻塞（Head-of-Line Blocking），这对于实时语音对话至关重要。
*   **回声消除与流控**：WebRTC 协议栈内置了成熟的拥塞控制、抖动缓冲（Jitter Buffer）和回声消除协商机制，适合双向流媒体传输。
</details>

<details>
<summary>2. 什么是“Ephemeral Token”（临时凭证），为什么车端不能直接硬编码 OpenAI API Key？(点击展开)</summary>

**Hint**: 车辆可能会被破解、拆解，Key 泄露的后果。

**参考答案**：
*   **安全性**：如果车端硬编码主 Key，一旦车辆被拆解或系统被 Root，Key 就会泄露，导致配额被盗用或预算耗尽。
*   **Ephemeral Token 机制**：车机通过双向认证连接到车厂后端，请求一个仅在短期（如 15 分钟）有效的 Token。Realtime API 使用该 Token 鉴权。即使泄露，攻击窗口也极短。
</details>

<details>
<summary>3. 在架构图中，“Vehicle Control Gateway”处于哪个信任域？它应该信任 Agents SDK 发来的指令吗？(点展开)</summary>

**Hint**: Zero Trust（零信任）原则。

**参考答案**：
*   **信任域**：它处于高信任域（Vehicle Trust Zone），通常运行在安全性要求较高的 MCU 或网关 SOC 上。
*   **不应完全信任**：它应将 Agents SDK 视为“不可信源”。即使 Agents SDK 逻辑正确，云端也可能被入侵。因此 Gateway 必须独立校验每条指令的合法性（如参数范围、当前车速限制、权限冲突）。
</details>

### 挑战题

<details>
<summary>4. 架构设计：如果用户在信号极差的地库，语音交互完全不可用，系统应如何实现“无缝降级”？请描述车端与云端的配合流程。(点击展开)</summary>

**Hint**: VAD 本地检测、离线命令词识别、状态同步。

**参考答案**：
1.  **连接状态感知**：Connection Manager 检测到连接断开或丢包率 > 20%。
2.  **路由切换**：音频流不再发往 Realtime API，而是路由至车机本地的轻量级 ASR/NLU 引擎（通常仅支持几百个固定指令）。
3.  **用户感知**：UI 上显示“离线模式”，TTS 切换为本地合成引擎（音色可能略有不同）。
4.  **功能限制**：仅响应车控指令（“打开车窗”），对于 RAG 问答或闲聊，反馈“网络不可用”。
5.  **恢复同步**：当网络恢复，本地引擎将离线期间发生的车控状态变更（如“车窗已开”）作为一个 `state_update` 事件发送给云端 Session，更新上下文。
</details>

<details>
<summary>5. 思考题：Realtime API 的“Function Calling”并不直接执行代码。在车控场景中，这种“意图”到“执行”的分离有什么架构上的优势？(点击展开)</summary>

**Hint**: 解耦、硬件差异化、安全拦截。

**参考答案**：
1.  **硬件抽象（HAL）**：云端模型只需要输出通用的 `set_temperature(24)`，不需要关心这辆车是特斯拉还是比亚迪，也不需要关心底层 CAN ID 是多少。具体的协议转换由车端或 Agents SDK 适配层完成。
2.  **安全拦截**：因为模型不直接执行，我们可以在“意图”生成后、“执行”发生前插入一层安全校验逻辑（Human-in-the-loop 或规则引擎）。
3.  **可测试性**：我们可以单独测试模型的意图识别准确率，而不需要连接真实的车辆硬件。
</details>

<details>
<summary>6. 多模态挑战：当用户指着窗外说“那是什么楼”时，系统架构如何保证摄像头刚好拍到了用户指的地方，且模型能理解？(点击展开)</summary>

**Hint**: 时间同步、多摄融合、视线追踪（Gaze Tracking）。

**参考答案**：
1.  **触发机制**：DMS 摄像头检测到用户有“手指指向”动作或 VAD 检测到代词“那”、“这个”。
2.  **空间对齐**：通过 DMS 计算用户视线向量（Gaze Vector）和手指向量。
3.  **摄像头选择**：根据向量方向，Perception Adapter 自动选择最匹配的外部摄像头（前视、侧前或侧后）。
4.  **时间同步**：由于视频处理有延迟，需要有一个 Ring Buffer 缓存最近几秒的视频帧。系统回溯到用户说“那”的时间戳对应的视频帧。
5.  **发送与推理**：裁剪出视线区域（ROI），发送给 Realtime API 进行视觉问答。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 "幻听" 与回声泄漏 (Echo Leakage)
*   **现象**：AI 刚说完半句话，突然自己打断自己，或者不断重复“对不起”、“请再说一遍”。
*   **原因**：AEC（回声消除）失效。车机播放 AI 的声音被麦克风录入，又传回给了 Realtime API。模型把自己的声音当成了用户的插话。
*   **调试技巧**：
    *   检查 `Server VAD` 事件日志。如果在 AI 说话期间 VAD 频繁触发，说明 AEC 有问题。
    *   在车端强制启用硬件 AEC（DSP）。
    *   在 Realtime API 侧使用 `conversation.item.truncate` 事件来精确处理打断逻辑，而不是仅仅依赖音频流。

### 5.2 状态竞争 (Race Conditions)
*   **现象**：用户说“打开车窗，哦不对，关上”。车窗先打开了一半，然后卡住，最后也没关上。
*   **原因**：两条指令几乎同时发出。`open_window` 工具调用正在执行，`close_window` 紧随其后。车端硬件可能无法处理这种快速反转。
*   **设计建议**：
    *   **指令队列**：在车控网关实现一个指令队列（FIFO）。
    *   **互斥锁**：同一执行器（如左前窗电机）在同一时间只能被一个任务占用。
    *   **原子化**：Agent 逻辑应具备“最新指令优先”原则，收到新指令时尝试取消旧指令（Cancel Token）。

### 5.3 上下文爆炸 (Context Overflow)
*   **现象**：对话进行十分钟后，响应越来越慢，最终报错。
*   **原因**：车舱内产生了大量的 Perception Event（如 DMS 每秒上报疲劳度）或工具调用日志，填满了 Context Window。
*   **设计建议**：
    *   **定期摘要**：Agents SDK 应后台定期对历史对话进行摘要（Summarization）。
    *   **事件过滤**：不要把每秒的“车速=80,81,80...”都发给模型。仅发送显著变化或应答需要的快照。
    *   **Ephemeral Events**：将感知事件标记为“瞬时”，在下一轮对话中自动丢弃，不计入长期历史。
