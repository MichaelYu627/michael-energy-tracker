# 快速捕获 + 批量同步重设计

**日期**：2026-04-24
**作者**：Michael + Claude（brainstorming）
**状态**：设计定稿，待转实施计划

---

## 1. 背景与问题

当前 app 采用"每日一张大表单 + 零散追加按钮"的结构，假设用户每天坐下来填一次。实际使用模式是：

- **随发生随记录**：早上起床记睡眠、中午吃完饭记餐食、下午小睡记补能、睡前记复盘
- **一天打开 5-8 次**，每次只想记当下那一件事
- **Obsidian 是决策中枢**：里面的 Claude 读全部历史，相当于第二大脑；数据必须能推过去

当前结构造成的体验问题：

1. 打开 app 看到 20+ 字段的大表单，只想记一件事时心智负担过重
2. "主记录 vs 追加"两套心智模型并存，用户困惑
3. 每次追加都会立刻触发 Obsidian 跳转，浏览器被抢焦 N 次
4. 折叠/展开状态隐藏、没反馈；编辑历史记录时表单被折叠导致"点了没反应"
5. 自定义协议跳转 + setTimeout 的技术实现脆弱

## 2. 目标

- 打开 app 只看到当下要记的那件事，字段不超过 4 个
- 保存是纯本地、毫秒完成、永远响应
- Obsidian 同步和记录解耦：默认攒着、坐到电脑前一键推；偶尔有需要可单条立刻推
- 所有现有数据零丢失、向后兼容
- 统计/图表/电池逻辑保持不变

## 3. 非目标（明确不做）

- 不引入构建工具（保持单文件 HTML）
- 不接 Obsidian Local REST API（继续用 `obsidian://` URL scheme）
- 不清理废弃字段（`deepWorkTasks, adminTasks, morningEnergy, afternoonEnergy, eveningEnergy`）—— 数据保留、UI 隐藏，留待未来
- 不做离线 PWA / Service Worker
- 不改变"一天一个 Obsidian 笔记"的文件组织

## 4. 信息架构：7 种事件类型

| # | 事件 | 频率 | 数据行为 | 字段 |
|---|---|---|---|---|
| 1 | 😴 睡眠 | 一天一次 | 覆盖 | `sleepScore, bedtime, wakeTime, wakeups` |
| 2 | 🍱 餐食 | 一天多次 | 追加到 `mealItems[]` | `time, type, food, cooked, healthy, ateOut` |
| 3 | 💪 训练 | 隔天一次 | 覆盖整组 | `workoutItems[], exercised=true` |
| 4 | 🛋 补能 | 一天多次 | 追加到 `recoveryItems[]` | `time, type, boost, note` |
| 5 | ⚖️ 体重 | 一天一次 | 覆盖 | `weight` |
| 6 | 🌙 复盘 | 一天一次 | 覆盖 | `mood, energy, dailyStory` |
| 7 | 💭 心情（新增）| 一天多次 | 追加到 `moodLogs[]`（**新字段**）| `time, tag, note` |

**划分原则**：一天只有一个真实值的 → 覆盖；可能多次发生的 → 追加。

## 5. 页面布局

```
┌──────────────────────────────────────┐
│ 🔋 能量电池 85% · 绿色区间           │  ← Hero（保留）
├──────────────────────────────────────┤
│ ⏳ 待同步到 Obsidian：3 条 [📤 推]   │  ← 橙色胶囊（新增 · 仅队列非空时显示）
├──────────────────────────────────────┤
│ 📰 Jarvis 每日简报 ▸                  │  ← 保留
├──────────────────────────────────────┤
│ 快速捕获：                            │
│ [😴][🍱▾][💪][🛋][⚖️][🌙][💭]        │  ← 7 个胶囊按钮（新增）
│                                       │
│ ┌─ 当前展开：🍱 记一餐 ──────────┐  │
│ │ 时间 12:15  餐次 午餐 ▾        │  │
│ │ 吃了什么：[文本框]             │  │  ← 事件面板（新增）
│ │ ☑ 自己做  ☑ 健康  ☐ 外食       │  │
│ │ [💾 保存]  ☐ 立刻推到 Obsidian  │  │
│ └─────────────────────────────────┘  │
├──────────────────────────────────────┤
│ 📅 今日时间线                         │
│ ✓ 07:15 😴 85 分 / 22:30-06:45       │  ← 已同步
│ ⏳ 07:20 ⚖️ 83.2 kg                   │  ← 待同步
│ ⏳ 12:15 🍱 午餐 牛肉青菜米饭         │
├──────────────────────────────────────┤
│ 📊 7 天 / 30 天统计 ▸                 │  ← 保留
│ 💾 最近记录 / 数据管理 ▸              │  ← 保留
└──────────────────────────────────────┘
```

### 关键交互规则

