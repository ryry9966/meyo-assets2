---
title: Agent 记忆系统设计：让助手真正记住你的偏好
feedId: 38264
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 默认的记忆机制很朴素：workspace 里的 MEMORY.md 做长期索引，memory/ 目录按天记流水，会话压缩时靠摘要续命。日常够用，但当你希望助手“记住我偏好 pnpm、commit message 用中文、别在回复里贴大段日志”时，纯文本流水很快失效——要么记不下来，要么捞不出来。这篇聊聊我折腾一段时间后沉淀下来的一套可落地设计。

## 问题：三种常见失败模式

1. **全塞进 SOUL.md / system prompt**：偏好越攒越多，token 持续膨胀，且所有会话都被污染，不相关的项目也生效。
2. **只靠对话历史检索**：历史里九成是过程噪音，召回回来的多半没用，还挤占上下文。
3. **让模型自由发挥地“记”**：无结构、无去重、无失效机制，三个月后记忆库内部互相矛盾，模型随机站队。

## 做法：分层 + 双通道读写

先把记忆拆成三类：

- **偏好层（profile）**：结构化键值，如 `package_manager: pnpm`、`reply_language: zh`，挂 namespace 作用域（全局 / 某项目）。
- **事实层（semantic）**：关于环境的稳定事实，如“这个仓库用 Node 20，测试跑 vitest”。
- **事件层（episodic）**：按天流水，只作召回素材，不直接进 prompt。

**写入侧**：不要在主对话里顺手记。用独立的 memory-write 提示词，在会话结束或显式触发时抽取，强制 schema：

```json
{
  "type": "preference | fact | episode",
  "scope": "global | project:xxx",
  "key": "package_manager",
  "value": "pnpm",
  "confidence": 0.9,
  "updated_at": "..."
}
```

confidence 低于 0.7 不落盘；同 key 新值**覆盖**旧值（supersede），不是 append。

**读取侧**：构建 prompt 时按 scope 硬过滤。偏好层小而稳定，可全量注入；事实层、事件层走召回，给记忆留固定 token 预算（建议 ≤ 上下文 5%），按相关度 × 时间衰减 × 命中频率排序。

**遗忘**：每条记忆带 `updated_at` 和 `hit_count`，超 60 天未命中且置信度低的进归档，不删只降权。

## 踩坑点

- **一次性指令不是偏好**。“这次用 3000 端口”不该进偏好层，写入提示词里必须明确区分“就这一次”和“以后都这样”。
- **冲突没处理，等于没记**。必须显式 supersede，旧值标记 outdated 留档，否则两个版本的偏好同时在库里。
- **召回范围过宽是隐性问题**。跨项目串记忆后，助手会在 A 项目里搬 B 项目的约定。namespace 隔离要硬做，不能靠提示词求它。
- **记忆不可见最致命**。用户不知道助手记了什么、为什么这么做，信任直接崩。保持明文文件、可查看可手改。
- **别一上来就上向量库**。SQLite + FTS5 在千条规模内完全够用，先跑通写入-召回-纠错闭环。

## 可复用建议

- 偏好层控制在 50 条以内，超了说明该合并语义了。
- 建一个 10 条左右的召回评测集（“这种场景应该想起来”），每次改动后跑一遍，防止越调越聋。
- 写入频率宁低勿高：错误记忆的代价远大于漏记。
- 用 MCP 把记忆暴露成 read / write / list 三个工具，list 必须能读原文，方便排查和人工修正。

## 总结

记忆系统的难点不在存储，而在**写入纪律和读取边界**：结构化 schema、显式覆盖规则、硬性 scope 隔离、用户可见可改。这四件事做对之后，向量检索只是锦上添花。建议从 OpenClaw 现有的 memory 文件起步，外面套一层 schema 和 namespace，一天内就能跑通最小闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/a2d34e4f476d4b2c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/2339a2a372c34585.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/fcf4b28360d1999d.png)

