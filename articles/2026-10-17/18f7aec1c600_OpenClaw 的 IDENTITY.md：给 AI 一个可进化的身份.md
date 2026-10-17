---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41805
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

用过一段时间 OpenClaw 的人都知道，agent 的 workspace 里躺着几个核心 Markdown 文件：SOUL.md、IDENTITY.md、USER.md、MEMORY.md。很多人只在初始化时瞥了一眼 IDENTITY.md，填个名字就再没打开过。这篇帖子想聊的是：为什么值得把它当成一项长期维护的配置资产，而不是一次性登记表。

## 问题

不认真维护 identity，常见三个症状：

1. **人格漂移**：周一精炼克制，周五满屏表情包，你却说不清哪里变了。
2. **调教全靠吵架**：不满意就当场在对话里训斥模型，效果只维持几轮。
3. **多渠道不一致**：同一个 agent 在 Telegram 和邮件里像两个人。

根因是身份被散落在系统提示、临时对话和记忆里，缺少一个可版本化、可 review 的单一事实来源。

## 做法

IDENTITY.md 就是那个单一事实来源，位于 workspace 根目录（默认 `~/.openclaw/workspace/`）。建议这样起步：

```markdown
# IDENTITY.md
- 名字：Claw
- 一句话定位：工程师的日常副驾驶
- 语气：先结论后细节，不堆形容词
- 边界：不确定就直说"我不确定"，不替用户做承诺
- 工具习惯：执行有副作用的操作前，先简述打算做什么
```

然后是三条工程化习惯：

1. **workspace 纳入 git**。每次调 persona 就是一次 commit，diff 一眼看出"它为什么变啰嗦了"。
2. **保持声明式**。写"是什么"，不写"当用户说 X 时你要 Y"——流程类规则归 AGENTS.md，身份只管稳定特质。控制在 30 行以内，形容词越少，遵循越稳。
3. **建立进化回路**。发现不满意的言行，不当场骂，先记下来，周末改成一行具体约束，commit 注明原因。每月花十分钟 review 一次，像开 retro。

## 踩坑点

- **文件太长**。写过 80 行"性格说明书"，模型开始自相矛盾，砍到 20 行反而听话。
- **和 SOUL.md / USER.md 抢地盘**。身份文件只写"它是谁"，用户偏好写 USER.md，事件写 MEMORY.md，混写必然漂移。
- **热编辑不生效**。部分运行方式下，改完文件需要重置会话或重启 gateway 才会重读。别误判为"模型不听话"——它根本没看到新文件。
- **让 agent 自己改自己**。开放自编辑很诱人，但没有 review 的自我修改是漂移加速器。至少要求它先在对话里 propose diff，由你确认后再落盘。

## 可复用建议

- 把 IDENTITY.md 当代码对待：小步 diff、有 commit message、有人 review。
- 团队场景：在 repo 里放一份身份模板，新 agent 初始化时填空，保证多 agent 的风格基线一致。
- 渠道差异不要写进身份，放到各渠道配置里做增量提示，身份保持唯一。
- 身份不只影响聊天语气，也影响工具调用风格（是否先汇报再执行、如何报错），值得专门写一行。

## 总结

IDENTITY.md 便宜到容易被忽视，但它是把 agent 从"每次会话重来"变成"长期共事者"的那块拼图。它的价值不在一次写对，而在可进化：小 diff、有版本、可回滚。下次发现 agent 行为不对，先别在聊天框里发火，去 workspace 改一行，commit，然后观察一周。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/8656a84b1eb2d0fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/07d3153a61c1f3a3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/c610cf419c765949.png)

