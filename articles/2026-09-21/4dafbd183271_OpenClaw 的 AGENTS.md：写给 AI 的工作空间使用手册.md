---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 38299
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

AGENTS.md 是近几年逐渐被各家 agent 工具接受的一个约定：在工作空间根目录放一份 Markdown，agent 启动会话时优先读取它，当作这个项目的操作规程。可以理解为 README 的姊妹篇——README 写给人看，AGENTS.md 写给 agent 看。

在 OpenClaw 的 workspace（默认 `~/.openclaw/workspace`）里，AGENTS.md 与 MEMORY.md 等文件并列，各管一层：记忆归 MEMORY.md，操作规程归 AGENTS.md。agent 每次会话的上下文都是空白的，它对这个项目的全部认知，要么来自自主探索，要么来自你提前写好的这份文件。

## 问题：没有它时，agent 在靠猜

没有像样的 AGENTS.md，典型行为是：

- 靠猜构建命令：`npm run build`、`make` 还是 `cargo build`，猜错就浪费一轮工具调用；
- 把 `dist/`、`*_pb2.py` 这类生成物当手写代码去"修复"；
- 为验证一行小改动跑全量测试，慢，还容易撞上无关的既有失败；
- 每个新会话重新探索一遍目录，token 花在重复劳动上。

更隐蔽的是行为不一致：同一个任务今天改对明天改错，因为约束只在你脑子里，没落到文件。

## 做法：一份能落地的 AGENTS.md

OpenClaw 初始化时自带模板，但默认模板是通用的，真正起作用的是你按自己项目改写后的版本。建议骨架如下，控制在 60 行以内：

1. **项目一句话**：是什么、什么技术栈，一行即可；
2. **常用命令**：build / test / lint / 本地启动，必须是可直接复制执行的完整命令，不要写"运行测试套件"这类描述；
3. **目录约定**：核心目录各放什么，明确标注哪些是生成物、禁止手改；
4. **硬性约束**：不改 lockfile、不升级依赖、不动已有 migration——禁止事项比期望事项更重要；
5. **验证方式**：改动完成后 agent 如何自查，例如"跑 `pnpm test --filter xxx`，只需覆盖改动模块"。

写完做一次验证：开一个全新会话，派一个小改动任务，观察它是否按文档行动。没按，多半是文档没写清楚，而不是 agent 不听话。

## 踩坑点

- **写成 README 二号**。铺满架构愿景和设计哲学，但 agent 需要的是命令和约束，不是愿景；
- **命令不可执行**。"运行格式化工具"没用，`pnpm biome check --write .` 才有用；
- **过长**。几百行之后关键约束会被淹没，agent 实际遵循的会大打折扣；
- **过时**。构建脚本改名后忘了更新文档，agent 会一直撞墙——过时信息比没有更糟；
- **根目录与子目录打架**。monorepo 场景下注意约定优先级，避免两份 AGENTS.md 互相矛盾。

## 可复用建议

- **让 agent 起草第一版**：让它探索仓库后生成草稿，你人工修订，通常比你手写的更全；
- **把踩坑变成飞轮**：agent 犯一次错，就把对应约束写回 AGENTS.md。这份文件的价值来自你真实踩过的坑，不来自模板；
- **随代码走 PR**：改了构建方式或目录结构，同一条 PR 里更新它，像 review 代码一样 review 文档；
- **定期冷启动测试**：每隔一两周用全新会话跑一次典型任务，确认文档仍然有效。

## 总结

AGENTS.md 的本质，是把团队里口口相传的隐性约定第一次显性化。直接收益是 agent 少犯错、少烧 token；间接收益是新人 onboarding 读的也是同一份文件。不必一次写到位——先放五条命令和三条禁令，剩下的靠日常踩坑慢慢补。规则写进文件，才算真正交给 agent。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/849dcb01ef191682.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/ef21ca45b9349660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/cfdd9129b7b99de8.png)

