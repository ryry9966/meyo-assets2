---
title: HEARTBEAT.md 实践：让 Agent 从"等你提问"变成"定期巡检
feedId: 37664
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

绝大多数 Agent 交互是拉模式：你发消息，它回答，会话结束它就"睡着"。但一个合格的值守型 Agent，价值大头在推送侧——定期巡检、汇总、提醒、监控。OpenClaw 内置了心跳机制：网关按固定间隔（`HEARTBEAT_INTERVAL_MS`，默认 30 分钟）唤醒一次 Agent。而 **HEARTBEAT.md** 就是决定"醒来之后干什么"的那份配置文件。

## 问题

没有 HEARTBEAT.md 时，心跳只是一次存活探测：Agent 被唤醒、确认没事、继续睡。你想要的"主动行为"只能靠两种别扭的方式实现：一是在聊天里反复口头叮嘱，上下文越长越容易漂移；二是在 Agent 体系外写 cron 脚本，只有死规则、没有模型判断力。两者共同的问题是：**任务逻辑和 Agent 的上下文、工具、判断力是割裂的**。

## 做法

1. 在工作区根目录（如 `~/clawd/workspace/`）创建 `HEARTBEAT.md`。
2. 用 checklist 语法写任务，每条做到"可判定、可执行、可跳过"：

```markdown
# Heartbeat

## 每次心跳检查
- 检查 ~/logs/error.log 末尾 50 行，若有新 ERROR 且上次心跳未报告过，
  摘要推送给我的主通道；否则不要说话
- 扫描 GitHub watched repos，若有新 release，汇报标题和 breaking changes

## 静默规则
- 没有任何待办时，回复 HEARTBEAT_OK，不要打扰我
```

3. 按需调整间隔。巡检类任务 30 分钟足够，别为了"及时"调到 5 分钟——那是给自己制造噪音。
4. 运行机制：每次心跳，HEARTBEAT.md 全文作为 prompt 注入，Agent 逐条评估、执行到期项、通过主动通道推送结果；无事则静默。文件不存在时，心跳退化为最简存活 ping。

## 踩坑点

- **写成散文而不是 checklist**。心跳 prompt 要的是可扫读条目，段落式描述会让 Agent 每次自由发挥，行为不稳定。
- **文件太长**。内容每个 tick 都计 token，10 行以内是合理上限。低频任务挪到 cron，心跳只留轮询和监控。
- **时间语义不可靠**。"每天早上九点发日报"这类条目不要放心跳里，Agent 对真实时间的感知有限。准点任务交给 cron/调度插件，心跳负责模糊周期任务。
- **不幂等**。最常见的翻车：同一件事每个 tick 重复推送。条目里务必写清去重条件（"仅当上次心跳未报告时"），或在状态文件里记录已上报的指纹。
- **每个 tick 都回话**。用 `HEARTBEAT_OK` 约定或响应抑制配置，做到"无事不扰"。
- **别放密钥**。这个文件会注入到每次心跳 prompt 里。

## 可复用建议

- 把 HEARTBEAT.md 当配置即代码：进 git，改动走 commit，方便回溯"它当时为什么这么做"。
- 推荐三段式模板：**每次检查（every-tick）/ 条件触发（conditional）/ 静默规则（silence）**。
- 冷启动只放 1–2 条，跑一周观察推送质量和 token 消耗，再逐步加。
- 心跳负责"轮询 + 判断"，cron 负责"准点触发"，两者配合而非互相替代。
- 每月 review 一次，删掉长期不触发的僵尸条目。
- 条目里尽量指定用哪个 MCP 工具/插件执行，减少 Agent 每次自选工具带来的方差。

## 总结

HEARTBEAT.md 的本质，是把"你希望 Agent 定期做什么"从聊天记录里抽出来，变成一份可版本化、可审查的常驻任务清单。工程上它就是一份被周期性注入的 prompt，所以写法和写 prompt 的纪律一样：**短、明确、幂等、默认静默**。做好这四点，Agent 就能从"问答机"变成真正意义上的值守助理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/9de2fe19327af83f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/c592419a0d10367c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/2f929da0165fae28.png)

