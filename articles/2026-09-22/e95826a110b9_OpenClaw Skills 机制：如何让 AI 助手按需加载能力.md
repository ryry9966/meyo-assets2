---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38491
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

Agent 框架最常见的问题就是上下文膨胀：为了让助手"什么都会"，把所有工具说明、操作手册、领域知识一股脑塞进 system prompt。结果上下文动辄几万 token，模型注意力被稀释，真正用到的能力不到一成，而且每轮对话都在为闲置内容付费。

OpenClaw 的 Skills 机制用"渐进式披露"（progressive disclosure）来解决这个问题：会话启动时只注入每个技能的元数据（name + description，各一行），完整的 SKILL.md 正文只有当模型判断当前任务相关时才会读取。本质上是把能力说明书从常驻内存改成了按需分页。

## 做法

1. **建目录。** 每个技能一个独立目录，入口固定是 `SKILL.md`，放在 agent workspace 的 skills 目录下。
2. **写 frontmatter。** 关键字段只有 `name` 和 `description`。description 是模型决定"要不要加载"的唯一依据，必须写成触发条件的描述，而不是功能宣传。例如："use when the user asks to export weekly reports to Excel, generate xlsx files, or format spreadsheets"——具体动词、具体产物、具体文件类型。
3. **写正文。** SKILL.md 正文控制在 200 行以内，只写模型不知道的事：本项目的路径约定、命令用法、边界条件。模型已知的通用知识不要重复。
4. **挂辅助文件。** 确定性操作写成脚本（如 `scripts/export.py`），正文里只写"何时调用、参数含义"。大段参考资料放 `references/`，正文指引模型按需读取。
5. **重启验证。** 用一句明显触发该技能的话测试，看日志确认 SKILL.md 是被触发后才读取的，而不是启动时全量加载。

## 踩坑点

- **description 写成营销文案。**"强大的 Excel 处理技能"不会触发任何东西；"export xlsx / pivot table" 才会。
- **把全部逻辑塞进 frontmatter。** 元数据是每轮对话的常驻成本，塞得越多亏得越多。
- **一个技能干三件事。** 触发条件混在一起，模型要么全加载要么不加载，拆成三个小技能命中率更高。
- **脚本路径写死。** 绝对路径换台机器就失效，用相对 workspace 的路径，并确认可执行权限。
- **技能数量失控。** 几十个技能的元数据本身就是一份不小的索引，定期删掉三个月没触发过的。

## 可复用建议

- description 模板："use when \<具体场景\>, involving \<具体文件/命令/产物\>"。
- 能用脚本表达的不要用自然语言描述，脚本是零歧义的。
- 把 skills 目录放进 git，改动走 commit，可回溯、可回滚。
- 排查"技能不触发"时，先看日志里模型实际看到的元数据列表——多数情况是 description 没写对，而不是机制坏了。

## 总结

Skills 机制的核心价值不是"能力更多"，而是把能力的加载成本从常驻改为按需。写好一个技能的投入，80% 应该花在那两行 description 上——它直接决定这个技能到底会不会被用到。先从把你最常用的一段操作手册拆成一个小技能开始，观察触发日志，再逐步迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/2d78701f11c05047.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/9bda8c78dc6d2a81.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/d7d304f542ec1bee.png)

