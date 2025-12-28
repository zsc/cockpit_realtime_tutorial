# Chapter 6｜RAG 检索与知识系统设计（RAG & Knowledge System）

## 1. 开篇段落与设计目标

### 1.1 背景
在车载环境中，驾驶员与乘客对信息的获取有着极高的要求：**准确（Accuracy）**、**即时（Immediacy）**和**简明（Conciseness）**。通用大模型（Foundation Models）虽然具备常识，但缺乏特定车辆（Unique VIN）的配置知识。例如，同一款车型的“2023款”和“2025款”在自动驾驶开启方式上可能完全不同。如果模型产生幻觉，不仅影响体验，更可能引发安全事故。

### 1.2 核心目标
本章旨在设计一个**特定于车辆上下文（Vehicle-Context-Aware）**的检索增强生成系统。
*   **高精度**：通过严格的元数据过滤，确保只回答当前车辆配置相关的问题。
*   **低延迟**：适配 Realtime API 的节奏，从意图识别到语音输出的首字延迟（Time to First Token）控制在可接受范围内。
*   **可溯源**：所有回答必须提供手册章节或文档引用，支持“有据可查”。
*   **多模态融合**：支持“看到仪表盘报警灯”直接触发检索。

---

## 2. 知识数据源与治理（Data Ingestion & Governance）

数据处理流水线（ETL）是 RAG 系统的灵魂。垃圾进，垃圾出（Garbage In, Garbage Out）。

### 2.1 异构数据源分类
车载知识库远不止一本 PDF 手册，它是一个异构数据的集合：

| 数据类型 | 来源示例 | 更新频率 | 特点 | 处理策略 |
| :--- | :--- | :--- | :--- | :--- |
| **核心静态数据** | 车主手册、保修手册、快速指南 | 年/款更新 | 结构化极强，含大量表格和警告框 | PDF 解析 + 布局分析 + 表格重构 |
| **动态服务数** | 故障码库 (DTC)、TSB (技术通报) | 月/周更新 | 代码 (P0300) 对应 文本解释 | 键值对索引 (Keyword Search) |
| **运营/生活数据** | 车机 App 权益、充电桩指引、保险条款 | 实时/日更新 | 碎片化、营销话术多 | 网页抓取 + 语义分块 |
| **多模态索引** | 仪表图标、中控按钮图、机舱部件图 | 车型换代更新 | 图像数据 | 图像向量化 (CLIP/ViT) 或分类标签化 |

### 2.2 深度文档解析与分块策略 (Advanced Chunking)

简单的“按字符数切分”在手册场景下是灾难性的。我们需要**语义感知分块**。

#### 2.2.1 布局感知解析 (Layout-Aware Parsing)
PDF 手册中常有跨页的表格、侧边栏的警告（Warning/Caution）。
*   **表格处理**：必须将表格还原为 Markdown 表格或 JSON，嵌入到对应的文本块中。如果表格被拆散，模型将无法理解“前轮”对应“2.5 bar”。
*   **警告关联**：将“警告框”与其紧邻的操作步骤强绑定，不可拆分。

#### 2.2.2 父子索引策略 (Parent-Child Indexing)
为了兼顾检索的**精准度**和生成上下文的**完整性**，采用父子块策略：
*   **Child Chunk (用于检索)**：细粒度切分（如 128 tokens），包含具体的关键词和单一语义点。
*   **Parent Chunk (用于生成)**：当 Child 被检索命中时，向 LLM 投喂其所属的更大的 Parent Chunk（如 512-1024 tokens），包含完整章节上下文。

### 2.3 元数据分类体系 (Metadata Taxonomy)
这是车规 RAG 最关键的设计。每个 Chunk 必须打上以下 Tag：
*   `vin_series`: 车型系列 (e.g., Model X)
*   `model_year`: 年款 (e.g., 2024)
*   `trim_level`: 配置级 (e.g., Plaid, LongRange)
*   `region`: 市场区域 (e.g., CN, EU, NA) - 法规不同导致功能说明不同
*   `software_version`: 软件版本范围 (e.g., >v11.0)

---

## 3. 检索架构设计（Retrieval Architecture）

我们采用**混合检索 + 重排**的工业级架构，以平衡语义理解和确匹配。

### 3.1 架构数据流 ASCII 图

