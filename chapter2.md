# Chapter 2｜用户体验与对话交互 (UX & Conversation Design)

## 1. 开篇段落

本章是整个座舱语音机器人系统的灵魂所在。技术架构决定了机器人“能做什么”，而交互设计决定了用户“愿不愿意用”。在时速 120km/h 的移动封闭空间内，任何一次错误的交互（如误解指令、延迟过高、喋喋不休）都可能转化为驾驶员的怒气，甚至安全事故。

本章将深度拆解基于 **OpenAI Realtime API** 的流式对话体验。我们将不再满足于传统的“一问一答”，而是追求一种**“共同在场” (Co-presence)** 的体验——即 AI 像副驾一样，既能听到你说什么，也能看到你看到的（路况/屏幕），并能感知你的情绪和状态。我们将重点讨论如何利用 Realtime 的低延迟特性来处理**话转换 (Turn-taking)**、**多模态打断**以及**高风险动作的原子性确认**。

**学习目标**：
1.  **深入理解车载认知模型**：如何在毫秒级时间内评估驾驶员的负荷并动态调整 AI 的行为。
2.  **掌握 Realtime 流式交互范式**：从 VAD 策略、回声消除到智能打断的完整链路设计。
3.  **构建多模态语义场**：利用摄像头（DMS/OMS/7V）和屏幕上下文（UI Tree）解决指代消歧（Grounding）。
4.  **设计容错与降级机制**：在网络抖动或感知失败时的优雅降级策略。

---

## 2. 体验原则：驾驶优先与认知负荷

在设计对话之前，我们必须量化“打扰”。

### 2.1 认知负荷金字塔 (Cognitive Load Pyramid)

车载交互设计的核心矛盾是：**丰富的功能 vs 有限的注意力**。

```ascii
      [ 红色区域 ]
     /  高风险区  \   <-- 倒车、变道、大曲率过弯、暴雨
    /--------------\
   [  黄色区域    ]  <-- 拥堵跟车、红绿灯路、导航指令密集区
  /------------------\
 [    绿色区域      ] <-- 高速巡航(开ACC)、停车休息、充电中
/----------------------\
```

*   **原则 1：动态冗余度 (Dynamic Verbosity)**
    *   **绿色区域**：AI 可以稍微啰嗦，提供更丰富的信息（如介绍餐厅评分、念读新闻）。
    *   **红色区域**：AI 必须**“闭嘴”**或**“极简”**。如果用户在倒车时问“明天天气”，AI 应暂缓回答或只报“晴天”，绝对不能念大段文字。
    *   *实现提示*：系统需订阅车辆 CAN 总线的 `Gear_State` (档位)、`Steering_Angle` (转角) 和 `Speed` (车速) 以及 `Wiper_Status` (雨刮) 作为 Realtime Session 的 Context 变量，动态调整 Prompt 的 `instruction`（例如注入：“User is in high cognitive load, be extremely concise.”）。

*   **原则 2：扫视即得 (Glanceability)**
    *   语音对话必须配合屏幕辅助。当 AI 说“为您找到这几个充电桩”时，屏幕显示列的停留时间应足够长，字体应足够大。
    *   **2秒法则**：驾驶员视线离开路面不能超过 2 秒。所有 UI 反馈必须在 2 秒内可被理解。

---

## 3. 语音交互模型：超越“指令-响应”

利用 OpenAI Realtime API，我们旨在构建 **“类人全双工” (Pseudo-Full-Duplex)** 体验。

### 3.1 听觉链路的“三重门”

为了让 AI 听得准且反应快，需要处理好端侧与云侧的配合：

1.  **第一道门：回声消除 (AEC) 与降噪 (NS)**
    *   *问题*：车内是一个巨大的回声腔（玻璃反射），且伴随风噪、胎噪、音乐声。
    *   *设计*：Realtime API 能够处理一定的环境音，但**强烈建议**在车端 DSP（数字信号处理）层先做硬件级 AEC。
    *   *Reference Signal*：车机发出的所有声音（音乐、TTS、导航音）必须作为参考信号（Reference）喂给 AEC 模块，从麦克风输入中“减去”。否则，AI 会听到自己说的话（自激）。

