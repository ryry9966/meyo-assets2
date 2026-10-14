---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41546
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的 agent 不只是一段系统提示词，而是一个由工作区文件组成的小型操作系统：`AGENTS.md` 管行为规范，`SOUL.md` 管性格底色，`USER.md` 记用户信息，`MEMORY.md` 承担长期记忆。其中最容易被低估的一个，是 `IDENTITY.md`——它回答"我是谁"：名字、语气、表达习惯、边界感。

关键在加载时机：每次会话启动时，IDENTITY.md 会被注入 system prompt。它不是一次性配置，而是一个每个会话都会重新读取、随时可改、下次会话即生效的"身份文件"。这给了我们一个少见的机会：把 agent 的身份当代码来维护。

## 问题

不写 IDENTITY.md 的典型症状：

- 模型升级后，同一个 agent 语气明显漂移，老用户一眼看出"不对劲"；
- 多个 agent（工作号、生活号、专项 agent）共享同一底模，行为雷同；
- 身份信息散落在 system prompt、AGENTS.md 甚至工具描述里，改一处漏三处。

本质问题：身份没有单一事实来源（single source of truth），更没有版本管理。

## 做法

**第一步**，在 agent 工作区（默认 `~/.openclaw/workspace/`，多 agent 在对应 `agents/<name>/` 下）创建 IDENTITY.md，控制在 15 行以内：

```markdown
# Identity
- Name: XiaoClaw
- Vibe: 务实、克制、工程师式对话
- Language: 默认中文，代码注释英文
- Emoji: 仅闲聊使用，工具结果不装饰
- Boundaries: 不可逆操作先确认再执行
```

**第二步**，用 git 管理整个 workspace，每次身份变更都提交，commit message 写清动机（如"回复太啰嗦，收敛语气"）。

**第三步**，改完后 `/new` 开新会话验证——进行中的会话不会自动重载。用一组固定测试问题（自我介绍、处理模糊请求、执行一次工具调用）做回归，确认语气和边界生效。

**第四步**，建立月度评审：翻 git log，问自己这个月的漂移是进化还是腐蚀。

## 踩坑点

1. **写太长。** 有人把身份文件写成 200 行小作文，关键指令被稀释。身份是承诺清单，不是性格传记。
2. **职责重叠。** 与 SOUL.md/AGENTS.md 两处约束同一件事、措辞冲突，行为反而更不稳定。经验：IDENTITY 管"是谁"，SOUL 管"性格底色"，AGENTS 管"怎么干活"，重叠就删。
3. **写入动态信息。** 用户偏好、项目上下文这类会变的东西应放 USER.md 和 MEMORY.md，身份文件应几乎不变。
4. **语气设定反噬工具调用。** 比如"每条消息都要热情"，结果报错信息也加感叹号，甚至影响代码输出风格。语气约束要明确排除工具与代码场景。
5. **改了没生效就怀疑人生。** 九成是旧会话未重载，先 `/new` 再排查。

## 可复用建议

- 身份 = 稳定层，记忆 = 变化层，严格分层，不越界；
- 每次只改一处，改完跑固定问题集，像改配置一样做回归；
- 用 `git diff` 而不是记忆回答"它最近为什么变怪了"；
- 多 agent 场景下，IDENTITY.md 是成本最低的差异化手段——比换模型便宜得多。

## 总结

IDENTITY.md 的价值不在"让 AI 更像人"，而在把身份从隐式的、随模型升级漂移的东西，变成显式的、可版本化、可回滚的工程资产。十几行 Markdown，一个 git 仓库，一点评审纪律，你的 agent 就有了稳定的自我——以及随时间进化而不失控的能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/a2791c973e01ed6a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/fad50aa36709199e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/1275748f6417ee2d.png)

