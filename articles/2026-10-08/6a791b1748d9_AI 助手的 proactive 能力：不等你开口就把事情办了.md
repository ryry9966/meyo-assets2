---
title: AI 助手的 proactive 能力：不等你开口就把事情办了
feedId: 40892
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

绝大多数 agent 的默认形态是"被动应答"：你发消息，它回消息。OpenClaw 因为常驻网关加上 cron、heartbeat、webhook 这些触发机制，天然具备主动发起的条件——问题在于，大部分人把主动任务配上一周后，就把通知渠道静音了。主动能力做不好，比不做更糟。

## 问题：主动 ≠ 多说话

失败的 proactive 通常死于三件事：

1. **频率失控**：heartbeat 每 20 分钟跑一次，每次都觉得"有话要说"；
2. **重复告警**：同一个 CI 挂了，推了五遍；
3. **越权执行**：agent 自己决定帮你回邮件、改文件，用户吓得直接停用。

三件事指向同一个根因：把"调度器"当成了 proactive 的全部，缺一层决策。

## 做法：把感知和打扰分开

我把它拆成三层，触发层只负责收集信号，绝不直接触达用户。

**触发层**：cron 定时巡检（release、数据源、日报素材）、webhook 接外部事件（CI 结果、git push）、heartbeat 低频自检。

**决策层**：一个精简的判定 prompt + 一份 state 文件（JSON），分三级：

- **L0 静默**：写进 state，用户永远看不到；
- **L1 汇总**：攒着，进每天固定时段的 digest；
- **L2 立即推送**：只对白名单条件生效（比如生产分支 pipeline 失败）。

简化示意（cron 任务 + 去重）：

```json
{
  "name": "repo-watch",
  "schedule": "0 9 * * *",
  "tz": "Asia/Shanghai",
  "prompt": "检查 watch-list 仓库的新 release，与 state.json 中已上报的 tag 对比；无新增则只更新 state 并回复 nothing；有新增走 digest；白名单条件之外不要立即推送"
}
```

关键是允许 agent 回答"无事"——判定 prompt 里明确写：不满足 L1/L2 条件时只更新 state、输出 nothing、不展开分析。仅这一条就把 heartbeat 的 token 成本压到了零头。

**执行层**：推送走通知渠道，附上下文和"为什么是现在"；任何写操作（发消息、改文件、调外部 API）默认不进 proactive 路径，需用户显式确认后才升级。

## 踩坑点

- **heartbeat prompt 太重**。一开始我把完整系统提示喂给每次心跳，一晚上烧掉几万 token。单独写一个三行的精简判定 prompt，效果反而更稳定。
- **去重缺失**。对事件做 hash（repo+tag、branch+commit）存进 state，上报前先查一遍。
- **时区**。cron 不写显式时区，digest 会在奇怪的时间到。统一 `Asia/Shanghai`。
- **主动会话污染主上下文**。巡检跑在独立 session，结论只以推送或 state 落地的形式进入主对话。
- **agent 自评"紧急"**。什么算 L2 必须是枚举出来的白名单条件，不能让它自己发挥。

## 可复用建议

1. **只读先行**：先跑两周纯监控 + 汇报，观察误报率，再考虑放开写操作。
2. **每日配额**：L2 推送上限写死在决策 prompt 里（比如 3 条），超出自动降级为 L1。
3. **决策留痕**：每次 L1/L2 判定结果追加到日志，周末复盘调阈值——proactive 调优和告警系统调优是同一门手艺。
4. **声明式配置**：触发条件和白名单全部放在可 review 的配置文件里，别散落在自然语言 prompt 中。

## 总结

Proactive 的价值不在"它动了"，而在"它挑对了时机"。工程上真正要花力气的不是调度器，而是那个薄薄的决策层：去重、分级、配额、留痕。把它当成一个信号处理问题来做，agent 才能长期留在你的通知列表里，而不是躺在被静音的那个分组。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/f28ba9436442a842.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/3b4522d4e56acc43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/17835bc7b01dc12c.png)

