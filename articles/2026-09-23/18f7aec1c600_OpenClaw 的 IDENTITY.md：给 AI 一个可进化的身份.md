---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38622
source: 综合讨论
publishedAt: 2026-09-23
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

OpenClaw 的 workspace 里有几个不起眼的小文件：AGENTS.md 管操作规范，SOUL.md 管性格底色，而 IDENTITY.md 回答的是最基础的问题——"我是谁"。名字、形象、emoji、头像，加上你自己扩展的边界与口径。文件通常只有十几行，但它会被注入每次会话的上下文，本质上是身份层的"配置文件"。

## 问题

很多人默认 agent 的自我描述散落在各处：系统提示词里写一句，某个插件里写一句，聊天中口头纠正一句。结果是：

- 同一个助手今天自称 A，重启后口径又变了；
- 多 agent 场景下互相串人格；
- 想调整语气或边界时，找不到该改哪一行。

长期跑下来你会发现，行为不可控的根因往往不是模型能力，而是身份定义从未被当作一个可管理的资产。

## 做法

**第一步，定位文件。** 默认在 workspace 根目录（`~/openclaw/workspace/IDENTITY.md`）。多 agent 时每个 workspace 各一份，不要共享。

**第二步，写最小版本：**

```markdown
# IDENTITY.md
- **Name:** 小爪
- **Creature:** 一只务实的机械猫，回答简短、工程化
- **Emoji:** 🐾
- **Avatar:** avatars/claw.png

## 边界
- 不确定就先问，不编造
- 删除、支付类操作必须二次确认

## 口径
- 中文回复，技术名词保留英文
```

**第三步，纳入 git。** 每次修改一次 commit，message 写清动机——"为什么改"比"改了什么"重要。

**第四步，建立进化机制。** agent 可以起草 diff（"最近确认太啰嗦，建议把边界第 2 条改成……"），但合并权在人。每月或一个大版本后统一 review，而不是随手热改。

## 踩坑点

1. **写成简历或营销文案。** 三百字的"使命愿景"会稀释注意力，行为反而漂移。30 行以内是经验值。
2. **职责重叠。** 身份文件里写工具用法、AGENTS.md 里写人设，三处描述同一件事，改一处忘两处。原则：IDENTITY.md 只回答"我是谁、边界在哪"，操作流程归 AGENTS.md 和 TOOLS.md。
3. **无人审查的自我进化。** 让 agent 自主改身份而不 review，一次失败的自我总结就可能固化成错误人格。
4. **只换 emoji 和头像就当完成"身份升级"。** 视觉身份是表层，行为身份（边界、口径、确认规则）才是大头。
5. **多 agent 用错 workspace 路径。** 身份串门，排查半天才发现读的是隔壁的文件。

## 可复用建议

- 把身份当代码管理：小步提交、可回滚、有历史可查。
- 排障顺序固定：agent 行为异常时，先问"是工具问题，还是身份模糊？"再看日志。
- 给 agent"提议权"而非"修改权"，这是安全与进化的平衡点。
- 换模型或升版本后，重读一遍 IDENTITY.md——不同模型对同一段人设的服从度不同，常需要微调。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把"AI 是谁"从口口相传变成了版本化的配置：小、可 diff、可回滚、可复盘。这正是长期运行的 agent 能保持行为一致又持续进化的朴素基础。如果你还没动过这个文件，建议今天花十分钟写一版，跑两周再看。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/4e16e62a2da5f98a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/3911d4fb197caa1e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/1b6705cfa3e2f03c.png)