```ascii
[User Audio Stream] 
       | (OpenAI Realtime API)
       v
[User Text/Transcript] + [Context: VIN, Location, Speed]
       |
       v
[Query Understanding Module]
| 1. Query Rewrite: "那个红色的圈圈灯" -> "制动系统故障警告灯"
| 2. Intent Classification: Manual_QA vs. ChitChat vs. Car_Control
| 3. Metadata Extraction: Extract filter tags based on current car state
       |
       v
[Hybrid Search Engine] <=== (Parallel Execution) ===>
|                                                  |
|-- [Vector Search (Dense)]                        |-- [Keyword Search (Sparse/BM25)]
|   (Captures "meaning")                           |   (Captures exact codes/names)
|   Query: Embedding(Text)                         |   Query: "Isofix", "P0300", "5W-40"
|   Filter: model='X' AND year='2024'              |   Filter: model='X' AND year='2024'
|   Return: Top-20 Chunks                          |   Return: Top-20 Chunks
|                                                  |
       v                                           v
[Ensemble & Deduplication] (Merge results)
       |
       v
[Reranker (Cross-Encoder)]
|  Input: (Query, Chunk_1), (Query, Chunk_2)...
|  Action: Deep semantic scoring (Slow but accurate)
|  Output: Top-3 Chunks with Re-rank Score
       |
       v
[Context Window Builder]
|  Format:
|  "Context 1 [Ref: Manual Sec 3.1]: ...text..."
|  "Context 2 [Ref: DTC DB]: ...text..."
       |
       v
[LLM Generation (Realtime API Output)]
```

### 3.2 关键组件详述

#### 3.2.1 查询重写 (Query Rewriting)
用户口语通常不规范。
*   用户说：“车头有个这种标志” -> 重写为：“车辆前部标志说明”。
*   用户说：“怎么连手机” -> 重写为：“蓝牙连接配对步骤”。
*   此步骤通常由一个轻量级 LLM (如 GPT-3.5-turbo 或微调后的 SLM) 快速完成，或作为 Agent 的思维链一部分。

#### 3.2.2 混合检索 (Hybrid Search)
*   **向量检索**：擅长回答概念问题（“为什么空调不制冷？”）。
*   **关键词检索**：擅长回答实体参数（“胎压是多少？”、“保险丝 F34 控制什么？”）。
*   **权重调节**：通常设为 `alpha * Vector_Score + (1-alpha) * BM25_Score`。

#### 3.2.3 重排 (Reranking)
向量相似度（Cosine Similarity）只能衡量“大概像”，无法衡量“逻辑蕴含”。Cross-Encoder 重排模型（如 BGE-Reranker）能显著提升 Top-1 准确率，剔除相关但错误的文档（如剔除“后排空调设置”当用户问“前排空调”时）。

---

## 4. 与 Realtime API 的集成策略

RAG 在此处不再是简单的“请求-响应”，而是**会话中的工具调用（Tool Call）**。

### 4.1 接口契约 (Tool Definition)
在 Realtime API session 初始化时，定义如下工具：

```json
{
  "name": "search_vehicle_manual",
  "description": "Search the vehicle owner's manual regarding features, warnings, or maintenance. Use this for 'how to', 'what is', or technical questions.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Refined search query" },
      "category": { "type": "string", "enum": ["feature", "warning", "maintenance"] },
      "requires_visual_match": { "type": "boolean", "description": "Set true if user refers to a visible icon/button" }
    },
    "required": ["query"]
  }
}
```

### 4.2 延迟掩盖与流式体验 (Latency Masking)
RAG 检索耗时（300ms - 800ms）可能导致语音对话出现尴尬的空白。
*   **Filler Words**: 在检测到 Tool Call 后，Realtime API 可以配置立即输出填充语（如“让我查一下手册…”、“正在检索相关信息…”），填补检索时的静默期。
*   **Async Processing**: 虽然 Realtime API 目前主要基于事件，但在服务端实现中，应尽可能并行处理检索与音频流的缓冲。

### 4.3 输出策略：语音摘要 + 屏幕详情
车载语音助手不应朗读长篇大论。
*   **System Prompt 令**：
    > "基于检索到的上下文回答用户。请用**口语化、简短（2句以内）**的语言概括核心步骤或结论。必须在回答末尾明确说明引用来源。如果操作复杂，请引导用户看屏幕。"
*   **多模态输出**：
    *   **Audio**: “要调整座椅高度，请使用座椅左侧的水平开关，向上或向下推动。”
    *   **Screen Payload**: 向车机发送结构化数据（JSON），车机端渲染出一张包含图文步骤的卡片，甚至高亮对应的 UI 按钮。

---

## 5. 多模态 RAG (Visual Grounding)

解决“这（指着屏幕/仪表盘）是什么？”的问题。

### 5.1 视觉检索路径
1.  **截图/推流**：触发对话时，捕获当前驾驶员视角的帧（DMS/OMS 或中控屏截图）。
2.  **视觉解析 (VLM)**：
    *   利用 GPT-4o 的视觉能力，将图像转化为文本描述：“仪表盘上有一个黄色圆圈括起来的感叹号图标”。
