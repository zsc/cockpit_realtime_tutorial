# Chapter 4｜Realtime 会话层设计 (Realtime Session Design)

## 4.1 开篇与目标
本章聚焦于**语音实时对话**的核心管道设计。与传统的“ASR（识别）→ NLP（理解）→ TTS（合成）”级联式架构不同，OpenAI Realtime API 提供了一个端到端的、流式的多模态交互环境。在车载场景下，这意味着更低的延迟、更自然的打断（Barge-in）以及听觉与视觉的并发处理。

**本章学习目标**：
1. 理解并选择适合车载环境的 Realtime API **接入拓扑**（WebRTC vs WebSocket）。
2. 掌握**会话生命周期管理**，包括鉴权、配置下发与断连恢复。
3. 设计**音频流处理策略**，重点解决回声消除（AEC）与打断机制。
4. 设计**多模态注入机制**，协调视觉帧与音频流的同步。
5. 建立**工具调用（Function Calling）**与实时语音流的协同模式。

---

## 4.2 接入架构与连接拓扑

在车载环境中，直接从车机端（Client）连接 OpenAI 存在安全隐患（API Key 泄露）和逻辑编排困难。因此，推荐采用 **BFF (Backend for Frontend) Relay 模式**。

### 4.2.1 架构拓扑选型

我们主要通过 **WebSocket** 或 **WebRTC** 进行连接。

*   **WebSocket (Raw Audio)**: 适合服务端中转，控制力强，便于做日志留存、敏感词过滤和协议转换。
*   **WebRTC**: 延迟极低，适合即时通讯，但服务端中转实现复杂（需搭建 SFU/MCU 或 TURN），且车载防火墙配置较严。

**Rule-of-Thumb**: **在车端与云端网关之间使用成熟的私有协议或 WebSocket，在云端网关（Relay Server）与 OpenAI 之间使用官方推荐的 WebSocket 链接。** 除非对延迟有极致要求（<300ms 端到端）且网络环境极其优越，否则避免车端直连。

#### 架构 ASCII 示意图

```ascii
+--------+       (1) Audio/Events      +---------+       (2) Signed WS     +----------+
|  Vehicle | <========================> |  Relay  | <====================> |  OpenAI  |
|  (Client)|       (Custom/WS)         | (Server)|                        | Realtime |
+--------+                             +---------+                        +----------+
    |                                       |                                   |
    | [Mic Input] -> DSP(AEC/NS) -> Enc     | [Auth & Session Mgmt]             | [Model]
    | [Spk Output] <- Dec <- Buffer         | [Tool Execution (Car Ctrl)]       | [Audio In/Out]
    | [Screen/Cam] -> Base64/Binary         | [RAG / Knowledge Fetch]           | [Function Call]
    +                                       +                                   +
```

### 4.2.2 安全鉴权策略
1.  **车端认证**：车机通过 mTLS 或 OAuth2 令牌连接 Relay Server。
2.  **OpenAI 认证**：Relay Server 持有 `OPENAI_API_KEY`，该 Key 永不下发到车端。
3.  **会话隔离**：每个 WebSocket 连接对应唯一的 `Session ID`，连接断开即销毁上下文（除非使用了持久化记忆机制，见 Chapter 5）。

---

## 4.3 会生命周期管理 (Lifecycle)

Realtime API 是有状态的。管理好“连接—配置—交互—销毁”的闭环至关重要。

### 4.3.1 初始化与配置 (Session Update)
连接建立后的第一件事是发送 `session.update` 事件。
*   **Voice**: 设置音色（推荐 `alloy` 或 `shimmer` 等清晰音色）。
*   **Instructions**: 注入 System Prompt（角色设定、安全边界）。
*   **Turn Detection**: 配置 VAD（语音活动检测）。建议开启 `server_vad` 模式，让模型决定用户何时说完，但也需允许车端强制发送“截断信号”。
*   **Tools**: 下发当前车辆可用的工具定义（根据车型配置动态生成）。

### 4.3.2 事件流驱动 (Event Loop)
Realtime API 完全由事件驱动。Relay Server 需维护一个事件泵。

*   **上行（To OpenAI）**：
    *   `input_audio_buffer.append`: 持续发送音频块（Base64 PCM）。
    *   `input_audio_buffer.commit`: 强制提交（当车端 VAD 判定说话结束时）。
    *   `conversation.item.create`: 注入非语音消息（如：系统提示“前方路况拥堵”或 RAG 检索到的文本）。

*   **下行（From OpenAI）**：
    *   `response.audio.delta`: 音频流片段（需即时转发给车端播放）。
    *   `response.audio_transcript.done`: 文本实录（用于在屏幕显示字幕）。
    *   `response.function_call_arguments.done`: 工具调用请求。

