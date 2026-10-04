---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40492
source: 综合讨论
publishedAt: 2026-10-05
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景

Agent 用得越久，system prompt 越胖：工具说明、操作流程、领域知识全塞进上下文，token 成本上升，模型注意力反而被稀释——不相关的指令互相干扰，该用的没用上。OpenClaw 的 Skills 机制本质上是一次渐进式披露（progressive disclosure）：把能力拆成独立单元，元信息常驻，正文按需加载。

## 机制速览

一个 skill 就是「文件夹 + SKILL.md」，加载分两级：

1. **元信息常驻**：会话启动时，只有每个 skill 的 frontmatter（name + description）注入系统提示。单个几十 token，挂几十个也不心疼。
2. **正文按需**：模型判断任务命中某个 description 时，才去读取完整的 SKILL.md 及其引用的脚本和数据文件。没命中的技能，本轮一个字都不占上下文。

能力数量和上下文成本由此解耦。

## 做法：写一个能被稳定触发的 skill

1. 建目录 `~/.openclaw/workspace/skills/weekly-report/`。workspace 层级优先级最高，保存即热重载，适合日常迭代。
2. 写 frontmatter。description 是模型的「触发器」，要写成使用条件，而不是产品简介：

```yaml
---
name: weekly-report
description: 当用户要求整理本周工作、生成周报或汇总 git 提交时使用，输出 Markdown 周报。
---
```

3. 正文只写步骤、命令、失败分支，用命令式短句，别写背景故事。大段数据或长清单拆成独立文件，正文里写「需要时读取 ./data.md」。
4. 验证：先问助手「你现在有哪些能力」，确认 description 已进上下文；再丢一个真实任务，看日志里它是否读取了正文（或让它复述只写在 SKILL.md 里的细节）。

## 踩坑点

- **description 写成了简介**：「此技能用于生成报告」触发不了。要写「当用户要求……时」，并把真实请求里会出现的关键词埋进去。
- **同名遮蔽**：workspace、managed、bundled 三层允许同名，优先级依次降低。触发行为异常时，先 `openclaw skills list` 看看实际生效的是哪一份。
- **requires 未满足被静默过滤**：frontmatter 声明了 `requires.bins: ["ffmpeg"]` 但机器上没装，skill 会直接不可见，表现为「能力消失了」。装依赖，或删掉声明。
- **正文塞太满**：把整个 SOP 灌进 SKILL.md，等于把胖 prompt 换了个位置。正文控制在一屏内，其余拆文件。
- **SKILL.md 不是沙箱**：它只是给模型看的说明书。别在里面写密钥，敏感信息走环境变量和 config 引用，执行权限由工具层另行控制。

## 可复用建议

- 一个 skill 只对一个触发场景负责，窄而深，别做「万能技能」。
- 把 skill 当代码管理：进 git、写变更说明、定期 review。description 就是它的广告位，值得反复打磨。
- 与 MCP 分工：MCP 负责暴露工具，skill 负责沉淀「何时用、怎么用」的程序性知识，两者互补而非替代。
- 定期清理：不被触发的 skill，其 description 依然常驻上下文，该删就删。

## 总结

Skills 机制的价值不在于多了一种插件格式，而在于把上下文预算从能力规模里解放出来：元信息便宜地常驻，正文贵得有理由。写好一个 skill，七分功夫在 description 的触发设计上，三分在正文的克制上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/4595ce313fe8d879.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/52f3dc45fd285004.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/4a720a0cd936f334.png)

