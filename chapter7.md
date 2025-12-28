# Chapter 7｜多模态感知输入设计 (Multimodal Perception Input)

## 7.1 开篇与目标

**多模态感知（Multimodal Perception）** 是将车载语音助手从“被动听令工具”升级为“主动式智能伙伴”的关键。在 OpenAI Realtime API 的加持下，模型不仅能处理音频，还能原生理解视觉信息。然而，车舱环境具有特殊性：**带宽受限**（车机 4G/5G 流量昂贵）、**隐私敏感**（车内是私密空间）、**实时性要求极高**（驾驶场景不容延迟）。

本章不再仅仅讨论“接入摄像头”，而是聚焦于构建一个智能的 **Perception Adapter（感知适配层）**。该层位于端侧（Edge），负责对 DMS（驾驶员监控）、OMS（乘员监控）、车外 7V 摄像头以及屏幕（IVI Screen）的信号进行**预处理、融合、过滤和对齐**，最终以最经济、最安全的方式注入到云端会话中。

**学习目标**：
1.  **架构设计**：掌握车端感知适配层（Perception Adapter）的职责与数据流向。
2.  **输入源治理**：深入理解 DMS/OMS/7V 及屏幕读取（UI Tree vs Pixel）的数据特性。
3.  **带宽与成本控制**：学会设计“按需采样”、“ROI 裁剪”和“事件驱动”的上传策略，防止 Token 爆炸。
4.  **时空对齐**：解决“语音”与“视觉”在时间戳和空间坐标系上的同步问题。
5.  **隐私计算**：确立端侧脱敏与云端推理的安全边界。

---

## 7.2 文字论述

### 7.2.1 核心架构：感知适配层 (The Perception Adapter)

直接将车上 10+ 路摄像头的 RTSP 流推给云端是不现实的。我们需要在车机（SoC）或座舱域控制器上部署一个中间件——**感知适配层**。

**设计原则**：*Heavy Lifting on Edge, Semantic Reasoning in Cloud.*（端侧做重活，云端做理解。）

```ascii
[ Vehicle Environment ]               [ Edge Computing (Car SoC) ]                [ Cloud / OpenAI ]
+---------------------+               +--------------------------+              +-------------------+
| 1. Sensors          |               | Perception Adapter       |              | Realtime Session  |
| - DMS Camera (IR)   |--Raw Video--->| +----------------------+ |              |                   |
| - OMS Camera (Fish) |--Raw Video--->| | Vision Pre-processor | |              | +---------------+ |
| - 7V Ext Cameras    |--Raw Video--->| | (Face Det/Dewarp)    | |              | | Vision Model  | |
| - Microphone Array  |--Audio Stream>| +----------+-----------+ |              | | (GPT-4o)      | |
| - IVI Screen        |--UI Tree----->|            |             |              | +-------+-------+ |
+---------------------+               | +----------v-----------+ |              |         ^         |
                                      | | Event & Frame Buffer | |              |         |         |
[ Vehicle Bus (CAN) ]                 | | (Sync Timeline)      | |              |         |         |
+---------------------+               | +----------+-----------+ |   WebRTC/WS  |         |         |
| 2. Signals          |               |            |             | <==========> |         |         |
| - Gear/Speed        |--Signal------>| +----------v-----------+ |   (Audio)    |         |         |
| - GPS/Map Data      |--Data-------->| | Context Injector     | | <==========> |         |         |
+---------------------+               | | (Filter & Optimize)  | | (Events/Img) |         |         |
                                      +--------------------------+              +-------------------+
```

#### 组件职责：
1.  **Vision Pre-processor (视觉预处理)**：
    *   **畸变校正 (Dewarping)**：将 OMS 的鱼眼图像展平，否则模型无法识别空间关系。
    *   **ROI (Region of Interest)**：根据 DMS 视线向量，裁剪 7V 画面中的关注区域。
    *   **Privacy Masking**：在端侧对车牌、人脸进行高斯模糊。
2.  **Event & Frame Buffer (事件与帧缓冲)**：
    *   维护一个短时（如 5秒）的环形缓冲区。当 VAD（语音活动检测）触发时，能够回溯提取用户“开口前 1 秒”的画面，解决“看见再说”的时间差。
3.  **Context Injector (上下文注入器)**：
    *   决定何时发送图像（Message），何时发送文本描述（System Instruction），何时仅发送信号（Function Call）。

### 7.2.2 输入源详解与处理策略

