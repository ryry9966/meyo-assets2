---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37643
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw 的 agent 是常驻进程：接消息渠道、跑定时任务、调各种工具。能力一多，最直接的做法是把所有使用说明都堆进 system prompt——一开始没问题，几周后上下文里躺着几十段"当你需要 X 时应该……"，每轮对话都在为这些大概率用不上的内容付费，模型的工具选择准确率也在下降。

Skills 机制就是为了解决这个问题：把每类能力拆成独立的目录包，运行时按需加载，而不是常驻上下文。

## 核心机制：三层渐进式披露

Skills 的设计核心是 progressive disclosure，理解这三层，其余都是细节：

1. **第一层（常驻）**：会话启动时，只有每个 skill 的 name 和 description 被注入上下文，单个 skill 占用几十 token。
2. **第二层（按需）**：任务匹配某个 skill 的 description 时，agent 才读入该 skill 的 SKILL.md 正文。
3. **第三层（更深的按需）**：SKILL.md 引用的 references/ 文档、scripts/ 脚本只在真正用到时才读取——脚本甚至是执行而非读入，基本不占上下文。

这个分层决定了写 skill 的优化方向：description 决定触不触发，SKILL.md 决定执行质量，脚本决定成本。

## 实操步骤

**1. 建目录结构**

```
~/.openclaw/skills/pdf-report/
├── SKILL.md
├── scripts/
│   └── render.py
└── references/
    └── style-guide.md
```

**2. 写 frontmatter，重点是 description**

description 不是"这是什么"，而是"什么时候用"。写成 `Use when the user asks to generate weekly PDF reports from Markdown files.` 这类句式，给模型一个明确的触发条件。

**3. 正文保持精简**

SKILL.md 只写操作流程：步骤、边界情况、该跑哪个脚本。超过几百行就该拆——参考手册、参数表、示例统统下沉到 references/，正文里留一行"详见 references/xxx"即可。

**4. 能脚本化的都脚本化**

模型读 300 行脚本要花 token，跑 `python render.py --input xx.md` 只要一行。确定性逻辑放脚本里，SKILL.md 只负责告诉 agent 怎么调用。

**5. 真实任务验证**

改完 skill 用 3~5 个真实场景的 prompt 测试：该触发的有没有触发，不该触发的有没有误伤。

## 踩坑点

- **description 含糊**：写"处理文档相关任务"这种描述，模型要么不触发要么乱触发。触发条件要具体到动词和对象。
- **SKILL.md 写成百科**：第二层一加载就顶掉正事。SKILL.md 是操作手册，不是知识库。
- **脚本依赖没声明**：skill 依赖 Python 3.10+ 或某个包却没写清楚，生产环境第一次跑就报错。在 SKILL.md 开头写明依赖和安装方式。
- **与 MCP 工具职责重叠**：MCP 管的是"连接外部服务"，skill 管的是"流程和领域知识"。两者重叠时模型选择会摇摆，规划时先划清边界。
- **skill 目录不进版本管理**：skills 是代码的一部分，进 git，改动可回滚、可审查。

## 可复用建议

- 一个 skill 只解决一类问题，宁可多个小 skill，不要一个大杂烩
- 把 SKILL.md 当成"给新同事的入职文档"来写：假设读者聪明但不了解你的业务
- 脚本做到幂等、支持 dry-run，方便排查
- 定期回顾 description：agent 的实际使用记录会告诉你触发边界准不准

## 总结

Skills 机制本质上是上下文经济学：描述常驻、正文按需、脚本执行，让每一份 token 花在真正需要的环节。对实践者来说，投入产出比最高的一步是把每个 description 打磨成清晰的触发条件——这一步做好了，后面的整条加载链路才会顺畅。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/bbae90e1f81b3867.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/fcb4bb22059fe8ac.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/863b36128f568969.png)