### 4.3.3 断连与恢复
**场景**：车辆驶入隧道，网络中断 5 秒。
**策略**：
1.  **车端**：暂停录音上传，提示音（可选），尝试重连。
2.  **Relay 端**：检测到车端断开，保持与 OpenAI 连接一段超时时间（如 60s）。
3.  **恢复**：车端重连后，Relay 下发最后一条未播放完的音频，或发送 `response.cancel` 重置状态，避免播放过时的回复。

---

## 4.4 音频链路与打断机制 (Barge-in)

车载环境最大的痛点是**噪声**和**回声**。Realtime API 的“全双工”特性要求极高的音频处理量。

### 4.4.1 AEC (回声消除)
如果车机播放 AI 的声音被麦克风录入并传回 OpenAI，模型会听到自己说话，导致死循环或“鬼畜”。
*   **硬件 AEC**：依赖车机 DSP 芯片（首选）。
*   **软件 AEC**：如果硬件不支持，需在 Android/Linux 音频 HAL 层集成 WebRTC 的 AEC 模块。
*   **Gotcha**：**绝对不要**把 AI 的回复音频作为 `input_audio_buffer` 发回给 OpenAI。

### 4.4.2 打断策略 (Interruption Handling)
用户在 AI 说话时插嘴，AI 必须立即停止。

**流程设计**：
1.  **检测**：车端 VAD 检测到用户人声（User Speech Start）。
2.  **截断**：
    *   车端立即停止 TTS 播放队列，清空缓冲区。
    *   车端发送 `input_audio_buffer.clear` 事件给 Relay。
    *   Relay 向 OpenAI 发送 `response.cancel` 事件。
3.  **对话**：车端开始流式发送新的用户语音。

**ASCII 流程图：打断**
```ascii
Time | User (Car)              | Relay Server          | OpenAI
-----+-------------------------+-----------------------+----------------
t0   | [Listening]             |                       | [Generates Audio A...]
t1   | <Plays Audio A part 1>  | <Fwd Audio A part 2>  |
t2   | "Hey wait!" (Talks)     |                       |
t3   | [Stop Playback!]        | -> input_audio.append |
t4   | -> VAD Triggered        | -> response.cancel    | [Stop Generation]
t5   | -> input_audio.append   | -> input_audio.commit | [Process New Audio]
```

---

## 4.5 多模态输入注入 (Multimodal Injection)

Realtime API 支持图像输入。车载场景下，摄像头（DMS/OMS）和屏幕内容是关键上下文。

### 4.5.1 图像采样策略
不要发送视频流！带宽和 Token 消耗都扛不住。
*   **按需触发**：用户问“我看那个灯是什么意思？”时，截取当前帧。
*   **低频采样**：每 10 秒采样一次低分辨率关键帧（如有变化），更新上下文。

### 4.5.2 注入协议
使用 `conversation.item.create` 消息，类型为 `message`，内容包含 `type: "image_url"` (Base64)。

**图像预处理 Rule-of-Thumb**：
*   **分辨率**：缩放到 512x512 或更低，除非需要看清仪表盘微小文字。
*   **灰度化**：DMS（红外）本身是黑白的，无需转换。
*   **裁剪**：只发送 ROI（感兴趣区域），例如只裁剪仪表盘区域发送，保护隐私并减少数据量。

---

## 4.6 工具调用与语音流的协调

当 Realtime 模型决定调用工具（如 `set_seat_heater`）时，音频流会发生什么？

### 4.6.1 模式 A：边做边说 (Asynchronous)
*   模型输出音频：“好的，正在为您打开座椅加热...”
*   同时下发 `function_call`。
*   Relay 执行车控指令。
*   执行完毕后，Relay 将 `function_call_output` 发回模型。
*   模型根据结果补充：“已开启。”

### 4.6.2 模式 B：先做后说 (Blocking)
*   模型停止生成音频。
*   等待 Relay 执行结果。
*   Relay 返回结果。
*   模型生成：“座椅加热已启。”
*   *缺点*：会有静默期（Dead Air）。建议在客户端播放“处理音效”或“正在思考”的动效。

### 4.6.3 工具回执注入
工具执行的结果（JSON）需要转化为自然语言反馈。
*   **Strategy**：将工具返回结果作为 `function_call_output` item 提交，然后触发 `response.create`，模型会自动将 JSON 结果总结为语音回复。

---

## 4.7 常见陷阱与错误 (Gotchas)

