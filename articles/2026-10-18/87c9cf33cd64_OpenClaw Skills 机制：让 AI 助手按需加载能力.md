---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 41948
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 的 agent 启动时只有一份系统提示和一组基础工具，真正的扩展能力来自 Skills：每个 Skill 是一个带 `SKILL.md` 的目录，YAML frontmatter 声明元信息，正文写具体操作流程。它的关键设计是**渐进披露**——启动时上下文里只注入所有 skill 的名称加一句 description 摘要，正文只在任务匹配时才被读入。理解这一点，才能把机制用对。

## 问题

常见两种反模式：一是把所有操作流程塞进系统提示或 `AGENTS.md`，上下文成本高，不同流程的指令还互相干扰；二是装了一堆 skill 却从不触发，等于纯占目录。

我的实际场景：想让助手在服务器上做三件事——渲染 PDF 周报、批量压缩图片、查询自建服务状态。全写进提示词约 1.5k token 常驻，而且图片压缩的指令偶尔会被错误套用到 PDF 任务上。

## 做法

1. 建目录：`~/.openclaw/workspace/skills/pdf-report/SKILL.md`。workspace 级 skill 优先于内置版本，便于覆盖默认行为。
2. 写 frontmatter：

```markdown
---
name: pdf-report
description: 将 Markdown 周报渲染为 PDF 并校验；仅在用户明确要求导出 PDF 报告时使用。
metadata:
  openclaw:
    requires:
      bins: [pandoc]
      os: [darwin, linux]
---
```

3. 正文只写「agent 不知道而机器知道」的内容：pandoc 命令模板、输出路径约定、校验方式（用 `pdfinfo` 查页数），以及「什么时候不要用」。控制在 40 行以内。
4. 用 `openclaw skills list` 检查状态：requires 不满足的 skill 会被标记为 unavailable 并从提示中剔除，避免带着坏依赖上线。
5. 做两组对话实测：一句明确说「导出 PDF」应触发；一句只说「总结本周」不应触发。

## 踩坑点

- **description 是唯一的路由信号。** 我第一版写「处理报告」，结果纯总结任务也触发了它。改成「仅在用户明确要求导出 PDF 报告时使用」后，边界立刻清晰。正例要写，反例更要写。
- **requires.bins 必须声明。** 没写时机器缺 pandoc，agent 会凭训练知识幻觉出一套调用，排查半小时；声明后条件不满足直接不装载。
- **正文别贪长。** 曾把整份报告模板贴进 SKILL.md，token 又膨胀回去了。解法：模板放同目录附属文件，正文只写「读取 ./template.md」。
- **改动不热生效。** 修改 SKILL.md 后需重启 gateway 或开新会话，改了没反应先怀疑这个。
- **多 agent 部署时 workspace skill 全员可见**，不希望共享的能力要在 agent 配置里做 allow/deny 收敛。

## 可复用建议

- 粒度对齐「一个明确意图」，不要做「大而全的工具箱 skill」。
- 优先包装稳定的 CLI 而非自然语言流程：命令可校验、可用 requires 做环境闸门。
- 每个 skill 正文末尾加「不要用于」一节，对触发边界的收益比想象大。
- 公共素材一律走附属文件，SKILL.md 只当索引和入口。

## 总结

Skills 的价值不在于「装了多少能力」，而在于上下文里常驻的只有索引。把 description 当路由、requires 当闸门、正文当按需文档，三件事分开写清楚，助手的行为边界就会稳定很多。剩下的问题，交给实测对话去收敛。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/a298f499883fdb2e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/a6e387068f9482f7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/73dda2b048a42b93.png)

