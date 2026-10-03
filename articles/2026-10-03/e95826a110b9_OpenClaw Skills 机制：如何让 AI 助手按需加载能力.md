---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40216
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

用 OpenClaw 时间长了，瓶颈往往不是模型不够聪明，而是上下文被塞满：MCP 工具的 schema、系统提示、各类常驻指令，每轮对话都要原样带一遍。工具一多——七八个 MCP server、几十个 tool——token 开销上去了，模型选工具的准确率反而下来。

Skills 是另一种组织能力的方式：把一段"何时用、怎么用"的知识打包成一个文件夹，启动时只注入每个技能的名称和一行描述，模型判断当前任务相关时才加载正文。本质是渐进式加载，用"按需"换"常驻"。

## 问题

我最初想用它解决三件事：

1. 常驻工具太多，系统提示臃肿；
2. 低频但重要的流程（发布前检查、日志排查 SOP）每次都要人工重述；
3. 团队的 prompt 片段散落在聊天记录里，无版本、无复用。

## 做法

最小的一个 skill 就是一个文件夹加一个 SKILL.md（路径以默认安装为例）：

```
~/.openclaw/skills/release-check/
├── SKILL.md
└── scripts/
    └── pre_check.sh
```

SKILL.md 用 frontmatter 声明元数据：

```markdown
---
name: release-check
description: 发布前检查。当用户提到发布、上线、release 时使用。
---

1. 运行 scripts/pre_check.sh，直接执行，不要先读全文
2. 失败项逐条确认后再继续
3. 报告格式见 references/report.md，仅在需要时读取
```

三个要点：

- **description 是路由依据。** 网关只把 name + description 放进系统提示，模型靠它决定是否加载。写"什么场景用 + 能做什么 + 触发词"，比罗列功能有效。
- **正文控制在一两百行。** 细节拆到 references/，按需再读；确定性步骤写成脚本放 scripts/，执行即可，不必进上下文。
- **验证加载行为。** `openclaw skills list` 确认被识别，再抛一个应触发的问题，从日志确认 SKILL.md 是否真的被读入——别只看回答像不像，要看加载记录。

## 踩坑点

- "skill 不生效"九成出在 description：太泛永不触发，太窄乱触发或不触发。先查它，再怀疑模型。
- frontmatter 缩进错误或缺 name 会静默失效，症状和"模型不肯用"一模一样。
- 把 SKILL.md 写成三千字长文，等于换个地方全量加载，按需就失去了意义。
- skill 是提示层约定，不是 MCP 那种硬接口，模型有概率不调用，描述要贴近用户的真实说法。
- 第三方 skill 里的脚本会以你的权限执行，装之前先通读一遍。

## 可复用建议

- 分工：结构化 API、实时数据走 MCP；流程知识、操作规范、团队 SOP 走 skill。
- 从对话记录里挖需求：重复解释过两次的流程，都值得沉淀成 skill。
- 定期审计从未触发的技能：改 description，或直接删。
- 技能目录进 git，review 流程当代码对待。

## 总结

Skills 解决的问题很朴素：让低频能力不占高频上下文。它和 MCP 是互补而非替代。如果你的聊天记录里已经躺着大量复制粘贴的 prompt 片段，值得花一个下午迁过去——迁移成本不高，收益是长期的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/664905bdf1233782.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/fe2d678bebed2271.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/bd613b1477521115.png)

