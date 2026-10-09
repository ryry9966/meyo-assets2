---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 41024
source: 综合讨论
publishedAt: 2026-10-09
---

# 背景

OpenClaw 里有两套“定时”机制：**cron** 和 **heartbeat**。前者是传统定时任务：到点触发、跑一段固定 prompt、可投递到指定会话或频道；后者是主 agent 的周期性自唤醒：每隔一段时间被网关叫醒一次，看看工作区的 HEARTBEAT.md 里有没有要处理的事，由模型自己判断做不做。

很多新用户会问：都能定时，用哪个？结论先放这里：**时间驱动用 cron，条件驱动、需要判断的用 heartbeat**。下面展开。

# 问题：一个真实场景

我最初把所有事都塞进了 heartbeat：每天早报、定时提醒、监控某个页面有没有更新。结果两个问题：

1. 早报要求“每天 8 点”，heartbeat 是固定间隔轮询，触发时间会漂移，8:07、8:31 都出现过；
2. heartbeat 每次唤醒都消耗 token，即使大多数唤醒最后什么都没做，一天的空转成本并不低。

反过来也有反面案例：把“盯着收件箱，有重要邮件就告诉我”写成 cron，每小时跑一次固定 prompt。但它拿不到主会话上下文，判断标准全靠 prompt 里那几行字，很快变成噪音制造机。

# 两者的本质区别

- **cron**：到点执行，prompt 固定，运行在隔离会话里，不依赖对话记忆。适合内容和时间都确定的任务。
- **heartbeat**：到点唤醒，prompt 很轻，agent 带着主会话上下文和 HEARTBEAT.md 自己决定是否行动、行动多少。适合“有事才动”的模糊任务。

一句话：cron 是“叫它干活”，heartbeat 是“叫它看看有没有活”。

# 做法与步骤

1. **先分类。** 列出手头所有定时需求，分两列：触发时间和输出内容都能提前写死的（早报、周报、定时提醒）进 cron；需要看情况判断的（盯收件箱、盯文件变化、检查任务是否卡住）进 heartbeat。
2. **配 cron。** 用 cron 工具添加任务，写清楚：schedule 表达式、完整的自包含 prompt（别指望它记得上次说了什么）、投递目标。新任务先用短间隔（比如每 5 分钟）跑通，再改成正式频率。
3. **配 heartbeat。** 在网关配置里设置唤醒间隔（默认 30 分钟够用，不需要就显式关掉）。HEARTBEAT.md 保持精简，每条一行：条件 + 动作，条目控制在 5 条以内。
4. **两者配合。** cron 负责在固定时间产出结果，heartbeat 负责闲时巡检，发现异常再调用工具深入处理。

# 踩坑点

- **时区。** 容器里 cron 按容器时区跑，和你本地差 8 小时是常见事故，上正式任务前先确认时区。
- **heartbeat 当广播用。** 在 HEARTBEAT.md 里写“每次唤醒都汇报一下”，等于自费买噪音。heartbeat 的默认行为应该是没事就静默。
- **cron prompt 写得太“诗意”。** “帮我看看今天有什么值得关注的”，agent 会自由发挥，输出结构每次不一样。prompt 里要明确输出格式，并写上“无事则说无事”。
- **HEARTBEAT.md 越养越肥。** 条目多了每次唤醒都变重，token 成本线性上涨。每周清一次，从来没触发过的条目删掉。

# 可复用建议

- 判断规则一句话：**精确时刻 + 固定产出 → cron；需要上下文和判断 → heartbeat。**
- 分钟级的高频轮询别用 heartbeat，用 cron 短间隔任务，或外部事件触发（webhook / MCP）。
- cron 的 prompt 当独立脚本来写：输入、动作、输出格式、兜底行为，四要素齐全。
- 心智模型：heartbeat 是“值班巡检”，cron 是“闹钟”。

# 总结

两者不是替代关系。cron 解决“什么时候做”，heartbeat 解决“要不要做”。把确定性的调度交给 cron，把不确定的判断交给 heartbeat，成本和噪音都会明显下降。建议从一条 cron 加三条 heartbeat 条目的小配置起步，跑一周看日志再调整，比一上来铺满配置靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/aea3c8a56ebbaeb9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/204bf3b02cc10346.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/385f95374e02e8a2.png)

