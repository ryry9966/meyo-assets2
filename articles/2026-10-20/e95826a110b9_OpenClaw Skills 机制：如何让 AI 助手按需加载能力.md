---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42048
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 用到后期，接的 MCP server 和自定义工具越来越多，一个典型的现象是：system prompt 里的能力清单越写越长，每次会话都把全部工具定义塞进上下文。结果就是 token 成本上升、模型在几十个工具里挑错、响应变慢。Skills 机制就是为了解决这个问题：把「能力」从「常驻上下文」改成「按需加载」。

## 核心思路

一个 Skill 本质上是一个目录：

- `SKILL.md`：带 frontmatter（name、description）。平时模型只能看到这份轻量索引里的 description；
- 若干支撑文件：详细指令、模板、脚本。

任务匹配到某个 description 时，模型才读取完整 `SKILL.md`；里面引用的脚本和深层文件，只在真正需要时进一步加载。这就是渐进式披露（progressive disclosure）——索引常驻，细节下沉。

## 具体做法

1. 项目下建 `skills/` 目录，一个子目录一个 skill；
2. 写 `SKILL.md`，frontmatter 的 description 用「什么时候用我」的句式，例如：*「当用户要求把数据库查询结果整理成周报表格时使用」*；
3. 正文控制在 500 行以内：确定性、流程性的操作写成脚本引用，`SKILL.md` 只留判断逻辑和入口；
4. 在 OpenClaw 配置里注册 skills 目录，重启会话后用一句自然语言触发验证，确认能命中。

## 踩坑点

- **description 写成「这是什么」而不是「什么时候用」**。模型靠语义匹配选 skill，不是靠名字。写成名词解释后触发率明显下降。
- **细节全堆进 SKILL.md**。曾把一个 skill 写到上千行，模型读了前 200 行就开始自由发挥；拆成三个引用文件后输出稳定很多。
- **职责重叠**。两个 description 都含「生成报告」，模型会随机挑一个，行为不可复现。边界要切干净。
- **脚本路径问题**。忘了加执行权限、或写死绝对路径，换机器就失效。统一使用相对 skill 目录的路径。
- **skill 数量失控**。超过三四十个后，索引本身又开始膨胀，该归并的归并。

## 可复用建议

- 一个 skill 只做一件事，宁可拆小；
- 确定性操作用脚本，模糊判断用自然语言指令；
- skill 入 git 管理，改动有据可查，方便回滚；
- 定期审计：哪些 skill 三个月没被触发过？要么 description 写得不好，要么真没用了。

## 总结

Skills 的价值不在于「能力更多」，而在于「上下文更干净」。把索引做薄、把触发条件写清、把细节下沉到按需加载的文件，Agent 的工具选择准确率和运行成本都会有可感知的改善。建议从现有 prompt 里挑一段高频流程先迁移成 skill 试水，再逐步铺开。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/199ebd22d29497a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/e8fbf09635186e21.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/bf3e01ba21d44059.png)

