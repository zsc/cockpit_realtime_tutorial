# Chapter 9｜GUI Agent：车载 App 自动化（点餐/购物/预约）

## 9.1 开篇与目标

在智能座舱生态中，应用的增长速度远超 API 接口的标准化速度。车企无法与成千上万个 App（外卖、停车、充电、票务）逐一对接后端 API。**GUI Agent（图形界面智能体）** 旨在解决这一痛点：它赋予语音助手像人类一样“看懂屏幕”并“操作屏幕”的能力，从而实现对任意安卓 App 的零接口控制。

本章重点设计一个基于 **OpenAI Realtime API (多模态理解)** 和 **Agents SDK (任务编排)** 的自动化系统，实现从语音指令到 App UI 操作的闭环。

**本章学习目标：**
1.  **架构设计**：掌握感知（Perception）、规划（Planning）、执行（Action）三层解耦架构。
2.  **混合感知**：深度理解 UI 树（Accessibility Tree）与视觉（Vision）的融合策略（SoM）。
3.  **鲁棒性工程**：设计针对 App 弹窗、加载慢、布局漂移的自动恢复机制。
4.  **安全人机协同**：界定 AI 操作与人类接管的边界（尤其是支付与驾驶安全）。

---

## 9.2 系统核心架构 (System Architecture)

GUI Agent 并非简单的脚本录制，而是一个具备动态推理能力的智能系统。它运行在车机 Android 系统之上，作为一个特权服务存在。

### 9.2.1 逻辑架构图

```ascii
+-----------------------------------------------------------------------+
|                       Agent Orchestrator (Cloud/Edge)                 |
|   (OpenAI Agents SDK: State Management, History, Reasoning)           |
+-----------------------------------------------------------------------+
        ^                       | (Plan: "Open Starbucks -> Click Buy")
        | (Observation)         v
+-------+-------+       +-------+-------+       +-------+-------+
|  Perception   |       |   Reasoning   |       |   Executor    |
|   (The Eye)   |       |  (The Brain)  |       |  (The Hand)   |
+-------+-------+       +-------+-------+       +-------+-------+
        ^                       ^                       |
        | Raw Data              | Decision              | Actions
        |                       v                       v
+-------+-----------------------+-----------------------+-------+
|                 Vehicle Android OS Framework                  |
+---------------------------------------------------------------+
|  AccessibilityService  |  InputManager  |  ActivityManager    |
|  (UI Tree / Events)    |  (Inject Event)|  (Package State)    |
+------------------------+----------------+---------------------+
|        Target App (e.g., Meituan, Starbucks, Maps)            |
+---------------------------------------------------------------+
```

### 9.2.2 核心组件职责

1.  **Perception Module (感知模组)**
    *   **UI Dump**: 通过 `AccessibilityInteractionController` 获取当前窗口的 UI 节点树（XML/JSON）。
    *   **Screen Cap**: 获取当前帧的像素截图（Bitmap）。
    *   **Pre-processor**: 执行敏感信息遮盖（Masking）、UI 树精简（去除无效节点）、视觉标记（Set-of-Mark）。

2.  **Reasoning Engine (推理引擎 - LLM)**
    *   基于当前的 Observation（截图 + UI 树）和 Memory（用户刚才说的话），生成下一步的 **ToolCall**。
    *   负责处理“找不到元素”、“页面加载中”等逻辑分支。

3.  **Action Executor (执行器)**
    *   将语义化的 ToolCall（如 `tap(element_id="btn_submit")`）转换为底层的 Android 输入事件（`MotionEvent`）。
    *   负责坐标映射、手势模拟（Swipe/Scroll）和文本输入（IME）。

---

## 9.3 混合感知策略 (Hybrid Perception)

单纯依赖视觉会面临高延迟和幻觉问题，单纯依赖 UI 树会面临 WebView/游戏引擎无法解析的问题。我们采用 **"UI 树为主，视觉为辅"** 的策略。

### 9.3.1 Set-of-Mark (SoM) 增强技术

为了让大模型准确理解屏幕位置，我们不直接发送原始截图，而是先在截图上“画框”。

**处理流程：**
1.  **提取**：从 UI 树中过滤出所有 `clickable=true` 或 `focusable=true` 的节点。
2.  **绘制**：在截图上对应坐标绘制半透明 Bounding Box，并标注数字 ID（1, 2, 3...）。
3.  **Prompt**：提示词中不再问“点击什么按钮”，而是问“点击哪个 ID”。

**优势**：
*   **Token 压缩**：不需要模型去猜测坐标，只需输出 ID。
*   **抗幻觉**：如果 ID 不存在，模型无法输出，避免了点击空白区域。

### 9.3.2 感知降级与回退 (Fallback Strategy)

