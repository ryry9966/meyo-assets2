---
title: 让 Agent 真正记住你的偏好：一套可落地的记忆系统设计
feedId: 38120
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

用 OpenClaw 把 Agent 跑起来之后，最先遇到的落差往往不是能力，而是"记性"：明明上周说过提交信息用中文、测试只跑 pytest，新开会话它又来一遍。模型本身是无状态的，所有"记住"都依赖外挂系统。这篇帖子记录我给自己那套 OpenClaw 实例做记忆层的过程，思路可以直接搬到其他 Agent / MCP 项目。

## 问题：三种常见做法为什么不行

1. **全塞进 system prompt**：开始能用，两周后记忆段膨胀到几千 token，互相矛盾的老偏好没人清理。
2. **只靠会话内"你记一下"**：模型口头答应，会话结束即失忆，没有任何持久化。
3. **上来就上向量库 + 抽取 pipeline**：工程量大，而第一批坑通常不在检索，在写入和更新。

## 做法：三层结构 + 受控读写

**第一层：用户档案（explicit）。** 一个 markdown 文件，frontmatter 存元数据，正文是人可读的偏好条目。只收两类：用户明确说过的、手动改过的。放进 git，改动可 diff、可回滚。条目结构大致是：

```yaml
id: commit-style
content: 提交信息用中文，遵循 Conventional Commits
source: explicit      # explicit / inferred
scope: global         # global / project:xxx
confidence: 1.0
updated: 2025-06-01
status: active        # active / stale / pending
```

**第二层：推断偏好（inferred）。** 会话结束时跑一个独立的小抽取 prompt，产出候选条目，带置信度和作用域。低于阈值不落盘；与已有条目冲突时不覆盖，标记 `pending` 等确认。

**第三层：情景记忆（episodic）。** 按项目分目录存任务片段，读取时混合检索：grep 关键词打底，向量召回补充。这层只影响当前任务上下文，不进全局 prompt。

**读写路径：**
- 写：通过 MCP 暴露 `memory_write` 工具，Agent 只能写第二、三层，写操作打日志供审计。
- 读：会话启动时组装记忆块，预算 400–500 token。排序用简单公式：相关度 × 置信度 × 时间衰减，超预算直接截断。
- 改：用户说"别再这样做"时，触发旧条目失效，而不是追加新条目。

## 踩坑点

- **记性太好也是灾难**。初期让 Agent 记录所有纠正，两周后记忆块挤掉一半预算。后来只收"可复用且稳定"的条目，一次性上下文走第三层。
- **推断过度泛化**。某天说"今天别发语音总结"，Agent 推断出"用户讨厌语音"。给 inferred 条目加低默认置信度和作用域限制后才缓解。
- **静默写入最伤信任**。有段时间 Agent 自己改档案，行为变得不可解释。改成对第一层的任何写入都在回复里附带一句"已记录：xxx"。
- **纯向量检索漏精确匹配**。项目代号、命令名靠 embedding 召回不稳，grep 兜底必不可少。

## 可复用建议

1. 先用文件 + 明确写入起步，够用很久；向量库等检索真成为瓶颈再加。
2. 每条记忆带齐四元组：来源、置信度、时间戳、作用域，缺一个后面就没法清理。
3. 抽取用独立小 prompt，别塞进主循环，否则成本和延迟都难看。
4. 给用户一个"查记忆"的命令或插件面板，可见性比聪明的合并算法重要。

## 总结

Agent 记忆不是 prompt 技巧，而是一个带数据治理属性的小工程：分层存储、受控写入、带预算的读取、可审计的变更。我的版本全部用文件实现，两百来行代码，配合 OpenClaw 的 MCP 工具注册即可跑通。先让"记住"这件事透明可控，再谈聪明。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/1aa39d0a40f45a5c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/95dedaef04719b2c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/a86f8301365b641d.png)

