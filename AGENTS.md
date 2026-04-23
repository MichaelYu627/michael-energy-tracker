# AGENTS.md

## 项目协作规则

- 这是 Michael 的个人能量追踪器，核心是把睡眠、饮食、训练、工作、情绪和 Obsidian 记录串成一个轻量系统。
- 修改前先理解现有 `index.html` 的单文件结构，避免破坏 `localStorage` 数据兼容、Obsidian 写入和 GitHub Pages 部署。
- 做代码修改后，至少运行一次内联脚本语法检查：

```bash
/bin/zsh -lc 'awk "/<script>/{flag=1;next}/<\/script>/{flag=0}flag" index.html > /tmp/michael-energy-tracker-check.js && node --check /tmp/michael-energy-tracker-check.js'
```

## 可用技能

### Caveman

- 来源：`/Users/yusheng/.claude/skills/caveman/caveman/SKILL.md`
- 触发：用户说“caveman mode”、“use caveman”、“less tokens”、“be brief”、`/caveman`，或明确要求压缩表达。
- 用途：进入超压缩沟通模式，减少废话和 token，但保留技术准确性。
- 默认强度：`full`。
- 可切换强度：`/caveman lite`、`/caveman full`、`/caveman ultra`、`/caveman wenyan-lite`、`/caveman wenyan-full`、`/caveman wenyan-ultra`。
- 停止：用户说“stop caveman”或“normal mode”。
- 注意：安全警告、不可逆操作确认、复杂多步说明需要恢复清晰表达，完成后再继续 caveman 风格。

### Humanizer-zh

- 来源：`/Users/yusheng/.claude/skills/Humanizer-zh/SKILL.md`
- 触发：用户要求润色、改写、去 AI 味、变自然、变像人写，尤其是中文文本。
- 用途：识别并去除 AI 写作痕迹，让中文更自然、更有真实语气。
- 核心原则：
- 删除填充短语和套话。
- 打破公式化结构，避免三段式、金句式、过度升华。
- 混合句子长短，让节奏更像真人。
- 保留原意，但加入更具体、更有人味的表达。
- 避免宣传腔、模糊归因、过度连接词和“不是……而是……”式排比。

## 使用方式

- 当任务明显匹配以上技能时，先读取对应 `SKILL.md`，再按技能说明执行。
- 如果用户只提到技能名但目标不明确，先按当前任务合理判断，不要求用户重复说明。