2.  **二道门：语音活动检测 (VAD)**
    *   *Server VAD vs. Client VAD*：
        *   **Server VAD (OpenAI)**：准确率高，能区分人声和背景音。建议作为默认方案。
        *   **Client VAD (本地)**：用于省流量和极速打断（Local Barge-in）。
    *   *策略*：当本地 VAD 判定有人说话时，**立即**通过 WebSocket 发送 `input_audio_buffer.append`。

3.  **第三道门：唤醒词 (Wake Word)**
    *   *隐私屏障*：Realtime API 计费昂贵且涉及隐私，不可 24 小时常开。
    *   *本地唤醒*：车端运行轻量级唤醒引擎（如 "Hi, ChatGPT"）。
    *   *One-Shot (一句连说)*：支持用户说“Hi ChatGPT 把空调开到26度”，不需要等唤醒词回应。这需要车端缓存唤醒词后的音频流，并在连接建立后迅速 `commit` 给 Realtime API。

### 3.2 智能打断 (Smart Barge-in)

Realtime API 的核心优势是可打断，但“怎么打断”有讲究。

*   **物理打断 (Hard Cut)**：
    *   检测到用户说话 -> 立即停止 TTS -> 清空 buffer。
    *   *适用*：用户说“停！”、“不对！”、“换一首”。
*   **语义打断 (Semantic Barge-in / Backchanneling)**：
    *   用户发出“嗯”、“对”、“然后呢”等附和词（Backchannel）。
    *   *设计*：此时**不应**打断 AI 的陈述。这需要模型具备极快的情感/意图判断能力。
    *   *实现难点*：目前的 Realtime API 默认 VAD 可能会把“嗯”视为打断。
    *   *Workaround*：可以通过 Prompt 告知模型：“If the user makes brief agreement sounds like 'hmm' or 'yes', ignore the interruption and continue speaking.”（但这也依赖模型对 `input_audio_buffer.commit` 后推理的反应速度，通常很难完美，目前工程上多采用**物理打断**作为基准，因为它更可控）。

### 3.3 延迟预算 (Latency Budget)

为了达到“实时”感，端到端延迟（Ear-to-Ear）应控制在 **700ms - 1000ms** 以内。

| 环节 | 预算 (ms) | 优化策略 |
| :--- | :--- | :--- |
| **车端采集+VAD+编码** | 50-100 | 使用 Opus 编码，20ms 帧长 |
| **网络传输 (上行)** | 50-200 | 5G/LTE 优先，QoS 保障 |
| **OpenAI 推理 (TTFT)** | 300-600 | Time To First Token，模型性能关键 |
| **网络传输 (下行)** | 50-200 | 流式接收，首包即播 |
| **车端解码+JitterBuf** | 50-100 | 动态抖动缓冲，避免卡顿 |

> **Rule-of-Thumb**: 如果预测延迟 > 1.5秒（如弱网），立即播放本地的“思考音效”（Filler Sound），如“Hmm... Let me check...”，掩盖静默期。

---

## 4. 多模态对话策略：视觉的语义化

车舱对话的一大痛点是“指代不明”。多模态输入（Video/Images）能解决此问题，但带宽和 Token 消耗巨大，不能无脑传视频流。

### 4.1 触发式视觉注入 (Triggered Vision Injection)

不要一直上传视频帧。只在以下时刻截取关键帧（Keyframe）并上传：
1.  **显式视觉询问**：用户说“看这里”“这是什么”、“前面的车”。
2.  **DMS 视线触发**：用户长时间（>1s）注视某处并开口说话。
3.  **主动关怀触发**：DMS 检测到打哈欠/闭眼 -> 截帧 -> 上传 -> AI 说“你看起来很累”。

### 4.2 屏幕上下文 (Screen Context)

当用户说“点这个”时，Realtime API 需要知道屏幕上有什么。

*   **方案 A：截图 (Screenshot)**
    *   *优点*：通用，不论什么 App 都能看。
    *   *缺点*：Token 消耗大，OCR 精度受光照/分辨率影响，无法获取控件 ID。