| 陷阱类型 | 描述 | 解决方案 |
| :--- | :--- | :--- |
| **自激啸叫 (Audio Loop)** | 模型听到了自己刚才说的话，误以为是用户在重复，导致无限复读。 | 严格检查 AEC 效果；在客户端实现“AI 说话时麦克风静音”作为保底（但这会牺牲打断体验，仅作降级）。 |
| **VAD 过于敏感** | 风噪、胎噪被误认为是人声，导致模型频繁被打断或胡言乱语。 | 调高 VAD 阈值；在车端做本地预处理（RNNoise）；在 System Prompt 强调“忽略背景噪音”。 |
| **工死锁** | 模型调用工具后，Relay 迟迟不返回 Output，导致会话卡死。 | 设置工具执行超时（如 3s）；超时后自动返回 "Status: Timeout" 给模型，让模型以此回复用户。 |
| **Token 爆炸** | 长期会话积累了大量历史音频和文本，导致延迟增加、费用飙升。 | 实现 Context Window 滚动机制；每隔 N 轮对话，将旧的历史项从 `session` 中剔除或总结。 |

---

## 4.8 本章小结

本章构建了车舱语音助手的“实时传输层”。核心要点包括：
1.  **架构**：采用 Client -> Relay -> OpenAI 的三层架构保障安全与逻辑编排。
2.  **音频**：AEC 是基础，Clear Buffer + Cancel Response 是实现打断的关键组合。
3.  **多模态**：图像应按需、采样、裁剪后注入，避免流式上传。
4.  **状态**：Relay Server 必须维护会话状态机，处理工具调用的异步回调。

---

## 4.9 练习题

<details>
<summary><strong>基础题 1：连接协议选择</strong></summary>

**题目**：为什么在车端不建议直接持有 OpenAI API Key 并建立 WebRTC 连接？列出至少两点原因。

**提示**：考虑安全性和业务逻辑位置。

**参考答案**：
1.  **安全性**：APK/二进制文件容易被反编译，导致主 API Key 泄露，带来巨额资损。
2.  **业务编排**：车控逻辑、RAG 检索通常位于私有云后端。如果车端直连 OpenAI，工具调用（Function Call）需要车端处理，这会导致车端逻辑过于厚重，且难以进行敏感操作审计（Audit）。
</details>

<details>
<summary><strong>基础题 2：打断机制实现</strong></summary>

**题目**：当用户打断 AI 说话时，为了最低延迟停止 AI 的声音，车端应该先做什么？是先等待服务器确认，还是先本地停止？

**提示**：UX 优先级。

**参考答案**：
**先本地停止**。
车端检测到打断意图后，应立即：
1. 停止音频播放器的缓冲区读取。
2. 清空本地未播放的音频缓。
3. *同时*向服务器发送 `input_audio_buffer.clear` 和打断信号。
不能等待服务器回包才停，否则会有几百毫秒的延迟，用户体验极差。
</details>

<details>
<summary><strong>挑战题 1：多模态上下文同步</strong></summary>

**题目**：用户指着窗外问：“左边那个是什么楼？”。此时系统需要摄像头的图像。请设计一个时序流程，确保图像和语音在时间上是对齐的。如果图像上传慢了 2 秒，会有什么后果？

**提示**：时间戳、事件顺序。

**参考答案**：
**设计流程**：
1. 用户说话开始（VAD Start）。
2. 车端记录此时的时间戳 T0，并触发摄像头抓拍（附带时间戳 T0）。
3. 音频流持续上传。
4. 图像在后台压缩、编码。
5. 在用户说话结束（VAD End）提交 `input_audio_buffer.commit` **之前**，务必通过 `conversation.item.create` 将图像注入到会话中。或者，在 System Prompt 中设定“如果需要看，请请求截工具”，但这通常太慢。
**后果**：
如果图像上传慢了 2 秒，且在音频 Commit 之后才到达，模型处理音频时可能还没看到图，会导致：
1. 回答“我没看到您说的是什么”。
2. 或者基于上一帧过期的图像回答（幻觉）。
**修正策略**：在 Relay 端做一个短时间的“对齐缓冲”，等待图像上传完毕（或超时）后再向 OpenAI 提交 Commit 信号。
</details>

<details>
<summary><strong>挑战题 2：弱网下的工具执行</strong></summary>

**题目**：用户说“打开空调”，指令上传成功，Realtime 模型返回了 `function_call`。此时车辆进入无信号区。Relay Server 执行了车控指令（通过长连接通道下发给车辆 T-Box），但无法将结果返回给 OpenAI Realtime API（因为连接断了）。请问如何设计恢复流程，避免“空调开了，但 AI 没给反馈”或“AI 以为没开，重试导致重复操作”？

**提示**：幂等性、本地反馈。

**参考答案**：
1.  **幂等性**：车控接口必须设计为幂等的（调用 5 次“开空调”等于调用 1 次）。
2.  **本地反馈**：车端 GUI/音频收到 T-Box 的执行成功信号后，可以直接播放本地的“空调已开启”提示音/UI 弹窗，而不必死等 AI 的语音流。
3.  **状态同步**：网络恢复后，Relay Server 重建 Realtime 会话，并将当前的车辆状态（Context）作为 System Prompt 更新进去，或者作为一条隐藏的 System Message 告诉 AI：“刚才在离线期间，空调已成功开启”。
</details>
