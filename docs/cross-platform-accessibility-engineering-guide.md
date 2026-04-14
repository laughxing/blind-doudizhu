# 跨平台无障碍游戏工程与架构指南

> 本文档面向客户端与前端工程团队、测试团队，重点阐述跨平台游戏（微信小游戏、iOS、Android、Web 等）在无障碍落地时的架构设计、模块拆分、平台适配与测试验收方案。

## 一、工程落地的核心判断

### 1. 游戏主体往往是引擎驱动，而不是原生页面驱动

很多跨平台游戏运行在 `Cocos`、`Unity`、自绘 `View`、`Canvas/WebGL` 等环境中。对这类项目来说：
- 真正的交互状态、焦点顺序、播报时机，通常由游戏逻辑层控制
- 原生页面或标准 DOM 往往只承载外壳、设置或部分标准 UI
- 不能假设平台天然知道“当前什么是重点对象、什么是下一步动作”

### 2. 语音播报更适合采用“游戏逻辑 + 平台语音能力”的组合模式

在大多数可落地的方案里：
- 发声能力来自平台原生的读屏、TTS 或 Web 语音接口
- 播报什么、何时播报、是否打断、播报完成后是否推进状态，由游戏逻辑统一管理

### 3. 跨平台架构必须把“语义层”放在平台层之上

推荐把系统无障碍 API 当作末端输出通道，而不是业务逻辑本身：
- 业务状态层负责产出真实游戏状态
- 无障碍语义层负责把状态转成可读、可导航、可执行的节点
- 平台适配层负责把这些节点映射到 `VoiceOver / TalkBack / Web ARIA / 自定义 TTS`

---

## 二、推荐的统一架构

推荐采用如下分层：

```text
游戏状态层
  -> 无障碍语义层
    -> 焦点与导航层
      -> 播报调度层
        -> 平台适配层
          -> 系统读屏 / 平台 TTS / 音效 / 震动
```

### 1. 游戏状态层
- 提供真实的游戏数据，不夹杂平台逻辑。
- 暴露当前回合、阶段、对象状态、可执行动作、倒计时、异常状态。

### 2. 无障碍语义层
- 把原始游戏状态转换为“用户能理解的节点”。
- 决定哪些视觉细节保留，哪些压缩掉。
- 生成摘要、详情、动作、状态标签。

### 3. 焦点与导航层
- 决定节点遍历顺序。
- 支持分区跳转。
- 处理“摘要 -> 明细 -> 动作”的浏览路径。

### 4. 播报调度层
- 合并、去重、延迟、打断、排队播报。
- 管理语音与音效、震动之间的优先级。
- 提供 `utteranceId` 或等价机制，让播报结束后可回调业务层。

### 5. 平台适配层
- 把统一语义映射到每个平台最合适的通道。
- 决定走系统读屏、原生 TTS、Web Speech、音效还是震动。
- 处理平台限制和回调差异。

---

## 三、推荐的工程实施方案

本文推荐采用一套统一方案：
> **共享无障碍内核 + 平台薄适配层 + 原生标准 UI 复用**

这套方案的关键思想是：
- 游戏只维护一份“可访问语义树”
- 平台只负责“如何暴露”和“如何播报”
- 标准 UI 尽量继续使用平台原生可访问能力
- 主玩法场景由共享层自己管理焦点、动作和摘要

### 1. 模块拆分建议

| 模块 | 归属层 | 主要职责 |
| ------ | ------ | ------ |
| `AccessibilityRuntime` | 共享层 | 无障碍总入口，协调各子模块 |
| `SemanticBuilder` | 共享层 | 把游戏状态转换为可访问节点与摘要 |
| `FocusEngine` | 共享层 | 管理焦点顺序、区域跳转、上下文恢复 |
| `ActionRouter` | 共享层 | 把“激活/切换/确认/返回”等动作映射到游戏意图 |
| `AnnouncementQueue` | 共享层 | 统一排队、打断、节流、去重、回调 |
| `AccessibilityProfile` | 共享层 | 管理语速、提示级别、自动摘要、重听策略 |
| `PlatformAdapter` | 平台层 | 暴露宿主能力，接收共享层调用 |
| `NativeBridge` | 平台层 | 处理引擎与原生之间的桥接细节 |

### 2. 核心接口建议

下面这组接口是跨平台项目里最值得先定下来的部分：

