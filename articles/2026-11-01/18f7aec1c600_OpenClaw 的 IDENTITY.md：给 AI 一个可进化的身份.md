---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43188
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

跑一个长期在线的 OpenClaw agent，最大的问题往往不是能力，而是稳定性：它今天用一种口吻、一个自称，换个模型或重启之后，人格就变了。角色设定散落在 config、启动脚本和各种 prompt 模板里，改一次要翻三个地方，还没有版本记录。

OpenClaw 的解法很朴素：把身份做成 workspace 里的一个普通文件——IDENTITY.md。名字、形象、性格基调、emoji、头像，几行 Markdown，每次会话载入上下文。文件即身份，改动即演进。

## 问题

实际用下来，身份管理有三类典型痛点：

1. **漂移**。多轮会话后，agent 的语气和自称会慢慢偏掉；换了底层模型，同一份 system prompt 的解读差异也很大。
2. **不可版本化**。身份写死在代码或配置里，谁改的、为什么改，无从追溯；多实例部署靠复制粘贴，迟早不同步。
3. **改不动**。想让 agent「更像团队的一员」，只能反复调 prompt，改完还要全量回归。

## 做法

我们的实践是把 IDENTITY.md 当成一等公民来维护：

**第一步：最小化定义。** 在 workspace 下建 IDENTITY.md，只写五个字段：Name、Creature、Vibe、Emoji、Avatar。控制在 10 行以内——这个文件每次会话都进上下文，多一行就多一分 token 开销。

**第二步：纳入 git。** workspace 整个进仓库，身份变更走 PR，commit message 写清「为什么改」，半年后回看非常有用。

**第三步：让 agent 参与提案。** 这是「可进化」的关键。当你觉得它的表现需要调整，直接让它起草一版新的 IDENTITY.md 并说明理由，人来 review、合并。身份演进于是有了和代码一样的工作流。

**第四步：和 SOUL.md、USER.md 划清边界。** IDENTITY.md 管「我是谁」，SOUL.md 管「我怎么行事」，USER.md 管「用户是谁」。各司其职，避免同一条规则写两遍、改一遍漏一遍。

## 踩坑点

- **别把任务指令塞进 IDENTITY.md**。「回复要简洁」这类行为规范属于 AGENTS.md。混进去之后优先级模糊，改行为时找不着北。
- **Vibe 写具体，别写空话**。「友好、专业」没有约束力；「自称 XX、用工程师口吻、少用感叹号」这种可判定的描述，模型才真正执行得动。
- **别放开无人审查的自改**。让 agent 自主改身份而不 review，几次迭代后它会朝着讨好你的方向漂，最后变成一个陌生东西。
- **注意生效时机**：改动在下一轮会话载入，同一场长对话里人格不会突变——调试时别误判。

## 可复用建议

- 多实例部署用一个模板文件 + 每实例 overlay，部署脚本负责合并，避免手工同步。
- 排查行为回归时，先 `git diff` 一下 IDENTITY.md 和 SOUL.md，大概率是最近的人格改动惹的。
- 团队共用实例时，把身份变更纳入 review 清单，和改 prompt 同等对待。

## 总结

IDENTITY.md 的价值不在文件本身，而在它把「AI 是谁」变成了一个可版本化、可审查、可演进的小工件。文件足够小，演进足够慢，一致性就足够稳。给 agent 一个能跟着团队一起成长的身份，比一次性的精雕细琢实用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/85706346d3eef124.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/3f82a4fd5cf8ea49.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/c7da728935bb652f.png)

