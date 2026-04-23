# AGENTS.md

## 通用定位

- 这是 Michael 的通用代理协作文件，可以复制到微信小程序、公司网站、个人工具、自动化脚本等不同项目中使用。
- 这个文件优先记录“长期可复用的工作方式”和“可调用的 skills”，项目特定规则放在最后单独维护。
- 新项目开始前，先阅读本文件，再阅读项目自己的 README、配置文件和代码结构。
- 修改代码前先理解现有数据结构、部署方式和用户真实使用路径，避免为了改功能破坏已有工作流。
- 涉及不可逆操作、删除数据、改部署、改账号权限时，必须先确认。

## 通用工作方式

- 先定位问题，再动手修改。
- 小步提交，说明改了什么、为什么改、如何验证。
- 不覆盖用户已有数据和未提交改动。
- 能本地验证就先验证，再部署。
- 如果有 AGENTS.md、README、技能文件、项目说明冲突，优先遵守更靠近当前项目根目录的规则。

## 通用 Skills

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
- 核心原则：删除填充短语和套话。
- 核心原则：打破公式化结构，避免三段式、金句式、过度升华。
- 核心原则：混合句子长短，让节奏更像真人。
- 核心原则：保留原意，但加入更具体、更有人味的表达。
- 核心原则：避免宣传腔、模糊归因、过度连接词和“不是……而是……”式排比。

## Skill 使用规则

- 当任务明显匹配某个 skill 时，先读取对应 `SKILL.md`，再按技能说明执行。
- 如果用户只提到技能名但目标不明确，按当前任务合理判断，不要求用户重复说明。
- 后续新增 skills 时，按同样格式加入“通用 Skills”区，记录来源、触发、用途和注意事项。

## 当前项目规则：Michael 能量追踪器

- 当前项目是 Michael 的个人能量追踪器，核心是把睡眠、饮食、训练、工作、情绪和 Obsidian 记录串成一个轻量系统。
- 主文件是 `index.html`，这是单文件前端应用，数据保存在浏览器 `localStorage`。
- 修改时必须注意兼容旧数据字段，避免破坏已有手机记录。
- Obsidian 写入依赖 `obsidian://new` URL scheme，改保存逻辑时要确认“本地保存”和“发到 Obsidian”都生效。
- GitHub Pages 是当前线上部署方式，推送后要检查 Pages 构建状态和线上 HTTP 状态。
- 做代码修改后，至少运行一次内联脚本语法检查：

```bash
/bin/zsh -lc 'awk "/<script>/{flag=1;next}/<\/script>/{flag=0}flag" index.html > /tmp/michael-energy-tracker-check.js && node --check /tmp/michael-energy-tracker-check.js'
```
