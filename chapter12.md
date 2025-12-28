# Chapter 12｜可观测性、评测与持续迭代 (Observability, Evals & Iteration)

## 1. 开篇段落

在传统的 Web 服务中，如果服务挂了，用户刷新一下就好。但在时速 120km/h 的车舱内，语音助手的任何迟钝、幻觉或误操作都可能演变成安全事故或严重的信任危机。基于 OpenAI Realtime API 和 Agents SDK 的系统引入了新的复杂性：**长连接的 WebSocket 状态流、非确定性的 LLM 推理、以及跨越端（车机）- 边（网关）- 云（模型）的物理链路**。

本章的目标是构建一个系统的“神经系统”：
1.  **可观测性 (Observability)**：不仅仅是看 Log，而是通过分布式追踪（Tracing）和多维指标（Metrics），让每一秒的延迟去向、每一次工具调用的上下文都清晰可见。
2.  **评测 (Evaluation)**：从“听感不错”转变为“数据驱动”，利用 Model-as-a-Judge 技术建立自动化评分体系。
3.  **迭代闭环 (Iteration Loop)**：通过影子模式（Shadow Mode）和灰度发布，在保障行车安全的前提下快速演进模型与策略。

---

## 2. 核心设计论述

### 2.1 全链路分布式追踪 (Distributed Tracing)

车载语音交互是典型的**异步事件驱动**架构。一个用户的意图可能触发多个下游 Agent 的协作。如果仅依赖分散的日志文件，排查问题无异于大海捞针。

#### 2.1.1 追踪上下文传播 (Trace Context Propagation)
必须建立唯一的 **`Trace_ID`** 贯穿整个生命周期。

*   **生成点 (Root Span)**：**必须在车端（Client）生成**。如果从云端网关开始生成，我们将丢失“车端 VAD 结束”到“请求到达网关”这段关键的弱网传输耗时。
*   **跨越 WebSocket**：OpenAI Realtime API 是基于 WebSocket 的持久连接。传统的 HTTP Header 追踪方式失效。需要在 WebSocket 的 `session.update` 或自定义应用层协议中携带 `client_trace_id`。
*   **工具侧透传**：当 Agent 发起 ToolCall（如控制空调）时，必须将 `Trace_ID` 作为元数据传递给车控网关，以便将物理执行结果（Action Result）与语音请求关联。

**Rule of Thumb**: **“无 Trace，不日志”。** 任何 Log 必须包含 `trace_id` 和 `span_id`。如果在代码中发现裸写的 `print` 或 `logger.info` 且不带上下文，这就是代码审查（Code Review）的阻断项。

#### 2.1.2 关键 Span 定义 (Critical Spans)

```ascii
[Trace Timeline: 1.5s Total]
|-----------------------------------------------------------------|
[Client: VAD & Encode] (50ms)
|--|
   [Network: Upstream] (150ms - unstable)
   |------|
          [Gateway: Auth & Route] (20ms)
          |-|
            [Realtime API: Inference & Tool Plan] (400ms)
            |------------------|
                               [Agent: Logic Check] (10ms)
                               |-|
                                 [Car Gateway: Execute Tool] (300ms)
                                 |-------------|
                                               [Realtime API: Response Gen] (200ms)
                                               |--------|
                                                        [Network: Downstream] (100ms)
                                                        |----|
                                                             [Client: Buffer & Play] (50ms)
                                                             |--|
```

### 2.2 多模态日志与数据脱敏 (Multimodal Logging)

Realtime API 的交互不仅仅是文本，还包含音频流和可能的截屏流。

#### 2.2.1 日志分级策略
由于车机流量昂贵且带宽有限，不能全量上传所有数据。
*   **L1 (Metric Only)**：仅上传计数器、延迟数值、错误码。**量开启**。
*   **L2 (Text/Meta)**：上传 ASR 转录文本、Tool 参数、Token 消耗。**全量开启（脱敏后）**。
*   **L3 (Payload Snapshot)**：上传关键帧图片、工具返回的 JSON 完整体。**按需开启（如错误触发）**。
*   **L4 (Raw Audio/Video)**：原始 PCM 音频、完整视频流。**仅灰度用户或特定 Debug 模式开启**。