| 场景 | UI 树状态 | 视觉状态 | 策略 |
| :--- | :--- | :--- | :--- |
| **原生 App** | 完整可用 | 辅助校验 | **优先使用 UI 树**。直接通过 ID 或 Text 查找控件，速度最快 |
| **Webview/H5** | 仅有容器节点 | 内容丰富 | **切换为纯视觉模式**。OCR 识别文字 + 图像理解图标，估算相对坐标。 |
| **游戏/地图** | 空白或乱码 | 内容丰富 | **纯视觉模式**。依赖图像识别。 |
| **黑屏/加载** | 包含 Loading | 包含 Loading | **等待模式**。暂停 1-2秒，重试感知，不消耗 LLM 推理次数。 |

---

## 9.4 任务编排与对话设计

### 9.4.1 显式规划与槽位填充 (Slot Filling)

GUI 操作极其昂贵（时间长、屏幕闪烁）。**Rule-of-Thumb**：绝对不要在 App 里一边问一边点。

**错误示范（交互太碎）：**
> User: "点杯咖啡" -> Agent 打开 App -> Agent: "哪家？" -> User: "星巴克" -> Agent 搜星巴克 -> Agent: "喝啥？" ...

**正确设计（One-Shot Execution）：**
1.  **Voice Interaction Layer**: 在对话层先将意图参数补齐。
    *   Agent: "好的，点咖啡。请问是星巴克还是瑞幸？要送到现在的定位吗？"
    *   User: "星克，冰美式，送到公司。"
2.  **Planning Layer**: 生成完整任务链。
    *   `Goal: Order "Iced Americano" from "Starbucks", Address="Office"`
3.  **Execution Layer**: Agent 沉默执行，只在关键节点播报。

### 9.4.2 运行时的用户反馈

在长达 10-30 秒的操作中，用户需要知道 Agent 是“死机了”还是“在思考”。

*   **UI 覆盖层 (Overlay)**: 在屏幕上方显示一个浮动的 Agent 状态条。
    *   状态：`正在搜索店铺...` -> `正在选购商品...` -> `准备结算...`
*   **语音伴随 (Voice Check-in)**:
    *   每执行 3-4 个动作，或遇到耗时加载时，通过 Realtime API 插入一句短语：“正在打开菜单，稍等”、“网速有点慢，正在加载”。

---

## 9.5 动作空间与执行器设计 (Action Space)

执行器不仅是点击，还需要处理复杂的交互逻辑。

### 9.5.1 定义 Tool 接口

```typescript
// 定义给 LLM 调用的工具集
type GUI_Tools = {
  // 基础操作
  tap: (element_id?: number, x?: number, y?: number, description?: string) => Result;
  scroll: (direction: "up"|"down"|"left"|"right", distance: "screen"|"half") => Result;
  type_text: (text: string, clear_first: boolean) => Result;
  
  // 导航操作
  go_back: () => Result;
  go_home: () => Result;
  
  // 流程控制
  wait: (seconds: number, reason: string) => Result; // 显式等待加载
  finish_task: (outcome: string) => Result; // 任务完成，交还控制权
  ask_user: (question: string) => Result; // 遇到无法决断的问题
};
```

### 9.5.2 复杂动作的原子化

*   **滑动查找 (Scroll-to-find)**:
    *   LLM 往往不能直接看到屏幕下方的元素。
    *   设计一个复合动作 `find_text_and_tap(text)`：执行器内部自动循环 `[检测 -> 未找到 -> 滑动 -> 检测]`，直到找到目标或超时，无需 LLM 参与每一帧的决策。
*   **输入法处理**:
    *   不要模拟点击键盘上的字母（容易出错）。
    *   直接调用 Android `Instrumentation.sendStringSync` 或通过 ADB 注入文本事件，绕过软键盘 UI。

---

## 9.6 鲁棒性与异常恢复 (Robustness)

GUI 自动化最大的挑战是环境的不确定性（AB 实验、广告弹窗、版本更新）。

### 9.6.1 常见异常模式与对策

1.  **幽灵弹窗 (Pop-ups)**:
    *   *现象*：刚要点击“下单”，突然弹出一个“领红包”遮住了按钮。
    *   *对策*：**视觉验证**。每次点击前，检查目标区域是否有 Z-index 更高的遮挡物。如果点击后页面未发生预期跳转（通过 UI 树 Hash 或截图相似度判断），判定为被拦截，触发“寻找关闭按钮”子任务。
2.  **元素ID变更**:
    *   *现象*：App 更新后，`resource-id` 变了。
    *   *对策*：**语义定位**。优先使用 `text` 内容（如 "去结算"）或 `content-desc` 进行定位，而不是死板的 ID 或坐标。
3.  **无限加载 (Spinner)**:
    *   *现象*：网络差，转圈超过 10 秒。
    *   *对策*：设置**超时熔断**。如果页面 UI 树在 5 秒内只有 Loading 节点，Agent 语音播报：“网络信号不佳，请稍后再试”，并主动终止任务。

