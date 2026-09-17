---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 38011
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 的 agent 每次被唤醒，面对的都是同一个工作空间：笔记、脚本、下载的文件、各种 skill 的产物。人类新人入职有 README 和入职文档，agent 却常常只能靠系统提示词和当次对话去"猜"这个空间怎么用。AGENTS.md 就是补上这一块的约定：放在工作空间根目录，每次会话启动时随工作区上下文一起注入，作为 agent 理解"这个空间如何运作"的长期规则。

它和 README 的区别一句话说清：README 写给人看，AGENTS.md 写给 AI 看。它不是提示词模板，更像一份操作规程。

## 问题

没有 AGENTS.md 时的常见症状：

- 同一件事每次都要口头交代（"生成的文件放临时目录，别污染根目录"）；
- agent 自作主张建目录，几周后工作空间长成灌木丛；
- 多实例部署时每台机器行为不一致；
- 换模型或升级版本后，之前调教出的习惯全部丢失。

本质是：会话记忆解决不了空间级约定，系统提示词又不适合塞个人和项目细节。

## 做法

1. 在工作空间根目录创建 AGENTS.md，控制在百行以内。上下文窗口很贵，长文档会稀释关键指令。
2. 分区块写，推荐五个部分：空间结构（哪个目录放什么、哪些只读）；命名与格式约定；工具与技能使用规则（何时用哪个 skill、调 MCP 工具的前置条件）；安全边界（哪些路径不写、删除类命令先确认）；输出习惯（回复语言、日志追加而非覆盖）。
3. 给个最小示例：

```markdown
# Workspace Conventions
- generated files → /tmp/scratch, never pollute workspace root
- daily notes → notes/YYYY-MM-DD.md, append only
- any file-deleting shell command: ask before run
- browser skill only for docs you have whitelisted
```

4. 写完用真实任务验证：让 agent 做一次"整理上周笔记"这类常规操作，观察遵守情况。不遵守就改写措辞——祈使句、放靠前位置、补一句原因。
5. 注意分层：工作空间级 AGENTS.md 管"家里怎么住"，进入某个 git 仓库干活时，仓库自己的 AGENTS.md 管"这个项目怎么改"，两者叠加生效。

## 踩坑点

- **写成介绍文档而不是行为规则。** 大段"我的项目是什么"没用，要写可执行的指令。
- **规则互相冲突。** 既写"自动备份"又写"删前确认"，agent 只能随机选一个，冲突规则等于没有规则。
- **太长。** 超过几百行后遵守率明显下降，重点被淹没。
- **不进版本管理。** AGENTS.md 也要进 git，改了什么、为什么改，要有记录，否则排障时无从对照。
- **把秘密写进去。** API key、内网地址不要写，这个文件会完整进入模型上下文。

## 可复用建议

- 把它当团队规约维护：agent 犯一次错就补一条规则，迭代优于一次写全。
- 与 SOUL.md、MEMORY.md 分工明确：身份人格放 SOUL.md，演变中的事实放 MEMORY.md，静态空间规约放 AGENTS.md，三者不重复。
- 公共约定抽成模板，新工作空间复制后只改差异段。
- 定期让 agent 复述它理解的规则，和文件对不上的地方，就是你写得模糊的地方。

## 总结

AGENTS.md 的价值不在"多写了一份文档"，而在把散落在对话里的口头约定，固化成版本化、可迁移的空间级配置。对 OpenClaw 用户来说，这是成本最低、见效最快的调优手段之一：一次编写，每次会话生效，随 git 走。先从十行规则开始，比追求完美文档更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/391c8a8de4f0b9ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/11c3b968ac3faa36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/2317c26209ed23e4.png)

