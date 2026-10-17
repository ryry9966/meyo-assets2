---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41847
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

在 OpenClaw 里跑一个常驻 agent，最直接的扩展方式是把所有能力都写进系统提示词：工作流、输出格式、常用命令……但提示词越长，模型注意力越稀释，token 成本也线性上涨。Skills 机制提供的是另一条路：每个能力以独立目录存在，模型平时只看到一份“目录索引”，真正用到时才读取全文。本质上是渐进式披露（progressive disclosure）在 agent 上下文管理上的落地。

## 问题

我之前把团队的周报生成流程、部署检查清单、日志分析套路全塞在一个约 6000 token 的系统提示词里，出现三个症状：

- 无关任务也会带出无关指令，让模型写代码时它总想顺手“生成周报”；
- 指令之间互相冲突，改一条容易碰坏另一条；
- 每轮对话固定多烧几千 token。

## 做法

1. **建目录**：在 `~/.openclaw/skills/weekly-report/` 下放一个 `SKILL.md`，这是唯一必需的文件。
2. **写 frontmatter**：`name` 保持短小；`description` 是关键——它会被注入系统提示词作为触发依据，应写成“何时使用”而不是名词标题。例如：“当用户要求整理群聊记录、生成周期性总结或格式化汇报文档时使用”。
3. **正文写过程**：指令控制在几十行内，超出的内容拆到 `reference.md`，并注明“需要细节时先读取该文件”。
4. **确定性操作脚本化**：模板渲染、格式校验放进 `scripts/`，SKILL.md 里只写调用方式和参数约定。模型负责判断，脚本负责执行。
5. **验证触发**：重启 gateway 后用自然语言提需求，观察 agent 是否主动读取了 SKILL.md；没触发就回头改 description。

## 踩坑点

- description 写成“周报工具”这类名词，模型很难判断何时该用；改成触发条件后命中率明显上升。
- frontmatter 的值里带冒号没加引号，skill 直接没被加载，且报错很隐蔽——排障先查这里。
- 一度把 API key 写进 SKILL.md，全文被读取时会进入上下文。敏感信息一律走环境变量。
- 同类 skill 描述重叠时模型会选错，我把三个文档类 skill 合并成一个、内部分支后反而稳定。
- 超长 SKILL.md 会把渐进式披露的收益吃掉，正文超过一屏就该拆。

## 可复用建议

- 粗略阈值：同一条指令重复交代三次以上，就该抽成 skill。
- 判断类知识用散文写，执行类步骤写成脚本，不要用提示词模拟确定性逻辑。
- skill 与 MCP 不冲突：MCP 解决“连外部系统拿动态数据”，skill 沉淀“怎么做事的过程知识”。我的部署 skill 就依赖一个 MCP 提供的集群状态。
- skill 目录进 git，和 workspace 一起版本化，方便回溯每次改 description 后触发率的变化。
- 定期翻日志看哪些 skill 从未触发过，删掉或合并，索引本身也是上下文成本。

## 总结

Skills 的价值不在于“多了一套插件系统”，而在于把上下文预算花在刀刃上：索引常驻、细节按需。单个 skill 的成本就是一个 markdown 文件，但配合克制的 description 和必要的脚本化，agent 的行为稳定性和 token 开销都有可感知的改善。建议从一条你重复交代最多的指令开始迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/73b7456f0931f214.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/b21b34f3a04c18c7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/cd6173560068f98f.png)

