---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40813
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

用 Agent 做日常自动化一段时间后，最常见的困境不是"能力不够"，而是"能力都堆在上下文里"。把 SOP、命令模板、领域知识全部塞进 system prompt，或者注册十几个 MCP server，结果是常驻 token 越滚越大，模型在选择工具时反而更容易犹豫。

OpenClaw 的 Skills 机制解决的就是这个问题：每个技能平时只在上下文里保留一段 `name + description`，当任务描述命中时，Agent 才去读完整的 `SKILL.md` 正文。本质上是一次"渐进式披露"——用一次文件读取换取常驻上下文的大幅缩减。

## 问题

我之前的做法是把日志排查流程、部署检查清单全写进 CLAUDE.md 类的记忆文件。实际后果：

1. 无关任务也要付出这部分 token 成本；
2. 流程之间互相"抢戏"，模型偶尔在错误场景调错流程；
3. 改一处要重启会话才稳定生效。

## 做法与步骤

**1. 建目录**，一个技能一个文件夹：

```
skills/
  log-triage/
    SKILL.md
    scripts/error_stat.sh
    references/alert_thresholds.md
```

**2. 写 SKILL.md**，frontmatter 的 description 是触发开关，按"什么时候用"来写，而不是"我能做什么"：

```markdown
---
name: log-triage
description: 当用户要求排查服务日志、统计错误率、定位报错时间段时使用。包含 grep/awk 模板与告警阈值参考。
---
```

**3. 正文控制在 300 行以内**：写 5–10 步 SOP，确定性步骤封装成脚本放 `scripts/`，长参考资料拆到 `references/`，正文里只留引用路径。

**4. 验证触发**：直接问 Agent "当前可用的 skills 有哪些"，再用一句真实的任务描述做触发测试，观察它是否主动打开了正文。

## 踩坑点

- **description 写成功能介绍**。写"支持日志分析"基本不会触发；写"当用户贴出一段报错日志或要求统计错误率时使用"才稳定命中。要放用户真实会说的关键词。
- **正文贪长**。SKILL.md 写到上千行，等于把 prompt 膨胀问题从"常驻"变成"按次付费"，频繁触发的技能反而更贵。细节拆 references，正文只当路标。
- **技能数量失控**。30 个以上的技能，frontmatter 列表本身就开始吃 token。相近领域果断合并。
- **脚本环境问题**。`chmod +x`、写死解释器绝对路径，别假设 Agent 的工作目录固定——脚本内部用绝对路径或显式 `cd`。
- **密钥泄漏**。SKILL.md 会被读进上下文，任何密钥、内网地址写进去都等于公开，走环境变量。
- **改完没生效**。修改后确认热加载是否生效，必要时重启会话，以 Agent 实际读到的版本为准。

## 可复用建议

- description 三段式模板：**触发场景 + 能力关键词 + 产出物**，一句话说清。
- 坚持三层结构：`SKILL.md` 做路标、`references/` 放细节、`scripts/` 放确定性步骤——凡是"每次执行结果应该一样"的事都脚本化，别让模型每次即兴发挥。
- skills 目录用 git 管理，变更走 commit，出问题能回滚。
- 维护一份固定测试用例（三五句典型任务描述），每次加技能后回放一遍，统计触发率。

## 总结

Skills 的价值不在"堆了多少能力"，而在"按需"。把常驻上下文压缩成几行描述，命中时才付出完整读取的成本；把易变的流程从记忆文件里拆出来，改起来也更干净。实践下来，我的常驻 prompt 瘦了一半以上，工具选择的准确率反而上升——少即是多，在 Agent 工程里是字面意义的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/80e873e90b4a4225.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/69d9a77943bc4115.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/61d834036b5bbdf4.png)

