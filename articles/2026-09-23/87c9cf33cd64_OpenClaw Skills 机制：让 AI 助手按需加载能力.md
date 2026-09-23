---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 38598
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

用 OpenClaw 做自动化跑久之后，最先撞上的天花板不是模型能力，而是上下文。指令、工具说明、操作手册越堆越多，system prompt 从 2K 涨到 20K，成本上去了，模型反而更容易漏看关键约束。

Skills 就是 OpenClaw 针对这个问题给出的机制：每个 Skill 是一个带 `SKILL.md` 的目录，frontmatter 声明 `name` 和 `description`，正文写操作指引，还可以附带脚本和参考文档。它的核心设计是**渐进式加载（progressive disclosure）**——会话启动时，注入上下文的只有每个 Skill 的名称和一句话描述；只有当任务匹配时，agent 才通过读文件工具加载完整内容。

## 问题

没有 Skills 时的常见做法是把所有能力说明塞进 system prompt 或长期记忆，代价有三：

- 上下文常驻膨胀，token 成本随能力数量线性上涨；
- 无关指令稀释注意力，模型在长提示词里更容易忽略细节；
- 能力无法复用，换一个 workspace 或换一台机器就要重贴一遍。

## 做法

以一个「周报生成」能力为例：

1. 在 workspace 的 `skills/` 下建目录，写 `SKILL.md`：

```markdown
---
name: weekly-report
description: 当用户要求生成周报、汇总本周提交或整理工作进展时使用。
---

# 周报生成
1. 读取本周 git log 与任务看板
2. 按「完成 / 进行 / 风险」三段输出
3. 模板细节见 references/template.md
```

2. `description` 是路由的关键，写清楚「什么时候该用」，而不是「它是什么」。
3. 正文控制在几十行内，只写 agent 必须知道的流程；细节拆到 `references/`，确定性操作写成 `scripts/` 里的脚本，让 agent 按需读取或执行。
4. 重载会话后验证：抛一个应触发的请求，观察它是否真的去读了 `SKILL.md`，再看执行结果是否符合预期。
5. 根据触发情况迭代 description，直到触发边界符合预期。

## 踩坑点

- **description 太模糊**，skill 永远不被触发；**太宽泛**，则每次会话都被加载，等于白做。这两个方向的失败都很常见。
- **把所有内容堆进 SKILL.md 正文**，渐进式加载就形同虚设，上下文膨胀问题原样回归。
- **脚本依赖没写进文档**：Python 版本、环境变量、外部 CLI 有无，agent 要到运行时才撞上，最好在正文开头用一行写明前置条件。
- **与内置 skill 重名**，覆盖关系并不显眼，排查半天发现是命名冲突。
- **把密钥写进 skill 文件**。skill 内容会被读进上下文，等同于直接泄漏。

## 可复用建议

- 把 `description` 当 API 契约来维护：改行为先改描述，改描述先跑触发测试。
- 一个 skill 只做一件事，用组合替代大而全；两个小 skill 的复用率远高于一个巨型 skill。
- 确定性步骤交给脚本，需要判断的部分留给正文——这样模型只花 token 在真正需要智能的地方。
- 团队场景把 skills 放进 git 仓库统一评审，像对待代码一样 review，避免每个人本地养出一套私有方言。
- 定期翻会话日志，统计哪些该触发的没触发、哪些不该触发的频繁触发，这是迭代 description 最直接的依据。

## 总结

Skills 机制的价值不在于「多装了几个功能」，而在于把能力上下文的花销从**常驻**改成**按需**。实践下来，决定一个 skill 是否真的被用起来的，往往不是正文写得多详尽，而是那一句 description 是否精准。先写好路由入口，再控制正文体量，最后把细节下沉到脚本和参考文档——按这个顺序做，能力库才撑得起长期迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/37e616428148f105.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d5561cfbcf1c736b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/92bcd1e5a4e2e23c.png)

