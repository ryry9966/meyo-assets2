---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 39501
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 定位是常驻的个人 AI 助手网关：下面接模型，上面接消息渠道，中间靠工具和技能扩展能力。能力一多，最先撞上的是上下文预算——每个 MCP server 的工具 schema 都是常驻 token，十几个 server 挂上去，还没开始干活，系统提示词就已经很重了。Skills 就是针对这个问题设计的补充机制。

## 问题

把所有能力都做成常驻工具或塞进系统提示词，有三个代价：

1. **token 成本**：工具 schema 逐字计入每次请求，而大部分会话里 90% 的工具根本用不到；
2. **选择噪音**：可选项太多，模型容易选错工具，或在功能相近的工具之间摇摆；
3. **维护成本**：很多"能力"其实只是一段操作流程——用什么命令、注意什么坑，是纯知识，不值得为它维护一个服务。

## 做法：三层渐进加载

Skills 的核心思想是渐进披露（progressive disclosure）。一个技能就是一个文件夹：

```
~/.openclaw/skills/
└── weather-report/
    ├── SKILL.md        # 入口：元数据 + 使用说明
    └── scripts/        # 可选：辅助脚本、参考文档
```

SKILL.md 用 frontmatter 声明元数据：

```markdown
---
name: weather-report
description: 查询指定城市天气并生成中文日报。当用户提到天气、气温、降雨时使用。
metadata:
  requires:
    env:
      - WEATHER_API_KEY
---
```

加载分三层：

1. **常驻层**：只有 name + description 进系统提示词，每个技能几十个 token，网关把它们汇总成紧凑清单；
2. **触发层**：模型判断当前任务匹配某个描述时，自己去读 SKILL.md 正文——步骤、命令、注意事项；
3. **引用层**：正文再引用 scripts/ 或 references/ 下的文件，用到才读。

效果：一百个技能的常驻开销，约等于过去塞一页工具说明。

动手步骤：

1. 在 workspace 技能目录建文件夹、写 SKILL.md，description 里放用户可能说的话；
2. 声明 requires（环境变量、依赖的二进制），网关做可用性检查，缺依赖会被标记而非报错；
3. 用 `openclaw skills list`（或对话内 /skills）确认技能已加载、状态 OK；
4. 用一两句真实请求触发验证，看日志里模型是否读取了 SKILL.md。

## 踩坑点

- **description 写成功能介绍而非触发条件**。模型靠它判断"现在该不该用"，写"这是一个天气工具"基本不会命中；要写"当用户提到 X/Y/Z 时使用"。
- **技能职责重叠**。两个 description 都能匹配同一句话时，命中不稳定，要么合并，要么划清边界。
- **requires 的环境变量没配**，技能长期显示不可用，还以为写错了——先看状态再排查。
- **把密钥写进 SKILL.md**。正文会被模型读取、也可能被借鉴，密钥只走环境变量。
- **乱装第三方技能**。技能本质是注入给模型的指令，存在提示词注入风险，来源不明的先人工审一遍再进目录。
- **该用 MCP 的场景用了 Skills**。Skills 擅长流程性知识；需要结构化参数校验、强类型接口的，还是走工具协议。

## 可复用建议

- 一个技能只做一件事，description 控制在一两句，内嵌触发词；
- 正文超过一屏就拆文件，SKILL.md 只留主干，细节进 references；
- 个人工作流放 `~/.openclaw/skills`，团队共享的进版本库——技能就是 Markdown，code review 成本为零；
- 给不同 agent 配 skills 允许清单，客服 bot 没必要看到部署类技能。

## 总结

Skills 不是替代 MCP，而是把"常驻工具"和"按需知识"分开：结构化接口交给工具协议，操作流程交给 Markdown。描述写好、边界划清、依赖声明完整，技能目录就是一份可供模型按需检索的运行手册。建议先从一两个高频流程开始抽技能，验证这套机制是否适合你的场景，再考虑规模化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/903e13cd61800e53.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/e7ab4c0b425871af.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/6edb2962f768992f.png)

