# 快速捕获 + 批量同步重设计 · 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把主表单拆成 7 个事件快速捕获面板 + 本地队列 + 一键推 Obsidian，按照 [2026-04-24-quick-capture-redesign-design.md](../specs/2026-04-24-quick-capture-redesign-design.md) 落地。

**Architecture:** 所有改动落在单文件 `index.html` 内（沿用现有 IIFE 架构）。新增 3 个模块 `EventModel / SyncQueue / PanelManager`，复用 `EntryModel / DateUtils / StorageService / UI`。保存为纯本地同步操作；Obsidian 推送用 `window.location.href = obsidian://...` 同步调用。

**Tech Stack:** 原生 HTML + CSS + JS（单文件），localStorage 持久化，Chart.js（已有），`obsidian://` URL scheme。

**Verification Strategy:** 项目无自动化测试框架（spec §11 明确）。每个任务的"验证步骤"由**手动在浏览器 DevTools / Console 里执行**，用 localStorage、window 对象、页面交互来核对。完成后 commit。

---

## Prep · Task 0：清理工作区

**Files:**
- Modify: `/Users/yusheng/量身定制小程序/michael-energy-tracker/.gitignore`
- Commit: 现有未提交的 `index.html` 修改

**Background:** 当前 index.html 有 108 行未提交的改动（之前的 3 处 bug 修复）。这些改动在重设计后会被新代码替换（`Handlers.submit`、`saveRecord` 等），但保留提交作为历史记录更干净。另需忽略 `.superpowers/` 目录（brainstorming 缓存）。

- [ ] **Step 1：追加 .gitignore**

Edit `.gitignore`，在末尾追加：

```
.superpowers/
```

- [ ] **Step 2：提交当前待定 fixes**

```bash
cd /Users/yusheng/量身定制小程序/michael-energy-tracker
git add .gitignore index.html
git commit -m "$(cat <<'EOF'
fix: 修复日期硬编码 + submit handler + 编辑展开

- getBootstrapTodayEntry 用 DateUtils.getTodayISO() 替代硬编码
- 清空私人 dailyStory，避免数据泄露复刻
- Handlers.submit 用 Handlers.saveXxx 直接引用（this 绑定修复）
- saveToObsidian 拆成 saveRecord 纯本地保存
- recordActions edit 时强制展开表单
EOF
)"
```

- [ ] **Step 3：验证工作树干净**

Run: `git status --short`
Expected: 空输出（没有任何 `M` / `??`）

---

## Phase A · 数据与模块基础

### Task 1：新增 `moodLogs[]` 字段到 EntryModel

**Files:**
- Modify: `index.html`（搜索 `normalizeMealItems\|normalizeRecoveryItems` 附近）
- Modify: `index.html`（搜索 `normalize(raw)` 的返回对象）

**Background:** 新增事件类型"💭 心情"要追加到每日 entry 的新数组 `moodLogs[]`。模仿现有的 `normalizeMealItems` / `normalizeRecoveryItems` 模式。

- [ ] **Step 1：新增 `normalizeMoodLogs` 辅助函数**

在 `index.html` 里找到 `normalizeRecoveryItems(items)` 方法（约 line 2960），在它下方紧接着添加：

```js
normalizeMoodLogs(items) {
  if (!Array.isArray(items)) return [];
  return items
    .map((item) => {
      if (!item || typeof item !== "object") return null;
      const time = String(item.time || "").trim();
      const tag = String(item.tag || "").trim();
      const note = String(item.note || "").trim();
      if (!tag) return null;
      return { time, tag, note };
    })
    .filter(Boolean);
},
```

- [ ] **Step 2：在 `normalize(raw)` 返回对象里加 `moodLogs`**

找到 `normalize(raw)` 方法的返回对象（约 line 2985，在 `recoveryItems` 之后，`healthyFood` 之前）加一行：

```js
moodLogs: this.normalizeMoodLogs(raw.moodLogs || []),
```

- [ ] **Step 3：浏览器验证**

打开 `index.html`（本地双击）→ F12 Console → 输入：

```js
localStorage.getItem("michael-energy-tracker-v1")
```

复制输出。然后输入：

```js
JSON.parse(localStorage.getItem("michael-energy-tracker-v1"))[0].moodLogs
```