#### 1. DMS (Driver Monitoring System)
*   **特性**：近红外，关注人脸与视线。
*   **信号化处理**：
    *   **高频低宽带**：不要上传 DMS 视频。在端侧计算出 `Gaze Vector` (视线向量，如 `{yaw: 15, pitch: -5}`) 和 `Emotion` (情绪)。
    *   **策略**：将结构化数据写入 Session 的 `session.attributes` 或作为 Tool Call 的参。
    *   **例外**：仅在识别出复杂手势（如“手在空中比划”）且置信度低时，上传截图求助云端。

#### 2. 7V / 车外摄像头 (External Cameras)
*   **特性**：高分辨率，多视角（前/后/侧），受光照影响大。
*   **场景**：“左边那个是什么楼？”
*   **处理流程**：
    1.  DMS 确认驾驶员看向“左侧”。
    2.  适配层选取“左视摄像头”数据。
    3.  利用 GPS 获取当前位置与朝向。
    4.  **关键点**：仅上传单帧（Snapshot），附带 GPS 文本元数据。
    *   **Rule of Thumb**: 永远不要流式传输车外景物，除非用户明确要求开启“风景解说模式”。即使那样，也建议 0.5fps 抽帧。

#### 3. 屏幕上下文 (Screen Context) - GUI Agent 的眼睛
屏幕读取是 GUI Agent 的基础。我们采用 **混合模式 (Hybrid Mode)**。

| 模式 | 数据源 | 适用场景 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- | :--- |
| **语义模式** | Accessibility Tree / DOM | 点餐、设置、列表选择 | 极小带宽 (<5KB)，精确坐标，文本可搜 | 无法理解美学、图片内容、图表 |
| **视觉模式** | Screenshot / Frame Buffer | 地图路况、相册、海报理解 | 通用性强，所见即所得 | 带宽大 (>100KB)，需 OCR，有幻觉风险 |

*   **注入策略**：
    *   默认保持 **语义模式**。每当 UI 发生变动（且静止 500ms 后），提取 UI Tree，精简为轻量级 JSON（去除样式属性，保留 ID、Text、Type、Bounds），作为 `background_context` 更新。
    *   当意图识别涉及“**看起来**怎么样”、“这张图”时，触发 **视觉模式**，上传截图。

### 7.2.3 多模态时空对齐 (Spatiotemporal Alignment)

#### 时间对齐 (Temporal Sync)
用户说“**这个**（t1）很好看”时，手指向的时间点可能是 t0.8。如果云端收到音频是 t2，收到图片是 t3（因上传延迟），就会导致错位。
*   **方案**：
    *   车端统一时钟（UTC）。
    *   Audio Buffer 和 Image Message 均携带 `capture_timestamp`。
    *   Prompt 工程：提示模型“用户所指的物体出现在时间戳 X 的画面中”。

#### 空间对齐 (Spatial Sync)
用户说“帮我点那个红色的按钮”。UI Tree 里没有颜色信息，只有 `bounds`。
*   **方案**：
    *   如果使用截图：模型输出像素坐标 `(x, y)`。
    *   车端映射：将 `(x, y)` 映射回 UI Tree 中最近的 `Clickable` 元素。
    *   **坐标归一化**：所有上传的图片和 UI Tree 坐标均归一化为 `0.0 - 1.0`，避免分辨率差异。

### 7.2.4 隐私与脱敏 (Privacy by Design)

在数据离开车端之前，必须经过以下管道：

1.  **PII Scanner (个人信息扫描)**：
    *   文本流：正则匹配手机号、身份证号，替换为 `[PHONE_REMOVED]`。
2.  **Visual Masking (视觉遮罩)**：
    *   车外：检测并模糊车牌（License Plates）和行人面部。
    *   车内：除当前对话者外，模糊其他客（除非处于多人会议模式）。
3.  **Screen Sanitization (屏幕清洗)**：
    *   检测到输入法键盘弹出、密码框（`inputType="password"`）、银行卡号显示区域，直接在截图上覆盖纯黑 Mask。

---

## 7.3 本章小结

1.  **分层架构**：不要让 Realtime API 直接裸连摄像头。**感知适配层**是必须的中间件，负责降噪、融合和缓冲。
2.  **数据节约**：默认传输**结构化数据**（DMS 事件、UI Tree），仅在 VAD 激活且意图明确时传输**视觉数据**（截图/ROI）。
3.  **屏幕双模态**：GUI Agent 应当优先依赖 Accessibility Tree 进行精准控制，仅在视觉任务中使用截图。
4.  **时钟同步**：解决“指哪打哪”的关键在于端侧打时间戳，而非依赖服务器接收时间。
5.  **安全红线**：所有敏感数据（密码、车牌、旁人隐私）必须在车端完成脱敏，云端只能看到清洗后的数据。

