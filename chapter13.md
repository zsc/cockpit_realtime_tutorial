# Chapter 13｜部署与运维（Deployment & Operations）

## 13.1 开篇段落
车载 AI 系统的部署是一个极具挑战的工程领域，它处于**高安全性要求（车规级）**与**极速迭代要求（AI 模型）**的矛盾交汇点。本章将详细阐述如何构建一个既能满足车辆行驶安全标准，又能灵活适配 OpenAI Realtime API 和 Agents SDK 快速演进的混合架构。

与传统的 Web 服务不同，车载部署必须面对：
1.  **不稳定的网络拓扑**：车辆会在 5G、4G、2G 和无信号区域间穿梭。
2.  **异构的硬件环境**：同一车系的高低配车型，其车机算力（CPU/NPU）和内存可能有倍数级差异。
3.  **不可逆的物理风险**：错误的部署可能导致车辆功能异常。

**本章习目标**：
*   掌握“车端-网关-模型端”的三层部署架构及其流量转发机制。
*   实现基于临时凭证（Ephemeral Token）的零信任安全接入方案。
*   设计细粒度的动态配置（Config Dispatch）系统，解耦 OTA 与 AI 迭代。
*   建立全链路可观测性体系，精准定位“语音延迟”的物理归属。

---

## 13.2 部署架构详解 (Deep Dive Architecture)

### 13.2.1 三层拓扑结构
我们采用**Tethered Architecture（系留架构）**，即车机作为感知与执行的触手，云端作为大脑，中间通过高可用的网关连接。

```ascii
[ Layer 1: Vehicle Edge ]       [ Layer 2: Managed Gateway ]       [ Layer 3: Intelligence Provider ]
(Android/QNX/Linux)             (Cloud Cluster / K8s)              (OpenAI & Vector DB)

+-----------------------+       +--------------------------+       +-------------------------+
|  Realtime Client SDK  |       |  A. Authentication       |       |                         |
| +-------------------+ | HTTPS |    - VIN Validation      | HTTPS |   OpenAI API Platform   |
| | Audio Buffer &    | |<----->|    - Token Issuance      |<----->|   (Model Inference)     |
| | Codec (Opus/PCM)  | |       |                          |       |                         |
| +-------------------+ |       +--------------------------+       +-------------------------+
| +-------------------+ |       |  B. Traffic Proxy        |  WSS  |                         |
| | Function Registry | |  WSS  |    - Rate Limiting       |<----->|   Realtime API Nodes    |
| | (Car Controls)    | |<=====>|    - PII Scrubbing (Txt) |       |   (GPT-4o-Realtime)     |
| +-------------------+ |       |    - Session Router      |       |                         |
| +-------------------+ |       +--------------------------+       +-------------------------+
| | Offline Fallback  | |                                          |                         |
| | (Local Command)   | |       +--------------------------+       |   RAG Knowledge Base    |
| +-------------------+ |       |  C. Agent Orchestrator   |<----->|   (Pinecone / Milvus)   |
+-----------^-----------+       |    (Agents SDK Runtime)  |       |                         |
            |                   |    - State Machine       |       +-------------------------+
      Hardware Bus              |    - Tool Resolver       |
      (SOME/IP, DDS)            +--------------------------+
```

### 13.2.2 组件职责细分

1.  **Layer 1 - 车端 (Vehicle Edge)**
    *   **音频栈**：负责回声消除 (AEC)、噪声抑制 (NS) 和本地端点检测 (VAD)。**Rule of Thumb**：VAD 必须在车端做，只有检测到人声才推流，否则会浪费 90% 的 Token 费用在传输背景噪音上。
    *   **连接管理**：维护 WebSocket/WebRTC 长连接，处理指数退避重连。
    *   **安全沙箱**：存储车机身份证书（Client Certificate），并在可信执行环境（TEE/Keystore）中签名请求。

2.  **Layer 2 - 业务网关 (Managed Gateway)**
    *   **BFF (Backend for Frontend)**：这是我们对自己基础设施的控制点。**严禁车机直连 OpenAI**。
    *   **会话中继 (Session Relaying)**：代理 WebSocket 流量。在此层可以注入 System Message（例如注入当前的车辆型号、用户昵称），对车机屏蔽 Prompt 细节。
    *   **审计与清洗**：记录所有 Tool Call 的参数；尝试识别并掩盖文本流中的敏感信息（Audio 流清洗较难，主要依赖合规协议）。