#### 2.2.2 隐私红线与脱敏 (Redaction)
*   **实体识别 (NER)**：在边缘侧（车机或网关）运行轻量级 NER 模型，将人名、手机号替换为 `<PERSON>`, `<PHONE>`。
*   **视觉脱敏**：上传截屏日志时，必须对人脸区域、车牌区域进行高斯模糊，或仅上传提取后的 UI DOM 树结构。
*   **鉴权信息**：自动扫描日志中的 `sk-xxx`、`Bearer tokens` 并进行掩码处理。

### 2.3 指标体系设计 (Metrics Architecture)

指标是系统的仪表盘。除了通用的 CPU/内存，我们需要关注三类特定指标。

#### 2.3.1 性能与体验指标 (Performance & UX)
*   **TTFS (Time to First Sound)**：从 VAD 截断到扬声器振动的端到端耗时。这是用户感知“快慢”的唯一标准。
*   **Interruption Rate (打断率)**：系统在播报时被用户打断的比例。
    *   *高打断率可能意味着*：回答太啰嗦、回答错误、或者 VAD 过于敏感（把噪音当人声）。
*   **Jitter Buffer Underrun**：音频播放缓冲区的欠载次数。直接对应用户听到的“卡顿”或“电音”。

#### 2.3.2 业务质量指标 (Business Quality)
*   **Task Success Rate (TSR)**：
    *   *车控*：ToolCall 参数正确且车辆执行成功的比例。
    *   *GUI*：成功抵达目标页面并完成点击的比例。
*   **RAG Evidence Ratio**：回答中包含有效引用（Citation）的比例。
*   **Fallback Rate**：系统回复“对不起，我不明白”、“请再说一遍”的比例。

#### 2.3.3 安全与风险指标 (Safety & Risk)
*   **DMS Trigger Count**：由于驾驶员分心/疲劳导致系统主动拒绝执行复杂任务的次数。
*   **High-Risk Tool Rejection**：用户请求高危动作（如行驶中开门）被系统拦截的次数。

### 2.4 评测体系 (Evaluation Pipeline)

不要相信“我觉得这个版本好多了”。

#### 2.4.1 离线评测 (Offline Eval)
建立“黄金数据集”（Golden Dataset），包含数千条覆盖各种场景的 Query。
*   **确定性评测**：针对车控。Query: "打开主驾座椅加热三档"。Expectation: `tool="seat_control", seat="driver", action="heat", level=3`。完全匹配才算过。
*   **语义评测**：针对闲聊/RAG。利用 GPT-4o 作为裁判（Judge），根据 Rubric 打分（1-5分）。
    *   *Rubric 维度*：准确性、简洁性（车内不宜长篇大论）、无害性、语气（是否过于顺从或过于生硬）。

#### 2.4.2 在线评测 (Online Eval)
利用用户行为作为隐式反馈。
*   **即时修正 (Correction)**：用户在 T+1 轮次说“不是”、“错了”、“取消”。 -> **负样本**。
*   **打断 (Barge-in)**：用户在系统说话的前 20% 时间内打断。 -> **疑似负样本**（不想听废话）。
*   **静默确认 (Silence)**：系统执行动作后，用户不再追问。 -> **正样本**。

### 2.5 持续迭代与实验 (CI/CD & Experimentation)

#### 2.5.1 影子模式 (Shadow Mode)
在发布新的 Prompt 或模型版本（如 gpt-4o-realtime-preview-new）之前，先开启影子模式。
1.  **流量复制**：将线上真实用户的音频流分发一份给影子模型。
2.  **Mock 执行**：影子模型进行推理、生成 ToolCall，但**不真正发给车控网关**，而是记录“如果执行，参数会是什么”。
3.  **Diff 分析**：对比 V1（线上）和 V2（影子）的 ToolCall。
    *   *Improved*：V1 没听懂，V2 输出了正确参数。
    *   *Regressed*：V1 正确，V2 输出了错误的温度。

#### 2.5.2 A/B 测试
在非安全关键路径（如 RAG 问答、推荐服务）进行 A/B 测试。
*   **实验组**：使用更泼的 System Prompt 或新的检索策略。
*   **对照组**：保持原状。
*   **观测指标**：多轮对话深度（Turn count）、用户满意度调研。

---

## 3. 本章小结