Expected: `[]`（每条老数据已自动补了空数组，没报错）。

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat(data): 新增 moodLogs[] 字段到每日 entry"
```

---

### Task 2：新增 `SyncQueue` 模块

**Files:**
- Modify: `index.html`（搜索 `const StorageService = {` 上面那行）
- Modify: `index.html`（搜索 `storageKey: "michael-energy-tracker-v1"` 附近的 CONFIG）

**Background:** 独立的 localStorage 键 `michael-energy-tracker-pending-sync-v1` 存储待同步事件队列。SyncQueue 封装加入/移除/按日期筛/去重/清今日的操作。

- [ ] **Step 1：在 CONFIG 里新增 storage key**

找到 `CONFIG = {`（约 line 2600），在 `templateSettingsKey` 那行下面加：

```js
pendingSyncKey: "michael-energy-tracker-pending-sync-v1",
```

- [ ] **Step 2：定义 `SyncQueue` 模块**

找到 `const StorageService = {`（约 line 3151），在它**上面**插入新模块：

```js
const SyncQueue = {
  load() {
    const raw = localStorage.getItem(CONFIG.pendingSyncKey);
    if (!raw) return [];
    try {
      const parsed = JSON.parse(raw);
      return Array.isArray(parsed) ? parsed : [];
    } catch (error) {
      console.error("读取待同步队列失败:", error);
      return [];
    }
  },

  save(queue) {
    localStorage.setItem(CONFIG.pendingSyncKey, JSON.stringify(queue));
  },

  add(event) {
    // 去重：同 id 的事件就地更新，不重复入队
    const queue = this.load();
    const existingIndex = queue.findIndex((item) => item.id === event.id);
    if (existingIndex >= 0) {
      queue[existingIndex] = event;
    } else {
      queue.push(event);
    }
    this.save(queue);
    return queue;
  },

  removeById(id) {
    const queue = this.load().filter((item) => item.id !== id);
    this.save(queue);
    return queue;
  },

  filterByDate(date) {
    return this.load().filter((item) => item.date === date);
  },

  clearByDate(date) {
    const queue = this.load().filter((item) => item.date !== date);
    this.save(queue);
    return queue;
  },

  countByDate(date) {
    return this.filterByDate(date).length;
  }
};
```

- [ ] **Step 3：浏览器验证**

刷新 `index.html` → Console：

```js
// 加一条测试事件
SyncQueue.add({ id: "test-1", eventType: "meal", date: "2026-04-24", time: "12:15", capturedAt: Date.now(), data: { food: "test" } });
SyncQueue.countByDate("2026-04-24"); // 应该 >= 1
SyncQueue.load(); // 看到刚加的条目
SyncQueue.removeById("test-1");
SyncQueue.countByDate("2026-04-24"); // 回到之前的数
```

但等等 —— `SyncQueue` 在 IIFE 里，Console 访问不到。先临时挂到 window 验证：在 `initialize()` 函数末尾临时加 `window.SyncQueue = SyncQueue;`，测完**删掉**。

Expected: add → count+1，removeById → count-1，值持久化到 localStorage。

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat(sync): 新增 SyncQueue 模块管理待同步事件"
```

---

### Task 3：新增 `EventModel` 模块 · 7 种事件类型定义

**Files:**
- Modify: `index.html`（搜索 `const EntryModel = {` 上面）

**Background:** 每种事件类型的 schema、验证、应用到 entry 的逻辑、markdown 渲染模板集中在 EventModel 里。后续面板全部调用它，不自己造逻辑。

- [ ] **Step 1：在 EntryModel 上方插入 EventModel**

找到 `const EntryModel = {`（约 line 2980），在它**上面**插入：

```js
const EventModel = {
  // 每种事件的定义：emoji / 显示名 / 字段列表 / 验证函数 / 应用逻辑 / markdown 渲染
  types: {
    sleep: {
      emoji: "😴",
      label: "睡眠",
      mode: "overwrite",
      validate(data) {
        if (typeof data.sleepScore !== "number" || data.sleepScore < 0 || data.sleepScore > 100) {
          return "睡眠分需要在 0-100 之间";
        }
        if (!data.bedtime || !data.wakeTime) return "入睡和起床时间必填";
        if (typeof data.wakeups !== "number" || data.wakeups < 0 || data.wakeups > 10) {
          return "起夜次数需要在 0-10 之间";
        }
        return "";
      },
      applyToEntry(entry, data) {
        return EntryModel.normalize({
          ...entry,
          sleepScore: data.sleepScore,
          bedtime: data.bedtime,
          wakeTime: data.wakeTime,
          wakeups: data.wakeups
        });
      },
      toMarkdown(event) {
        const d = event.data;
        return [
          `### 😴 睡眠 ${event.time}`,
          `- 睡眠分：${d.sleepScore}`,
          `- 入睡：${d.bedtime} / 起床：${d.wakeTime}`,
          `- 起夜：${d.wakeups} 次`
        ].join("\n");
      }
    },

    meal: {
      emoji: "🍱",
      label: "餐食",
      mode: "append",
      validate(data) {
        if (!data.food || !data.food.trim()) return "请填写吃了什么";
        if (!data.type) return "请选择餐次";
        return "";
      },
      applyToEntry(entry, data) {
        const mealItems = [...(entry.mealItems || []), {
          time: data.time,
          type: data.type,
          food: data.food,
          cooked: !!data.cooked,
          healthy: !!data.healthy,
          ateOut: !!data.ateOut,
          note: ""
        }];
        return EntryModel.normalize({
          ...entry,
          mealItems,
          mealDetails: EntryModel.formatMealDetails(mealItems, entry.mealDetails),
          cooked: entry.cooked || data.cooked,
          healthyFood: entry.healthyFood || data.healthy,
          ateOut: entry.ateOut || data.ateOut
        });
      },
      toMarkdown(event) {
        const d = event.data;
        const tags = [d.cooked ? "自己做" : null, d.healthy ? "健康" : null, d.ateOut ? "外食" : null].filter(Boolean).join(" · ") || "无标签";
        return [
          `### 🍱 ${d.type} ${event.time}`,
          `- ${d.food}`,
          `- ${tags}`
        ].join("\n");
      }
    },

    training: {
      emoji: "💪",
      label: "训练",
      mode: "overwrite",
      validate(data) {
        if (!Array.isArray(data.workoutItems) || data.workoutItems.length === 0) {
          return "请至少添加一个训练动作";
        }
        return "";
      },
      applyToEntry(entry, data) {
        return EntryModel.normalize({
          ...entry,
          workoutItems: data.workoutItems,
          exercised: true
        });
      },
      toMarkdown(event) {
        const items = (event.data.workoutItems || []).map(
          (it) => `- ${it.name} · ${it.volume}${it.note ? "｜" + it.note : ""}${it.rating ? "｜⭐ " + it.rating : ""}`
        );
        return [`### 💪 训练 ${event.time}`, ...items].join("\n");
      }
    },

    recovery: {
      emoji: "🛋",
      label: "补能",
      mode: "append",
      validate(data) {
        if (!data.type) return "请选择补能类型";
        if (typeof data.boost !== "number" || data.boost < 1 || data.boost > 40) {
          return "补能加分需要在 1-40 之间";
        }
        return "";
      },
      applyToEntry(entry, data) {
        const recoveryItems = [...(entry.recoveryItems || []), {
          time: data.time,
          type: data.type,
          boost: data.boost,
          note: data.note || ""
        }];
        const nextBoost = Math.min(40, (entry.energyBoost || 0) + data.boost);
        return EntryModel.normalize({
          ...entry,
          recoveryItems,
          energyBoost: nextBoost
        });
      },
      toMarkdown(event) {
        const d = event.data;
        const lines = [
          `### 🛋 ${d.type} ${event.time}`,
          `- 补能加分：+${d.boost}%`
        ];
        if (d.note) lines.push(`- ${d.note}`);
        return lines.join("\n");
      }
    },

    weight: {
      emoji: "⚖️",
      label: "体重",
      mode: "overwrite",
      validate(data) {
        if (typeof data.weight !== "number" || data.weight < 20 || data.weight > 300) {
          return "体重需要在 20-300 kg 之间";
        }
        return "";
      },
      applyToEntry(entry, data) {
        return EntryModel.normalize({ ...entry, weight: data.weight });
      },
      toMarkdown(event) {
        return `### ⚖️ 体重 ${event.time}\n- ${event.data.weight.toFixed(1)} kg`;
      }
    },

    reflection: {
      emoji: "🌙",
      label: "复盘",
      mode: "overwrite",
      validate(data) {
        if (typeof data.energy !== "number" || data.energy < 1 || data.energy > 10) {
          return "今日整体能量需要在 1-10 之间";
        }
        if (!data.mood) return "请选择睡前状态";
        return "";
      },
      applyToEntry(entry, data) {
        return EntryModel.normalize({
          ...entry,
          mood: data.mood,
          energy: data.energy,
          dailyStory: data.dailyStory || ""
        });
      },
      toMarkdown(event) {
        const d = event.data;
        const lines = [
          `### 🌙 复盘 ${event.time}`,
          `- 睡前状态：${d.mood}`,
          `- 今日整体能量：${d.energy}/10`
        ];
        if (d.dailyStory) lines.push("", d.dailyStory);
        return lines.join("\n");
      }
    },

    mood: {
      emoji: "💭",
      label: "心情",
      mode: "append",
      validate(data) {
        if (!data.tag) return "请选择心情";
        return "";
      },
      applyToEntry(entry, data) {
        const moodLogs = [...(entry.moodLogs || []), {
          time: data.time,
          tag: data.tag,
          note: data.note || ""
        }];
        return EntryModel.normalize({ ...entry, moodLogs });
      },
      toMarkdown(event) {
        const d = event.data;
        const lines = [`### 💭 心情 ${event.time}`, `- ${d.tag}`];
        if (d.note) lines.push(`- ${d.note}`);
        return lines.join("\n");
      }
    }
  },

  // 生成唯一 id（timestamp + random 避免快速双击冲突）
  makeId() {
    return `evt-${Date.now()}-${Math.floor(Math.random() * 1000)}`;
  },

  // 构造 pendingSync 队列里的一条事件
  buildQueueItem(eventType, date, time, data) {
    return {
      id: this.makeId(),
      eventType,
      date,
      time,
      capturedAt: Date.now(),
      data
    };
  },

  // 把一组事件按 time 升序拼接成 markdown（多个事件之间空一行）
  renderBatchMarkdown(events) {
    const sorted = [...events].sort((a, b) => (a.time || "").localeCompare(b.time || ""));
    return sorted.map((event) => this.types[event.eventType].toMarkdown(event)).join("\n\n");
  }
};

