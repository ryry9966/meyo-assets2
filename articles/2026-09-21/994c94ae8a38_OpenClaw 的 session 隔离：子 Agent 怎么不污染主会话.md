---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 38340
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

跑多 Agent 任务时常见结构：主会话负责拆解任务、调度子 Agent、汇总结果；子 Agent 干脏活——批量检索、读长文档、试错性调工具。OpenClaw 里每个会话有独立的 context 和 session 记录，但很多人默认让子 Agent 直接"接着主会话聊"，或者把子 Agent 的完整输出原样倒回去，问题就来了。

## 问题：污染是怎么发生的

典型三种：

1. **中间过程回流**：子 Agent 的工具调用、报错重试、检索到的长文档被追加进主会话 history，主模型下一轮推理要"陪读"这些噪声。
2. **指令漂移**：子 Agent 的 system prompt 或 scratchpad 内容混入主上下文，主 Agent 行为被带偏，比如开始模仿子 Agent 的输出格式。
3. **状态覆写**：子 Agent 和主会话共享 memory 文件或 MCP 资源，把用户长期偏好、笔记直接覆盖掉。

后果不只是 token 变贵——上下文越长，主 Agent 对原始指令的注意力越稀释。很多人说"任务跑久了 Agent 变笨"，一半是这个原因。

## 做法

我们的约定很简单：**子 Agent 是黑盒，进出都走窄门。**

1. **会话隔离**。每个子 Agent 分配独立 session id，禁止复用主会话 id。父会话只记录指针：任务 id、结果文件路径，不落全文。
2. **输入走 brief**。给子 Agent 的 prompt 是自包含的任务简报：目标、约束、可用工具、交付格式。需要背景就摘录，不要"以防万一"把主会话完整 history 塞进去。
3. **输出走契约**。要求子 Agent 返回结构化结果（JSON 或固定 markdown 模板），长度上限写进 prompt。父 Agent 收到后做一次摘要落盘，不再展开。
4. **工作区隔离**。给子 Agent 单独的 scratch 目录，memory 和共享配置只读。需要写回的内容让它生成 patch 或建议，由主会话审核后合并。
5. **生命周期管理**。任务结束即归档或丢弃子会话；失败重试开新 session，不复用带脏状态的上一次。

## 踩坑点

- **子 Agent 返回自由文本**：主会话等于把子过程又吃了一遍，隔离白做。契约里必须同时限制格式和长度。
- **重试时部分合并**：上次失败的 transcript 残留一半在主会话里，比不隔离更糟。重试务必换 session。
- **UI 串流混淆**：有的前端把子 Agent 的轮次实时插进主会话流，视觉上像同一条历史。先确认持久化层里到底存了什么，再怀疑框架。
- **忘了销毁**：子会话没关，占存储和配额，还可能被下次调度误捞回来。

## 可复用建议

分享一个排查技巧：在主会话记录里 grep 只有子 Agent 才会用的工具名或 scratch 路径，命中就说明有泄漏。隔离做得好不好是可以验证的，不靠感觉。

建议沉淀成团队约定，四件套：brief 模板、输出契约、scratch 目录规范、session TTL。固定下来，新任务直接套，比每次现想规则省事得多。

## 总结

Session 隔离的本质是控制信息流向：父到子给摘要，子到父给结论，中间过程留在子会话里生死自负。OpenClaw 的 session 机制提供了做隔离的原材料，剩下的纪律靠约定。如果只改一件事，先从"子 Agent 必须返回结构化结果"做起，收益立竿见影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/8683834fbd67d90b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/8b53c65ee605a9ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/ac7f52aaec4df9f6.png)