*   **全链路追踪是基石**：Trace ID 必须从车端生成，跨越 WebSocket 和物理网关，是调试所有延迟和错误的前提。
*   **端到端指标是唯一真理**：用户不关心模型有多快，只关心从闭嘴到听到声音的 TTFS。
*   **数据驱动质量**：通过“影子模式”进行无风险验证，通过“Model-as-a-Judge”进行规模化离线评分。
*   **分级日志**：在成本、隐私和可调试性之间寻找平衡，默认仅开启 Metrics 和脱敏文本 Log。

---

## 4. 练习题

### 基础题

**Q1: 在计算“首字延迟”（TTFB）和“首声延迟”（TTFS）时，为什么必须区分两者？在 Realtime API 场景下它们分别受什么影响？**
*   提示：网络包到达 vs 播放器缓冲区填满。

<details>
<summary>点击展开答</summary>

**答案**：
*   **区别**：TTFB 是客户端收到第一个音频数据包的时间；TTFS 是扬声器实际发出声音的时间。两者之间存在**解码耗时**、**Jitter Buffer（抗抖动缓冲）** 和 **播放器启动耗时**。
*   **影响因素**：
    *   TTFB 主要受网络 RTT、模型推理首字生成速度影响。
    *   TTFS 额外受车机音频驱动（Audio HAL）、缓冲策略（为了平滑可能会故意缓存 200ms）影响。
*   **重要性**：用户感知的是 TTFS。如果 TTFB 很低但 TTFS 很高，说明车端播放策略有问题（如缓冲过大）。
</details>

**Q2: 系统日志中出现大量 `input_audio_buffer.speech_stopped` 事件，但紧接着没有任何 ToolCall 或 Response，且 Trace 显示 `server_vad` 触发。这可能是什么问题？**
*   提示：误触发、静音阈值。

<details>
<summary>点击展开答案</summary>

**答案**：
这通常意味着**误触发（False Positive）**。
1.  **原因**：车内噪音（风噪、空调声）或旁人说话被 Realtime API 的服务端 VAD 误判为有效语音指令。
2.  **现象**：系统截取了一段音频进行推理，但模型判断该音频无意义（或无法识别意图），因此输出了空结果或被后续逻辑过滤。
3.  **解决**：需要调优端侧 VAD 的门限（Gate），或者在 Prompt 中加强对无意义噪音的忽略指令。
</details>

**Q3: 在日志脱敏中，为什么建议在“边缘侧（车机/网关）”进行，而不是在“云端日志服务”中进行？**
*   提示：攻击面、合规性。

<details>
<summary>点击展开答案</summary>

**答案**：
1.  **传输合规**：某些法规（如 GDPR 或中国数据安全法）禁止未脱敏的 PII（个人身份信息）或敏感地理位置信息**离开设备**或**跨境传输**。
2.  **攻击面最小化**：如果在云端脱敏，意味着原始敏感数据必须先经过网络传输，这增加了中间人攻击或云端存储泄露的风险。
3.  **带宽节省**：去除不必要的原始数据（如高分辨率视频背景）可以显著降低上传带宽。
</details>

### 挑战题

**Q4: 设计一个针对“GUI Agent”的自动化回归测试方案。App 的 UI 经常变，如何避免测试脚本频繁失效？**
*   提示：不要依赖坐标 (x, y)，考虑语义匹配。

<details>
<summary>点击展开答案</summary>

**答案思路**：
1.  **避免硬编码**：严禁使用 `click(x=500, y=200)` 或绝对 XPath。
2.  **语义定位**：测试脚本应描述“意图”，如 `Action: Click Button with Text "下单"` 或 `Find Element: Coffee Latte`。
3.  **利用 Agent 自愈**：
    *   测试执行器不直接操作 UI，而是调用一个“Test Agent”。
    *   当 Test Agent 发现找不到“下单”按钮时，它会分析当前页面 DOM，寻找语义相似的元素（如“去支付”、“立即购买”），尝试执行并记录警告。
4.  **视觉回归 (Visual Regression)**：在关键步骤对比截图的结构似度，如果 UI 布局巨变（如改版），触发人工介入，而不是直接报错。
</details>

