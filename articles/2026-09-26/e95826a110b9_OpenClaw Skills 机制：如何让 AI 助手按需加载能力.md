---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38999
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景：系统提示词是一种预算

跑过一段时间 Agent 的人都会遇到同一个问题：操作手册越攒越多。把所有 SOP、命令片段、输出格式全塞进 system prompt，代价是三重的——token 成本上涨、模型注意力被稀释、一份巨型提示词改起来步步惊心。

OpenClaw 的 Skills 机制针对的就是这件事：把能力做成“技能文件夹”，启动时只注入一行摘要，任务真正命中时才读取全文。核心思路是渐进式披露——先给目录，再看正文。

## 机制拆解

一个技能就是一个目录，核心是其中的 SKILL.md：

```
skills/pdf-report/
└── SKILL.md
```

技能可以放在工作区的 `skills/` 目录或用户级目录（OpenClaw 兼容 `~/.claude/skills` 这类 Agent Skills 约定路径，认知可以直接复用）。frontmatter 只留两个关键字段：

```yaml
---
name: pdf-report
description: 当用户要求把数据整理成 PDF 周报时使用，含排版模板和生成命令
---
```

运行时分两个阶段：

- **启动阶段**：OpenClaw 扫描技能目录（工作区目录、用户级目录、配置追加的 paths），把每个技能的 name + description 压成一行注入系统提示词，单个技能通常只占几十 token。
- **运行阶段**：模型判断当前任务命中某个 description，用文件读取工具读完整的 SKILL.md，按里面的步骤执行；大型技能还可以引用同目录的参考文件，用到了再读。

也就是说，十个技能常驻提示词的成本，约等于过去一个中等章节。

## 动手写一个

1. 建目录写 SKILL.md，name 用 kebab-case，description 一句话写清触发场景。
2. 正文用命令式短句：步骤、约束、输出格式。目标是“读完就能照做”，不是“读完有感”。
3. 大块材料（长模板、参数表）拆成同目录的引用文件，正文写明何时读哪个。
4. 技能依赖外部工具或环境变量时，在 frontmatter 的 metadata 里声明 `requires.bins` / `requires.env`。环境缺依赖时技能会被标记不可用，不占提示词。
5. 重载后用 `openclaw skills list` 确认技能与依赖状态，再丢一句触发指令，从日志验证模型确实读了全文。

## 踩坑点

- **description 写成名词解释**。“这是一个处理 PDF 的技能”不如“当用户要求生成 PDF 周报时使用”。description 是唯一的路由键，用“何时用”句式写。
- **正文又写成了大 prompt**。SKILL.md 超过一两百行，渐进式披露就白做了。原则：正文放执行路径，细节下沉到引用文件。
- **read 工具没开**。模型拿不到读全文的手段，技能“看起来失效”，只靠摘要硬猜。兜底是在 frontmatter 加 `always: true` 强制全量注入，但这是例外不是默认。
- **同名技能被覆盖**。工作区和用户级目录都放同名技能时，优先级容易和直觉不一致，排查先看日志里实际加载的是哪份。
- **把密钥写进 SKILL.md**。全文会被读进上下文，等于明文进模型。`requires.env` 只用来声明变量必须存在，别把值写进去。
- **改了技能没重载**。以为生效了，实际还在跑旧版。验证改动时，重载是第一步。

## 可复用建议

- 一个技能一个职责，靠组合而不是堆砌。
- 技能进 git 团队共享，symlink 挂进技能目录，技能改动享受和代码同级的评审待遇。
- description 写完做一次自测：假设自己是模型，只看这一行，会触发吗？
- 验证手段日志化：别问“你加载了哪些技能”，直接看读取日志。
- 演进路径：先 Skills 后 MCP。某个技能高频、稳定、需要 API 级集成时再下沉为工具；不确定时，markdown 的试错成本低得多。

## 总结

Skills 是文档化的能力分发：写起来是 markdown，常驻成本是一行摘要，触发成本是一次文件读取。在明确需要 API 集成之前，先写技能、再考虑插件和工具，这是 Agent 能力扩展里投入产出比最高的一层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/23e3762aaaa211b4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/f6aec5f6f06e7dc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/2395f697f21fc13c.png)