*   **方案 B：UI 树 (Accessibility Tree/DOM)**
    *   *优点*：Token 少，精准获取控件文本、ID、可点击状态。
    *   *缺点*：需要 App 适配（类似 Android Accessibility 服务）。
*   **混合策略**：
    *   优先上传精简后的 UI 树（JSON 格式）。
    *   如果 JSON 描述不清（如图片按钮），再辅以局部截图（Crop）。

**Prompt 示例**：
> "User is looking at the center screen. Context: The screen shows a list of 3 restaurants. Item 1: KFC (Coordinates: 100, 200). Item 2: McDonald's. User says: 'Go to the first one'. Your Action: Call tool `navigate_to(target='KFC')`."

---

## 5. 安全确认与行动策略 (Action Policy)

车控涉及物理实体，必须防范 LLM 的“幻觉操作”。

### 5.1 意图与参数槽位 (Intent & Slots)

使用 Agents SDK 将自然语言转化为确定性调用。

*   **Case**: "我觉得有点冷"
*   **Reasoning**: 用户没说具体动作，但意图是提高温度。
*   **Tool Call**: `climate_control(action="increase_temp", value=2, unit="celsius")` (默认步进值需在 Prompt 中设定)。

### 5.2 风险确认流程 (The "Two-Key" Turn)

对于高风险操作（Level 2/3），采用**“两步确认法”**。

1.  **Step 1: 意图冻结 (Freeze Intent)**
    *   AI 解析出意图，但不执行。
    *   AI 生成确认话术：“您是想打开天窗吗？这会产生风噪。”
2.  **Step 2: 显式授权 (Explicit Grant)**
    *   户回答：“是的”、“打开吧”。
    *   **关键点**：这必须是一个新的 Turn。Agents SDK 在收到确认前，其状态机应停留在 `WAITING_CONFIRMATION` 状态。

### 5.3 影子模式 (Shadow Mode)
对于极其危险的操作（如“关闭大灯”在夜间），系统可以进入影子模式：
*   **AI**：“正在为您关闭大灯...”
*   **底层网关**：拦截指令，**不执行**，但检查车辆光感传感器。
*   **反馈**：如果环境光极暗，网关拒绝执行并返回 Error。
*   **AI (修正)**：“检测到环境光过暗，为了安全，大灯无法关闭。”

---

## 6. 多轮任务流设计模板

### 6.1 渐进式任务流 (Progressive Disclosure)

不要一次性问完所有问题。

*   **错误**：“请问您要去哪里？要走高速吗？避开拥堵吗？还需要顺路加油吗？”
*   **正确**：
    1.  **AI**: “去哪里？”
    2.  **User**: “上海中心。”
    3.  **AI**: “好的，找到两个路线。第一个走高速快10分钟，第二个不收费。选哪个？”
    4.  **User**: “快的那个。”

### 6.2 纠错与修改 (Correction)

支持在任务流中随时修改之前的参数。

*   **User**: “帮我订个明天晚上6点的闹钟。”
*   **AI**: “好的，明晚6点。”
*   **User**: “改为后天早上8点。”
*   **Logic**: Agent 必须能识别“改为”这个关键词，并更新 Context 中的 `time` 变量，而不是创建一个新闹钟。

---

## 7. 语气与人格 (Tone & Voice)

车载 AI 的人格设定应为：**靠谱的副驾 (Reliable Co-pilot)**。

### 7.1 语言风格规范
*   **杜绝废话**：不要说“作为一个人工智能模型...”、“好的，我明白了，正在为您处理...”。
*   **积极响应**：使用“好的”、“收到”、“没问题”作为简短的确认（Acknowledge）。
*   **不确定性表达**：当置信度低时，不要猜。
    *   *Bad*: “前面的车是宝马。” (其实看不太清)
    *   *Good*: “看起来像是一辆深色轿车，具体型号我看不清。”

### 7.2 声音合成 (TTS)
利用 Realtime API 的情感化 TTS：
*   **紧急情况**：语速加快，音调提高，去除多余停顿。“注意！前方急刹车！”
*   **舒缓模式**：夜间或播放音乐时，语速放缓，音量适度降低（Whisper mode 概念）。