3.  **Layer 3 - 智能与知识层**
    *   **Agents Runtime**：运行 Python/Node.js 编写的 Agents SDK 逻辑，处理复杂的任务编排。
    *   **RAG Service**：提供毫秒级向量检索。

---

## 13.3 安全接入与凭证管理 (Security & Identity)

车载安全的核心原则是：**假设车机环境是不可信的，假设设备已被物理攻破。**

### 13.3.1 临时会话凭证流程 (The Handshake)
我们不使用静态 Key，而是使用 OpenAI Realtime API 的 `Client Secret`（Ephemeral Token）模式。

```ascii
Sequence Diagram: Secure Session Establishment

Vehicle (Client)          Gateway (Server)            OpenAI API
       |                         |                        |
       | 1. Auth Request (HTTPS) |                        |
       | {VIN, Timestamp, Sign}  |                        |
       |------------------------>|                        |
       |                         |                        |
       | 2. Verify Signature     |                        |
       |    & Subscription       |                        |
       |                         |                        |
       |                         | 3. Create Session      |
       |                         |    POST /v1/realtime/sessions
       |                         |    {model, instructions}|
       |                         |----------------------->|
       |                         |                        |
       |                         | 4. Returns client_secret
       |                         |    (ephemeral token)   |
       |                         |<-----------------------|
       |                         |                        |
       | 5. Return Token         |                        |
       |    {token, expiry}      |                        |
       |<------------------------|                        |
       |                         |                        |
       | 6. Connect (WSS)        |                        |
       |    Authorization:       |                        |
       |    Bearer [token]       |                        |
       |=================================================>|
       |        (Encrypted Bidirectional Stream)          |
```

*   **步骤详解**：
    1.  车机使用出厂烧录的私钥对请求签名，向 Gateway 发起认证。
    2.  Gateway 确认该 VIN（车辆识别码）未欠费、未被列入黑名单。
    3.  Gateway 代表车辆向 OpenAI 申请一个**临时会话**。此处 Gateway 可以配置该会话的默认 Prompt 和工具集。
    4.  OpenAI 返回一个有效期极短（如 60秒用于建连）的 Token。
    5.  车机拿到 Token，建立 WebSocket 连接。

### 13.3.2 敏感数据隔离
*   **支付隔离**：涉及支付的 Tool（如 `pay_parking_fee`）不直接在 LLM 链路中完成扣款。LLM 仅负责生成“预支付订单号”，车机收到 Tool Call 后，拉起原生的安全支付收银台（Native UI），要求用户进行指纹或密码确认。

---

## 13.4 版本管理与动态配置 (Versioning & Config)

车机 OTA（Over-The-Air）升级周期通常为 1-3 个月，而 AI 模型的 Prompt 优化可能每天都在发生。

### 13.4.1 动态能力矩阵 (Capability Matrix)
车机在每次启动或建连前，应拉取一份 JSON 配置文件。

**Config Example:**
```json
{
  "config_version": "2025-12-01_v3",
  "features": {
    "enable_realtime_api": true,
    "enable_gui_agent": false, // 灰度关闭点餐功能
    "use_preview_model": false
  },
  "tool_whitelist": [
    "set_temperature",
    "play_music",
    // "open_sunroof"  <-- 此车型无天窗，配置中移除此工具
  ],
  "system_prompt_version": "v4.2.1",
  "timeouts": {
    "server_processing": 5000,
    "tool_execution": 3000
  }
}
```

*   **Rule of Thumb**：永远不要让模型去猜测车辆有哪些功能。通过 Prompt 或 Tool Definition 明确告诉模型“当前车辆具备的功能列表（Tools）”。如果车辆不支持座椅按摩，不仅不注册该 Tool，甚至在 Prompt 中都要暗示“我无法控制按摩”。

### 13.4.2 兼容性策略
*   **Prompt 向下兼容**：如果 Prompt 中新增了对某个新 Tool 的描述，必须确保旧版车机（没有该 Tool 的 Native 代码）在收到该 Tool Call 时能优雅地返回 `ToolNotFound` 错误，而不是 Crash。
*   **Agents SDK 版本锁定**：云端 Gateway 需要根据车机上报的 `client_sdk_version` 路由到不同版本的 Orchestrator 集群。

---