// 注：spec §7.3 的"（已修改）"后缀逻辑在本实施版本里**简化掉**。
// 本版只支持"编辑尚在队列的事件"（时间线只展示 pending）。
// 编辑已推 Obsidian 的事件暂不支持 —— 需在 Obsidian 里直接改。
// 后续若需要再引入 isEdit 标记和 eventLog 数据结构。
```

- [ ] **Step 2：浏览器验证**

刷新页面 → Console（临时 `window.EventModel = EventModel;` 在 initialize() 里挂一下）→ 输入：

```js
const ev = EventModel.buildQueueItem("sleep", "2026-04-24", "07:15", { sleepScore: 85, bedtime: "22:30", wakeTime: "06:45", wakeups: 1 });
EventModel.types.sleep.toMarkdown(ev);
// 期望输出：
// "### 😴 睡眠 07:15\n- 睡眠分：85\n- 入睡：22:30 / 起床：06:45\n- 起夜：1 次"
```

```js
EventModel.types.meal.validate({ food: "", type: "午餐" });
// 期望："请填写吃了什么"
EventModel.types.meal.validate({ food: "牛肉", type: "午餐" });
// 期望：""
```

删掉临时的 `window.EventModel = ...`。

- [ ] **Step 3：Commit**

```bash
git add index.html
git commit -m "feat(events): 新增 EventModel 定义 7 种事件类型"
```

---

## Phase B · UI 框架

### Task 4：HTML 骨架 · 快速捕获条 + 面板容器 + 时间线 + 待同步胶囊

**Files:**
- Modify: `index.html`（搜索 `<form class="entry-form"`）

**Background:** 新增所有 UI 骨架结构（暂无交互）。旧主表单先**整体 hide**（加 `hidden` 属性或 `display:none`），新 UI 显示。一次性建好空壳，后续 panel 任务往里填字段。

- [ ] **Step 1：注释掉（不删）旧"今日录入"整块**

找到 `<form class="entry-form" id="entryForm">`（约 line 2103），往上找到包含它的 `<article class="panel">`，给这个 article 加 `hidden` 属性：

```html
<article class="panel" hidden data-legacy="true">
```

（`data-legacy="true"` 标记稍后清理用）

- [ ] **Step 2：在这个 legacy article 之后插入新结构**

```html
<!-- 重设计：快速捕获面板 -->
<article class="panel" id="capturePanel">
  <div class="panel-content">
    <div class="section-title">
      <h2>快速捕获</h2>
      <span id="captureStatusLabel">点一个按钮展开当下要记的那件事</span>
    </div>

    <div class="capture-bar" id="captureBar">
      <button class="capture-pill" type="button" data-event-type="sleep">😴 睡眠</button>
      <button class="capture-pill" type="button" data-event-type="meal">🍱 餐食</button>
      <button class="capture-pill" type="button" data-event-type="training">💪 训练</button>
      <button class="capture-pill" type="button" data-event-type="recovery">🛋 补能</button>
      <button class="capture-pill" type="button" data-event-type="weight">⚖️ 体重</button>
      <button class="capture-pill" type="button" data-event-type="reflection">🌙 复盘</button>
      <button class="capture-pill" type="button" data-event-type="mood">💭 心情</button>
    </div>

    <div class="capture-panel-container" id="capturePanelContainer"></div>
  </div>
</article>

<!-- 待同步胶囊（仅当 pendingSync 今日非空时显示） -->
<div class="pending-sync-capsule" id="pendingSyncCapsule" hidden>
  <span class="pending-sync-text">⏳ 待同步到 Obsidian：<strong id="pendingSyncCount">0</strong> 条</span>
  <button class="btn-primary btn-slim" id="pushAllBtn" type="button">📤 一键推到 Obsidian</button>
</div>

<!-- 今日时间线 -->
<article class="panel" id="todayTimelinePanel">
  <div class="panel-content">
    <div class="section-title">
      <h2>今日时间线</h2>
      <span id="todayTimelineLabel">今天记的所有事件</span>
    </div>
    <div class="timeline-list" id="todayTimeline"></div>
  </div>
