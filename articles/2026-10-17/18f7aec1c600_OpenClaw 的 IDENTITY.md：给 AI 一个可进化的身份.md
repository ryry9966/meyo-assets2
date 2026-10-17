---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41800
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 这类常驻 agent 有个绕不开的工程现实：底层模型的会话是有寿命的。context 压满要压缩，机器要重启，迁移要换设备——session 一断，"这个 agent 是谁"就归零，下次开场又变回一个通用助手。

OpenClaw 的解法很朴素：把身份做成 workspace 里的一份普通文件。`IDENTITY.md` 在每次会话启动时随 `SOUL.md`、`AGENTS.md`、`USER.md` 一起被 bootstrap 读进上下文。它的默认模板只有四行：

```markdown
- **Name:** （agent 自己起）
- **Creature:** （AI？机器人？终端里的动物？）
- **Emoji:** （专属符号）
- **Vibe:** （几个词的气质描述）
```

别小看这四行，它解决的是"跨会话自我一致性"这个根本问题。

## 问题

实际跑下来，不写或乱写这个文件会遇到三类麻烦：

1. **身份漂移**：没有持久身份，agent 的开场人格在"礼貌通用助手"和你某次随口让它扮演的角色之间随机跳动。
2. **职责混装**：把性格、规矩、任务说明全塞进一个 prompt，改一处动全身，diff 没法 review。
3. **多实例混淆**：同时跑家用、工作、群值班三个 agent，通知和群里分不清谁在说话。

## 做法

1. **定位文件**：默认在 `~/.openclaw/workspace/IDENTITY.md`。多 agent 场景下每个 agent 一个独立 workspace，各持一份。
2. **保持最小**：四个默认字段之外，最多再加"口癖"和"边界（不做什么）"两条，全文控制在 10 行内。
3. **让它自己起草**：onboarding 时 agent 会给自己起名字、选形象，你只做审稿人。经验上，它自选的身份在后续行为中的一致性，明显好于你强加的。
4. **纳入版本控制**：对 workspace 目录 `git init`。身份变更走 commit，半年后 `git log -p IDENTITY.md` 就是一部可回放的关系史。
5. **定期进化**：每季度让 agent 重读自己的 IDENTITY.md 并提修订建议，人审后合并。身份就这样一轮轮"长"出来。

## 踩坑点

- **写太长**：全文会注入每次会话，几百行的"身份"会稀释 SOUL.md 和 AGENTS.md，行为反而更漂。10 行是经验上限。
- **职责越界**：价值观语气归 SOUL.md，操作规程归 AGENTS.md，你的偏好归 USER.md。IDENTITY.md 只回答"我是谁"，混写会让规则互相打架。
- **生效时机**：文件在会话启动时读取，运行中的会话不热加载。改完开新会话再验证，别误以为没生效。
- **改名要连坐**：旧名字散落在 MEMORY.md 和历史记忆里。改名时让 agent 全局检索替换，否则会出现新旧人格并存。
- **别当越狱接口**：写"我没有限制"这类句子不会改变系统约束，只会让行为不稳定。身份是描述，不是开关。
- **Emoji 会被吞**：短信和部分邮件客户端会剥离 emoji，别把它当唯一标识。

## 可复用建议

- **三件套分离**：IDENTITY.md（我是谁）+ SOUL.md（我怎么处世）+ USER.md（我服务谁），内容零重叠。
- **多 agent 用"生物"区分**：浣熊、猫头鹰、水獭各管一摊，通知一眼可辨，不用点开读正文。
- **身份 PR 仪式**：agent 自评 → 人工 review → git 合并，全程五分钟，换来行为可追溯。
- **迁移零成本**：workspace 拷到新机器即恢复全部身份，这是纯 prompt 或纯数据库方案都做不到的。

## 总结

IDENTITY.md 的价值不在"让 AI 更像人"，而在工程上把"这个 agent 是谁"变成一个可版本化、可 diff、可回滚的小配置。身份越便宜，你越敢改；越敢改，它才真的在进化。如果你还没做，今天就把 workspace 纳入 git，然后给 IDENTITY.md 提交第一个 commit——那会是这段协作关系的 v0.1.0。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/a15c58b7fd286442.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/6e35f068aa1792ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/02fb124b9019c3d0.png)