3.  **文本检索**：将描述作为 Query 输入 RAG 系统。
4.  **图像相似度检索 (可选)**：如果建立过图标库的向量索引（CLIP Embedding），可以直接用图像 Embedding 搜索最相似的图标 ID，获取其含义。

### 5.2 策略选择
*   **路径 A (高精度/高延迟)**：上传图片 -> GPT-4o 分析 -> 生成答案。适合复杂场景（如不认识的机械部件）。
*   **路径 B (低延迟/限定域)**：本地分类器识别常见 Icon ID -> 查表 -> 生成答案。适合仪表盘 60 个标准警告灯。

---

## 6. 安全、防幻觉与合规 (Safety & Compliance)

### 6.1 引用与归因 (Attribution)
*   **强制引用**：模型回答必须包含 `[Source: Manual_V2_Pg45]`。
*   **UI 呈现**：车机屏幕上显示引用来源的缩略图，用户点击可跳转到电子手册原文。

### 6.2 拒答机制 (Refusal Strategy)
*   **阈值控制**：如果 Rerank 的最高分低于设定阈值（如 0.6），视为“未找到相关信息”。
*   **安全兜底话术**：“我在当前手册中没有找到关于‘X’的确切说明。为了安全起见，建议您联系售后服务或查看纸质手册索引。”
*   **禁止推断**：System Prompt 中明确禁止根据常识推断胎压、油号、扭矩等敏感参数。

### 6.3 法律免责
对于涉及维修、改装、急救的回答，必须附带简短的免责声明（可在语音中略去，但必须在屏幕文字中展示）。

---

## 7. 性能优化 (Performance Optimization)

### 7.1 多级缓存 (Multi-level Caching)
1.  **L1 本地缓存 (In-Car)**：高频静态问答（Top 50 FAQ，如“怎么连蓝牙”、“胎压复位”）。零延迟，断网可用。
2.  **L2 语义缓存 (Edge/Cloud)**：使用 Redis + Vector。当新 Query 与历史 Query 相似度 > 0.95 时，直接返回历史答案。

### 7.2 边缘计算与离线能力
虽然 Realtime API 依赖云端，但 RAG 索引可以做**切片下发**。
*   **场景**：车辆进入地下车库。
*   **方案**：车机本地部署轻量级向量库（如 SQLite-VSS）和 Small Language Model (SLM)。虽然对话流畅度不如云端，但能保证核心故障查询功能的可用性。

---

## 8. 本章小结

本章构建了一个适应车载严苛环境的 RAG 系统。其核心差异点在于：
1.  **元数据为王**：通过 VIN 码和配置过滤，解决“通用回答”的危害。
2.  **深度解析**：保留表格和警告框的语义结构。
3.  **口语化输出**：适应 Realtime API，区分“听的内容”和“看的内容”。
4.  **安全围栏**：无证据不回答，低置信度即拒答。

---

## 9. 练习题 (Exercises)

### 基础题

<details>
<summary><strong>Q1: 为什么在车载 RAG 中，重排（Reranking）步骤几乎是必须的？</strong></summary>

> **Hint**: 向量检索（Embedding）通常基于语义相似，而不是逻辑正确。
>
> **答案**: 
> 向量检索只能找到“语义相关”的内容。例如用户问“后视镜加热”，向量检索可能召回“后视镜调节”、“后视镜折叠”和“后镜加热”。如果不进行重排，模型可能会被混杂的上下文干扰。Cross-Encoder 重排模型可以精确计算 Query 和 Document 的相关性得分，将真正的“加热说明”排在第一位，并剔除得分低的干扰项，从而大幅减少幻觉风险。
</details>

<details>
<summary><strong>Q2: 描述“父子索引（Parent-Child Indexing）”如何解决检索粒度与上下文完整性之间的矛盾。</strong></summary>

> **Hint**: 容易被搜到的块不一定容易被读懂。
>
> **答案**: 
> 矛盾在于：为了精准检索，我们需要很小的分块（Child，包含特定关键词）；但为了让 LLM 理解并生成连贯回答，我们需要较大的上下文（Parent，包含前后文、警告、前置条件）。
> 解决方案：索引时对 Child 建立向量索引；检索时，命中 Child 后，取出其对应的 Parent 块喂给 LLM。这样既保证了召回率，又保证了生成的准确性和完整性。
</details>

<details>
<summary><strong>Q3: 针对表格数据（例如“轮胎气压表”），最推荐的预处理方式是什么？</strong></summary>

> **Hint**: 直接转成纯文本通常会丢失行列对应关系。
>
> **答案**: 
> 最推荐的方式是将表格转换为 **Markdown 格式** 或 **JSON 对象**，并保留表头信息。对于复杂的跨页表格，需要进行合并处理。如果表格极其复杂，可以生成表格的文本摘要（Summary）用于检索，同时保留原始结构数据用于生成。
</details>

