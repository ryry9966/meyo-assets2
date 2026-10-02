---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40091
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

OpenClaw 的 workspace 里，真正“定义 agent 是谁”的不是代码，而是几个 markdown 文件：AGENTS.md 管行为规则，SOUL.md 管性格与选择，MEMORY.md 管记忆。IDENTITY.md 是其中最不起眼的一个——多数人初始化时填个名字和 emoji 就再没打开过。用久了会发现，它其实承担着一个关键职责：让 agent 在长期运行中保持自我一致。

## 问题

没写好 IDENTITY.md 的 agent，常见三种症状：

1. **人格漂移**：上下文被压缩或会话重启后，agent 的自称和语气开始随机变化，这次是“小助手”，下次又回到默认口吻。
2. **身份僵硬**：反过来，一次性写死一个详细人设，三个月后使用场景变了，agent 还在按旧设定说话，改起来又怕牵一发动全身。
3. **多实例串台**：跑多个 agent 共用 workspace 时，两个实例抢同一个名字，自我介绍互相污染。

本质矛盾是：身份既要稳定（否则形同虚设），又要可变（否则跟不上需求），中间缺一个受控的演进机制。

## 做法

我现在的 workspace 结构和迭代流程：

1. **骨架放 IDENTITY.md，只放稳定项**：name、emoji、creature（一个短隐喻，比如“住在终端里的寄居蟹”）、description。描述用第三人称写具体特征，避免“乐于助人”这类正确但无信息量的话。
2. **易变项外置**：语气偏好和边界感归 SOUL.md；“我负责 XX 项目的运维”这类场景身份写进对应项目目录的局部说明。IDENTITY.md 控制在 20 行以内——它会被注入上下文，写长了挤占实际工作空间。
3. **用 MEMORY.md 收集摩擦**：日常使用中人设让你不舒服的点（太啰嗦、自称别扭），让 agent 记进 daily notes，打上身份相关标签。
4. **定期小步改写**：每两周 review 一次这些记录，把确认的结论合回 IDENTITY.md。全程 git 管理，commit message 写清楚“为什么改”，形成一条可回溯的身份变更史。
5. **冒烟测试**：改完开新会话，问三个固定问题——你是谁、你如何称呼自己、你的边界是什么——对照 diff 确认行为符合预期。

## 踩坑点

- **IDENTITY 和 SOUL 写重了**：同一句话出现在两个文件，改了一处忘另一处，agent 行为忽好忽坏。分工要清楚：IDENTITY 回答“它是什么”，SOUL 回答“它怎么选择”。
- **绝对化人设**：写了“永远热情、从不拒绝”，跟安全规则冲突时会出现诡异的拧巴言行。用倾向性表述，比如“默认友善，遇到风险明确说‘不’”。
- **改了不生效**：IDENTITY.md 在会话开始时注入，改完当前会话不会变。要么重开会话，要么显式让它重读文件。
- **中英混杂**：文件用英文写、日常对话用中文，agent 自我介绍会夹生。中文用户建议在描述里显式写明自称和语言习惯。

## 可复用建议

把 IDENTITY.md 当代码对待：进 git、走 review、留变更记录。团队场景可以为不同角色的 agent（运维、code review、客服）做身份模板，只改差异字段。身份进化不需要聪明算法，只需要一条便宜的反馈闭环：摩擦记录 → 定期 review → 小步提交 → 冒烟测试。

## 总结

IDENTITY.md 的价值不在“让 AI 更像人”，而在用几十行文本解决长期一致性问题。稳定骨架、外置易变项、版本化迭代——“可进化的身份”工程含量就这么多，但便宜到没有理由不做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/c6d5b2eed6caaf08.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/c22ca921c815e0b2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/aec4b6a74426156c.png)