## 13.5 弱网与韧性设计 (Resiliency & Fallback)

### 13.5.1 连接状态机
车机端须维护一个健壮的状态机：

*   **Idle**：待机，VAD 监听中。
*   **Connecting**：正在获取 Token 并建立 WSS。
*   **Connected (Realtime)**：正常语音对话中。
*   **Degraded (Weak Net)**：丢包率 > 10%，自动降低音频采样率或切换到离线模式。
*   **Offline**：断网。

### 13.5.2 降级策略 (Graceful Degradation)
1.  **音频缓冲 (Jitter Buffer)**：Realtime API 对网络抖动敏感。接收端需维护 200ms-400ms 的播放缓冲，以平滑网络抖动带来的卡顿，但这会增加感官延迟，需动态调整。
2.  **本地兜底 (On-device Fallback)**：
    *   当 Ping > 1000ms 或连接断开时，系统自动切换至本地 NLU 引擎。
    *   本地引擎仅支持基础指令：“打开空调”、“导航回家”、“上一首”。
    *   UI 显性提示：“网络不佳，仅支持基础语音指令”。

---

## 13.6 可观测性与监控 (Observability)

不能依赖用户投诉来发现问题，必须建立端到端的监控。

### 13.6.1 关键延迟指标 (Latency Breakdown)
我们需要知道“慢”在哪里。Trace ID 必须从车机生成，透传至云端。

| 指标段 (Segment) | 定义 | 正常阈值 (p99) | 常见瓶颈 |
| :--- | :--- | :--- | :--- |
| **VAD Latency** | 用户停止说话到车机判定结束 | 300-500ms | 本地算法过于保守 |
| **Upload RTT** | 车机音频包 -> 网关 -> OpenAI | 100-300ms | 4G 信号差 |
| **Inference Time** | OpenAI 收到音频到首个 Token 生成 | 300-600ms | 模型负载高、Context 过长 |
| **Download RTT** | OpenAI -> 网关 -> 车机 | 100-300ms | 下行带宽拥塞 |
| **Player Buffer** | 车机收到音频到扬声器出声 | 200ms | 播放器缓冲设置过大 |
| **E2E Latency** | **用户停顿 -> 听到声音** | **< 1.5s** | 上述总和 |

### 13.6.2 告警阈值 (Alerting Rules)
1.  **Safety Critical**：
    *   `ToolCall_Error_Rate` > 1%（车控失败率飙升，立即介入）。
    *   `Unauthorized_Access` > 5次/分钟（可能遭攻击）。
2.  **User Experience**：
    *   `E2E_Latency` > 3s 持续 5分钟。
    *   `Reconnection_Rate` > 10%（大面积断网）。

---

## 13.7 本章小结

*   **架构分层**：车端做“耳与手”（输入输出），云端做“脑”（模型），网关做“盾”（安全与流控）。
*   **安全至上**：采用 Ephemeral Token 机制，杜绝车端存储长效密钥；车控指令需经过严格的云端校验与本地二次确认。
*   **配置解耦**：利用 Feature Flags 系统应对车机碎片化和慢速 OTA 问题。
*   **数据驱动**：建立包含 VAD、网络、推理、渲染的全链路延迟监控，用数据指导优化。

---

## 13.8 练习题

### 基础题

**Q1: 在车载环境中，为什么推荐在车端（Edge）进行 VAD（语音活动检测）而不是将所有音频流式传输给云端？**
<details>
<summary>点击查看提示与参考答案</summary>
*   **提示**：流量成本、延迟、隐私。
*   **参考答案**：
    1.  **带与成本**：车内大部分时间是静音或噪音声。上传所有音频会消耗大量 4G 流量，并产生巨额的 Realtime API 费用（按分钟计费）。
    2.  **延迟**：云端 VAD 需要上传-判断-下发打断指令，增加了“打断”的延迟，体验不如本地检测灵敏。
    3.  **隐私**：上传非对话期间的音频涉及严重的隐私合规风险。
</details>

**Q2: 描述“临时会话凭证（Ephemeral Token）”相比于“静态 API Key”在车载场景的两个核心优势。**
<details>
<summary>点击查看提示与参考答案</summary>
*   **提示**：Key 泄露后果、权限控制。
*   **参考答案**：
    1.  **时效性安全**：临时凭证有效期极短（如仅限本次会话）。即使车机被黑客攻破并导出内存中的 Token，该 Token 也很快失效，无法用于长期盗刷。
    2.  **最小权限范围**：静态 Key 通常是 Project 级别的，权限过大。临时 Token 可以在生成时绑定特定的 Context工具白名单，实现单次会话的权限隔离。
