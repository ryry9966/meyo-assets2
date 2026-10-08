---
title: 不等你开口就把事办了：Proactive Agent 的触发层与决策门实践
feedId: 40961
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

大部分 Agent 部署是纯响应式的：用户发消息，Agent 回答。这个模式天花板很明显——大量有价值的事情（CI 失败分拣、证书到期提醒、晨会摘要）根本不需要有人在旁边问。所谓 proactive，拆开看就三件事：**什么时候触发、要不要出手、出手了怎么兜底**。三者缺一，主动就会变成打扰。

## 问题

最省事的思路是给 Agent 挂个 cron 定时干活。我们试过，结果通常两种：要么频繁推送制造"狼来了"，用户两周后开始无视所有通知；要么某天触发器静默挂掉，Agent 该干的活没干，没人知道。根源不在触发器，而在触发之后的**判断和治理缺失**。

## 做法

**1. 触发层只负责叫醒。** 在 OpenClaw 里，时间驱动用 cron，事件驱动用 webhook 或文件监听，以插件或 MCP server 的形式暴露给 Agent。payload 保持结构化、最小化：触发来源、时间窗、过滤条件，不要在这里塞上下文。

**2. 决策门。** Agent 拿到触发后不直接执行，先输出结构化判断：

```json
{
  "action": "act | propose | ignore",
  "confidence": 0.82,
  "reason": "main 分支连续 3 次 CI 失败，非 flaky 特征",
  "proposed_steps": ["定位 commit", "生成 triage 草稿"]
}
```

这一层是整个 proactive 体系的核心，也是后续统计与回归的基础。一个带 schema 的工具调用就够了，不需要额外组件。

**3. 三档放权，按类目灰度。** notify（只说不做）→ propose（动作准备好，人一键确认）→ act（仅限预授权类目，如给 flaky 任务打标重跑）。任何类目至少跑两周 notify-only，接受率达标才升档。

**4. 全量留痕。** 每次运行记录 trigger payload、context、decision、result 四元组，既能回放排障，也是调判断门的样本。

一个实际例子：每天 8:30 cron 拉昨晚 CI 失败，决策门分拣——flaky 特征的走 act 直接标注重跑（已预授权），疑似真回归的走 propose 生成 triage 草稿，其余 ignore。上线一个月，晨会前人均省 15 分钟左右，误报率约 8%，可接受。

## 踩坑点

- **自触发回路。** Agent 写入的文件又触发了监听器，几分钟跑爆预算。解法：触发层排除 Agent 自身输出路径，并加冷却窗口。
- **触发器静默死亡。** 比"不触发"更糟。触发层自身要有心跳上报，超时告警打到人。
- **上下文贪多。** 把全量日志塞给决策门，成本翻倍、判断反而变差。给最小结构化上下文，细节让它在 propose 阶段按需拉取。
- **权限复用。** proactive 会话的 tool 权限应比交互会话更窄，别图省事直接复用。

## 可复用建议

1. 核心指标是**建议被采纳率**，不是触发次数——触发多只说明你吵。
2. 决策输出固定 schema，方便做 A/B 和回归测试。
3. 每个类目单独设频率上限和 token 预算，防单类目失控。
4. 从低错误成本场景切入（摘要、分拣、草稿），别一上来就碰生产资源。

## 总结

Proactive 的工程量八成在触发层之外：判断门的设计、放权的节奏、可观测性。先把 notify 做准，让用户信任它说什么，才轮得到 act 替用户做什么。"不等你开口"的前提，是用户闭着眼也敢让它动手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/69f0608e7e54168e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/2984aeb29c409077.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/ab542945503335ee.png)

