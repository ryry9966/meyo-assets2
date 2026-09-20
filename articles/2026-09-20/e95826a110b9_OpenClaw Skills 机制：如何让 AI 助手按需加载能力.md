---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38256
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

用 OpenClaw 做长期运行的 agent，能力边界会不断扩张：MCP 工具、定时任务、内部脚本、团队约定。如果把这些操作说明全部塞进 system prompt，很快会遇到三个问题：上下文持续膨胀、指令互相干扰、模型注意力被稀释——它反而记不住任何一条。

Skills 机制就是为了解决这件事：把"某类任务怎么做"封装成独立目录，平时只在提示词里保留一行描述，真正用到时才把正文加载进上下文。这就是所谓的渐进式披露（progressive disclosure）。

## 问题

实际使用中常见两类失败：装了一堆 skill，模型从不主动用；或者 skill 写得太重，一触发就把几千 token 灌进上下文。根源是没有理解它的两层结构——`SKILL.md` 的 frontmatter（`name` + `description`）常驻上下文充当索引，正文按需读取。索引写不好，加载机制等于白搭。

## 做法

1. 在 workspace 的 skills 目录下建一个文件夹，写 `SKILL.md`，frontmatter 示例：

```yaml
---
name: log-triage
description: 当用户要求排查服务日志、定位报错原因时使用。输出按时间线整理的结论。
---
```

2. 正文写三样东西：触发条件、执行步骤、约束（比如哪些操作需要先确认）。
3. 长参考资料（参数表、runbook）拆成同目录下的独立文件，正文里只写"需要时读取 `references/xxx.md`"。
4. 重载 gateway，用一个真实任务验证：观察 agent 是否主动引用该 skill。
5. 不触发就改 description 的措辞；误触发就补"仅在……时使用"的限定。

## 踩坑点

- **description 写成功能清单，而不是触发条件。** 模型靠语义匹配决定是否读取正文，"能做 A/B/C"远不如"当出现 X 场景时使用"有效。
- **正文写成百科全书。** 一加载就是几千 token，不如外置成参考文件，正文只留主干流程。
- **把需要确定性结果的操作写成纯文字流程。** 重复性操作应配脚本：skill 只说明何时、如何调用脚本，脚本保证幂等和输出稳定。
- **多个 skill 的 description 语义重叠。** 会互相抢触发，命名和措辞要能划清边界。
- **frontmatter 格式错误导致 skill 静默失效。** YAML 缩进错了不报错，只是列表里没有它。改完用诊断命令确认 skill 出现在已加载列表中。

## 可复用建议

- 一个 skill 只做一类事，名字即边界。
- description 套模板："**\<触发场景\>时使用。做\<什么\>，产出\<什么\>。**"
- 分工原则：流程性知识进 skill，数据接口走 MCP。MCP 给 agent "手"，skill 给 agent "手册"，两者互补而非替代。
- 把 skills 目录纳入 git：团队共享、变更可回滚、出问题能 diff。
- 每个 skill 配一个固定的验证 prompt，改版后做回归测试，防止措辞一改触发就失效。

## 总结

Skills 的价值不在"装了多少"，而在"按需加载"。它的本质是上下文经济学：常驻的只有一行索引，昂贵的正文只在需要时支付成本。实践中，写好一行 description 比堆十个 skill 更重要——先让模型在正确的时机想起它，再谈内容质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/ace69a861fa2a979.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/45edb9f1356cda83.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/15e086d2978d9cbb.png)

