---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 38215
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的主会话承载长期交互和记忆检索，是整个 agent 的"主上下文"。跑批量抓取、代码实验、长耗时任务时，我们习惯 spawn 子 agent 去干。但子 agent 如果和主会话共享上下文或工作区，污染几乎是必然的——而且是那种几轮之后才显现、很难定位的慢性污染。

## 问题：污染通常有三种形态

1. **上下文回流**：子 agent 的工具输出（网页全文、几百行日志）直接进主会话，token 膨胀，后续轮次被无关内容带偏。
2. **抢答**：子 agent 拿到向频道发消息的能力，用户看到主 agent 和子 agent 两份回答。
3. **写穿**：子 agent 往主 workspace 的 `memory/`、`AGENTS.md` 里写中间产物，长期记忆从此混入垃圾，之后每轮检索都能捞到。

## 做法

核心思路是三件事：**上下文不回流、副作用不外溢、生命周期可回收**。

1. **显式隔离 session**。spawn 时确认子 agent 使用独立 session id，而不是继承主会话上下文。默认行为不等于你要的行为，值得花一分钟看一眼配置。
2. **结果只回传摘要**。把结果投递模式设为 summary，并限制返回长度。子 agent 的完整 transcript 留在它自己的 session 里，主会话只收 800 字以内的结论。
3. **工具白名单**。子 agent 只挂任务必需的 tools，明确禁用 `channels.send`、`memory.write` 这类有外溢能力的工具。
4. **工作区分离**。给子 agent 单独的 workspace 子目录（如 `workspaces/task-<runid>`），遵循"读多写少"：可以读主工作区，写入只进自己的沙箱。
5. **用完即归档**。任务结束就清理子 session；不要 resume 旧子会话去续新任务——resume 会把全部旧上下文带回来，等于白隔离。

示意配置（字段名以你所用版本的仓库文档为准）：

```json
{
  "spawn": {
    "session": "isolated",
    "result": { "mode": "summary", "maxChars": 800 },
    "tools": ["read", "shell", "fetch"],
    "workspace": "workspaces/task-<runid>",
    "deny": ["channels.send", "memory.write"]
  }
}
```

## 踩坑点

- `result.mode` 设成 full/transcript，等于没做隔离，只是换了个 session 名义。
- 子 agent 并发写同一个 workspace 子目录，`<runid>` 忘了带时间戳，任务互相覆盖。
- 长任务用同步等待，主会话整个卡住，心跳也受影响——spawn 后应走异步通知。
- 调试时图省事直接复用了上次的子 session，排查了半天"为什么它知道上周的事"。

## 可复用建议

- 约定 session key 命名：`sub-<task>-<date>`，审计日志里主/子会话一目了然。
- 把上面那份隔离配置做成团队 spawn 模板，新任务改参数不改结构。
- 重要结果落盘为结构化文件，靠文件传递结果，而不是指望会话上下文记住。
- 定期清理 sessions 目录，子会话默认不进长期记忆索引。

## 总结

session 隔离不是加一个开关，而是一组约束的组合：只回摘要、工具白名单、沙箱写入、用完归档。四件事都做到，子 agent 才是真正意义上的"临时工"——干完活走人，不留痕迹。默认配置只能保证功能可用，不能保证上下文干净，这一点值得每次 spawn 前想一遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/79df51d833218a33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/757eddc9456a0d11.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/490208f2d0eeb859.png)

