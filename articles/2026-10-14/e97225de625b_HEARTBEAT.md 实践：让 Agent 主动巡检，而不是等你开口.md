---
title: HEARTBEAT.md 实践：让 Agent 主动巡检，而不是等你开口
feedId: 41550
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

用 OpenClaw 这类常驻 Agent 时，默认交互是拉模式：你发消息，它才动。但很多事不需要"你问"，只需要"定时看一眼"——CI 红了没、仓库里有没有忘 push 的提交、今天的日历上有什么。OpenClaw 的心跳机制就是为这个准备的：Agent 每隔一段时间被唤醒一次，读工作区里的 `HEARTBEAT.md`，有事就做，没事就回 `HEARTBEAT_OK`，网关把它静默吞掉。文件本身极小，但把交互模型从 pull 变成了 push。

## 问题：被动 Agent 的三个空白

1. **周期性任务没有落点。** cron + 脚本能做轮询，但没有判断力，告警要么哑要么吵。
2. **上下文不在场。** 定时脚本跑的时候不知道你昨天的对话、不知道项目状态；Agent 知道。
3. **动作链路断。** 脚本发现问题后还是要人来决定"要不要说、说多严重"，Agent 可以自己收敛这件事。

## 做法

**1. 在工作区根目录建 `HEARTBEAT.md`**，每条写成"条件 + 阈值 + 动作 + 沉默条款"：

```markdown
# HEARTBEAT.md
- ~/projects/api 有未 push 的本地提交 → 提醒一次
- 用 MCP 邮件工具查未读，只有 reviewer 来的才报告
- 09:00 后的第一次心跳：汇总今天的日历
- 其余情况：HEARTBEAT_OK
```

**2. 配置节奏**（openclaw.json，具体字段以当前版本文档为准）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "direct",
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    }
  }
}
```

**3. 重启网关或 reload 配置，看日志验证。** 第一次故意制造一个条件（比如留一个未 push 的提交），确认它真的会说话；再确认无事时它真的保持沉默。

## 踩坑点

- **写成"汇报体"就是轰炸。** "随时汇报近况"会让每次心跳都发消息。所有条目都要有沉默条件，沉默才是默认态。
- **别把间隔调太猛。** 每次心跳是一次完整的模型调用，5 分钟一次纯烧 token。30 分钟起步，除非有硬需求。
- **条目要具体。** "盯着项目"会产出噪音；"测试连续失败 2 次才报，报过之后当天不再重复"才可用。给失败限流，否则工具一挂它每半小时哭一次。
- **心跳默认跑在主会话里，输出会污染上下文。** 条目保持一行以内，重活让 Agent 去调工具，不要在心跳里长篇分析。
- **target 和 activeHours 一定要设。** 否则凌晨三点它可能往你的群或私聊里发"今日待办"。

## 可复用建议

- 把它当 runbook-lite，条目 ≤5 条。超了说明该拆：高频监控交给 Prometheus/CI，心跳只做低频、有判断的巡检。
- 搭配 MCP 工具才有内容可查：邮件、日历、GitHub、磁盘、电池，接进来心跳才不空转。
- `HEARTBEAT.md` 纳入 git，每次改条目走 commit，出问题可回溯。
- 实测好用的场景：每日晨报、未 push 提交提醒、PR 待审汇总、RSS 摘要、服务器磁盘水位。

## 总结

HEARTBEAT.md 本质上是给常驻 Agent 一个带判断的 cron：低频唤醒、读清单、要么安静、要么干一件具体的事。它不是监控系统，替代不了告警体系；它的价值在于把"你想到才去问"变成"它定时看、必要时才开口"。清单短、条件硬、默认沉默——做到这三点，它就是那种配完就不用管的自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/ffb7815ae63bc6b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/dfdbca0322c98743.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/4678257850f11458.png)