---

## 7.4 练习题

### 基础题

**Q1: 在车舱语音交互中，为什么说“事件驱动的图像传输”优于“视频流传输”？请列举三个具体的收益。**
<details>
<summary>参考答案</summary>

1.  **成本大幅降低**：视频流每秒产生大量 Token（即使是 GPT-4o 也非常昂贵），事件驱动仅在需要时（如用户提问）传输单帧，成本相差几个数量级。
2.  **降低系统延迟**：实时视频流占用大量上行带宽，可能导致语音包排队拥塞，增加对话延迟（Latency）。
3.  **隐私合规**：不持续上传视频意味着不持续记录用户隐私，降低了数据泄露风险，更容易通过合规审查。
</details>

**Q2: 解释什么是 Accessibility Tree（辅助功能树/UI 树），以及为什么它是 GUI Agent 的首选输入？**
<details>
<summary>参考答案</summary>

*   **定义**：Accessibility Tree 是操作系统（Android/Linux/QNX）提供的一种层级结构，描述了屏幕上所有 UI 控件的属性（文本、类型、位置、状态ID），原本用于盲人读屏软件。
*   **首选原因**：
    1.  **机器可读性**：直接提供文本和控件 ID，无需 OCR 和目标检测，准确率 100%。
    2.  **极低带宽**：一个复杂页面的 UI Tree JSON 往往只有几 KB，而截图需要几百 KB。
    3.  **包含隐藏信息**：能提供截图无法显示的属性（如按钮是否 `disabled`，列表是否 `scrollable`）。
</details>

**Q3: DMS 检测到用户视线指向“右后视镜”，同时麦克风收到“帮我收起来”。请描述感知适配层应该如何构造发送给 Realtime API 的消息？**
<details>
<summary>参考答案</summary>

感知适配层不应发送右后视镜的**图像**（因为意图是控车，不是视觉问答），而是发送**上下文信息**。
1.  **构建 Context**：`{ "user_gaze": "side_mirror_right", "gaze_confidence": 0.95 }`。
2.  **发送消息**：
    *   音频流：用户的语音“帮我收起来”。
    *   Function/Tool Definition：确保 `fold_mirror(mirror_id)` 在可用工具列表中。
3.  **模型推理**：模型结合语音意图和 `user_gaze` 上下文，推断出 `mirror_id = right`，并调用 `fold_mirror("right")`。
</details>

### 挑战题

**Q4: 设计一个处理“多乘客并发冲突”的感知策略。主驾说“打开窗户”，副驾同时说“太冷了别开”。OMS 和麦克风阵列如何配合？**
<details>
<summary>参考答案</summary>

1.  **声源定位 (SSL)**：麦克风阵列通过波束成形（Beamforming）识别出两个声源方向：Zone 1 (主驾) 和 Zone 2 (副驾)。
2.  **OMS 验证**：OMS 摄像头确认 Zone 1 和 Zone 2 均有乘客，且嘴唇在动（Lip Activity Detection）。
3.  **身份标记**：语音转文字时，将两条流分别标记为 `[Driver]: Open window` 和 `[Passenger]: Don't open`。
4.  **仲裁逻辑（System Prompt）**：
    *   系统提示词中设定优先级：安全相关指令（如有人喊停） > 主驾指令 > 副驾指令。
    *   模型输出决策回复“收到，副驾觉得冷，那我们先不开窗。”
</details>

**Q5: 针对 7V 摄像头的“所见即所得”功能，设计一套“缓存-回溯”机制，以解决用户说完话时车已经开过目标物体的问题。**
<details>
<summary>参考答案</summary>

1.  **环形缓冲区 (Ring Buffer)**：在内存中维护一个 5-10 秒的视频帧/元数据缓冲区，覆盖 7 个摄像头。
2.  **VAD 标记**：记录用户语音开始的时间戳 $T_{start}$ 和结束时间戳 $T_{end}$。
3.  **回溯策略**：
    *   当意图识别为“查询路况/景物”时，不要截取 $T_{end}$（说完话）时的画面。
    *   计算最佳帧时间 $T_{best} = T_{start} + \Delta$（通常用户是在看到物体后 200-500ms 开始说话）。
    *   如果车速 $V$ 很高，需根据 $V \times (T_{now} - T_{best})$ 计算位移，甚至调用后视摄像头（如果物体已经甩到车后）。