<details>
<summary><strong>Q4: Realtime API 是流式的，如何在 RAG 检索耗时较长（例如 2秒）时保持良好的用户体验？</strong></summary>

> **Hint**: 避免“死寂”。
>
> **答案**: 
> 1. **注入填充语 (Filler Words)**：Agent 在决定调用搜索工具时，先立即下发一段音频（如“我查一下手册…”）。
> 2. **非阻塞式思考**：如果支持，播放填充语的同时后台并行检索。
> 3. **视觉反馈**：在车机屏幕上显示加载动画或“正在搜索知识库”的状态。
</details>

### 挑战题 (开放性思考)

<details>
<summary><strong>Q5: 设计一个“主动式 RAG”场景。当车辆发生故障码 P0300（多缸失火）时，系统如何在用户开口前做好准备？</strong></summary>

> **Hint**: 结合车端信号与云端缓存。
>
> **答案**: 
> 1. **信号触发**：车端网关监测到 DTC P0300 上报。
> 2. **预取 (Prefetch)**：车端 Agent 或云端服务立即以 "P0300" 和 "Engine Misfire" 为 Key，触发 RAG 检索。
> 3. **缓存热加载**：将检索到的解释、危害等级、建议措施加载到当前会话的 Context 缓存中。
> 4. **主动交互**：当用户询问“车怎么抖了”或“那个灯亮了”时，LLM 无需等待检索，直接利用缓存信息秒回，甚至主动发起对话：“我检测到发动机有异常震动（P0300），建议您...”
</details>

<details>
<summary><strong>Q6: 假设用户驾驶的是一辆 10 年车龄的老车，手册早已更新多次。如何确保 RAG 回答的是当年那个版本的知识？</strong></summary>

> **Hint**: 数据版本控制与车辆档案。
>
> **答案**: 
> 1. **知识库版本化**：在数据入库时，不只是覆盖旧数据，而是建立多版本索引（Versioned Index）。每个 Chunk 必须包含 `effective_date_start` 和 `end`，或者关联特定的 `model_year`。
> 2. **车辆画像 (Vehicle Profile)**：会话开始时，注入车辆的 VIN 和出厂日期。
> 3. **过滤条件**：检索时强制带上 `filter: model_year == 2015`。严格禁止跨年代检索，因为老车的物理按键操作可能在新车上变成了屏幕菜单操作。
</details>

---

## 10. 常见陷阱与错误 (Gotchas)

### 10.1 陷阱：PDF 解析丢失空间语义
*   **错误**：直接使用 PyPDF2 等库提取文本。
*   **后果**：多栏排版（Multi-column layout）被按行读取，导致左栏的文字和右栏的文字混杂在一起，逻辑完全错乱。
*   **调试技巧**：使支持 Layout Analysis 的工具（如 Microsoft Azure Document Intelligence, Amazon Textract 或开源的 LayoutLM），确保按“阅读顺序（Reading Order）”而非“物理行”提取文本。

### 10.2 陷阱：过度依赖向量检索匹配专有名词
*   **错误**：仅使用 Dense Vector Search 搜索“ISOFIX”。
*   **后果**：向量模型可能认为“儿童座椅接口”与“ISOFIX”相似，但也可能因为训练数据原因，匹配到“安全带”等宽泛概念，而忽略了包含精准 "ISOFIX" 字符串但语义描述较少的文档块。
*   **调试技巧**：必须实现混合检索（Hybrid Search）。对于缩写、代码（DTC codes）、专有名词，BM25/Keyword Search 的权重应动态调高。

### 10.3 陷阱：忽略否定词
*   **错误**：用户问“能不能用拖车绳拖拽？”，文档写“**切勿**使用拖车绳拖拽”。
*   **后果**：由于语义相似度高（都包含“拖车绳”、“拖拽”），该块被召回。如果 LLM 只是摘抄而忽略了“切勿”，会造成严重事故。
*   **调试技巧**：
    1.  使用高质量的 Reranker，它对否定逻辑更敏感。
    2.  在 System Prompt 中强调：“特别注意警告（Warning）和禁止（Do not）标志，必须明确告知用户潜在风险。”

### 10.4 陷阱：多模态“张冠李戴”
*   **错误**：用户指着中控屏上的“空调按钮”，但摄像头视野较广，同时也拍到了“危险警报灯按钮”。
*   **后果**：系统错误地解释了危险警报灯。
*   **调试技巧**：
    1.  利用 **Screen Reading (Accessibility Tree)** 作为辅助，获取当前屏幕焦点的元素文本，比纯视觉更准。
    2.  如果是物理按键，增加交互确认：“您是指中间红色的那个三角形按钮吗？”
