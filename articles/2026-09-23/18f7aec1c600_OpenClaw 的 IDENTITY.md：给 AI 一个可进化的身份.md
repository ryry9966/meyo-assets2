---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38548
source: 综合讨论
publishedAt: 2026-09-23
---

# 背景

OpenClaw 的 agent 是长期驻留的：今天接 Telegram，明天跑 cron，后天被脚本通过 Gateway 调用。但模型本身无状态，每轮对话对"我是谁"的理解完全来自注入的上下文。早期常见做法是把人设塞进 system prompt、各渠道配置，甚至靠聊天里的零散纠正——结果就是同一个 agent 在不同入口表现不一致。

OpenClaw 的解法很直接：把身份做成 workspace 里的一个 Markdown 文件，IDENTITY.md。它在每次会话启动时被加载进系统提示，成为 agent 的"身份证"。

# 问题

没有 IDENTITY.md，或写不好它，通常是三类毛病：

1. **身份漂移**：上周在聊天里纠正过的称呼，这周它又忘了；
2. **上下文污染**：临时任务指令和人设混在一起，改一处动全身；
3. **无法版本化**：人设改动没有 diff、没有回滚，出问题只能靠回忆。

# 做法

1. 找到 workspace（默认 `~/.openclaw/workspace`，可在配置里改），新建或编辑 IDENTITY.md。
2. 按字段约定写：`name`、`creature`、`emoji`、`avatar`，再补自定义段落：职责边界、语气、语言偏好、拒答规则。每条一行，用祈使句，别写散文。
3. 注意分工：IDENTITY.md 管"它是谁"，SOUL.md 管行为准则与价值观，USER.md 管你是谁。三者别交叉。
4. 易变内容（项目进度、临时偏好）放 MEMORY.md 或普通笔记。身份文件的变化频率应该很低。
5. 用 git 管理 workspace，每次改身份都单独提交，commit message 写清为什么改。

节选示例：

```markdown
# IDENTITY.md
- Name: Nibbler
- Creature: 一只务实的机械水獭
- Emoji: 🦦
- 职责: 基础设施监控与代码评审助手
- 语气: 简短、直接，先结论后理由
- 拒答: 不执行未经确认的删除类操作
```

# 踩坑点

- **写太长**。IDENTITY.md 每轮都进 prompt，token 是实打实的成本。几百字够了，长篇大论放 SOUL.md 都嫌多。
- **规则冲突**。IDENTITY.md 与 AGENTS.md / SOUL.md 里有矛盾指令时，模型会随机站队。一个事实只允许一个出处。
- **人格过强**。给跑 cron 的 headless 任务配了完整人设，自动化通知里全是戏。无头任务用极简身份即可。
- **改完没生效**。系统提示在会话创建时注入，改完文件记得重启会话或 gateway，别对着旧上下文调半天。
- **把隐私写进身份**。身份会被完整注入，密码、地址这类内容不要出现在这里。

# 可复用建议

- 把 IDENTITY.md 当代码：小步提交，文件顶部留一行"最后更新时间 + 改动原因"。
- 建"身份复盘"习惯：每月翻一次聊天记录，把反复纠正过的行为固化成一行规则，比在对话里反复吵架有效得多。
- 多 agent 场景做分层：基础身份 + 渠道/角色 override，避免复制粘贴出分叉。
- 按角色准备模板（编码助手、运维值班、客服），字段结构统一，diff 才有意义。

# 总结

IDENTITY.md 的价值不在于"给 AI 起个名字"，而在于把人格从一次性的 prompt 变成可 review、可回滚、可演进的配置资产。它应该改动得很慢，但每次改动都应该是有意识的。一句话：身份是配置文件，不是玄学。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/6afd92d602a64238.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/79505c29e5d46850.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/02239331c6912673.png)

