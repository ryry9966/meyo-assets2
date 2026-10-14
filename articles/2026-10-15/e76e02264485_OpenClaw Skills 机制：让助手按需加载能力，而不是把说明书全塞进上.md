---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是把说明书全塞进上下文
feedId: 41566
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

用 OpenClaw 做日常自动化的人大多经历过同一个阶段：把所有操作说明、命令清单、流程规范全写进 system prompt，或者一股脑挂上一堆 MCP server。结果上下文被常驻内容占满——token 开销高，模型注意力被稀释，反而容易漏看关键指令。

问题的本质是：agent 的能力边界不取决于"你教过它多少"，而取决于"它此刻能想起多少"。

Skills 就是 OpenClaw 对这个问题的回答：把能力拆成独立目录，平时只在上下文里留一行摘要，任务命中时才加载完整说明。和 MCP 对比着看更清楚——MCP 解决"能不能调到工具"，Skills 解决"什么时候该想起这个能力"，两者是叠加关系。

## 机制速览

一个 skill 就是一个文件夹：

```
~/.openclaw/workspace/skills/pdf-report/
└── SKILL.md
```

`SKILL.md` 的 frontmatter 里，真正决定触发的是 `name` 和 `description`：

```yaml
---
name: pdf-report
description: 当用户要求生成或整理 PDF 周报时使用
---
```

加载分三层：

1. **元数据层**：所有技能的 name + description 常驻 system prompt，每个只占几十 token；
2. **触发层**：模型判断当前任务命中某条 description，主动读取完整 `SKILL.md`；
3. **深层引用**：`SKILL.md` 里可以再指向 `scripts/`、`references/` 下的文件，用到才读。

这就是渐进式披露：装 50 个技能，常驻成本也只是 50 行摘要。

## 实操步骤

1. 在 `workspace/skills/` 下建目录，写 `SKILL.md`，description 用"什么时候用我"的句式描述触发条件；
2. 正文写成操作流程：步骤、参数、边界情况，能引用脚本就不要用文字堆命令；
3. 需要外部命令或环境变量的，在 metadata 里声明依赖，网关会据此判断技能是否可用；
4. `openclaw skills list` 确认被扫到，开新会话实际触发一次验证；
5. 稳定后可通过 ClawHub 分享或安装社区技能（装到 `~/.openclaw/skills/` 托管目录），不要直接改内置技能。

## 踩坑点

- **description 太泛会误触发，太窄会永不触发**。"处理文件"不行，"处理 2024 年 Q3 的文件"也不行。写"用户意图 + 场景"，不写具体参数；
- **一个目录塞多个主题**，触发判断会变模糊，宁可拆成多个小 skill；
- **改完不重启就验证**。技能列表在会话构建时注入，改完 `SKILL.md` 要重启 gateway 或新开会话，别在旧会话里反复问"你会不会 X"；
- **缺依赖后置暴露**。依赖 pandoc 之类外部命令的技能，正文第一行先写检测和回退方案，否则触发后才在报错里发现环境缺东西；
- **技能数量失控**。技能多了，description 总量本身也会变成新的上下文负担，定期清理不再触发的。

## 可复用的写法

- 把 skill 当"给新同事的 onboarding 文档"写，不是当配置文件写；
- 确定性逻辑下沉到 `scripts/`，`SKILL.md` 只保留判断和编排，模型负责选路，脚本负责执行；
- 团队内统一 description 的句式和粒度，触发率会明显稳定；
- 长文档拆进 `references/`，主文件控制在模型一次能完整读完的长度。

## 总结

Skills 机制的价值不在"多装了几个技能"，而在把上下文预算从"常驻说明"换成"按需知识"。建议的起步方式：把你 system prompt 里最长的那段说明拆成第一个 skill，观察一周实际触发情况，再决定是否铺开。先解决一个高频场景，比一次装二十个技能有效得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/1598ea7fb1f0598d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/a32bc9cbeba886ed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/22f043236cbd2402.png)