**Q5: 你的系统上线后，发现“Undo 率”（用户撤销操作）在“高速公路”场景下比“市区”场景高出 3 倍。请利用可观测性体系推断可能的原因，并设计验证实验。**
*   提示：环境噪音、认知负荷、风险规避。

<details>
<summary>点击展开答案</summary>

**答案思路**：
**推断原因**：
1.  **噪音干扰**：高速路胎噪/风噪大，导致 ASR 错误率飙升（如把“开窗”听成“开强风”）。
2.  **驾驶员紧张**：高速驾驶时用户语速变快或含糊，且对系统的任何错误容忍度极低（稍微不对立即撤销）。
3.  **VAD 切割错误**：持续的底噪导致 VAD 无法准确判断语音结束，录入了过长的尾部噪音，影响意图识别。

**验证实验**：
1.  **数据切片**：在日志系统中筛选 `Speed > 80km/h` 的 Session，分析其 ASR 转录文本的 Word Error Rate (WER)。
2.  **听测**：抽样听取高速场景的原始音频，确认是否包含大量风噪。
3.  **影子模式实验**：针对高速场景部署专门的 Prompt（如“在嘈杂环境下，请更倾向于反问确认而不是直接执行”），观察 Undo 率变化。
</details>

**Q6: 如何计算 RAG 系统的“无知率”（Ignorance Rate）与“幻觉率”（Hallucination Rate）？这两个指标有什么区别？**
*   提示：诚实回答 vs 胡编乱造。

<details>
<summary>点击展开答案</summary>

**答案思路**：
*   **无知率 (Ignorance Rate)**：
    *   *定义*：系统明确表示“我不知道”、“未在手册中找到相关信息”的比例。
    *   *性质*：这是**好**的失败。说明系统边界清晰，未检索到知识时选择了诚实。
    *   *计算*：正则匹配回答中的拒识关键词。
*   **幻觉率 (Hallucination Rate)**：
    *   *定义*：系统给出了回答，但答案是错误的，或者引用了错误的文档（如引用“空调章节”回答“轮胎问题”）。
    *   *性质*：这是**坏**的失败。具有误导性。
    *   *计算*：需要 `Model-Based Eval`。
        *   Prompt Judge: "Does the answer strictly follow the provided context? If no, label as Hallucination."
        *   检查 `Citation` 链接的有效性。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 "Heisenberg" 日志效应
*   **错误**：为了 Debug 偶发的卡顿问题，在生产环境开启了 `Level=Debug` 日志，同步写入磁盘或上传。
*   **后果**：大量的 I/O 操作抢占了 CPU 时间片，导致音频解码线程饥饿，**原本偶发的卡顿变成了必现的卡顿**。
*   **对策**：车机端的日志写入必须是**异步、非阻塞**的，且在环形缓冲区（Ring Buffer）中进行，仅在关键事件触发时刷盘。

### 5.2 盲目信任 "Status: 200"
*   **错误**：监控看板显示 HTTP/WebSocket 错误率为 0%，认为系统很健康。
*   **后果**：Realtime API 连接正常，但模型一直输出“......”或者在循环重复上一句话。
*   **对策**：实施**语义监控**。检测输出文本的**熵（Entropy）**、重复率、以及是否命中“兜底话术库”。

### 5.3 忽略了“端侧时钟偏差”
*   **错误**：直接用 Server 接收到日志的时间减去 Client 发送日志的时间来计算延迟。
*   **后果**：由于车机时间可能不准（或与服务器未同步），可能计算出“负数延迟”或“巨大延迟”。
*   **对策**：使用 NTP 同步，或者在计算延迟时仅依赖**单调时钟（Monotonic Clock）的差值**（Duration），而不是绝对时间戳的减法。或者在握手阶段计算 `Clock_Offset` 进行校正。

### 5.4 影子模式的“副作用”
*   **错误**：影子模式虽然拦截了车控指令，但忘记拦截 **副作用（Side Effects）**，例如“发送短信”、“创建日历日程”或“调用第三方 API（如点餐下单）”。
*   **后果**：用户只是问问，结果后台默默下了两单咖啡，或者给老板发了奇怪的测试短信。
*   **对策**：对所有外部 Tool 实施严格的 **Mock 隔离**。在影子模式下，Tool Executor 必须被替换为 Dummy Executor，只返回模拟数据，绝对不能发生网络 IO。
