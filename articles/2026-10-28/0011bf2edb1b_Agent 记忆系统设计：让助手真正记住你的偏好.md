---
title: Agent 记忆系统设计：让助手真正记住你的偏好
feedId: 42863
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

LLM 本身无状态，每次调用都是"失忆"的。Agent 框架解决的是编排问题，记忆得自己搭。社区里常见做法是接一个向量库就宣布"有记忆了"，用两周就会发现：助手记得你上周聊过的项目，却始终记不住"commit message 用中文"这种最基本的偏好。

## 问题

偏好记忆和普通对话记忆性质不同：

1. 偏好是小而稳定的结构化事实，向量化后语义信号很弱，"不要用 any"这类否定式偏好检索命中率很低；
2. 全量塞进 system prompt，token 成本线性上涨，注意力被稀释，模型反而更容易忘；
3. 偏好会演变（换编辑器、换包管理器），只增不删的记忆库最终变成噪声源。

## 做法

**1. 分层。** 我把记忆拆成三层：工作记忆（当前会话上下文）、情景记忆（做过什么事，走向量检索、按需召回）、偏好记忆（稳定的用户事实，数量少，会话启动时全量注入）。关键是把偏好和情景分开存储，别混在同一个 collection 里。

**2. 抽取。** 会话结束后跑一个轻量抽取 pass，判断本轮是否出现偏好信号，输出结构化条目：

```json
{
  "type": "preference",
  "content": "commit message 用中文，conventional commits 格式",
  "source": "explicit",
  "confidence": 0.9,
  "evidence": "用户明确纠正过一次",
  "status": "active",
  "created_at": "2025-01-12",
  "last_hit_at": null
}
```

`source` 区分 explicit（用户明说）和 implicit（行为推断），implicit 初始 confidence 压到 0.5 以下。

**3. 注入与召回。** 会话启动时取 status=active 的条目按 confidence 排序，前 30 条直接拼进 system prompt——总量通常不到 1k token，比重检索可靠得多。向量检索只留给情景记忆。

**4. 冲突处理。** 新证据与旧条目矛盾时不覆盖，把旧条目置为 deprecated 并记录被谁替代，保留追溯链。用户显式纠正（"我说过用 pnpm"）视为最高置信信号，立即写入并提升优先级。

**5. 生命周期。** 每次命中更新 last_hit_at，连续 30 天未命中的 implicit 条目自动降权，人工确认后清理。

我把读写封装成 MCP tool（memory.search / memory.write），在 OpenClaw 侧以插件形式注册，换客户端也不用重写记忆层。

## 踩坑点

- **implicit 推断太激进**：用户用了一次 vim 就记下"偏好 vim"，比不记更烦。低置信条目应在下次自然出现时向用户确认一句，再转正。
- **记忆条目不记来源**：三个月后没法判断哪条能删，只能全库清空重来。
- **把记忆当可信输入**：记忆内容源自对话，可能被注入。拼进 prompt 前过一遍白名单字段，别把自由文本原样注入。
- **没有回归测试**：留一组固定场景（模拟某用户画像跑十轮对话），断言偏好被遵守，否则记忆逻辑改一版崩一版且无感知。

## 可复用建议

- 个人 Agent 先别建系统，一个 markdown 偏好文件 + 启动时注入，能覆盖 80% 需求；
- 要升级时先加"结构化偏好层"，最后才考虑向量化；
- 每条记忆必须带 source / confidence / 时间戳三件套；
- 给用户一个查看和编辑记忆的入口（哪怕只是 /memory 命令），可解释性决定信任度。

## 总结

偏好记忆的核心不是"存得下"，而是"取得准、错能改、可审计"。分层存储解决取得准，置信度与废弃机制解决错能改，来源记录加回归测试解决可审计。先从最土的方案跑通闭环，再逐步加复杂度，是这类系统最省力的路径。欢迎在评论区交流你们记忆层踩过的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/78c2cf64a3e8d4f5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/34673088c4ec99b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/3665823ebceb60ca.png)

