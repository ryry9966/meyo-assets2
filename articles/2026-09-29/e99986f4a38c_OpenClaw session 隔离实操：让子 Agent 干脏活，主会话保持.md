---
title: OpenClaw session 隔离实操：让子 Agent 干脏活，主会话保持干净
feedId: 39460
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的主会话是长命的：gateway 会为每个对话维护一份持续累积的 transcript（存放在 `~/.openclaw/agents/<agentId>/sessions/*.jsonl`），你说过的话、工具调用的完整输出、报错堆栈，全都在里面。上下文一满就触发 compact，早期指令被压成摘要，Agent 行为开始漂移——这是所有长跑 Agent 的通病。

OpenClaw 给的解药是子 Agent：通过 `sessions_spawn` 起一个独立 session 的子 Agent，它有自己的 transcript、自己的上下文窗口，跑完只把最终结论回传主会话。

## 问题

不做隔离的典型症状：

- 让主 Agent 直接读几个大文件做分析，几轮之后主会话被文件内容灌满，compact 频繁触发；
- debug 循环（改代码 → 跑测试 → 读报错）产生的中间输出全记在主会话里，噪音远大于有效信息；
- 最终早期的人格设定、项目约定被压没了，Agent 开始“忘记”你定过的规矩。

## 做法

1. **会产生大量中间输出的活，spawn 出去。** 任务描述写成自包含的 mini-spec：目标、边界、产出格式。子 Agent 看不到主会话历史，“按刚才说的办”这种写法必挂。
2. **控制回传。** 默认子 Agent 结束后把 final message 投回当前对话；不需要打扰时设 `deliver: false`，结果留在任务列表里，用 `/subagents` 查看。
3. **需要继承上下文时用 fork，而不是共享 session。** spawn 支持 fork 当前会话，把 transcript 克隆一份作为子 Agent 的起点——读得到，但写不回主会话。
4. **按任务配子 Agent。** `agents` 配置里给执行型子 Agent 配独立 workspace、更便宜/更快的模型；涉及不可信命令的走 sandbox。
5. **定期清理。** `openclaw sessions` 看堆积情况，给子 Agent 会话配 idle 清理，避免 jsonl 文件无声吃磁盘。

## 踩坑点

- **隔离的是上下文，不是资源。** 子 Agent 默认和主 Agent 共享 workspace 和工具，文件冲突、端口占用照样发生。要文件级隔离，得配独立 workspace 或 git worktree。
- **默认不是并行。** spawn 之后主 Agent 通常在等结果；想真并行需要后台跑并主动收割，注意别留一堆没人管的后台任务。
- **套娃有成本。** 子 Agent 里再 spawn，每层都是完整上下文加 token 计费，两层封顶比较稳。
- **compact 不是删文件。** 主会话 compact 后 transcript 仍在磁盘上，只是进模型的内容变短；以为 compact 省了磁盘是误解。

## 可复用建议

- 一条判断标准：**过程大于结论的活给子 Agent，依赖对话历史的决策留在主会话。**
- 把常用子 Agent（验证、批量执行、代码 review）固化为配置里的具名 agent，spawn 时直接引用，不要每次手写任务模板。
- 观察 `~/.openclaw/agents/*/sessions/` 的体积和 compact 频率，这是主会话健康度的直接指标，比等出问题再排查划算。

## 总结

session 隔离的本质是控制信息流：让“过程”烂在子会话里，只把“结论”带回来。把主会话当成珍贵的、低噪音的决策上下文来经营，长跑 Agent 的稳定性问题就解决了一大半。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/95a6b35a8cf99848.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/f1f46774848b45c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/61c3cc203445bc94.png)

