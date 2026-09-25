---
title: IDENTITY.md：给 AI 一个可进化的身份
feedId: 39013
source: 综合讨论
publishedAt: 2026-09-26
---

# IDENTITY.md：给 AI 一个可进化的身份

## 背景

OpenClaw 的 agent 不只是一段 system prompt。它的工作区里有几个常驻的 Markdown 文件——`SOUL.md`、`IDENTITY.md`、`USER.md`、`AGENTS.md`——每次会话启动时读进上下文。`IDENTITY.md` 是其中最不起眼的一个：默认模板只有几行，很多人装完就直接跳过。

但它解决的是一个长期问题：这个 agent 到底"是谁"。

## 问题

没有认真维护 IDENTITY.md 时，常见三种状况：

1. **身份散落在各处**。人设写在内置提示、写在某次对话的临时指令里。想改个称呼，要翻配置、重启网关，改完还不知道是否生效。
2. **多 agent 无法区分**。当你同时跑三四个 agent（一个写代码、一个管自动化任务、一个守群聊），日志和消息里全是同一种口吻，出了问题不知道该找谁复盘。
3. **身份不会演化**。要么写死不再动，要么随手乱改，三个月后回看，连自己都不认识这个 agent 的定位。

## 做法

我的实践是把 IDENTITY.md 当成**配置文件**维护，而不是一段提示词。

**第一步，放一个最小结构**，十行以内足够：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 一只住在终端里的机械螃蟹
- Emoji: 🦀
- Vibe: 话少，直接，先给结论再给理由
- Attributes: 务实、谨慎、对破坏性操作高度敏感
```

**第二步，划清分工**。IDENTITY.md 只回答"我是谁"：名字、形象、一句调性。行为边界和回复规则放 SOUL.md / AGENTS.md，用户偏好放 USER.md。文件之间不互相抢活，这是后续能稳定演进的前提。

**第三步，用 git 管理工作区**。workspace 整体纳入版本控制。每次发现 agent 哪里不对味——比如群里太啰嗦、该拒绝时太客气——只改 IDENTITY.md 的一行，commit message 写清楚为什么改。

**第四步，多 agent 各持一份**。每个 agent 独立 workspace、独立 IDENTITY.md，字段结构保持一致，靠 emoji 和名字区分。消息里一眼能认出是谁在说话，复盘时也能按身份归档。

## 踩坑点

- **把行为规则塞进 IDENTITY.md**。"深夜不发消息""调用工具前先确认"这类内容应在 SOUL.md / AGENTS.md，混进来会让文件持续膨胀、优先级混乱。
- **写太长**。几百行的人设模型记不住重点，还烧 token。实测十行加一句 vibe，效果不比长文差。
- **让 agent 自己改身份、不设 review**。自我修改叠加几次，人设会漂移甚至自相矛盾。所有改动走 git，人工确认后再合入。
- **以为改完立即生效**。IDENTITY.md 在下一次会话加载时读取，进行中的会话不会重读，改完要开新会话验证。
- **只在出问题时才改**。演化应低频、小步，一次只调一个维度，否则说不清是哪处改动起了作用。

## 可复用建议

1. 把身份当配置：小、稳定、版本化，不当成自由发挥的作文。
2. 演进史交给 git log，不必单独维护变更文档——`git log -p IDENTITY.md` 就是这个 agent 的成长记录。
3. 每月固定 review 一次，删掉过时描述，比随手乱加更有效。
4. 多 agent 场景统一模板字段，降低长期维护成本。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把"agent 是谁"从散落的提示词里抽出来，变成一个可 review、可版本化、可小步演化的对象。写好第一版只需十分钟，真正的收益来自之后几个月里那些一行的、有记录的修改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a498681ff55ad4d1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/3dc7118919cdc6e9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/42aa6e52b0932c23.png)