1. **快速捕获条**：同一时刻只能有一个面板展开。点另一个 → 当前收起、新的展开。
2. **保存是纯本地同步操作**：点击 → 立即写 `entries` + push `pendingSync` → 毫秒内完成 → 状态条提示"已保存"。
3. **"立刻推到 Obsidian" 勾选**：保存后该事件的 markdown 立即写到今日笔记，然后从 `pendingSync` 移除。
4. **顶部橙色胶囊**：仅当 `pendingSync` 有今日事件时显示。点胶囊 = 一键推所有今日待同步。
5. **今日时间线**：按 `time` 升序展示今日所有事件。已同步 ✓、待同步 ⏳。点一条可编辑（编辑后重新入队）。

## 6. 数据模型

### 6.1 `entries` 结构（保留）

现有 `normalize()` 全部字段保留。新增一个字段：

```js
{
  // ... 所有现有字段 ...
  moodLogs: []  // 新字段，默认空数组；每条 { time, tag, note }
}
```

`normalize()` 里加一行：`moodLogs: this.normalizeMoodLogs(raw.moodLogs || [])`，相应地增加 `normalizeMoodLogs` 辅助函数（参照现有 `normalizeMealItems`）。

### 6.2 `pendingSync` 新增 localStorage 键

**Key**：`michael-energy-tracker-pending-sync-v1`

**值**：
```js
[
  {
    id: "evt-1777002118296",      // 唯一 id（时间戳即可）
    eventType: "meal",              // 7 种之一
    date: "2026-04-24",             // 对应的日期
    time: "12:15",                  // 面板里填的事件时间
    capturedAt: 1777002118296,      // 保存时的 Date.now()
    data: {                          // 事件特有快照
      food: "牛肉 青菜 米饭",
      cooked: true,
      healthy: true,
      ateOut: false,
      type: "午餐"
    }
  }
]
```

### 6.3 模块组织

代码仍在同一个 IIFE 里，但新增 3 个模块：

- `EventModel`：每种事件类型的字段验证、数据序列化、markdown 渲染
- `SyncQueue`：管理 `pendingSync` 队列（加入、移除、按日期筛、拼 markdown）
- `PanelManager`：管理当前展开的面板、面板之间的切换

现有模块保留：`DateUtils, Analytics, EntryModel, StorageService, ChartRenderer, UI, App, Handlers`。

## 7. 同步机制（Hybrid）

### 7.1 保存流程

```
用户点 💾 保存
  ↓
校验字段（事件特有的 validate）
  ↓ 通过
Apply to entries（覆盖 or 追加）
  ↓
Push to pendingSync
  ↓
UI.render()（顶部胶囊 +1、时间线新增 ⏳ 条）
  ↓
若勾选"立刻推"：
  ↓
  构建 markdown → window.location.href = obsidian://...
  ↓
  从 pendingSync 移除该事件
  ↓
  时间线：⏳ → ✓
```

### 7.2 一键推流程

```
用户点 📤 胶囊
  ↓
pendingSync.filter(e => e.date === today)
  ↓
按 time 升序排序
  ↓
拼接 markdown（每个事件一个 H3 section）
  ↓
构建单一 obsidian:// URL
  ↓
window.location.href = url（同步）
  ↓
pendingSync 移除今日全部条目
  ↓
UI.render()（胶囊消失、时间线全部 ✓）
```

### 7.3 编辑历史事件

时间线上点某条 → 面板预填该条数据 → 用户修改 → 保存

- 在 `entries` 里原位更新
- 原事件若还在 `pendingSync`：就地更新其 `data`
- 原事件若已被推过（不在队列）：**重新入队**，顶部胶囊 +1，提示"Obsidian 那边需要更新"

**局限性**：`obsidian://` URL scheme 只支持 append 不支持 replace-section。所以如果一个已推事件被编辑后再次推送，Obsidian 笔记里会出现**两个 section**（旧的 + 新的）。对策：在重推的事件 markdown 前加标记 `### {emoji} {eventLabel} {time} （已修改 {HH:MM}）`，让 Obsidian Claude 和用户都能识别哪个是最新版本。手工清理旧 section 的责任留给用户（Obsidian 里可快速删除）。

## 8. Obsidian Markdown 格式

### 8.1 文件路径

继续沿用：`Michael&Jarvis/我的训练日志/训练日志 YYYY-MM-DD.md`

### 8.2 单事件格式模板

```markdown
### {emoji} {eventLabel} {time}
- {field1}：{value1}
- {field2}：{value2}
...
```

具体每种事件（见 §4）各自的渲染模板由 `EventModel.toMarkdown(event)` 生成。

### 8.3 URL 构造

