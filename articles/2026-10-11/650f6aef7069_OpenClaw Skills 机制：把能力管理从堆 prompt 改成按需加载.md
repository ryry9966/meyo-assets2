---
title: OpenClaw Skills 机制：把能力管理从堆 prompt 改成按需加载
feedId: 41235
source: 综合讨论
publishedAt: 2026-10-11
---

# 背景

OpenClaw agent 的能力来自三层：模型本身、MCP 工具、以及 Skills。不少人初装时习惯把所有说明塞进 system prompt——工具用法、话术规范、领域知识全堆一起。跑几周后 prompt 涨到上万 token，出现两个典型症状：模型开始“忘事”，中间的指令经常被忽略；成本上去了，触发反而不准。

Skills 机制就是冲着这个问题来的：能力不预装，按需加载。注意它和 MCP 不冲突——MCP 提供“能调用的手”，Skills 提供“什么时候调、怎么调”的说明书。

# 问题本质

核心矛盾是 context 预算有限 vs 潜在能力无限。50 个技能的说明书不可能常驻上下文，但 agent 又得知道有哪些技能存在。解法是渐进式披露，分三级：

1. 启动时只注入每个 skill 的 name + description，几十 token 一个；
2. agent 判断当前任务命中某个 skill，主动读取完整 SKILL.md；
3. skill 引用的脚本、模板等资源，执行时才碰。

# 做法

目录结构，一个能力一个目录：

```
skills/
  pdf-report/
    SKILL.md
    scripts/render.py
    assets/template.md
  web-clip/
    SKILL.md
```

frontmatter 示例：

```yaml
---
name: pdf-report
description: 当用户要求把网页/表格导出为 PDF 周报，或提到"周报""导出 PDF"时使用。先抓取数据，再调用 scripts/render.py。
---
```

步骤：
1. 在 `~/.openclaw/skills/` 下建目录；
2. frontmatter 只写 name 和 description；
3. 正文按“何时用 → 步骤 → 边界情况”组织，控制在 200 行内；
4. 可执行逻辑放 scripts，SKILL.md 里写清调用方式和前置依赖；
5. 开新会话，用几个真实问法验证触发。

# 踩坑点

- **description 决定生死。** 写“这是一个 PDF 工具”基本不会触发；要写成“用户要求 X 或提到 Y 时使用”，把真实口语里的关键词放进去。
- **技能职责重叠会互相抢触发。** 两个 skill 都声称处理“导出”，模型会随机选一个。先在纸面上划清边界再动手。
- **SKILL.md 本身也占 context。** 写 800 行等于把问题换个地方。目标应是读完就能干活，细节外移到资源文件。
- **脚本依赖是大头。** 绝对路径、Python 版本、权限问题占了我调试时间的大部分，SKILL.md 里务必写依赖检查步骤。
- 改完 skill 记得开新会话验证，旧会话可能还拿着缓存的清单。

# 可复用建议

- 把 description 当“检索 query”写：动作 + 场景 + 关键词，用真实用户原话测试。
- 一个 skill 只做一件事，宁可多建几个小的。
- 每个 skill 配 2~3 条触发用例，改动后跑一遍，防止回归。
- 通用逻辑下沉到共享脚本，skill 只留编排。

# 总结

Skills 机制的价值不在于“能装插件”，而在于把能力管理从“堆 prompt”变成“建索引”。清单常驻、正文按需、资源懒加载——这三层结构让能力数量增长时，context 成本基本持平。先写好 description，再谈其他。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/a6adb06bc89cf540.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/1e393bb310486363.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/8e51ada8e44044ec.png)

