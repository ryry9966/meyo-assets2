---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41920
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

Agent 长期运行有个现实问题：能力越多，提示词越臃肿。早期我把所有操作说明、注意事项全塞进 system prompt 和 AGENTS.md，结果上下文里常年躺着几千 token 的“说明书”，模型真正用到的不到一成，还会稀释注意力——该遵守的规则反而被忽略。

Skills 就是针对这个问题设计的机制，核心是**渐进式披露**：启动时只把每个技能的 `name + description` 注入上下文，正文（SKILL.md 的剩余部分）只在模型判断需要时才加载。一段描述几十 token，正文可以是几百行，相当于给能力加了一层懒加载。

## 技能长什么样

技能就是一个带 frontmatter 的 Markdown 文件：

```markdown
~/.openclaw/skills/weather/SKILL.md
---
name: weather-lookup
description: 查询指定城市的实时天气。当用户问天气、出行建议时使用。
---
## 步骤
1. 用 exec 工具执行 `scripts/fetch.sh <城市>`
2. 解析 JSON，只汇报温度、降水、风力
## 注意
- 查不到就明说失败，不要编数据
```

放 `~/.openclaw/skills/` 是全局技能，`~/.openclaw/workspace/skills/` 是工作区技能（适合跟项目走）。

## 做法：五步接一个技能

1. 建目录和 SKILL.md，frontmatter 必须有 `name` 和 `description`；
2. description 写成“触发条件 + 能力边界”，这是模型决定是否加载的**唯一依据**；
3. 正文只写具体步骤、脚本路径、失败处理，别写客套话；
4. 辅助脚本放进技能目录，正文用相对路径引用；
5. 新开会话后问一句“你现在有哪些技能”，验证识别情况。

需要精细控制时，在 openclaw.json 里用 `skills.allow` / `skills.deny` 按名字开关；多 Agent 场景下可以给不同 agent 配不同的技能集合。

## 踩坑点

- **description 太模糊**（“处理各种数据”）→ 永远不触发；**太宽泛**（“回答任何问题”）→ 频繁误加载。把它当函数签名写。
- 期望 SKILL.md “执行”代码是误解：技能只是指令，真正执行仍靠 exec 等工具，技能负责让模型“用得对”。
- 正文塞太长，触发一次就吃掉大几百 token。控制在两屏以内，细节放脚本注释里。
- 两个技能描述高度重叠会同时触发、互相打架，按职责切干净。
- 路径问题：exec 的工作目录未必是技能目录，脚本里先 `cd` 到自身所在目录更稳。
- 改了技能不生效，多半是旧会话还持着旧上下文，新开会话再看。

## 可复用建议

- 一个技能只做一件事，“什么时候用 / 什么时候不用”写进 description；
- 把团队沉淀的 SOP（部署流程、告警处置、数据导出）逐个固化为技能，比写长文档更容易被 agent 真正用上；
- skills 目录用 git 管理，PR 只评审 description——等于在给模型行为做 code review；
- 定期让 agent 列出已加载技能和触发频率，清掉长尾。

## 总结

Skills 的价值不在于“能力更多”，而在于让能力有了清晰的路由：廉价的元数据常驻，昂贵的细节按需进场。写好 description，等于给 agent 装了一张靠语义而非硬编码的路由表。对要让助手接手真实运维和日常流程的场景，这是目前性价比最高的扩展方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/d6620a0c80b23b66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/7b4ea8746fed21e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/67f07e538ac2f865.png)

