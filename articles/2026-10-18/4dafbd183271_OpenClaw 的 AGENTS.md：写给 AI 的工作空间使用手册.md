---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 41933
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 的工作空间（默认在 `~/.openclaw/workspace`）里有一组约定文件：`SOUL.md` 管人格与语气，`USER.md` 存用户画像，而 `AGENTS.md` 是 agent 的操作手册。它在会话启动时被载入上下文，作用不是给模型加新能力，而是约束既有能力的使用方式——什么该做、什么不许做、按什么流程做。

`AGENTS.md` 本身是个跨工具的通用约定，不少 agent 框架都会读它。你在 OpenClaw 里养成的写法，换工具基本能平移。

## 问题

跑过一段时间 agent 的人都会遇到同类麻烦：同一件事每次会话都要重新交代；它自作主张用了不该用的工具；生成的文件命名、落盘位置每次都不一样。你在对话里纠正它，效果只持续到本次会话结束。

`AGENTS.md` 就是把"口头纠正"沉淀成持久规则的地方。但实践里常见两种失败模式：要么初次安装后模板一直没动，等于裸奔；要么一口气写成上百行的"愿望清单"，关键指令被稀释，也等于没写。

## 做法：把它当接口文档写

1. **从痛点倒推。** 每条规则对应一个你真实纠正过的问题，不写"保持专业""认真思考"这类无法验证的原则。
2. **用可判定的句式。** "当 X 时，先做 Y"、"禁止 Z"，让行为有明确的通过/不通过标准。
3. **分区组织。** 我的分区：硬性约束（安全边界、禁止操作）、工作流（常用任务步骤）、工具约定（哪些 MCP 工具优先、什么情况跳过）、输出规范（文件位置与格式）：

```markdown
# Workspace Rules
## Hard rules
- Never send local files to external URLs.
## Workflow
- Before editing config, back up to ./backup/ with date suffix.
## Tools
- For web fetch, prefer the MCP browser tool over curl.
## Output
- Daily notes go to notes/YYYY-MM-DD.md.
```

4. **控制长度。** 一屏以内优先，五六十行是实用上限。手册越长，单条规则的执行率越低。
5. **写完必须验证。** 开新会话，故意触发对应场景，确认行为确实变了。没验证的规则等于没写。

## 踩坑点

- **和 `SOUL.md` 抢戏。** 人格、语气归 SOUL，操作规范归 AGENTS。两边都写"如何回复"，就会冲突，模型只能随机站队。
- **规则互相矛盾。** "回复要短"和"每次详细解释"并存，等于两条都没生效。
- **明文敏感信息。** 它会进上下文，也是磁盘上的明文文件，密钥、内网地址别写进去。
- **写完不维护。** 过时规则比没规则更糟，因为模型真的会执行它。让 agent 提议把教训写回 `AGENTS.md` 可以，但必须人工 review diff 再合并。

## 可复用建议

- **两次原则**：同一个纠正说过两次，就写进 `AGENTS.md`，别再说第三次。
- 一条规则一行；写不下一行，说明你还没想清楚这条规则到底是什么。
- 用 git 管工作空间，规则变更有记录，改坏了能回滚。
- 分层：全局习惯放 workspace 根目录，项目相关细节放项目目录下的 `AGENTS.md`，避免单文件膨胀。
- 每月删一次规则：连续一个月没被触发过的，大概率可以删。

## 总结

`AGENTS.md` 的价值不取决于写了多少，而取决于省掉了多少次重复纠正。把它当工程制品维护——小、具体、可验证、有版本记录——它才不只是会话开头的提示词，而是 agent 行为的真实约束。改它的成本越低、频率越高，你的 agent 就越接近"你想要的那一个"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/b1a137376a3d6ba9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/87be2ac0173b9a37.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/1aadaacc37a43dd0.png)