</details>

### 挑战题

**Q3: 设计一个“影子模式（Shadow Mode）”的发布流程，用于在不影响用户的情况下验证新的 System Prompt 是否会导致误控车。**
<details>
<summary>点击查看提示与参考答案</summary>
*   **提示**：双路运行，抑制输出。
*   **参考答案**：
    1.  **采样**：选取 1% 的线上活跃会话。
    2.  **分流**：用户的请求同时发送给 V1（当前生产版）和 V2（新版 Prompt/Agent）。
    3.  **执行与抑制**：
        *   V1 的结果正常返回给车机执行。
        *   V2 的结果在云端被捕获，**不发给车机**。
    4.  **比对**：在后台比对 V1 和 V2 的 Tool Call 差异。例如，如果 V1 没动作，但 V2 发出了 `open_trunk`（打开后备箱）指令，则标记为潜在风险（False Positive）。
    5.  **评估**：只有当 V2 的误触率低于阈值时，才转为灰度发布。
</details>

**Q4: 车处于 3G 弱网环境，Realtime API 连接频繁中断。请设计一套完整的用户体验降级方案，涵盖 UI、语音和功能三个维度。**
<details>
<summary>点击查看提示与参考答案</summary>
*   **提示**：预期管理、本地能力。
*   **参考答案**：
    1.  **UI 维度**：状态栏显示“网络不佳”图标；对话球变色（如从蓝色变为黄色），直观告知用户当前处于“低智模式”。
    2.  **语音维度**：
        *   合成音切换为本地 TTS（音质可能稍差但响应快）。
        *   对于无法处理的复杂请求，播放简短提示音或“请稍后再试”，避免长时间静默等待。
    3.  **功能维度**：
        *   自动屏蔽 RAG（知识问答）和 GUI Agent（点餐）意图，避免长时间 Loading 后失败。
        *   意图识别切换至本地正则/轻量级 NLU，仅响应“车窗、空调、媒体”等高置信度指令。
</details>

---

## 13.9 常见陷阱与错误 (Gotchas)

### 13.9.1 证书过期导致的“砖头”事故
*   **现象**：车机与 Gateway 认证用的 Client Certificate 有有效期（如 2 年）。到期后，车机无法认证，所有语音功能失效，且无法 OTA（因为 OTA 也需要认证）。
*   **调试/预防**：必须设计证书自动轮换（Rotation）机制。在证书过期前 3 个月，每次连接时 Gateway 检查有效期，并自动下发新证书。同时，必须保留一个物理后门或不需要证书的紧急 OTA 通道。

### 13.9.2 音频编解码不匹配
*   **现象**：Realtime API 推荐使用 G.711 或 PCM16，但为了省流，车机端使用了高压缩比的 Opus 或 AAC。如果网关层未做转码，或者 Sample Rate（24k vs 16k）不匹配，会导致模型听到全是杂音，输出乱码。
*   **预防**：端到端统一音频格式标准（推荐 PCM 24kHz 单声道用于高质量语音，或 G.711 用于低延迟）。如果在车端编码，务必确保云端有对应的解码器。

### 13.9.3 忽略了“地域围栏（Geo-fencing）”
*   **现象**：车辆跨境行驶（如从中国内地开到香港，或欧洲跨国）。由于数据合规要求（GDPR 等），原本连接的云端节点可能不再合法，或者 OpenAI 的服务在某些地区不可用。
*   **预防**：App 启动时检查 GPS 位置，根据位置连接至对应合规区域的数据中心（Region）。如果进入不支持的区域，应主动切断云端服务并提示用户。

### 13.9.4 Token 耗尽时的静默崩溃
*   **现象**：车主聊得太嗨，单次会话超过了 OpenAI 的 Context Window 限制，或者 Token 余额耗尽。API 返回 4xx 错误，车机端未处理该错误，导致用户说话后系统毫无反应。
*   **预防**：必须捕获 WebSocket 的 `error` 事件。遇到 Context Limit 时，自动截断历史消息或开启新 Session；遇到 Quota 问题时，播放特定的错误提示音（“服务暂时不可用”），而不是静默。
