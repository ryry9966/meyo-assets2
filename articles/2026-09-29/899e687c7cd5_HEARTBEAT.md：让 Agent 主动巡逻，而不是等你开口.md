---
title: HEARTBEAT.md：让 Agent 主动巡逻，而不是等你开口
feedId: 39371
source: 综合讨论
publishedAt: 2026-09-29
---

**TL;DR**：HEARTBEAT.md 是 workspace 里一个普通的 Markdown 文件，Agent 每隔 N 分钟读一次：有任务就执行，没任务就跳过。它是把"被动问答 Agent"改造成"主动巡检 Agent"成本最低的办法。

## 背景

大多数 Agent 的使用模型是"你问它答"。但在个人助理场景里，很多需求恰恰是反过来的：仓库出了新 issue 要提醒我、RSS 有重要更新要汇总、每天早上推一份日报。这些都属于"用户不说话，Agent 也该动"的事。

自己写脚本 + cron 调 openclaw 当然可行，但每条任务一段脚本，去重、状态、通知目标散落各处，写到第三条就不想维护了。

## 做法：一个文件加一个间隔

1. 在 `openclaw.json` 里打开心跳：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "activeHours": { "start": "09:00", "end": "23:00" }
      }
    }
  }
}
```

2. 在 workspace 根目录建 `HEARTBEAT.md`，任务用"条件 → 动作"一句式写：

```markdown
# HEARTBEAT

- 查看 <org>/<repo> 的 open issues，若出现带 P0 标签的新 issue，
  推送到 Telegram；否则保持沉默
- 每天第一次心跳时汇总未读 RSS 并推送，推送后把日期写入
  state/digest.txt；若已是今天则跳过
```

3. 两个省成本的设计要利用好：
   - **空文件即跳过**。文件缺失或只剩 `# HEARTBEAT` 一行时，这一拍不启动模型，零 token。想临时停掉所有主动任务，清空文件即可，不用改配置。
   - **心跳默认跑在隔离会话**，不会把半小时一次的检查记录塞进主对话上下文。

4. 心跳通知失败时，Agent 会按 `HEARTBEAT_FAIL.md` 的指引兜底，有洁癖可以自定义。

## 踩坑点

1. **没写去重条件就是定时刷屏**。任务必须写清"仅在变化时通知"，并让 Agent 把状态落盘到 workspace 文件——文件持久，重启也不丢。
2. **隔离会话看不到你们刚才聊了什么**。依赖对话状态的任务要么显式落盘，要么改 `session: main`（主会话会被心跳内容逐渐撑大，慎用）。
3. **间隔别贪快**。`every: 5m` 加每拍三五个工具调用，一个月后账单会教育你。先跑 30m 观察一周再调。
4. **心跳不是 SLA**。笔记本合盖、网关进程挂了，心跳就停。报警类、时间敏感的任务交给外部 cron 或真正的监控系统，心跳只做"氛围感知"。
5. **文件名写错 = 静默跳过**，表现和空文件一模一样。配完去日志确认每拍是 ran 还是 skipped。
6. 需要精确时刻的任务（"每天 8:30 整"）用 cron，别让模型在心跳里猜时间。

## 可复用建议

- 分三层：**固定时刻 → cron；条件触发 → HEARTBEAT.md；一次性 → 手动提问**，职责别混。
- 任务统一成模板：`检查 X；若满足 Y 则通知 Z；状态写入 W`。一行一个，别写成小作文。
- 成本粗算：每拍 ≈ 心跳 prompt + 任务工具调用，乘以每天拍数（30m 间隔约 96 拍）。`activeHours` 能直接砍掉夜间拍数。
- 从一个任务跑稳再叠加。一次加五个，你分不清哪条在烧钱。

## 总结

HEARTBEAT.md 的巧妙之处在于把"主动性"退化成了一个文件约定：文件有内容就干，没内容就闭嘴。配上 activeHours、隔离会话、状态落盘三个习惯，它能长期低成本地跑在一台常开的机器上。先用它接管一件你每天都会手动去查的事，再决定扩展——主动性的价值不在于多，而在于省掉你脑子里那根绷着的弦。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/aca6365a18e86f5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c2613f734fb44bf6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/9e7b37c2b47a1fc0.png)

