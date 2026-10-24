---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42458
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 的 agent 不靠硬编码 system prompt 拼人设，而是靠工作区里的一组 Markdown 约定文件：`AGENTS.md`、`SOUL.md`、`USER.md`、`TOOLS.md`、`MEMORY.md`，以及今天的主角 `IDENTITY.md`。每次会话启动，这些文件被注入上下文，agent 由此知道"我是谁、对面是谁、该怎么做"。

`IDENTITY.md` 通常只有几个字段：Name（名字）、Creature（原型）、Vibe（语气基调）、Emoji（头像）。首次运行的 bootstrap 流程会引导 agent 自己给自己起名——这是它自我认知的起点。

## 问题

实践中常见两类翻车：

1. **不写**。默认人格随会话漂移，今天自称"助手"、明天语气时冷时热，多人共用一个实例时尤其混乱。
2. **乱写**。一次性生成几百字小作文塞进去，关键指令被稀释；或者写得太卖萌，一到调用 MCP 工具、跑自动化任务的场景，agent 还在强行撒娇。

## 做法

我的流程四步，全部可复现：

**1. 字段最小化。** 只写名字、原型、语气、emoji，总量控制在 20 行以内。行为约束交给 `SOUL.md`，用户画像交给 `USER.md`——三个文件各管一摊，别互相污染。

**2. 中文直接写。** `IDENTITY.md` 是自然语言 Markdown，中英混排都能被理解。用中文描述语气（"简洁、克制、带一点冷幽默"），比套英文模板再翻译更贴合实际输出。

**3. 建立进化循环。** 这是"可进化"的关键。我设了一个每周例行动作：问 agent"根据最近两周的交互，`IDENTITY.md` 里哪一条已经不符合实际使用？给出修改建议"。它的提案我来审，改动走 Git 提交。三个月下来 diff 一拉，能清晰看到身份如何从"初版想象"收敛到"真实使用习惯"。

**4. 版本化。** workspace 本身就是 Git 仓库，每次变更都写清 commit message（例如"语气从活泼调整为中性，因常用场景转向运维"），人格演进全程可追溯。

## 踩坑点

- **换模型要重校准。** 同一段 Vibe 描述，在 A 模型下是"克制"，换到 B 模型可能变成"冷淡"。升级或更换模型后务必 review 一遍。
- **别写敏感信息。** 这个文件进每次 system prompt，真实姓名、内网地址、任何密钥都不要放。
- **多 agent 别共用。** 每个 agent 的 workspace 各持一份，共享会导致称呼和语气互相串。
- **emoji 别太具体。** 写了 🦀 之后，它在无关回复里也频繁输出螃蟹，工具调用场景显得不专业。现在我只把 emoji 当头像标识，不要求体现在回复里。

## 可复用建议

- 把 `IDENTITY.md` 当 API 对待：字段结构稳定，内容小步迭代，禁止一次性重写。
- 团队部署时准备一份填空模板加三行填写说明，比让每个人自由发挥一致得多。
- 身份注入只占几百 token，是所有个性化手段里性价比最高的：比堆 few-shot 例子便宜，比改代码零风险。

## 总结

`IDENTITY.md` 的价值不在"给 AI 起名好玩"，而在一致性：一段稳定的自我描述，换来稳定的输出风格。而"可进化"不是玄学，是三个工程动作——字段最小化、Git 版本化、定期让 agent 提案人工审核。文件即身份，diff 即成长史。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/062e8bf0c8ac74ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/d9becfb7972882d4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/b2ce0e4a2ad4f7a5.png)