4.  **数据上传**：提取 $T_{best}$ 时刻最符合视线方向的一帧上。
</details>

**Q6: 你的系统需要支持一个“相册管家”功能（用户在屏幕上浏览照片，语音让助手‘把这张发给老婆’）。如何兼顾隐私（不上传所有照片）和功能可用性？**
<details>
<summary>参考答案</summary>

1.  **本地元数据索引**：不要将照片像素传给云端。App 端提供当前照片的元数据（ID, 时间, 地点, 人物标签-本地计算）。
2.  **意图路由**：
    *   用户：“这是哪里拍的？” -> 上传元数据中的 GPS/地点信息给 LLM。
    *   用户：“照片里有几个人？” -> 上传元数据中的 Face Count 或本地 OCR 结果。
    *   用户：“把这张发给老婆” -> 触发 ToolCall `share_photo(photo_id, contact_name)`。
3.  **仅必要时上传**：只有当用户问具体的视觉细节（“他手里拿的是什么？”）且本地标签无法回答时，才申请用户授权上传经过人脸模糊处理的当前照片截图。
</details>

---

## 7.5 常见陷阱错误 (Gotchas)

### 1. 坐标系的巴别塔 (Coordinate System Mismatch)
*   **陷阱**：UI Tree 的坐标是像素绝对值（如 `1920x1080`），而截图上传给 OpenAI 后，模型可能在缩放后的图片（如 `512x512`）上进行归一化坐标预测。ToolCall 执行点击时，坐标偏离几百像素，点到了错误的按钮。
*   **调试技巧**：
    *   强制执行**归一化坐标 (0.0 - 1.0)** 标准。
    *   在 System Prompt 中明确告知模型：“Input image resolution is varied, please output normalized coordinates (0.0-1.0) for point [x, y].”
    *   在端侧执行器中，将归一化坐标 `x * screen_width` 还原。

### 2. "幽灵"元素污染上下文 (Ghost Elements)
*   **陷阱**：直接序列化 UI Tree，包含了大量不可见元素（`visibility=gone`）、被弹窗遮挡的底层元素、或者宽高为 0 的布局容器。这消耗了 Context Window，且让模型产生幻觉去点击不可见按钮。
*   **调试技巧**：
    *   编写一个严格的 `TreePruner`（树修剪器）。
    *   规则：剔除 `isVisibleToUser=false`、`width<=0`、`height<=0` 的节点。
    *   计算 Z-Order，如果一个按钮完全被另一个 `clickable` 的 View 覆盖，则从树中移除该按钮。

### 3. 时间戳漂移 (Timestamp Drift)
*   **陷阱**：使用 HTTP/WebSocket 发送图片的时刻作为“时间戳”。由于网络抖动（Jitter），图片到达云端可能比语音慢 2 秒，导致模型认为用户在评论 2 秒后的画面。
*   **调试技巧**：
    *   **不要依赖服务器接收时间**。
    *   在协议层（Protocol）中显式定义 `event_time` 字段。
    *   在调试面板中，将语音波形与关键帧图片按 `event_time` 对齐回放，检查是否同步。

### 4. 也是隐私：反射风险 (Reflection Risk)
*   **陷阱**：拍摄仪表盘或副驾屏幕时，光面屏幕反射出了驾驶员的脸或车外敏感环境。
*   **调试技巧**：
    *   这是一个极难完全避免的物理问。
    *   缓解措施：降低上传图像的分辨率和对比度，或者在端侧部署专门的“反射去除”算法（但这通常算力成本过高）。
    *   **合规话术**：在隐私协议中明确告知“屏幕截图可能包含环境反射”。

### 5. 令牌通胀 (Token Inflation)
*   **陷阱**：为了追求高精度，每次 GUI 操作都上传一张 1080P 高清截图。GPT-4o 处理高分图片的 Token 消耗是巨大的（一张图可能消耗 >1000 Tokens）。如果是多轮对话，费用惊人。
*   **Rule of Thumb**：
    *   UI Tree 优先（Token 极少）。
    *   如必须截图，先降采样到 512px 宽，或仅裁剪当前 App 的 Window 区域。
    *   对于连续操作，考虑只在“报错”或“页面剧烈变化”时才刷新截图。
