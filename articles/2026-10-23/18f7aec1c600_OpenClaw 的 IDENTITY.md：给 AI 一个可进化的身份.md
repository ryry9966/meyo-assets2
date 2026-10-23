---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42373
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 的 workspace 里有一组 markdown 文件承担"人格层"：SOUL.md 管行为性格，USER.md 管用户画像，MEMORY.md 管长期事实，而 IDENTITY.md 管"我是谁"——名字、物种、气质、emoji、头像、颜色这些字段。这些文件会在会话启动时注入上下文，是 agent 输出风格的第一锚点。

听起来很简单，但实践里这块恰恰是最容易失守的。

## 问题：两个极端

我在社区里见过两类典型故障：

1. **放任型**：IDENTITY.md 保持默认模板，跑了两周后发现 agent 的自称、语气随会话漂移，同一个 agent 今天高冷明天热情，用户侧观感割裂。
2. **堆料型**：把 IDENTITY.md 当 system prompt 用，塞满"永远不要……""你必须……"，结果和 SOUL.md 的性格描述、AGENTS.md 的行为约定大量重叠，两个文件互相打架，改一处崩一处。

核心矛盾是：**身份需要稳定，但身份也需要随使用场景演进**。没有演进机制的稳定会僵化，没有约束的演运会漂移。

## 我的做法

我的 workspace 结构和迭代流程大概是这样：

**第一步，收敛字段。** 只保留：name、creature、vibe（一句话，不超过 30 字）、emoji、avatar。字段越少越稳，避免把规则混进来。

**第二步，明确职责边界。** 一张自检表：

| 内容 | 归属文件 |
|---|---|
| 我是谁、怎么自称 | IDENTITY.md |
| 性格、语气、价值观 | SOUL.md |
| 用户偏好、称呼 | USER.md |
| 具体任务规则 | AGENTS.md / TOOLS.md |

同一件事只允许出现在一个文件里，出现两处必删一处。

**第三步，git 管 workspace。** IDENTITY.md 的每次改动走 commit，diff 就是变更记录，可回滚、可审计。这是"可进化"的前提——没有版本控制，进化就是赌博。

**第四步，回归验证。** 准备 3 个固定测试问题（一个日常闲聊、一个任务请求、一个故意越界的问题），每次改完 IDENTITY.md 跑一遍，检查自称、语气、边界响应是否符合预期。改动后输出明显跑偏，就 revert。

## 踩坑点

- **负向指令堆砌**："永远不要卖萌"这类写法，模型有时反而被激活相关模式。改成正向描述："语气克制、直接、工程化"效果好得多。
- **agent 自己改自己**：给 agent 写权限让它随手更新身份，几次之后身份就不是你认识的了。改身份必须是人的动作，agent 最多提案。
- **多 agent 复制同一份**：几个实例共用 IDENTITY.md，日志和消息里完全分不清谁是谁。每个实例至少 name 和 emoji 要区分。
- **avatar 改太勤**：头像字段高频变更，部分客户端有缓存，会出现"文字说改了但头像没变"的诡异观感。低频批量改。

## 可复用建议

- 把 IDENTITY.md 当**配置文件**而非文案来维护：小步改、有 review、能回滚。
- vibe 一句话就够，性格细节去 SOUL.md，别越界。
- 演进节奏固定下来：比如只在发现明确漂移或场景变化时改，每周最多一次。
- 中文场景额外注意：自称（"我"还是名字）在 IDENTITY.md 里定死一次，能省掉很多风格仲裁问题。

## 总结

IDENTITY.md 的价值不在于"给 AI 起个名字"，而在于给身份一条**可审计、可回滚的演进通道**。把它当成一份小而稳的配置来运营，agent 的风格稳定性会上一个台阶——这比任何 prompt 技巧都划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/1e73b4a6d00af508.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/f23e954627e9a0b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/76979f83b89c13d6.png)

