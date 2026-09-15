---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37753
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 的主会话承载对话状态、工具调用记录和记忆文件引用。跑长任务时的常见做法，是派生子 Agent 去处理检索、批处理、代码梳理这类“脏活”。隔离做得不好，子 Agent 的上下文、工具输出、甚至它写入的记忆条目会全部回流主会话——轻则 token 预算爆掉，重则主 Agent 开始把子 Agent 的中间产物当成事实。

## 问题在哪

实际踩过的三类污染：

1. **上下文污染**：子 Agent 的完整工具输出被拼回主线程。一次网页抓取几万 token，主会话立刻失焦，后续推理质量明显下降。
2. **记忆污染**：子 Agent 与主会话共用 MEMORY 路径，它写入的中间结论被主 Agent 在别的任务里当作已验证事实引用。
3. **状态污染**：复用同一个 session id，子 Agent 残留的系统提示和偏好设置影响下一轮对话，排查时非常隐蔽。

## 做法

我们团队现在约定四条：

1. **独立 session id + 独立 transcript 路径**。派生时生成 `main_id:sub-{task_hash}`，transcript 落到 `workspace/sub-sessions/`，主会话只保留最终结果文件的引用。
2. **显式交接契约**。子 Agent 的收尾指令固定为：输出结构化 JSON（结论、依据路径、剩余风险），字数上限写死，不回传原始工具输出。
3. **工具与权限收窄**。按任务配置 MCP 工具白名单；共享记忆目录对子 Agent 只读，单独给它一个可写的 scratch 目录。
4. **预算与超时**。每个子会话设 token 上限和 wall-clock 超时，超时即截断，并在结果 JSON 里标注 `status: timeout`，由主 Agent 决定重试还是降级。

spawn 伪配置大致长这样：

```json
{
  "spawn": {
    "inherit_history": false,
    "context_brief": "task_brief.md",
    "tools_allowlist": ["web.search", "fs.read"],
    "workspace": "sub-sessions/{task_hash}/",
    "memory_access": "ro",
    "return_contract": "json_summary",
    "max_tokens": 20000,
    "timeout_sec": 300
  }
}
```

## 踩坑点

- **别传完整历史“帮它理解上下文”**。`inherit_history` 一开，隔离等于没做。正确姿势是主 Agent 先蒸馏一份 task brief 再下发。
- **summary 本身也会膨胀**。没有字数上限的“最终总结”，跑几次就是新的污染源。
- **transcript 别删，要归档**。排障时唯一能还原子 Agent 决策过程的就是那份 transcript，删了等于丢现场。
- **只读挂载要真正生效**。有一次 scratch 目录被软链到共享路径，子 Agent 绕过了限制，白名单配了也白配。挂载后用一条只写测试验证一下，成本很低。

## 可复用建议

- 主会话定位成**调度器**，保持薄上下文：只装任务、约束和子 Agent 返回的结论。
- 把 spawn 配置做成模板入库，团队共用一套交接契约，code review 时有统一标准可对。
- 每周扫一遍 sub-sessions 目录，重点看异常长的 transcript——那通常是隔离失效的信号。

## 总结

Session 隔离不是开一个开关，而是一组显式接口：独立会话、收窄的工具面、单向的结构化回传、硬性预算。这四点做到位，子 Agent 才是在替你干活，而不是往主会话里倒垃圾。欢迎在评论区交流你们的隔离方案和翻车现场。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/0b8f370ce0f75f67.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/a3885c68c7658602.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/c43ea58afdc12e66.png)