---

## 8. 本章小结

本章构建了车载语音机器人的交互框架。
1.  **原则**：安全第一，所见即所得，动态适应驾驶员负荷。
2.  **听**：硬件 AEC + 混合 VAD + 智能打断 = 流畅对话。
3.  **看**：视觉作为对话的“锚点”，解决指代不清，但不滥用带宽。
4.  **做**：风险分级，高危动作必须显式确认，利用 Agents SDK 管理状态。

---

## 9. 练习题 (Exercises)

<details>
<summary><strong>Q1 (基础): 在 Realtime API 的连接中，如果车子进入隧道网络断开，并在 5 秒后重连。此时会话状态（Context）会发生什么？应该如何设计重连机制？</strong></summary>

*   **提示**：Realtime API 是基于 WebSocket 的长连接，也是有状态的。
*   **参考答案**：
    *   **现象**：WebSocket 断开，OpenAI 端的 Session 会在短暂超时后销毁（或保持极短时间，视 API 具体配置而定，通常认为是易失的）。之前的 `session.update` 设置的 instructions 和 context 可能会丢失。
    *   **设计策略**：
        1.  **客户端缓存**：车端 Client 必须在本地维护一份“当前会话状态镜像”（Conversation History, System Instructions, Current Tool Context）。
        2.  **重连恢复**：网络恢复后，重新建立 WebSocket 连接。
        3.  **状态注入**：在发送第一帧音频前，先发送 `session.update` 事件，将缓存的 Prompt、Context 和最近几轮对话历史（作为 user/assistant messages）重新灌入，恢复“记忆”。
</details>

<details>
<summary><strong>Q2 (进阶): 设计一个“处理后排儿童吵闹”的 Agent 交互流程。结合 OMS（摄像头）和声源定位。</strong></summary>

*   **提示**：主动关怀 vs 打扰驾驶。区分是谁在说话。
*   **参考答案**：
    1.  **感知**：麦克风阵列检测到后排高分贝非语言音频（哭闹）。OMS 识别出后排有儿童。
    2.  **决策**：判断持续时间 > 10秒（避免偶发尖叫触发）。
    3.  **交互（驾驶员侧）**：
        *   AI（温和语气）：“检测到宝宝好像有点不舒服，需要我播放一些儿歌或者开启‘儿童安抚模式’吗？”
    4.  **执行**：
        *   用户：“好的。”
        *   Action：后排音区播放儿歌，前排音区保持导航/静音（音区分离），同时调柔后排氛围灯。
</details>

<details>
<summary><strong>Q3 (挑战): 用户说“这首歌太难听了，切歌”。此时如果是（1）蓝牙音乐（2）车机自带音乐 App（3）FM 收音机。Agent 应如何处理工具调用的差异？</strong></summary>

*   **提示**：不同媒体源的控制协议不同。
*   **参考答案**：
    *   **意图识别**：统一识别为 `media_control(action="next")`。
    *   **网关适配 (Vehicle Control Gateway)**：
        *   **Case 1 (车机 App)**：调用 App 提供的 API `app.music.next()`。
        *   **Case 2 (蓝牙)**：通过 AVRCP 协议向手机发送 `Next` 指令。
        *   **Case 3 (FM)**：FM 没有“下一首”的概念，映射为“搜下一个电台”或 `tune.scan_up()`。
    *   **交互反馈**：
        *   如果 FM 搜索到了噪音，AI 需能解释：“已为您切换到下一个频率 101.7。”
</details>

<details>
<summary><strong>Q4 (场景设计): 设计“GUI Agent 辅助停车缴费”的异常流程。假设二维码加载不出来。</strong></summary>

