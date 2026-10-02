---
title: OpenClaw 的 session 隔离：让子 Agent 干脏活，主会话保持干净
feedId: 40178
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

OpenClaw 里每个 channel（Telegram、WhatsApp 等）对应一个主 agent 会话，对话记录、工具调用、MCP 的原始输出全都堆在同一个 session 文件里。上下文一旦写满就触发压缩，而压缩必然丢细节——丢的往往是你最不想丢的那部分。

## 问题

最常见的污染路径是让主会话直接干重活：遍历目录读二十个文件、连续调十几次 MCP 工具排查问题、跑一轮批量处理。每一步的中间输出都留在主上下文里，几轮之后主会话就被过程性垃圾塞满：响应变慢、压缩频繁、模型开始"忘事"。更麻烦的是失败任务会留下半截输出，之后的对话还会被这些残渣带偏。

## 做法

1. **重活一律走 `sessions_spawn`**。子会话有独立上下文和独立 session 文件，跑完后只把结果摘要注回主会话。
2. **spawn 提示词写死两件事**：任务目标和返回格式。明确要求"只返回结论，不超过 N 行"，禁止子 agent 把过程贴回来。
3. **异步任务**（爬取、批处理）spawn 之后用 `sessions_poll` 轮询或等回调，主会话该干嘛干嘛。
4. **定时任务和 heartbeat 触发的检查类工作**，同样配置成隔离会话执行，不要挤在主会话里跑。

## 踩坑点

- 子 agent 里又调用 `sessions_send` 往主会话写内容，隔离等于没做。spawn 时必须显式禁止。
- 为了"看看子 agent 干了啥"，在主会话里手动拉子会话的完整 `sessions_history`——全量日志瞬间进主上下文，白隔离。要看就只看摘要。
- 没设超时的子会话会僵死：任务卡住，模型调用一直挂着。每个 spawn 都给 timeout。
- 多个子 agent 并发写同一份文件或状态，结果互相覆盖。任务描述里划清各自的读写边界。
- session 文件只增不减，隔一两周用 `sessions_list` 过一遍，清掉僵尸会话。

## 可复用建议

把 spawn 提示词模板化，固定四段：**目标 → 边界**（允许动哪些文件和工具）→ **返回格式**（≤N 行结论）→ **禁止事项**（不 send 回主会话、不展开过程）。团队统一这个模板，新人也能写出干净的隔离任务，review 时也有据可查。

## 总结

session 隔离的本质是控制上下文的"写入权限"：主会话只接收结论，过程性输出留在子会话里自生自灭。OpenClaw 的 spawn 机制已经把管道铺好，剩下的纪律靠提示词和习惯——重活不进主会话，是成本最低、收益最直接的一条。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5e9597424ea740bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/1b29e8d5a32fa5b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/dc8a619ad24567a4.png)