### 9.6.2 自我反思循环 (Self-Correction Loop)

利用 Agents SDK 的循环特性，在每次 Action 后加入 Observation 检查：

```python
# 伪代码逻辑
while not task_done:
    obs = get_screen_state()
    plan = agent.think(obs, goal)
    
    if plan.action == "tap(checkout)":
        execute(plan.action)
        sleep(1.0)
        
        # 校验步：有没有进入下一个页面？
        new_obs = get_screen_state()
        if is_same_page(obs, new_obs):
            # 没变，说明点击失败或卡顿
            agent.memory.add("Attempted to tap checkout but screen didn't change. Is there a popup?")
            continue # 重新思考
```

---

## 9.7 安全、隐私与合规 (Safety & Compliance)

### 9.7.1 支付红线 (The Payment Red Line)

*   **原则**：Agent **严禁**触碰任何支付密码键盘、指纹确认框或面容识别确认框。
*   **实现**：
    1.  **关键词检测**：感知层检测到页面包含“请输入密码”、“指纹支付”、“FaceID”等字样。
    2.  **强制中断**：系统底层拦截 Agent 的点击信号。
    3.  **交接**：Agent 语音播报：“已帮您下好订单，请您亲自确认支付。”随后 Agent 退出前台，让用户接管。

### 9.7.2 驾驶分心管理 (Driver Distraction)

*   **行驶中策略 (Speed > 0)**:
    *   **静默操作**：Agent 在操作 App 时，将屏幕亮度压暗或显示遮罩（"AI 正在后台为您处理..."），避免界面快速跳转分散驾驶员注意力。
    *   **结果摘要**：仅在最终确认页或结果页恢复显示。
*   **高危动作拦截**:
    *   DMS 检测到驾驶员长视线偏离（看屏幕 > 2秒），立即暂停 GUI 自动化并语音提示：“请看路，操作已暂停。”

---

## 9.8 性能优化 (Performance)

目标：每一步操作延迟 < 1.5秒。

1.  **截图压缩**:
    *   不要传原图（如 2K 屏）。下采样至 720p 或 1080p，转为灰度图（如果颜色不重要），压缩为 JPEG (Quality 70)。
2.  **UI 树缓存与去噪**:
    *   过滤掉所有 `visible=false`、宽高为 0 或位于屏幕外的节点，减少 LLM 上下文长度。
3.  **并行处理**:
    *   截图线程与 UI 树抓取线程并行执行，最后通过时间戳对齐。

---

## 9.9 本章小结

*   **架构**：GUI Agent 是连接语音与封闭 App 生态的桥梁，采用“感知-规划-执行”闭环。
*   **核心技术**：SoM（标记集）技术解决了大模型“看不准”坐标的问题；混合感知解决了 UI 树缺失的问题。
*   **体验设计**：通过“槽位填充”减少交互轮数，通过“静默执行”减少驾驶干扰。
*   **底线**：支付环节必须人机解耦（Handoff），确保资金安全。

---

## 9.10 练习题

### 基础题

<details>
<summary><strong>Q1: 为什么在车载 GUI Agent 中，我们要对 UI 树（Accessibility Tree）进行预处理和精简？列举至少三个理由。</strong></summary>

> **答案：**
> 1.  **降低 Token 消耗**：原生 UI 树包含大量布局嵌套（FrameLayout, LinearLayout 等无意义容器），直接输入 LLM 会消耗大量 Token 并增加成本。
> 2.  **减少延迟**：Context 越长，LLM 首字生成（TTFT）越慢。精简后的树能显著提升响应速度。
> 3.  **提高准确率**：过多的噪声节点（如隐藏的菜单、背景图层）会干扰模型注意力，导致幻觉或选错元素。我们只关心 `clickable`, `text`, `content-desc` 等关键属性。
</details>

<details>
<summary><strong>Q2: 解释什么是“SoM (Set-of-Mark)”技术，以及它是如何解决 GUI 点击精度问题的？</strong></summary>

> **答案：**
> *   **定义**：SoM 是一种视觉提示技术，它利用 UI 树的坐标信息，在发送给视觉模型的截图上预先绘制带编号的边框（Bounding Box）。
> *   **原理**：它将回归问题”（预测 X,Y 连续坐标，难度大，易飘移）转化为“分类问题”（预测 ID 编号，难度小，准确率高）。模型只需要输出“点击 ID 5”，执行器再去查表找到 ID 5 的中心坐标进行点击。
</details>

<details>
<summary><strong>Q3: 当 Agent 正在操作 App 时，用户突然打断说“算了，不需要了”，系统应如何响应？</strong></summary>