```js
const content = events.map(EventModel.toMarkdown).join("\n\n");
const url = `obsidian://new?vault=...&file=...&append=true&content=${encodeURIComponent(content)}`;
window.location.href = url;
```

单事件推 vs 批量推使用同一条代码路径，区别只在 `events` 数组长度。

## 9. 迁移与向后兼容

### 9.1 数据迁移

- 打开升级版 app，`StorageService.loadEntries()` 读旧数据
- `normalize()` 给每条自动补 `moodLogs: []`
- 老的 `deepWorkTasks` 等字段保留在 JSON 里，UI 不展示
- **Bootstrap 逻辑**：`ensureBootstrapTodayEntry` 可以拿掉 —— 新架构下用户点了某个面板保存才会创建今日条目，首屏看到"今日暂无事件"是合理的

### 9.2 `pendingSync` 初始值

首次加载 = 空数组。历史上已录但未来得及推 Obsidian 的条目不入队（用户可通过"编辑"触发重新入队）。

### 9.3 旧入口迁移

| 旧 | 新 |
|---|---|
| 主表单 submit | 删除（按钮消失） |
| `addMealLogBtn` | 迁到 🍱 面板的保存按钮 |
| `addRecoveryLogBtn` | 迁到 🛋 面板 |
| `addNightReviewBtn` | 迁到 🌙 面板 |
| `toggleMainRecordBtn` | 删除（无折叠概念） |

## 10. 错误处理

| 场景 | 处理 |
|---|---|
| 必填字段没填 | 面板内红字，阻止保存 |
| 数值超出范围 | 同上 |
| `localStorage` 满（QuotaExceededError）| catch → 状态条红字"存储已满，请导出 JSON 后清理旧数据"→ 不入队 |
| Obsidian URL scheme 失败 | 无法探测。乐观清队列。用户发现后可通过编辑重推 |
| 同一事件保存两次（快速双击）| 按 `id` 去重（SyncQueue 入队前检查） |

## 11. 测试计划

手工测试清单（personal tool，暂不引入自动化）：

### 单事件路径
- [ ] 😴 睡眠保存 → entries 有 sleepScore、顶部胶囊 +1、时间线出现 ⏳
- [ ] 🍱 餐食连续保存 2 次 → mealItems 有 2 条、胶囊 +2
- [ ] 💪 训练保存 → 覆盖整组 workoutItems
- [ ] 🛋 补能保存 → recoveryItems 追加、energyBoost 累加
- [ ] ⚖️ 体重保存 → weight 覆盖
- [ ] 🌙 复盘保存 → mood/energy/dailyStory 覆盖
- [ ] 💭 心情保存 → moodLogs 追加

### 同步路径
- [ ] 勾选"立刻推" 保存 → Obsidian 打开 + 队列不增
- [ ] 不勾选保存 → 队列 +1
- [ ] 一键推今日 3 条 → Obsidian 打开写入 3 个 section、胶囊消失、时间线全部 ✓
- [ ] 推完再保存一条 → 胶囊重新出现
- [ ] 推 Obsidian 失败（手动取消协议）→ 队列已清，可通过编辑重推

### 编辑路径
- [ ] 时间线上点已同步事件 → 面板预填 → 修改保存 → 重新入队、提示 Obsidian 需更新
- [ ] 时间线上点待同步事件 → 修改保存 → 队列里该条 `data` 更新，不多一条

### 数据兼容
- [ ] 导入老版本 JSON（没有 moodLogs 字段）→ 正常加载，moodLogs 自动补 []
- [ ] 导出 JSON → 包含新字段 `moodLogs`，版本号 +1

### 跨日
- [ ] 昨日的 pending 事件不会出现在今日胶囊里
- [ ] 手工修改系统时间到次日 → 新事件写到新日期笔记

## 12. 实施范围

### 保留不动
- 电池计算逻辑
- Chart.js 睡眠趋势图
- Jarvis 简报面板
- 最近记录列表（下方）
- 导出/导入 JSON
- 数据概览（天数、日期范围）
- `bootstrapTodayEntry` 里的今日日期修复（之前已修）

### 重写
- HTML：删除主表单 + 折叠按钮 + 追加区；加快速捕获条 + 事件面板 + 时间线 + 待同步胶囊
- JS：新增 `EventModel / SyncQueue / PanelManager`；重写 `Handlers.submit` 和所有 append handlers
- CSS：新增面板切换、胶囊、时间线的样式

### 延后
- 废弃字段清理（数据先留着）
- 拆单文件为多文件
- PWA manifest + Service Worker
- Obsidian Local REST API 集成

---

## 附录 A：决策记录

| 决策 | 选项 | 取 | 理由 |
|---|---|---|---|
| 使用模式 | A/B/C | A | 用户"随发生随记录" |
| 整体方向 | A/B/C | B | 大表单不适配 A 模式 |
| 训练粒度 | 整次/单动作 | 整次 | 用户隔天一次训练 |
| 时段能量 | 保留/砍 | 砍 | 减少字段，只留复盘里"今日整体" |
| 深度工作/琐事 | 保留/砍 | 砍 | 用 Obsidian daily note 记 |
| 心情独立面板 | 加/不加 | 加 | 第 7 种事件 |
| 同步节奏 | 手动/自动/混合 | 混合 | 默认攒着 + 偶尔立刻推 |
| Obsidian 文件 | 一日一文 / 一事一文 | 一日一文 | Obsidian Claude 看一个文件就有完整上下文 |
| 技术选型（跳转）| location.href / a.click() / iframe | `location.href` 同步 | 之前踩过坑的唯一稳妥方式 |