```ts
type NodeRole = "summary" | "button" | "toggle" | "group" | "item" | "action" | "status" | "log";
type AnnouncementChannel = "system_screen_reader" | "custom_tts" | "sound" | "haptic";

interface SemanticNode {
  id: string;
  group: string;
  role: NodeRole;
  label: string;
  value?: string;
  hint?: string;
  enabled: boolean;
  selected?: boolean;
  priority: number;
  actions: ActionDescriptor[];
}

interface ActionDescriptor {
  id: string;
  label: string;
  kind: "activate" | "next" | "prev" | "confirm" | "cancel" | "open" | "back";
  payload?: Record<string, unknown>;
}

interface Announcement {
  id: string;
  text: string;
  channel: AnnouncementChannel;
  priority: number;
  interruptible: boolean;
  dedupeKey?: string;
  utteranceId?: string;
}

interface PlatformAdapter {
  isScreenReaderEnabled(): boolean;
  renderSemanticTree(nodes: SemanticNode[]): void;
  focusNode(nodeId: string): void;
  announce(input: Announcement): void;
  stopAnnouncement(channel?: AnnouncementChannel): void;
  vibrate(pattern: string): void;
  playCue(name: string): void;
  onUtteranceEvent(
    callback: (event: "start" | "done" | "cancel" | "error", utteranceId: string) => void
  ): void;
}
```

### 3. 谁拥有语音通道

推荐把“语音通道”的所有权明确交给 `AnnouncementQueue`，并遵守以下规则：
- **不要重复播报同一条语义**：如果当前信息已经由系统读屏准确朗读，就不要再由自定义 TTS 重复播报一次。
- **用系统读屏承载标准 UI**：菜单、设置、登录、弹窗优先交给平台。
- **用游戏播报承载动态游戏语义**：回合切换、战斗结果、棋步解析走游戏自己播报。
- **所有平台都要有“播报完成回调”**：否则游戏逻辑容易和语音节奏脱节。

---

## 四、平台能力映射与落地建议

### 1. 微信小游戏
- 把小游戏画布视为“语义不透明”的渲染层。
- 在游戏内部自己维护焦点树、摘要文本、动作列表和播报节奏。
- 重点依赖：明确的焦点顺序、高质量音效反馈、震动反馈、简短但结构化的语音资产。
- 谨慎使用：依赖宿主系统读屏理解画布内对象。

### 2. iOS
- 菜单、设置等尽量使用可访问原生 UI，用 `VoiceOver` 负责标准 UI 的浏览与激活。
- 主玩法运行在 Metal/Unity/Cocos 场景中时，需要自己构建可访问语义节点，并映射到可访问元素或辅助导航模型。
- 必须避免自定义 TTS 与 `VoiceOver` 形成双播。

### 3. Android
- 游戏或引擎层维护无障碍逻辑，原生层通过 `TextToSpeech` 做播报，播报完成后回调共享逻辑层继续推进状态。
- `TextToSpeech` 适合做：回合切换、棋步结果、牌型摘要、错误反馈。
- 不推荐为了实现“应用内无障碍”去额外开发 `AccessibilityService`。

### 4. Web
- Web 的最大陷阱：纯 Canvas 不等于可访问。
- 菜单、设置优先使用真实 DOM。
- 对核心互动对象维护一个 DOM 镜像或语义摘要面板。
- `aria-live`：`polite` 用于普通状态更新，`assertive` 仅用于高优先级必须打断的提醒。

---

## 五、工程实现细节建议

### 1. 不要让业务代码直接调用平台 TTS
统一由 `AccessibilityManager` 决定是否播报、如何播报、走哪个平台接口。否则会到处散落适配代码，无法做去重、节流、优先级控制。

### 2. 维护播报优先级与节流规则
- 同一类信息短时间内去重。
- 高优先级状态可打断低优先级补充说明。
- 同一帧或同一状态变更中，先合并再播报。

### 3. 把“已选中”“不可用”“危险”做成统一状态词表
不要在各处随意拼文案，建议维护统一词表和模板。这能保证多平台之间的语言体验一致。

### 4. 所有关键动作都要有完成态反馈
完成态反馈建议至少包含两层：一层语义确认（如“已出牌：对 K”），一层辅助反馈（如短震或确认音）。

---

## 六、测试与验收、团队协作

### 1. 必测组合
- 开启系统读屏 vs 关闭系统读屏仅保留游戏辅助播报
- 慢语速与快语速
- 高频状态变化与网络波动
- 后台切换或系统中断

### 2. 关键验收问题
- 用户能否在 5 到 10 秒内知道“现在轮到谁、我能做什么”？
- 用户是否必须依赖视觉坐标才能完成关键操作？
- 同一条信息是否会被重复播报？
- 页面切换后焦点是否会落到合理位置？（返回按钮陷阱）

### 3. 团队协作分工
- **产品**：定义无障碍范围，标出必须主动播报的关键时刻。
- **设计**：输出焦点顺序、摘要策略、播报模板、反馈规范。
- **工程**：建立统一语义层和平台适配层，建立播报优先级、去重、回调闭环。
- **测试**：建立真实辅助技术下的回归用例，验证系统读屏与游戏自定义播报是否互相打架。