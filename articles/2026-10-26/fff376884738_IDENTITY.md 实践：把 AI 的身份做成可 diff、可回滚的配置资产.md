---
title: IDENTITY.md 实践：把 AI 的身份做成可 diff、可回滚的配置资产
feedId: 42602
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

OpenClaw 的会话是逐次重建的：每次对话启动时，gateway 会从 agent 的 workspace 读取一组 Markdown 文件（AGENTS.md、SOUL.md、IDENTITY.md、USER.md 等）拼进上下文。其中 IDENTITY.md 的职责很窄——回答"你是谁"：名字、形象、气质、默认风格。它不装行为规则，也不装记忆。

理解这一点很重要：身份文件是**每次会话都会注入**的常驻上下文，改一行，影响的是之后所有会话。

## 问题

我们最初把它当一次性注册表：起好名字、选完 emoji，之后再没动过。几个月下来暴露三个问题：

1. **人设漂移。** 实验期往 SOUL.md 塞了大量语气规则，和 IDENTITY.md 里早期的性格描述互相打架，同一个 agent 在不同会话里表现不一致。
2. **无法回滚。** 某次把 vibe 改得过于"商务腔"，体验明显变差，但没人记得上一版写了什么，只能凭印象重写。
3. **多 agent 串味。** 一台机器跑工作和生活两个 agent，workspace 没隔离，身份字段互相覆盖。

## 做法

**1. 最小化填写。** 当前 `~/.openclaw/workspace/IDENTITY.md`：

```markdown
# IDENTITY.md
- **Name:** 小钳
- **Creature:** 机械龙虾
- **Vibe:** 简洁、直接、偶尔冷幽默
- **Emoji:** 🦞
- **Avatar:** assets/avatar.png
```

控制在 15 行以内，长篇人设留给 SOUL.md。

**2. 整个 workspace 进 git。**

```bash
cd ~/.openclaw/workspace
git init && git add IDENTITY.md && git commit -m "identity: v1 baseline"
```

每次改动单独 commit，重要节点打 tag（如 `identity-v2`）。回滚就是 `git checkout`，不再凭记忆。

**3. 单字段演进循环。** 一次只改一个字段 → 观察 2~3 天日常使用 → 在 journal 里记一句感受 → 决定保留还是 `git revert`。不要攒一波大改，否则无法归因。

**4. 多 agent 隔离。** 每个 agent 独立 workspace、独立 IDENTITY.md，从源头避免串写。

**5.（可选）受控自编辑。** 如果想让 agent 参与身份演进，给它一个只能写 `IDENTITY.draft.md` 的 skill，由你 diff 后手动合并。不要让模型直接改自己的 IDENTITY.md。

## 踩坑点

- **把人设小作文写进 IDENTITY.md。** 它每会话注入 system prompt，30 行以上基本是浪费预算，还会挤占工作上下文。行为规则归 AGENTS.md/SOUL.md，用户事实归 USER.md。
- **vibe 写矛盾形容词**（"极简"又"热情洋溢"），模型会在两极间随机取边，表现不稳定。
- **Avatar 路径写错会静默回退**到默认图，注意它相对 workspace 解析，改完重开会话确认。
- **改完不生效就下结论。** 运行中的会话未必重读文件，改完重开会话再验证。
- **让模型自改身份。** 几轮之后它会把自己改得更"讨喜"，这是自反馈回路，方向不可控。

## 可复用建议

- 把 IDENTITY.md 当"一屏简历"：≤15 行，名字 + 3~5 个气质词 + 1~2 条边界。
- 身份里程碑用 git tag 管理，复盘靠 diff 不靠记忆。
- 固定节奏（每周或双周）review 一次：`git diff` 看变化，删掉过时特质。
- 新人设先在 staging agent 上试跑几天，再合入主力。

## 总结

IDENTITY.md 的价值不在起个好名字，而在把身份变成一份**可 diff、可回滚、有 review 流程**的配置资产。身份稳定是行为稳定的前提；演进要有节奏——一次一个字段，用一段时间真实使用的感受而不是一次对话的直觉，来决定保留还是回退。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/e63479a46816f1bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/20502b26f9660e3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/c36a4bb92e53e841.png)

