---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40076
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

OpenClaw 的 agent 是按会话运行的：每次冷启动，模型其实什么都不记得。真正让它"还是它"的，是 workspace 里几份会被注入 system prompt 的 Markdown 文件——AGENTS.md 管操作规则，SOUL.md 管价值观和做事边界，而 IDENTITY.md 管"我是谁"：名字、形态、性格基调、emoji、头像。

很多人装完就用默认身份，跑几周后才觉得"它说话像个客服"。这篇记录我把 IDENTITY.md 当成真正的配置资产来维护的过程。

## 问题

起初我以为身份就是个名字，实际踩了三个坑：

1. **身份散落各处**。开场白写在 AGENTS.md，语气写在 SOUL.md，名字又在别处提了一句，改一次要动三个文件，还经常漏。
2. **人格漂移**。我隔三差五凭感觉改措辞，没有版本记录。某天它突然自称全名加头衔，回溯半天才发现是某次手滑加的一句话，在每轮注入时被反复放大。
3. **Token 预算失控**。IDENTITY.md 每轮都进上下文。我一度写成小作文，长会话里这部分固定开销明显高于它的信息价值。

## 做法

我的 workspace 结构现在固定为：

```
~/.openclaw/workspace/
├── IDENTITY.md   # 我是谁（本篇主角）
├── SOUL.md       # 我怎么做事（价值观/边界）
├── USER.md       # 用户是谁
└── AGENTS.md     # 操作规则
```

IDENTITY.md 本体控制在 15 行以内：

```markdown
# IDENTITY

- **Name:** Moli
- **Creature:** 住在终端里的寄居蟹
- **Vibe:** 低功耗、先确认再执行、讨厌废话
- **Emoji:** 🐚
- **Avatar:** assets/avatar.png
```

具体步骤：

1. **先分工再动笔**：身份类信息（名字/形态/基调）进 IDENTITY.md；行为类（"回复不超过三句"）进 SOUL.md 或 AGENTS.md。判断标准：这条影响"它像谁"，还是只影响"它怎么干活"。
2. **用 git 管 workspace**：每次改身份单独一个 commit，写清动机。人格漂移时可以 diff 回溯，这是最重要的保险。
3. **小步快跑**：一次只改一个字段，跑一两天日常任务观察输出风格，确认没有副作用再改下一处。
4. **让 agent 参与进化**：我会定期让它自己提交一份"你觉得现在的身份描述哪里与实际行为不符"，人工 review 后再落盘。身份从单向配置变成双向协商。

## 踩坑点

- **emoji 不是装饰**。它几乎每条回复都会带，写个不符合气质的 emoji，整体观感立刻拧巴。
- **SOUL 和 IDENTITY 写重了会打架**：我在两处都写了"简洁"，结果它简洁到把确认步骤也吞了。重叠字段只保留一处。
- **avatar 用相对路径**，绝对路径换机器就静默失效。
- **多 agent 场景必须一人一份 workspace**，共用文件必然串人格。

## 可复用建议

- 把 IDENTITY.md 当代码对待：版本化、小步提交、认真写 commit message。
- 字段宁少勿多，15 行是硬预算。
- 身份迭代与功能迭代解耦，别在调试插件时顺手改人格。
- 每月 review 一次，用对话记录里的真实行为校准描述，而不是凭想象。

## 总结

IDENTITY.md 的价值不在"给 AI 起了个名字"，而在把身份变成一份可版本化、可回滚、可协商的活文档。会话是短的，身份是长的——把这个长状态认真管起来，agent 才会在一次次冷启动之后，仍然是那一只寄居蟹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/b3fa464010e27640.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/0402815b22d95bce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/54b06782bfd28f3f.png)

