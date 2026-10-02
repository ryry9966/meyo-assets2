---
title: 不等你开口：给 OpenClaw 助手加 proactive 能力的工程笔记
feedId: 40166
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

多数人用 Agent 还是"问答模式"：人发起，Agent 响应。但 OpenClaw 的架构其实给 proactive 留了口子——heartbeat 定时唤醒、webhook 事件回调、MCP 工具暴露的状态，都能当触发源。问题从来不是"能不能做"，而是"怎么做得不像个话痨"。

## 问题

Proactive 的本质不是"多说"，而是"在对的时机说"。落地时要回答三个问题：

1. **何时触发**——事件从哪来，可不可靠；
2. **值不值得说**——价值判断，避免通知疲劳；
3. **敢不敢做**——权限边界，避免自动执行出事故。

## 做法

我落地了三类场景：CI 失败告警、每日摘要、证书/磁盘阈值检查。结构统一，五步：

**1. 盘点事件源。** 三类：定时（heartbeat，低频，如每 30 分钟）、事件驱动（GitHub webhook 等）、状态轮询（包成 MCP tool，查磁盘、查队列积压）。原则：webhook 能覆盖的不用轮询，轮询能满足延迟的最小频率即可。

**2. 定义触发契约。** 每个触发携带结构化 payload：source、时间、事件类型、紧急度。别把原始日志直接喂给模型，先过滤截断，成本和误判率都会降。

**3. 评估与行动分离。** 第一次调用只判一件事："该不该说"，输出 speak/silent 加理由。silent 直接落日志，不惊动人；只有 speak 才进入第二阶段执行动作。这一刀切下去，噪音能砍掉八成。

**4. 给动作上预算。** Proactive 会话独立限额（如最多 5 次工具调用），敏感工具走白名单。删除、部署、支付类操作一律 dry-run 或要求确认，绝不自动执行。

**5. 留反馈通道。** 每条主动消息带来源标签，用户回一句"这类别报了"就能写回 mute 规则。

一条最小规则大致长这样：

```yaml
- name: ci-failure-alert
  trigger: webhook/github
  condition: run.conclusion == failure && repo in watchlist
  evaluate: require tool evidence, no speculation
  act: notify with failed job link
  budget: { max_tool_calls: 5, quiet_hours: "23:00-08:00" }
```

## 踩坑点

- **心跳频率 vs token 成本**：早期 5 分钟 heartbeat 带全量上下文，一周 token 翻了几倍。改成 heartbeat 只带 diff 和触发 payload，无事可做就直接短路返回。
- **幻觉式"报警"**：模型曾凭一次请求超时就推断服务挂了。后来加硬规则：主动发言必须引用工具输出作为证据，无证据的推断一律 silent。
- **webhook 重试导致重复触发**：同一事件 redelivery，通知发了两遍。按 event id 去重，已处理 id 保留 24 小时。
- **深夜轰炸**：积压的不重要事件凌晨三点全吐出来。加静默时段缓冲队列，能等到早上的事就攒到早上。

## 可复用建议

1. 从 1–2 个只读、高价值场景起步（CI 失败、证书过期），先建立信任，再谈写操作。
2. 规则配置化而非硬编码：每条规则 = trigger + condition + action + budget，可审查、可一键 mute。
3. 盯两个指标：触发次数和发言率。发言率长期低于 20%，说明规则太吵，先收紧 condition 而不是加新规则。
4. Proactive 会话与主会话上下文隔离，防止后台巡检污染你正在处理的主任务。

## 总结

Proactive 的价值不在于"说得多"，而在于过滤掉九成噪音后，那一成该说的话能在对的时机出现。OpenClaw 现成的触发机制（heartbeat / webhook / MCP）已经够用，真正花功夫的是价值判断与行为边界。我的建议浓缩成两句话：**先限动作，再开口；先限频率，再说话。** 一个一天只说一次但次次可信的助手，远比一个喋喋不休的助手有用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/9a535d77dfeada02.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/f2c13ebb849b88d1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/164c9903b5b322b7.png)