*   **提示**：不要让用户在这个时候手动输入复杂的车牌号。
*   **参考答案**：
    1.  **User**: “付一下停车费。”
    2.  **Agent**: 打开缴费 App。
    3.  **Error**: 屏幕读检测到页面显示“网络错误”或二维码区域为空白。
    4.  **Fallback**:
        *   AI：“缴费页面加载失败了。建议您直接驶离，使用出口的人工通道或 ETC。”
        *   **或者**（如果有能力）：AI 尝试点击页面上的“刷新”按钮（Retry）。
    5.  **禁止行为**：不要让用户语音报信用卡号，也不要让用户盯着屏幕手动输。
</details>

<details>
<summary><strong>Q5 (思考): 为什么说车载语音交互中，“沉默”也是一种有效的回答？请举例。</strong></summary>

*   **提示**：Backchanneling 和隐式确认。
*   **参考答案**：
    *   **场景 1：低风险指令的极速反馈**。用户：“打开阅读灯。” 灯亮了。此时 AI 不需要说“好的，阅读灯已打开”。**动作本身的物理反馈（灯亮）已经足够**。AI 保持沉默（或仅播放一个极短的 'Ding' 音效）能极大提升流畅感。
    *   **场景 2：高负荷路况**。用户在处理紧急变道，随口抱怨了一句“这车真多”。AI 保持沉默比搭话“是啊，交通状况确实不好”更安全，不占用驾驶员的听觉通道。
</details>

<details>
<summary><strong>Q6 (Prompt Engineering): 写一段 System Instruction，用于指导 Realtime Model 处理“带有方言口音且语速极快”的老年用户。</strong></summary>

*   **提示**：宽容度、确认策略、语速匹配。
*   **参考答案**：
    ```markdown
    Role: You are a patient, clear-speaking vehicle assistant.
    User Profile: The user may speak with a strong accent and fast pace.
    Strategy:
    1. Listen carefully. If the intent is ambiguous due to accent, ask for clarification politely ("不好意思，我没听清，您是说...吗？").
    2. Do NOT mimic the user's fast speed. Speak slightly slower and clearer than usual to establish a calm rhythm.
    3. For critical vehicle controls, paraphrase the request back to the user before executing (e.g., "您是要打开后备，对吗？").
    ```
</details>

---

## 10. 常见陷阱与错误 (Gotchas)

### 10.1 陷阱：音量“Ducking”处理不当
*   **现象**：AI 说话时，音乐声音没有降低，导致用户听不清 AI 说话；或者 AI 刚说完，音乐突然“炸”回原音量，吓人一跳。
*   **调试**：
    *   **Fade-in/Fade-out**：在 Ducking（压低背景音）恢复时，必须有 500ms-1s 的渐变过程。
    *   **优先级抢占**：导航播报 > 电话 > AI 对话 > 媒体音乐。当导航插嘴时，AI 应当暂停（Pause）而不是继续说但被压低声音。

### 10.2 陷阱：过度拟人化的恐怖谷
*   **现象**：Prompt 设置为“你是一个调皮的小妹妹”，结果 AI 在用户急刹车时开玩笑说“哎呀好刺激”。
*   **对策**：**Context-Aware Persona**。当 CAN 总线检测到急刹车（Deceleration > 0.6g）或气囊弹出时，强制切换 System Prompt 为“紧急救援模式”，语气严肃、精准、只确认安全状态。

### 10.3 陷阱：Realtime API 的“抢话”
*   **现象**：用户还在思考“呃...那个...”，AI 以为用户说完了，开始抢答。
*   **调试**：
    *   调整 Realtime API 的 `turn_detection` 参数。
    *   `silence_duration_ms`: 默认可能太短。在车内建议设为 **500ms - 800ms**。
    *   或者实现“动态阈值”：如果用户的前几个词是 Filler words ("呃", "让我想想")，则在客户端逻辑中抑制 `commit`，给予更多等待时间。

### 10.4 陷阱：忽略了离线能力
*   **现象**：车子进地库断网，用户喊“打开车窗”，AI 毫无反应。
*   **对策**：**Hybrid Intent Matching**。
    1.  ASR 转录文本同时发给（1）云端 Realtime，（2）本地正则匹配引擎。
    2.  如果云端超时（>1s）或网络不可用，且本地引擎命中了高频车控指令（开窗/关灯），则**立即由本地执行**并反馈。不要死等云端。
