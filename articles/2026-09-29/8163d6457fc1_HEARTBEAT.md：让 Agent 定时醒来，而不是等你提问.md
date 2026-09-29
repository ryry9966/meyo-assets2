---
title: HEARTBEAT.md：让 Agent 定时醒来，而不是等你提问
feedId: 39496
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

多数 Agent 的默认形态是"你问一句、它答一句"。但当你把 OpenClaw 当成日常值守助手时，真正的需求往往反过来：磁盘满了它先提醒、服务挂了它先发现、每天早上给你一份汇总。OpenClaw 为此留了一个很轻的口子——workspace 里的一份 `HEARTBEAT.md`，配合定时心跳，让 agent 周期性醒来、自己找事做。

## 问题

没有心跳时，这类需求只能靠外部 cron 调 API，或者你记得去问它。而粗暴地"每 30 分钟跑一次 agent"会掉进两个极端：不写约束，agent 每次醒来都汇报"一切正常"，刷屏且烧 token；写得太含糊，判断不稳定，误报漏报都有。

## 做法

**1. 建文件。** 在 workspace 根目录（默认 `~/.openclaw/workspace/`）新建 `HEARTBEAT.md`。心跳触发时，OpenClaw 会把这份文件连同固定的心跳提示词（含当前时间）一起发给 agent；文件为空则走默认行为——无事可做，静默跳过。

**2. 写一份克制的清单**，核心是"检查项 + 沉默条件"：

```markdown
## 每次心跳
- 检查 http://localhost:3000/health，非 200 才通知
- 磁盘占用 > 85% 才通知

## 每日 09:00（按当前时间判断）
- 汇总昨日 git commit 数，一行以内

## 规则
- 一切正常：不输出任何内容
- 仅异常或每日任务到点时发消息，不超过 3 行
```

**3. 配置节奏。** `openclaw.json` 里大致如下（字段名以你的版本为准）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "interval": "30m",
        "activeHours": { "start": "08:00", "end": "23:00" },
        "model": "某个便宜模型"
      }
    }
  }
}
```

`activeHours` 避免半夜打扰；`model` 给心跳挂个便宜模型——心跳大多是机械判断，不需要最强推理。

**4. 验证。** 把 interval 临时调成 1–2 分钟，看日志和目标会话：确认心跳真的读到了清单、异常判断符合预期、主动消息送达正确渠道，再调回去。这一步别省。

## 踩坑点

- **成本线性于频率。** 每个 tick 是一次完整推理。清单控制在十几行；精确到分钟的调度交给 cron 任务，HEARTBEAT.md 只放低频、粗粒度的值守项。
- **不写沉默条件必然吵。** "一切正常时不要输出"必须显式写进文件，否则它会非常尽职地每半小时问候你一次。
- **判断要可机械化。** 用阈值、退出码、HTTP 状态码，别写"看看服务是否健康"。心跳是同一段提示词反复执行，主观描述会带来随机误报。
- **上下文污染。** 心跳与你的会话共享记忆，长输出会挤占上下文，务必要求"异常时不超过三行"。
- **心跳也有工具权限。** 它醒来同样能执行 shell，清单里别放高风险命令，沙箱和 allowlist 照常配置。

## 可复用建议

- **分层**：值守清单放 HEARTBEAT.md，长期事实放 MEMORY.md，精确定时用 cron，三者别混。
- **心跳只做判断和转发**：重活写成本地脚本，agent 只读结果、比阈值、决定说不说话。
- **多 agent 拆岗**：每个 agent 有自己的 workspace，也就有自己的 HEARTBEAT.md——"盯服务"和"盯日历"可以是两个不同节奏的值守员。

## 总结

HEARTBEAT.md 的价值不在"自动化"本身，而在把"agent 何时该开口"从隐式行为变成一份可版本管理、可 review 的文字约定。小清单、便宜模型、明确的沉默条件，三件事做到位，主动式 agent 才是省心的，而不是费 token 的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/5c879bee9f28da67.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/3fc88d85c122a486.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ee79a4afe8035422.png)