> **答案：**
> 1.  **立即停止**：Realtime API 接收到 `input_audio_buffer.speech_started` 事件，立即发送信号终止当前的 Agent 执行线程。
> 2.  **状态回滚（Best Effort）**：如果可能，执行 `go_home()` 返回桌面，或者不进行任何操作保持现状。
> 3.  **确认反馈**：语音回复“好的，已停止操作”。
> 4.  **避免副作用**：确保不会因为延迟而执行之前排队的“点击下单”指令（需要清空 Action Queue）。
</details>

### 挑战题

<details>
<summary><strong>Q4: 场景设计：用户说“帮我这个订单分享给微信好友”。GUI Agent 需要跨两个 App（外卖 App -> 微信）操作。请设计这一流程，并指出最大的技术难点。</strong></summary>

> **答案：**
> *   **流程**：
>     1.  在外卖 App 点击“分享”按钮。
>     2.  系统弹出的 ShareSheet（系统级或应用级）中找到“微信”图标并点击。
>     3.  **难点跳转**：此时前台应用切换为微信。Agent 需要感知到包名变更为 `com.tencent.mm`。
>     4.  在微信“最近聊天”列表中寻找目标好友，或点击“创建新聊天”。
>     5.  点击“发送”。
> *   **最大技术难点**：
>     *   **上下文丢失**：跨 App 跳转时，UI 风格突变，Agent 需要快速适应新环境。
>     *   **隐私界限**：进入微信后，屏幕上全是私人聊天记录。Agent 必须严格执行隐私过滤（对非目标好友的头像/名字进行模糊处理），且只能操作用户指定的好友，不能随意浏览。
>     *   **启动延迟**：微信启动可能需要数秒，Agent 需要有智能等待机制。
</details>

<details>
<summary><strong>Q5: 针对“滑动查找”场景（例如在长列表中找‘麦当劳’），请设计一个具体的算法逻辑，确保 Agent 不会陷入死循环（上下反复滑）。</strong></summary>

> **答案：**
> **算法逻辑：**
> 1.  **初始化**：设置 `max_swipes = 5`，`visited_content_hashes = Set()`。
> 2.  **循环**：
>    *   **Step 1 检查**：获取当前屏幕所有文本。如果包含“麦当劳”，返回坐标，结束。
>    *   **Step 2 记录**：计算当前屏幕内容的哈希值（或提取关键文本集合）。如果该哈希值已存在于 `visited_content_hashes`，说明到底了或者没滑倒，**终止并报错**（防止死循环）。加入哈希表。
>    *   **Step 3 动作**：执行 `swipe_up()`（内容上移，浏览下方）。
>    *   **Step 4 计数**：`swipes++`。如果 `swipes > max_swipes`，语音播报“找不”，结束。
> 3.  **优化**：如果列表很长，可以使用 OCR 辅助判断是否有 ScrollBar 到底的视觉特征。
</details>

---

## 9.11 常见陷阱与错误 (Gotchas)

### 1. 坐标系地狱 (Coordinate Hell)
*   **问题**：UI 树返回的坐标是基于 App 窗口的（Window Coordinates），而 ADB `input tap` 需要的是屏幕绝对坐标（Screen Coordinates）。
*   **陷阱**：在分屏模式、画中画模式或有状态栏/导航栏的情况下，Y 轴往往有几十像素的偏差，导致点不准。
*   **调试技巧**：开启 Android 开发者选项中的 "Pointer Location"（指针位置），直观看到系统的点击位置与 Agent 意图位置的偏差，计算 Offset 进行补偿。

### 2. 软键盘遮挡 (IME Obstruction)
*   **问题**：Agent 点击输入框后，软键盘弹出，遮住了原本位于底部的“确定”按钮。Agent 继续根据旧截图点击底部，结果点到了键盘上的某个键。
*   **调试技巧**：
    *   **先收起**输入完文本后，习惯性发送 `Back` 键或 `HideKeyboard` 指令。
    *   **使用 Action 键**：直接发送 `Enter` / `Search` 的 KeyCode，触发表单提交，完全绕过点击按钮。

### 3. DOM 树不仅是树 (Virtual Nodes)
*   **问题**：Flutter 或 Unity 应用（如车机版 Bilibili 游戏中心）通常只暴露一个大的 View，没有 Accessibility 节点。
*   **陷阱**：依赖 UI 树的 Agent 在这些 App 里完全“失明”。
*   **调试技巧**：必须实现**自动降级**机制。当检测到根节点下无子节点，但截图非黑屏时，强制切换到纯视觉（Vision-only）模式。

### 4. 动态广告的干扰
*   **问题**：开屏广告倒计时、应用内浮层广告。
*   **陷阱**：Agent 试图点击广告上的“跳过”，但因为倒计时结束广告自动消失，导致点击穿透到了下层的内容，触发了意外操作。
*   **调试技巧**：在点击前进行二次 `Check`，或者设计更保守的等待策略（Wait for settle）。