</article>
```

**位置**：插在原 legacy article 之后、`<section class="panel">` 能量守恒提醒之前（约 line 2552）。可以先在原 article 关闭标签 `</article>` 后面搜索下一个 section 再插。

- [ ] **Step 3：新增 CSS**

找到 `<style>` 结束前（约 line 1995），在 `</style>` 上面加：

```css
/* 快速捕获重设计 */
.capture-bar {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin: 12px 0 16px;
}
.capture-pill {
  padding: 8px 14px;
  border-radius: 20px;
  border: 1px solid var(--border, #ccc);
  background: white;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.15s;
}
.capture-pill:hover {
  background: var(--bg-deep, #e8dcc6);
}
.capture-pill.active {
  background: var(--accent, #1e7b64);
  color: white;
  border-color: var(--accent, #1e7b64);
  font-weight: 600;
}
.capture-panel-container {
  min-height: 20px;
}
.capture-panel {
  background: #e8f5e9;
  border: 2px solid var(--accent, #1e7b64);
  border-radius: 10px;
  padding: 16px;
  margin-top: 4px;
}
.capture-panel h3 {
  margin: 0 0 12px;
  font-size: 15px;
}
.capture-panel .field-row {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin-bottom: 10px;
}
.capture-panel .panel-actions {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-top: 12px;
  flex-wrap: wrap;
}
.capture-panel .immediate-push-check {
  font-size: 13px;
  color: #555;
  display: flex;
  align-items: center;
  gap: 4px;
}

/* 待同步胶囊 */
.pending-sync-capsule {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #fff8e1;
  border: 1px solid #ffb300;
  border-radius: 10px;
  padding: 10px 14px;
  margin: 0 auto 12px;
  max-width: 960px;
}
.pending-sync-text {
  font-size: 13px;
  color: #5d4037;
}
.pending-sync-text strong {
  color: #e65100;
  margin: 0 2px;
}

/* 时间线 */
.timeline-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.timeline-item {
  display: flex;
  gap: 10px;
  padding: 8px 12px;
  background: white;
  border: 1px solid #eee;
  border-radius: 8px;
  cursor: pointer;
  font-size: 13px;
  transition: background 0.15s;
}
.timeline-item:hover {
  background: #fafafa;
}
.timeline-item .tl-status {
  width: 18px;
}
.timeline-item .tl-time {
  color: #999;
  width: 50px;
  flex-shrink: 0;
}
.timeline-item .tl-content {
  flex: 1;
}
.timeline-item.synced .tl-status::before { content: "✓"; color: #2e7d32; }
.timeline-item.pending .tl-status::before { content: "⏳"; }
.timeline-empty {
  padding: 20px;
  text-align: center;
  color: #999;
  font-size: 13px;
}
```

- [ ] **Step 4：浏览器验证**

刷新页面。应该看到：
- 旧的"今日录入"大表单消失了
- 新的"快速捕获"标题 + 7 个胶囊按钮
- 胶囊按钮 hover 变灰
- 下面的"今日时间线"空白
- 其他一切（电池、Jarvis、最近记录、统计）保持不变

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat(ui): HTML 骨架 · 快速捕获条 + 时间线 + 待同步胶囊"
```

---

### Task 5：`PanelManager` 模块 · 管理当前展开的面板

**Files:**
- Modify: `index.html`（搜索 `const SyncQueue = {` 下面 —— Task 2 创建的那个）

**Background:** 记录"当前激活的事件类型"。点胶囊 → 展开对应面板（关闭其他）。

- [ ] **Step 1：在 SyncQueue 之后插入 PanelManager**

搜索 `const SyncQueue = {` 找到 Task 2 创建的模块，在它的闭合 `};` 之后插入：

```js
const PanelManager = {
  activeType: null,

  open(eventType, options = {}) {
    this.activeType = eventType;
    this.render(options);
  },

  close() {
    this.activeType = null;
    this.render();
  },

  toggle(eventType) {
    if (this.activeType === eventType) {
      this.close();
    } else {
      this.open(eventType);
    }
  },

  render(options = {}) {
    // 胶囊按钮 active 状态
    document.querySelectorAll(".capture-pill").forEach((btn) => {
      btn.classList.toggle("active", btn.dataset.eventType === this.activeType);
    });

    const container = document.getElementById("capturePanelContainer");
    if (!container) return;

    if (!this.activeType) {
      container.innerHTML = "";
      return;
    }

    // 调用面板专属的渲染函数（下面 Phase C 各任务里定义）
    const renderer = PanelRenderers[this.activeType];
    if (!renderer) {
      container.innerHTML = `<div class="capture-panel"><p>面板 ${this.activeType} 待实现</p></div>`;
      return;
    }
    container.innerHTML = renderer(options.prefill || null);

    // 调用绑定函数（每个面板自己处理 input/save）
    const binder = PanelBinders[this.activeType];
    if (binder) binder(options.prefill || null);
  }
};

// 每个事件类型的面板渲染器：输入 prefill（可空）→ 返回 HTML 字符串
const PanelRenderers = {};

// 每个事件类型的面板事件绑定器：prefill 同上 → 给面板里的元素绑事件
const PanelBinders = {};
```

- [ ] **Step 2：在 `bindEvents()` 里绑定胶囊点击**

找到 `function bindEvents() {`（约 line 5005）。在函数体内**末尾**追加：

```js
document.getElementById("captureBar").addEventListener("click", (event) => {
  const pill = event.target.closest(".capture-pill[data-event-type]");
  if (!pill) return;
  PanelManager.toggle(pill.dataset.eventType);
});
```

- [ ] **Step 3：浏览器验证**

刷新页面 → 点任意一个胶囊按钮（如"😴 睡眠"）

Expected:
- 该按钮变绿（`.active`）
- 下面出现 "面板 sleep 待实现" 的临时提示（因为 PanelRenderers.sleep 还没实现）
- 再点一次 → 收起、按钮复位
- 点另一个胶囊 → 上一个收起，新的展开

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat(ui): PanelManager 管理面板互斥展开"
```

---

## Phase C · 7 个事件面板（每个一任务）

> **约定**：每个面板都实现 `PanelRenderers.<type>(prefill)` 和 `PanelBinders.<type>(prefill)` 两个函数。`prefill` 为对象（编辑历史时填），为 `null` 时用默认值。保存按钮的通用处理逻辑：
>
> 1. 从面板读取 data
> 2. `EventModel.types.<type>.validate(data)` → 验证
> 3. 建一个 queue item
> 4. `App.applyEvent(event)` 写 entries + push SyncQueue + render
> 5. 若勾选"立刻推" → `App.pushEventsToObsidian([event])` 并从队列移除
> 6. 关闭面板

### Task 6：`App.applyEvent` + `App.pushEventsToObsidian` 辅助方法

**Files:**
- Modify: `index.html`（搜索 `const App = {`）

**Background:** 这两个方法会被所有面板重用。先一次性写好再去写具体面板。

- [ ] **Step 1：在 App 里加两个新方法**

找到 `const App = {`（约 line 4457），在 `render()` 方法之后加：

```js
// 接收一条事件，应用到 entries + 入队 + 重绘
applyEvent(event) {
  const entries = state.entries;
  const existingEntry = entries.find((e) => e.date === event.date) || EntryModel.normalize({ date: event.date });
  const nextEntry = EventModel.types[event.eventType].applyToEntry(existingEntry, event.data);
  const nextEntries = entries.filter((e) => e.date !== event.date).concat(nextEntry);
  state.entries = StorageService.saveEntries(nextEntries);
  SyncQueue.add(event);
  UI.render(state.entries);
  UI.renderTodayTimeline();
  UI.renderPendingCapsule();
  return true;
},

// 把一组事件推到 Obsidian（单事件 or 批量共用）
pushEventsToObsidian(events) {
  if (!events.length) return;
  const date = events[0].date;
  const content = "\n\n" + EventModel.renderBatchMarkdown(events);
  const file = `${CONFIG.obsidianFolder}/训练日志 ${DateUtils.formatShortYearDate(date)}`;
  const url = `obsidian://new?vault=${encodeURIComponent(CONFIG.obsidianVault)}&file=${encodeURIComponent(file)}&append=true&content=${encodeURIComponent(content)}`;
  // 同步调用，自定义协议必须这样
  window.location.href = url;
  // 乐观清队列（无法从 URL scheme 得知成败）
  events.forEach((e) => SyncQueue.removeById(e.id));
  UI.renderTodayTimeline();
  UI.renderPendingCapsule();
},
```

- [ ] **Step 2：给 UI 模块加两个空的 render 占位（下一任务填充）**

找到 `const UI = {`（约 line 3500，在 `const ChartRenderer =` 之后），在 `render(entries)` 方法（在 UI 模块末尾，约 line 4442）**之前**加：

```js
renderTodayTimeline() {
  // Task 14 填充
},

renderPendingCapsule() {
  // Task 15 填充
},
```

- [ ] **Step 3：Commit**

```bash
git add index.html
git commit -m "feat(app): applyEvent + pushEventsToObsidian 通用方法"
```

---

### Task 7：睡眠面板

**Files:**
- Modify: `index.html`（搜索 `const PanelRenderers = {};`）

- [ ] **Step 1：实现 sleep 渲染器**

找到 `const PanelRenderers = {};` （Task 5 创建的空对象），改成：

```js
const PanelRenderers = {
  sleep(prefill) {
    const p = prefill || {};
    const today = state.entries.find((e) => e.date === DateUtils.getTodayISO()) || {};
    return `
      <div class="capture-panel" data-panel-type="sleep">
        <h3>😴 记睡眠</h3>
        <div class="field-row">
          <div class="field" style="flex:1;min-width:200px">
            <label>睡眠分 <strong id="sleepPanelScoreValue">${p.sleepScore ?? today.sleepScore ?? 80}</strong></label>
            <input type="range" id="sleepPanelScore" min="0" max="100" step="1" value="${p.sleepScore ?? today.sleepScore ?? 80}">
          </div>
          <div class="field">
            <label>起夜次数</label>
            <input type="number" id="sleepPanelWakeups" min="0" max="10" value="${p.wakeups ?? today.wakeups ?? 0}">
          </div>
        </div>
        <div class="field-row">
          <div class="field" style="flex:1">
            <label>入睡时间</label>
            <input type="time" id="sleepPanelBedtime" value="${p.bedtime || today.bedtime || '22:00'}">
          </div>
          <div class="field" style="flex:1">
            <label>起床时间</label>
            <input type="time" id="sleepPanelWakeTime" value="${p.wakeTime || today.wakeTime || '06:00'}">
          </div>
        </div>
        <div class="panel-actions">
          <button class="btn-primary" id="sleepPanelSave" type="button">💾 保存</button>
          <label class="immediate-push-check">
            <input type="checkbox" id="sleepPanelImmediate"> 立刻推到 Obsidian
          </label>
        </div>
        <div class="panel-error" id="sleepPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
      </div>
    `;
  }
};

const PanelBinders = {
  sleep(prefill) {
    document.getElementById("sleepPanelScore").addEventListener("input", (e) => {
      document.getElementById("sleepPanelScoreValue").textContent = e.target.value;
    });
    document.getElementById("sleepPanelSave").addEventListener("click", () => {
      const data = {
        sleepScore: Number(document.getElementById("sleepPanelScore").value),
        bedtime: document.getElementById("sleepPanelBedtime").value,
        wakeTime: document.getElementById("sleepPanelWakeTime").value,
        wakeups: Number(document.getElementById("sleepPanelWakeups").value)
      };
      const err = EventModel.types.sleep.validate(data);
      if (err) {
        document.getElementById("sleepPanelError").textContent = err;
        return;
      }
      const editId = prefill?.id;
      const event = editId
        ? { ...prefill, data, capturedAt: Date.now() }
        : EventModel.buildQueueItem("sleep", DateUtils.getTodayISO(), DateUtils.getCurrentTimeString(), data);
      App.applyEvent(event);
      if (document.getElementById("sleepPanelImmediate").checked) {
        App.pushEventsToObsidian([event]);
      }
      UI.showStatus(`已保存：😴 睡眠`);
      PanelManager.close();
    });
  }
};
```

- [ ] **Step 2：浏览器验证**

刷新页面 → 点 😴 睡眠 → 拖滑块 → 填时间 → 点 💾 保存

Expected:
- 面板关闭
- 底部状态条："已保存：😴 睡眠"
- Console 里 `JSON.parse(localStorage["michael-energy-tracker-pending-sync-v1"])` 可以看到新增一条
- `JSON.parse(localStorage["michael-energy-tracker-v1"])` 今日 entry 的 sleepScore/bedtime/wakeups 已更新

- [ ] **Step 3：Commit**

```bash
git add index.html
git commit -m "feat(panels): 😴 睡眠面板"
```

---

### Task 8：体重面板

**Files:**
- Modify: `index.html`（搜索 `const PanelRenderers = {`）

- [ ] **Step 1：在 PanelRenderers 里追加 weight 方法**

在 `sleep` 方法之后追加：

```js
weight(prefill) {
  const p = prefill || {};
  const today = state.entries.find((e) => e.date === DateUtils.getTodayISO()) || {};
  const defaultWeight = p.weight ?? today.weight ?? CONFIG.currentWeightDefault;
  return `
    <div class="capture-panel" data-panel-type="weight">
      <h3>⚖️ 记体重</h3>
      <div class="field-row">
        <div class="field" style="flex:1">
          <label>今日体重（kg）<strong id="weightPanelValue">${defaultWeight}</strong></label>
          <input type="range" id="weightPanelSlider" min="40" max="140" step="0.1" value="${defaultWeight}">
        </div>
      </div>
      <div class="panel-actions">
        <button class="btn-primary" id="weightPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="weightPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="weightPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

在 PanelBinders 里追加 weight：

```js
weight(prefill) {
  document.getElementById("weightPanelSlider").addEventListener("input", (e) => {
    document.getElementById("weightPanelValue").textContent = Number(e.target.value).toFixed(1);
  });
  document.getElementById("weightPanelSave").addEventListener("click", () => {
    const data = { weight: Number(document.getElementById("weightPanelSlider").value) };
    const err = EventModel.types.weight.validate(data);
    if (err) {
      document.getElementById("weightPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("weight", DateUtils.getTodayISO(), DateUtils.getCurrentTimeString(), data);
    App.applyEvent(event);
    if (document.getElementById("weightPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：⚖️ 体重`);
    PanelManager.close();
  });
},
```

- [ ] **Step 2：浏览器验证**

刷新 → 点 ⚖️ 体重 → 调滑块 → 保存 → 看状态条 + localStorage 数据

- [ ] **Step 3：Commit**

```bash
git add index.html
git commit -m "feat(panels): ⚖️ 体重面板"
```

---

### Task 9：餐食面板

**Files:**
- Modify: `index.html`

- [ ] **Step 1：PanelRenderers 追加 meal**

```js
meal(prefill) {
  const p = prefill?.data || {};
  const time = prefill?.time || DateUtils.getCurrentTimeString();
  return `
    <div class="capture-panel" data-panel-type="meal">
      <h3>🍱 记一餐</h3>
      <div class="field-row">
        <div class="field">
          <label>时间</label>
          <input type="time" id="mealPanelTime" value="${time}">
        </div>
        <div class="field" style="flex:1">
          <label>餐次</label>
          <select id="mealPanelType">
            <option value="早餐" ${p.type === "早餐" ? "selected" : ""}>早餐</option>
            <option value="上午餐" ${p.type === "上午餐" ? "selected" : ""}>上午餐</option>
            <option value="午餐" ${!p.type || p.type === "午餐" ? "selected" : ""}>午餐</option>
            <option value="加餐" ${p.type === "加餐" ? "selected" : ""}>加餐</option>
            <option value="晚餐" ${p.type === "晚餐" ? "selected" : ""}>晚餐</option>
            <option value="其他" ${p.type === "其他" ? "selected" : ""}>其他</option>
          </select>
        </div>
      </div>
      <div class="field">
        <label>吃了什么</label>
        <textarea id="mealPanelFood" placeholder="例如：牛肉 青菜 米饭" rows="2">${p.food || ""}</textarea>
      </div>
      <div class="field-row">
        <label class="immediate-push-check"><input type="checkbox" id="mealPanelCooked" ${p.cooked !== false ? "checked" : ""}> 自己做</label>
        <label class="immediate-push-check"><input type="checkbox" id="mealPanelHealthy" ${p.healthy !== false ? "checked" : ""}> 健康营养</label>
        <label class="immediate-push-check"><input type="checkbox" id="mealPanelAteOut" ${p.ateOut ? "checked" : ""}> 外食</label>
      </div>
      <div class="panel-actions">
        <button class="btn-primary" id="mealPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="mealPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="mealPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

- [ ] **Step 2：PanelBinders 追加 meal**

```js
meal(prefill) {
  document.getElementById("mealPanelSave").addEventListener("click", () => {
    const data = {
      time: document.getElementById("mealPanelTime").value,
      type: document.getElementById("mealPanelType").value,
      food: document.getElementById("mealPanelFood").value.trim(),
      cooked: document.getElementById("mealPanelCooked").checked,
      healthy: document.getElementById("mealPanelHealthy").checked,
      ateOut: document.getElementById("mealPanelAteOut").checked
    };
    const err = EventModel.types.meal.validate(data);
    if (err) {
      document.getElementById("mealPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, time: data.time, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("meal", DateUtils.getTodayISO(), data.time, data);
    App.applyEvent(event);
    if (document.getElementById("mealPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：🍱 ${data.type}`);
    PanelManager.close();
  });
},
```

- [ ] **Step 3：浏览器验证**

刷新 → 点 🍱 餐食 → 填"牛肉" → 保存 → 再点 🍱 → 填"汤" → 保存

Expected: localStorage 里今日 entry 的 `mealItems` 数组应该有 2 条。`pendingSync` 应该有 2 条。

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat(panels): 🍱 餐食面板"
```

---

### Task 10：补能面板

**Files:**
- Modify: `index.html`

- [ ] **Step 1：PanelRenderers 追加 recovery**

```js
recovery(prefill) {
  const p = prefill?.data || {};
  const time = prefill?.time || DateUtils.getCurrentTimeString();
  return `
    <div class="capture-panel" data-panel-type="recovery">
      <h3>🛋 记补能</h3>
      <div class="field-row">
        <div class="field">
          <label>时间</label>
          <input type="time" id="recoveryPanelTime" value="${time}">
        </div>
        <div class="field" style="flex:1">
          <label>类型</label>
          <select id="recoveryPanelType">
            <option value="午睡" ${!p.type || p.type === "午睡" ? "selected" : ""}>午睡</option>
            <option value="按摩" ${p.type === "按摩" ? "selected" : ""}>按摩</option>
            <option value="冥想" ${p.type === "冥想" ? "selected" : ""}>冥想</option>
            <option value="散步" ${p.type === "散步" ? "selected" : ""}>散步</option>
            <option value="其他补能" ${p.type === "其他补能" ? "selected" : ""}>其他补能</option>
          </select>
        </div>
        <div class="field">
          <label>补能加分（%）</label>
          <input type="number" id="recoveryPanelBoost" min="1" max="40" step="1" value="${p.boost ?? 10}">
        </div>
      </div>
      <div class="field">
        <label>备注</label>
        <input type="text" id="recoveryPanelNote" placeholder="例如：睡了 25 分钟，醒来舒服很多" value="${p.note || ""}">
      </div>
      <div class="panel-actions">
        <button class="btn-primary" id="recoveryPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="recoveryPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="recoveryPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

- [ ] **Step 2：PanelBinders 追加 recovery**

```js
recovery(prefill) {
  document.getElementById("recoveryPanelSave").addEventListener("click", () => {
    const data = {
      time: document.getElementById("recoveryPanelTime").value,
      type: document.getElementById("recoveryPanelType").value,
      boost: Number(document.getElementById("recoveryPanelBoost").value),
      note: document.getElementById("recoveryPanelNote").value.trim()
    };
    const err = EventModel.types.recovery.validate(data);
    if (err) {
      document.getElementById("recoveryPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, time: data.time, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("recovery", DateUtils.getTodayISO(), data.time, data);
    App.applyEvent(event);
    if (document.getElementById("recoveryPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：🛋 ${data.type}`);
    PanelManager.close();
  });
},
```

- [ ] **Step 3：浏览器验证 + Commit**

刷新 → 记一条午睡 → 检查 localStorage entry 的 recoveryItems 和 energyBoost。

```bash
git add index.html
git commit -m "feat(panels): 🛋 补能面板"
```

---

### Task 11：训练面板（复杂：模板 + 多动作）

**Files:**
- Modify: `index.html`

**Background:** 训练面板要复用现有的 `CONFIG.workoutTemplates`（常用七项）。用户可以一键导入、也可以逐个添加/编辑。**mode: overwrite** —— 保存时整组 workoutItems 覆盖 today entry。

- [ ] **Step 1：PanelRenderers 追加 training**

```js
training(prefill) {
  const p = prefill?.data || {};
  const today = state.entries.find((e) => e.date === DateUtils.getTodayISO()) || {};
  const existingItems = p.workoutItems || today.workoutItems || [];
  return `
    <div class="capture-panel" data-panel-type="training">
      <h3>💪 记训练</h3>
      <div class="panel-actions" style="margin-bottom:12px">
        <button class="btn-secondary btn-slim" id="trainingLoadTemplatesBtn" type="button">一键导入常用七项</button>
        <button class="btn-secondary btn-slim" id="trainingClearBtn" type="button">清空</button>
      </div>
      <div id="trainingItemsList" style="display:flex;flex-direction:column;gap:6px;margin-bottom:12px">
        ${existingItems.map((it, idx) => `
          <div class="training-item-row" data-idx="${idx}" style="display:flex;gap:6px;align-items:center">
            <input type="text" class="tr-name" value="${it.name || ""}" placeholder="动作" style="flex:1">
            <input type="text" class="tr-volume" value="${it.volume || ""}" placeholder="组数" style="flex:1">
            <input type="number" class="tr-rating" value="${it.rating || 8}" min="0" max="10" style="width:60px">
            <button class="btn-warning btn-slim" type="button" data-action="remove-tr-item">×</button>
          </div>
        `).join("")}
      </div>
      <button class="btn-secondary btn-slim" id="trainingAddItemBtn" type="button" style="margin-bottom:12px">+ 添加动作</button>
      <div class="panel-actions">
        <button class="btn-primary" id="trainingPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="trainingPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="trainingPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

- [ ] **Step 2：PanelBinders 追加 training**

```js
training(prefill) {
  const list = document.getElementById("trainingItemsList");

  function renderRow(item) {
    const idx = list.children.length;
    const row = document.createElement("div");
    row.className = "training-item-row";
    row.dataset.idx = idx;
    row.style.cssText = "display:flex;gap:6px;align-items:center";
    row.innerHTML = `
      <input type="text" class="tr-name" value="${item.name || ""}" placeholder="动作" style="flex:1">
      <input type="text" class="tr-volume" value="${item.volume || ""}" placeholder="组数" style="flex:1">
      <input type="number" class="tr-rating" value="${item.rating || 8}" min="0" max="10" style="width:60px">
      <button class="btn-warning btn-slim" type="button" data-action="remove-tr-item">×</button>
    `;
    list.appendChild(row);
  }

  document.getElementById("trainingLoadTemplatesBtn").addEventListener("click", () => {
    list.innerHTML = "";
    CONFIG.workoutTemplates.forEach((tpl) => renderRow({ name: tpl.name, volume: tpl.volume, rating: tpl.rating, note: tpl.note }));
  });

  document.getElementById("trainingClearBtn").addEventListener("click", () => {
    list.innerHTML = "";
  });

  document.getElementById("trainingAddItemBtn").addEventListener("click", () => {
    renderRow({ name: "", volume: "", rating: 8, note: "" });
  });

  list.addEventListener("click", (event) => {
    const btn = event.target.closest("[data-action='remove-tr-item']");
    if (!btn) return;
    btn.parentElement.remove();
  });

  document.getElementById("trainingPanelSave").addEventListener("click", () => {
    const rows = list.querySelectorAll(".training-item-row");
    const workoutItems = Array.from(rows).map((row) => ({
      name: row.querySelector(".tr-name").value.trim(),
      volume: row.querySelector(".tr-volume").value.trim(),
      rating: Number(row.querySelector(".tr-rating").value) || 0,
      note: ""
    })).filter((it) => it.name);

    const data = { workoutItems };
    const err = EventModel.types.training.validate(data);
    if (err) {
      document.getElementById("trainingPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("training", DateUtils.getTodayISO(), DateUtils.getCurrentTimeString(), data);
    App.applyEvent(event);
    if (document.getElementById("trainingPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：💪 训练 (${workoutItems.length} 个动作)`);
    PanelManager.close();
  });
},
```

- [ ] **Step 3：浏览器验证 + Commit**

刷新 → 点 💪 → 导入七项 → 改其中一个评分 → 保存 → 看 localStorage

```bash
git add index.html
git commit -m "feat(panels): 💪 训练面板（模板 + 多动作）"
```

---

### Task 12：复盘面板

**Files:**
- Modify: `index.html`

- [ ] **Step 1：PanelRenderers 追加 reflection**

```js
reflection(prefill) {
  const p = prefill?.data || {};
  const today = state.entries.find((e) => e.date === DateUtils.getTodayISO()) || {};
  return `
    <div class="capture-panel" data-panel-type="reflection">
      <h3>🌙 睡前复盘</h3>
      <div class="field-row">
        <div class="field" style="flex:1">
          <label>睡前状态</label>
          <select id="reflectionPanelMood">
            <option value="平静" ${!p.mood && !today.mood || p.mood === "平静" || today.mood === "平静" ? "selected" : ""}>平静</option>
            <option value="满足" ${p.mood === "满足" || today.mood === "满足" ? "selected" : ""}>满足</option>
            <option value="疲惫" ${p.mood === "疲惫" || today.mood === "疲惫" ? "selected" : ""}>疲惫</option>
            <option value="焦虑" ${p.mood === "焦虑" || today.mood === "焦虑" ? "selected" : ""}>焦虑</option>
            <option value="感恩" ${p.mood === "感恩" || today.mood === "感恩" ? "selected" : ""}>感恩</option>
            <option value="其他" ${p.mood === "其他" || today.mood === "其他" ? "selected" : ""}>其他</option>
          </select>
        </div>
        <div class="field">
          <label>今日整体能量（1-10）</label>
          <input type="number" id="reflectionPanelEnergy" min="1" max="10" value="${p.energy ?? today.energy ?? 7}">
        </div>
      </div>
      <div class="field">
        <label>想复盘什么</label>
        <textarea id="reflectionPanelStory" rows="5" placeholder="例如：今天哪里做得不错，哪里消耗了能量，明天醒来最重要的一件事。">${p.dailyStory ?? today.dailyStory ?? ""}</textarea>
      </div>
      <div class="panel-actions">
        <button class="btn-primary" id="reflectionPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="reflectionPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="reflectionPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

- [ ] **Step 2：PanelBinders 追加 reflection**

```js
reflection(prefill) {
  document.getElementById("reflectionPanelSave").addEventListener("click", () => {
    const data = {
      mood: document.getElementById("reflectionPanelMood").value,
      energy: Number(document.getElementById("reflectionPanelEnergy").value),
      dailyStory: document.getElementById("reflectionPanelStory").value.trim()
    };
    const err = EventModel.types.reflection.validate(data);
    if (err) {
      document.getElementById("reflectionPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("reflection", DateUtils.getTodayISO(), DateUtils.getCurrentTimeString(), data);
    App.applyEvent(event);
    if (document.getElementById("reflectionPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：🌙 复盘`);
    PanelManager.close();
  });
},
```

- [ ] **Step 3：浏览器验证 + Commit**

```bash
git add index.html
git commit -m "feat(panels): 🌙 复盘面板"
```

---

### Task 13：心情面板（新事件类型）

**Files:**
- Modify: `index.html`

- [ ] **Step 1：PanelRenderers 追加 mood**

```js
mood(prefill) {
  const p = prefill?.data || {};
  const time = prefill?.time || DateUtils.getCurrentTimeString();
  const selectedTag = p.tag || "🙂";
  const tags = ["😫", "😕", "😐", "🙂", "😄", "🤔", "😡", "🥺"];
  return `
    <div class="capture-panel" data-panel-type="mood">
      <h3>💭 记心情</h3>
      <div class="field-row">
        <div class="field">
          <label>时间</label>
          <input type="time" id="moodPanelTime" value="${time}">
        </div>
      </div>
      <div class="field">
        <label>此刻心情</label>
        <div id="moodPanelTagGroup" style="display:flex;gap:6px;flex-wrap:wrap">
          ${tags.map((tag) => `
            <button type="button" class="mood-tag-btn${tag === selectedTag ? ' active' : ''}" data-tag="${tag}"
                    style="font-size:20px;padding:6px 12px;border:1px solid ${tag === selectedTag ? '#1e7b64' : '#ccc'};background:${tag === selectedTag ? '#e8f5e9' : 'white'};border-radius:8px;cursor:pointer">${tag}</button>
          `).join("")}
        </div>
      </div>
      <div class="field">
        <label>想说什么（可选）</label>
        <input type="text" id="moodPanelNote" placeholder="例如：会议太多，没时间深度工作" value="${p.note || ""}">
      </div>
      <div class="panel-actions">
        <button class="btn-primary" id="moodPanelSave" type="button">💾 保存</button>
        <label class="immediate-push-check">
          <input type="checkbox" id="moodPanelImmediate"> 立刻推到 Obsidian
        </label>
      </div>
      <div class="panel-error" id="moodPanelError" style="color:#c62828;font-size:12px;margin-top:8px"></div>
    </div>
  `;
},
```

- [ ] **Step 2：PanelBinders 追加 mood**

```js
mood(prefill) {
  let selectedTag = prefill?.data?.tag || "🙂";
  document.getElementById("moodPanelTagGroup").addEventListener("click", (event) => {
    const btn = event.target.closest(".mood-tag-btn");
    if (!btn) return;
    selectedTag = btn.dataset.tag;
    document.querySelectorAll(".mood-tag-btn").forEach((b) => {
      const active = b.dataset.tag === selectedTag;
      b.classList.toggle("active", active);
      b.style.border = `1px solid ${active ? "#1e7b64" : "#ccc"}`;
      b.style.background = active ? "#e8f5e9" : "white";
    });
  });

  document.getElementById("moodPanelSave").addEventListener("click", () => {
    const data = {
      time: document.getElementById("moodPanelTime").value,
      tag: selectedTag,
      note: document.getElementById("moodPanelNote").value.trim()
    };
    const err = EventModel.types.mood.validate(data);
    if (err) {
      document.getElementById("moodPanelError").textContent = err;
      return;
    }
    const editId = prefill?.id;
    const event = editId
      ? { ...prefill, time: data.time, data, capturedAt: Date.now() }
      : EventModel.buildQueueItem("mood", DateUtils.getTodayISO(), data.time, data);
    App.applyEvent(event);
    if (document.getElementById("moodPanelImmediate").checked) {
      App.pushEventsToObsidian([event]);
    }
    UI.showStatus(`已保存：💭 ${data.tag}`);
    PanelManager.close();
  });
},
```

- [ ] **Step 3：浏览器验证 + Commit**

刷新 → 点 💭 → 选 😕 → 加备注 → 保存 → 看 localStorage 今日 entry.moodLogs 有 1 条

```bash
git add index.html
git commit -m "feat(panels): 💭 心情面板（新事件类型）"
```

---

## Phase D · 时间线 + 同步

### Task 14：今日时间线渲染

**Files:**
- Modify: `index.html`（搜索 `renderTodayTimeline()` 那个空占位）

- [ ] **Step 1：实现 `UI.renderTodayTimeline`**

找到 Task 6 加的空 `renderTodayTimeline()` 占位，改成：

```js
renderTodayTimeline() {
  const container = document.getElementById("todayTimeline");
  if (!container) return;
  const today = DateUtils.getTodayISO();
  const pending = SyncQueue.filterByDate(today);

  // 今日所有已录的事件：pending 队列里的（未推）+ 今日 entry 里的已持久化数据推导出的"已推"事件
  // 简化做法：只展示 pending 里的（未推 ⏳）；已推事件无源可查（URL scheme 不返回）。
  // 但用户编辑时仍从 pending 或 entry 取 → 所以时间线只展示 pending 就够，用户的心智是"今天还有几条没同步"。

  if (!pending.length) {
    container.innerHTML = `<div class="timeline-empty">今天还没有记录，点上面的胶囊开始。</div>`;
    return;
  }

  const sorted = [...pending].sort((a, b) => (a.time || "").localeCompare(b.time || ""));
  container.innerHTML = sorted.map((event) => {
    const def = EventModel.types[event.eventType];
    const summary = this.summarizeEvent(event);
    return `
      <div class="timeline-item pending" data-event-id="${event.id}">
        <span class="tl-status"></span>
        <span class="tl-time">${event.time || "--:--"}</span>
        <span class="tl-content">${def.emoji} ${def.label} · ${summary}</span>
      </div>
    `;
  }).join("");
},

summarizeEvent(event) {
  const d = event.data || {};
  switch (event.eventType) {
    case "sleep": return `${d.sleepScore} 分 · ${d.bedtime}-${d.wakeTime}`;
    case "meal": return `${d.type}：${d.food || "无内容"}`;
    case "training": return `${(d.workoutItems || []).length} 个动作`;
    case "recovery": return `${d.type} · +${d.boost}%`;
    case "weight": return `${d.weight?.toFixed(1) ?? "--"} kg`;
    case "reflection": return `${d.mood} · ${d.energy}/10`;
    case "mood": return `${d.tag}${d.note ? " · " + d.note : ""}`;
    default: return "";
  }
},
```

- [ ] **Step 2：`App.applyEvent` 已经调 UI.renderTodayTimeline，不用改**

（Task 6 里已经有 `UI.renderTodayTimeline()`）

- [ ] **Step 3：在 `UI.render(entries)` 里也 call 一次（初始加载）**

找到 `render(entries)` 方法（约 line 4442），在末尾加：

```js
this.renderTodayTimeline();
this.renderPendingCapsule();
```

- [ ] **Step 4：浏览器验证**

刷新 → 之前可能已经在 pending 队列里有几条测试数据。时间线应该列出来，每条按时间升序，带 ⏳ 图标。点一个胶囊记新事件 → 时间线立即新增一条。

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat(ui): 今日时间线渲染"
```

---

### Task 15：待同步胶囊 + 一键推

**Files:**
- Modify: `index.html`

- [ ] **Step 1：实现 `UI.renderPendingCapsule`**

找到 Task 6 加的 `renderPendingCapsule()` 空占位，改成：

```js
renderPendingCapsule() {
  const capsule = document.getElementById("pendingSyncCapsule");
  const countEl = document.getElementById("pendingSyncCount");
  if (!capsule || !countEl) return;
  const count = SyncQueue.countByDate(DateUtils.getTodayISO());
  countEl.textContent = count;
  capsule.hidden = count === 0;
},
```

- [ ] **Step 2：绑定"一键推"按钮**

找到 `bindEvents()` 里 Task 5 加的 captureBar 监听后面，继续加：

```js
document.getElementById("pushAllBtn").addEventListener("click", () => {
  const today = DateUtils.getTodayISO();
  const pending = SyncQueue.filterByDate(today);
  if (!pending.length) {
    UI.showStatus("今天没有待同步事件");
    return;
  }
  App.pushEventsToObsidian(pending);
  UI.showStatus(`已推 ${pending.length} 条到 Obsidian · ${DateUtils.formatShortDate(today)}`);
});
```

- [ ] **Step 3：浏览器验证**

刷新 → 记 2 条不同事件（不要勾"立刻推"）→ 顶部橙色胶囊出现 "待同步到 Obsidian：2 条"→ 点胶囊里的 📤 按钮

Expected:
- Obsidian 被拉起（会跳出协议处理提示，选允许）
- 内容含两条 H3 section
- 胶囊数字变 0、胶囊隐藏
- 时间线变空（pending 已清）

若 Obsidian 未安装：url 写到地址栏后会提示"无法打开"。此时可在 Console 里临时跑：

```js
const pending = SyncQueue.filterByDate(DateUtils.getTodayISO());
console.log(EventModel.renderBatchMarkdown(pending));
```

看输出是否符合 spec §8 的格式。

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat(sync): 待同步胶囊 + 一键推到 Obsidian"
```

---

## Phase E · 编辑已记录事件

### Task 16：点时间线 → 预填面板 → 编辑重入队

**Files:**
- Modify: `index.html`

**Background:** 从时间线上点一条事件 → PanelManager 打开对应面板并 prefill → 用户改完保存 → SyncQueue 里**同 id 就地更新**（`data` 被新值覆盖）→ 没推过的事件不会变成两条、已推过的事件本版不支持编辑（时间线只展示 pending）。

- [ ] **Step 1：绑定时间线点击**

在 `bindEvents()` 里加：

```js
document.getElementById("todayTimeline").addEventListener("click", (event) => {
  const item = event.target.closest(".timeline-item[data-event-id]");
  if (!item) return;
  const id = item.dataset.eventId;
  const existing = SyncQueue.load().find((e) => e.id === id);
  if (!existing) return;
  PanelManager.open(existing.eventType, { prefill: existing });
});
```

- [ ] **Step 2：验证编辑流程**

- 记一条餐食（不勾立刻推）
- 时间线上点该条
- 🍱 面板展开，字段预填了原数据
- 改"吃了什么"为新内容
- 点保存
- 时间线上该条的 summary 变成新内容，**不多一条**（同 id 就地更新）
- `SyncQueue.load()` 里该条的 `data` 是新版本，`capturedAt` 更新

- [ ] **Step 3：Commit**

```bash
git add index.html
git commit -m "feat(edit): 时间线点击预填面板 · 编辑就地更新队列"
```

---

## Phase F · 收尾

### Task 17：删除废弃代码

**Files:**
- Modify: `index.html`

**Background:** 之前 Task 4 只给旧"今日录入"表单加了 `hidden`。现在确认新流程走通，彻底删除对应 HTML 和已无引用的 JS handlers（saveRecord / submit / addMealLog / addRecoveryLog / addNightReview / toggleMainRecordEditor / fillForm 等）。

**⚠️ 这一任务删代码量最大，强烈建议在独立分支做、先手动跑完整测试清单再 commit。**

- [ ] **Step 1：删除 legacy article**

搜索 `<article class="panel" hidden data-legacy="true">`，从它的 `<article>` 到对应的 `</article>` 整块删除（这是原来的"今日录入"表单，约 600 行）。

- [ ] **Step 2：删除 `Handlers` 里已废弃的方法**

在 `const Handlers = {` 里删除这些方法（不再被引用）：
- `submit(event)`
- `saveRecord()`
- `addMealLog()`
- `addRecoveryLog()`
- `addNightReview()`
- `addWorkoutRoutine()`
- `addWorkoutItem()`
- `clearWorkoutItems()`
- `workoutEditorClick` / `workoutEditorInput` / `workoutToggleChange` / `workoutTemplateClick` / `workoutTemplateChange`
- `mealLogInputChange` / `cookedToggleChange`
- `sleepSliderInput` / `weightSliderInput` / `energyBoostSliderInput` / `bedtimeSliderInput` / `wakeTimeSliderInput`
- `resetForm`

**保留：** `recordActions`（旧记录列表的编辑/删除 —— 按旧逻辑可继续工作；或根据需要下一步适配）、`exportJson`、`importTrigger`、`importFile`、`resize`。

- [ ] **Step 3：删除 `bindEvents` 里对应的 addEventListener 行**

删除那些已废弃 handlers 的绑定。保留：form / recordList / export / import / resize。

- [ ] **Step 4：删除 `UI` 里已无引用的方法**

- `fillForm(entry, options)`
- `renderWorkoutTemplates()` + `renderWorkoutEditor(items)`
- `refreshMealDetails()`
- `syncAllSliders()`
- `setFormMode(mode)`
- `resetForm()`
- `toggleMainRecordEditor()`
- `renderMainRecordStatus(entries)`
- `renderDailyAppendList(entries)`（时间线取代它）

**保留：** `render(entries)`、`renderHeroBattery`、`renderTodaySummary`、`renderGrowthDashboard`、`renderMetrics`、`renderReminders`、`renderRecordList`、`renderDataOverview`、`renderTodayTimeline`、`renderPendingCapsule`、`summarizeEvent`、`showStatus`、`escapeHtml`、`formatWeight`、`getTodayMainRecordStatus`。

- [ ] **Step 5：删除 `ensureBootstrapTodayEntry` + 调用**

- 在 `StorageService` 里删 `ensureBootstrapTodayEntry()` 整个方法
- 删 `EntryModel.getBootstrapTodayEntry()` 方法
- 在 `initialize()` 里找到 `state.entries = StorageService.ensureBootstrapTodayEntry();` 删掉这行（保留前面的 `ensureSeedData`）

- [ ] **Step 6：删除对应 CSS**

在 `<style>` 里搜索并删除这些类（不再被任何 HTML 用）：
- `.entry-form`
- `.main-record-status*`
- `.workout-*`（训练模板相关的旧样式）
- `.meal-detail-*`
- `.append-log-*`
- `.daily-log-*`
- `.form-grid`
- `.slider-*`（如果新面板没用到）
- `.toggle-group`
- `.progress-chip-row` / `.progress-chip`
- `.checkbox-chip`

谨慎检查每个类是否真的无引用 —— 可以在 `index.html` 里 Grep 该类名。

- [ ] **Step 7：完整测试清单**

跑完 spec §11 的所有测试场景：
- 7 种事件各保存一次
- 立刻推一条
- 一键推多条
- 编辑历史
- 导入老 JSON（保留老版本文件）
- 跨日（手动改系统日期）

**任何一条失败都暂缓 commit，回滚有问题的 Step。**

- [ ] **Step 8：Commit**

```bash
git add index.html
git commit -m "refactor: 删除旧主表单 + append handlers + bootstrap 逻辑"
```

---

## 完成

**最终 commit 序列（期望）**：
```
Task 0: fix 修复日期硬编码 + submit handler + 编辑展开
Task 1: feat(data): 新增 moodLogs[] 字段
Task 2: feat(sync): SyncQueue 模块
Task 3: feat(events): EventModel 7 种事件
Task 4: feat(ui): HTML 骨架
Task 5: feat(ui): PanelManager
Task 6: feat(app): applyEvent + pushEventsToObsidian
Task 7: feat(panels): 😴 睡眠
Task 8: feat(panels): ⚖️ 体重
Task 9: feat(panels): 🍱 餐食
Task 10: feat(panels): 🛋 补能
Task 11: feat(panels): 💪 训练
Task 12: feat(panels): 🌙 复盘
Task 13: feat(panels): 💭 心情
Task 14: feat(ui): 时间线
Task 15: feat(sync): 胶囊 + 一键推
Task 16: feat(edit): 编辑重入队
Task 17: refactor: 删除废弃代码
```

18 次 commit，每次可回滚。每任务完成后：
1. 手动打开 index.html 验证
2. Git commit
3. 进入下一任务

如果执行到中间发现方案有问题，stop，回到 brainstorming 调整 spec。
