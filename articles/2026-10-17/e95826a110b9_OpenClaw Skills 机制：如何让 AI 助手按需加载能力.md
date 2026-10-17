---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41795
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

给 Agent 接的能力越多，prompt 越臃肿。早期我把 MCP 工具说明、常用操作 SOP、各类约定全塞进系统提示词，几十个工具的 schema 加上两千字工作流描述，对话还没开始，上下文已经吃掉大几万 token。结果模型选工具的准确率反而下降，成本还在涨。

OpenClaw 的 Skills 机制就是为了解决这个问题：能力以"技能"为单位组织，按需加载，而不是全量注入。

## 问题：全量注入是反模式

Agent 的注意力是稀缺资源。把所有能力的细节常驻在系统提示词里，会带来三个后果：token 成本线性上涨、模型在大量相似描述里选错工具、prompt 维护变成灾难。我们缺的不是能力，而是调度能力的方式。

## 做法：三层渐进式披露

Skills 的核心是 progressive disclosure，分三层：

1. **索引层**：会话启动时，只有每个技能的 name + description 进入系统提示词，几十个技能也就几百 token。
2. **正文层**：当任务命中某个技能的 description，Agent 才用 read 工具读取该技能完整的 `SKILL.md`。
3. **细节层**：`SKILL.md` 内部还可以引用脚本、模板、补充文档，这些文件只在执行时才被读取。

也就是说，常驻的是"目录"，不是"正文"。

落地一个技能，以我写的周报整理技能为例：

1. 建目录：个人技能放 `~/.openclaw/skills/weekly-report/`，随项目走的放 workspace 下的 `skills/` 目录，内含一个 `SKILL.md`。
2. 写 frontmatter，重点打磨 description：

```yaml
---
name: weekly-report
description: 当用户要求整理周报、汇总本周工作、生成周报草稿时使用。触发词：周报、本周总结、weekly report。
---
```

3. 正文写清楚：适用场景、执行步骤、输入来源（比如从哪个目录读日志）、输出格式、禁止事项。判断类逻辑用自然语言写，确定性步骤直接让它调用技能目录里的脚本。
4. 重启会话后用技能列表命令（如 `/skills`）确认已被索引，再用一句自然的话测触发。

## 踩坑点

- **description 写成功能说明而不是触发条件**。我第一版写"用于整理周报的工具"，几乎不触发；改成包含用户真实会说的话之后，命中率明显提升。description 本质是路由 key，是给模型做匹配用的。
- **SKILL.md 写太长**。有次正文写了两千字，等于把 prompt 臃肿换个地方。原则：正文只留判断逻辑和流程骨架，细节下沉到引用文件。
- **技能职责重叠**。两个 description 都含"总结"，模型随机选一个。后来按输出场景拆边界，一个技能只管一类结果。
- **脚本路径问题**。技能内引用脚本要用相对技能目录的路径，写成绝对路径换台机器就挂。
- **把 Skills 当代码**。`SKILL.md` 是指令不是程序，需要确定性输出的部分（格式化、统计）应该配真实脚本让模型调用，而不是让它"照着描述心算"。

## 可复用建议

- description 模板：`当用户要 X / 提到 Y / 给出 Z 时使用`，把触发词直接写进去。
- 单一职责，宁多勿混；`SKILL.md` + frontmatter 是通用约定，技能可以跨项目复用。
- 判断交给 prose，执行交给脚本，两者在同一技能里配合。
- 技能目录进 git，description 的每次改动当作一次路由调优来 review。
- 定期清理：不再触发的技能果断删，索引层也是要花 token 的。

## 总结

Skills 机制的本质，是承认 Agent 里最稀缺的资源不是能力，而是上下文和注意力。把能力做成目录 + 按需取用的文档包，模型才能在有限的认知预算里做对选择。写技能的功夫，七分花在 description 上，不为过。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/73586c6f95fe6e32.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/ece8c98a3b7eaf3e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/d554026a9143d70f.png)

