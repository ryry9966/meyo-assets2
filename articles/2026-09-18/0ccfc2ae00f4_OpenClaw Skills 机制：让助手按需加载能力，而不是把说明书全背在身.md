---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是把说明书全背在身上
feedId: 38016
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

用 OpenClaw 做日常自动化一段时间后，一个很实际的问题会浮出来：能力越接越多，system prompt 越来越胖。浏览器操作、定时任务、MCP server、各种自写脚本，每接一个就往提示词里塞一段说明。结果 token 消耗涨了，模型反而更容易选错工具，因为它要在一大堆指令里"猜"该用哪个。

## 问题

本质上是**能力数量与上下文预算的矛盾**。一次会话里真正用到的能力通常只有一两样，但传统做法是全量常驻。这不仅贵，还稀释了模型注意力。

## Skills 的加载机制

OpenClaw 的 Skills 采用两段式加载，思路类似"目录常驻、正文按需"：

- 每个 skill 是一个目录，核心是一个 `SKILL.md`；
- frontmatter 里的 `name` 和 `description` 会注入 system prompt，占用极小；
- 正文（操作步骤、脚本用法、注意事项）默认不进上下文，只有当模型判断当前任务匹配某个 skill 的描述时，才读取全文。

也就是说，装 20 个 skill 的常驻成本，可能只相当于过去塞一两个完整工具说明。

## 动手做一个最小 Skill

以"生成 PDF 周报"为例，在 `~/.openclaw/skills/pdf-report/` 下建 `SKILL.md`：

```markdown
---
name: pdf-report
description: 当需要把数据或 Markdown 整理成 PDF 报告时使用。触发词：周报、导出 PDF、生成报告。
---

# PDF 报告生成

## 步骤
1. 确认输入文件存在且非空
2. 执行 scripts/build.py --input <path> --out ./report.pdf
3. 输出完成后核对页数，异常则重跑

## 边界
- 输入为空时直接报错，不要生成空报告
```

几个要点：

1. **description 写"什么时候用"，不是"这是什么"**。它是触发的唯一依据，把典型说法（周报、导出 PDF）写进去。
2. 可执行脚本放在同目录的 `scripts/` 里，正文中只写调用方式。
3. 保存后重启 gateway（或用你版本对应的 reload 方式），用 `openclaw skills list` 确认被扫到。
4. 换三种说法测触发："帮我出个周报""把这段整理成 PDF"——都命中才算合格。

## 踩坑记录

- **描述太泛会乱触发**，太窄或缺关键词则永远不触发。宁可迭代两三轮 description，也不要指望模型自己理解。
- **正文别写太长**。触发即加载，写了两千字等于把省下的 token 又花回去。细节拆到 `references/` 单独文件，让模型按需再读。
- **脚本依赖要自查**。skill 常驻的是元信息，真正执行发生在 agent 宿主机上，缺 Python 依赖会"触发成功、执行失败"。脚本开头先做环境检查并给出可执行的安装命令。
- **改了不生效**多半是没重载。Skills 默认在启动时扫描，本地调试期建议固定一个"改完即重启"的动作。
- **和 MCP 工具别重复覆盖**。同一件事既有 MCP tool 又有 skill，模型选择会摇摆。一个能力二选一：需要结构化 API 走 MCP，需要操作流程文档走 skill。

## 可复用的写法建议

- description 固定用"触发条件 + 适用场景 + 触发词"三段式，团队内统一。
- 易变内容（路径、参数、密钥名）放正文或独立文件，frontmatter 保持稳定，避免小改动引发重载验证。
- skills 目录用 git 管理，跨设备同步和回滚都省心。
- 维护一份触发测试清单，每次改 description 就跑一遍。

## 总结

Skills 没有什么黑魔法，本质是**把能力文档化，再配合懒加载**。改动成本很低，收益却直接体现在两个硬指标上：token 开销下降，工具选择准确率上升。如果你正在被膨胀的 system prompt 困扰，建议从一个最痛的重复流程开始，先写一个 skill 跑通闭环，再决定要不要把现有工具说明逐步迁过去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/8e8e0ab3763a4a16.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/c639aebdead39c34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/3bc92bf45f714c9a.png)

