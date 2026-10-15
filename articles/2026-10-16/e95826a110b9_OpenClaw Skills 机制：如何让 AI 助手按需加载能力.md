---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41638
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

跑过一段时间 Agent 的人都有体感：能力越多，system prompt 越臃肿。早期的常见做法是把所有工具说明、操作手册全量塞进上下文，结果 token 成本线性上涨，模型注意力被稀释，反而更容易答非所问。OpenClaw 的 Skills 机制就是针对这个问题：把能力拆成一个个独立技能包，平时只暴露一句话摘要，用到时才加载正文。

## 问题

我们想要的效果很朴素：常驻上下文里只保留最小路由信息，同时保证模型在真正干活时能拿到完整操作指引——该调哪个 CLI、参数怎么填、失败怎么兜底。全量加载和完全不加载之间，需要一个按需加载的中间层。

## 做法

Skills 的本质是一个文件夹加一份 SKILL.md，加载分三层：

1. **会话启动**：只有 frontmatter 里的 `name` 和 `description` 进入系统提示词，单个技能成本几十 token；
2. **触发加载**：模型判断当前任务匹配某技能时，才把 SKILL.md 正文注入上下文；
3. **按需深读**：正文里引用的附属文件（模板、示例、参考文档）再按需读取。

一个最小可用技能长这样：

```markdown
---
name: pdf-extract
description: 从本地 PDF 提取文本与表格。当用户提到 PDF、发票、扫描件的解析或汇总时使用。
metadata:
  os: [darwin, linux]
  requires:
    bins: [pdftotext]
---

## 步骤
1. 用 pdftotext -layout 保留排版提取
2. 扫描件先判断是否需要 OCR
3. 输出统一为 markdown 表格，无法识别的字段标 TODO
```

放到工作区 `skills/` 目录或 `~/.openclaw/skills/` 即可，插件也可以携带技能。用 `openclaw skills list` 查看加载与 gating 状态，`skills info <name>` 会告诉你技能为什么被拦：缺二进制、缺环境变量或 OS 不匹配。

`requires` 是关键设计：声明依赖后，环境不满足的技能直接不进路由，避免模型拿着不存在的工具瞎编命令。

## 踩坑点

1. **description 写成宣传语**。它是路由依据，不是广告位。"强大的文档处理工具"这种描述基本不会触发；写清楚"做什么 + 什么场景用 + 用户会说什么词"才有效。
2. **一个巨型技能包打天下**。十几个不相关操作堆进一个 SKILL.md，触发条件模糊，正文过长还会稀释指令。按职责拆小。
3. **该做 MCP 的做成了 Skill**。需要 schema 校验、结构化返回、跨客户端复用的能力，MCP server 更合适；Skill 擅长的是流程性知识——教模型按什么顺序调 CLI、怎么处理边界情况。
4. **忽略正文是按需注入的**。不要假设模型时刻记得技能细节；但同会话内一旦加载就常驻，多个技能间别重复写大段相同内容。
5. **依赖没 gate**。不写 `requires` 时，环境里没有 pdftotext，模型会硬编一个近似命令，报错信息非常有迷惑性。

## 可复用建议

- description 公式：能力一句话 + 触发场景 + 触发关键词，控制在两行内；
- 从最小 SKILL.md 起步，跑几天真实对话，观察"该触发没触发 / 不该触发乱触发"，再迭代正文；
- skills 目录进 git，改动可回溯，团队可共享；
- 定期用 `skills list` 审计，删掉长期没触发过的技能；
- Skill 管流程、MCP 管工具，两者组合使用：Skill 正文里直接引用 MCP 工具名，是常见且顺手的写法。

## 总结

Skills 解决的不是"能力从哪来"，而是"能力何时进上下文"。把路由信息做到极致地小，把操作知识做到恰好够用，Agent 的上下文才能长期保持干净。如果你已经在用 MCP 和插件，值得花半小时把自己重复用的几段 prompt 固化成技能包——这笔投入的收益会随时间复利。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/246def621778e213.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/e99767c2fa434cd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/615c65cba7720830.png)

