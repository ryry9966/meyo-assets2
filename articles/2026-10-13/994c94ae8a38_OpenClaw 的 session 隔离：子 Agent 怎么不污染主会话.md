---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41362
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

OpenClaw 里主会话是长生命周期的：一个 agent 绑定一个 session key，历史消息、工具调用、系统提示全堆在同一个上下文里。跑自动化时经常需要子 agent 去干脏活——批量抓数据、跑脚本、试错调参。但如果子 agent 的过程全部留在主会话上下文里，几轮下来主会话就废了。这篇记录一下我们在实际任务里沉淀的隔离做法。

## 问题：两种"污染"

1. **上下文膨胀**：子 agent 的工具输出（网页抓取结果、命令日志、报错栈）直接灌进主会话，token 消耗陡增，模型注意力被稀释，后面正常对话质量肉眼可见地下降。
2. **行为带偏**：主会话历史里残留大量子 agent 的中间过程，主 agent 后续的回复风格和决策被这些噪音带着走。更隐蔽的是共享状态：子 agent 改了工作区文件或 memory，主 agent 下次读到的已经是被改过的东西。

## 做法

1. **子 agent 用独立 session key**。spawn 时给显式 key，比如 `agent:research:task-0611`，绝不复用 `agent:main`。子 agent 的所有工具调用和中间输出都落在自己的 session 文件里。
2. **只回传摘要**。spawn 配置选摘要投递模式，子 agent 结束后自己把结论压成一段话交回主会话，过程不逐条转发。
3. **工具面收窄**。子 agent 只挂任务必需的工具，prompt 里写死输出格式，比如"只输出 JSON，不超过 200 字"，限制它跑偏和产生噪音。
4. **工作区隔离**。子 agent 要写文件的，给它单独的子目录或独立 agentDir，避免和主 agent 互相覆盖。
5. **定时任务单独 session**。cron 任务绑定独立 session key，别和主会话混用，产物通过摘要或文件交接。

## 踩坑点

- **共享 memory 是重灾区**：子 agent 一次试错的结论被写进长期记忆，之后主 agent 一直"记得"这个错误结论。建议子 agent 禁写 memory，或者结束后人工审一遍 diff。
- **摘要本身不摘要**：子 agent 输出太自由，回传的"总结"三千字，隔离了个寂寞。摘要长度必须在 prompt 里量化。
- **key 命名随意**：排查时分不清哪个 session 是谁开的。建议固定 `agent:用途:日期` 的约定。
- **拆分过碎**：忘了子 agent 也消耗上下文，任务拆得太细，每个子 session 又各自膨胀。大任务先规划再拆。
- **结果没人收**：关掉投递后子 agent 白跑一遍，产物只躺在它自己的 session 文件里。

## 可复用建议

- 判断标准：任务会产生大量工具输出、需要试错 → 开子 agent；只是多问一句 → 主会话直接答，或者 `/new` 开个干净 session。
- 子 agent prompt 三要素：明确输入、限定工具、约束输出（格式 + 长度）。
- 定期看 session 文件大小，异常膨胀的直接重置，别硬续。

## 总结

Session 隔离的本质是控制信息流向：**过程留在子会话，结论回到主会话，状态落盘到文件而不是上下文**。OpenClaw 提供了 session key、spawn、摘要投递这些机制，剩下的纪律要自己守——子 agent 不是越多越好，能不污染的上下文就不要污染。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/a12d1f9bc2a8d312.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/38c2cd36576a9808.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/018d33737aa6ecc1.png)

