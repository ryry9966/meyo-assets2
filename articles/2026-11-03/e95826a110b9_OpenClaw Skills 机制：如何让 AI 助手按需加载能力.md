---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 43306
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

Agent 跑得久了，最典型的膨胀路径是这样的：接了十几个 MCP 工具、攒了几十条自定义指令、还有一堆 SOP 文档，最后全塞进 system prompt。结果是三件事同时变差：

1. 每次会话的固定 token 开销居高不下，长对话还没开始就占掉一大块上下文；
2. 工具注册得越多，模型选错的概率越明显——选项多了，命中率反而降；
3. 能力在多个 agent 之间靠复制粘贴同步，改一处漏三处。

OpenClaw 的 Skills 机制针对的就是这个问题，核心思路是**渐进式披露（progressive disclosure）**：把每项能力做成独立目录，启动时只注入名称和一句描述，正文指令和脚本等真正触发时才加载。

## 做法

一个 skill 的最小结构：

```text
skills/
  log-triage/
    SKILL.md          # frontmatter: name / description + 执行指令
    scripts/parse.py
  deploy-check/
    SKILL.md
```

SKILL.md 的 frontmatter 里，`description` 是唯一的"广告位"——模型完全靠这一行决定要不要加载这个 skill，正文本身在触发前不可见。

落地步骤：

1. **提炼**：挑重复执行三次以上的流程（日志排查、上线前检查、周报汇总），先写成步骤化文档；
2. **收敛**：转成 SKILL.md，正文只放执行该任务必需的指令、约束和输出格式；
3. **外置**：长配置、示例、脚本放同目录，正文里引用相对路径，触发后按需读取；
4. **注册**：目录放进 skills 路径，新会话生效。启动上下文里每个 skill 只占一行元数据，几十 token；
5. **验证**：拿真实任务跑，重点观察两件事——该触发时有没有触发，触发后步骤是否完整执行。

## 踩坑点

- **description 写成"是什么"而不是"什么时候用"**。写"当用户要求分析服务日志、定位错误峰值时使用"，命中率远高于"一个日志分析技能"。前者是触发条件，后者是标签。
- **正文贪长**。SKILL.md 超过一两页，一次加载就把省下的 token 吃回去了。原则：正文是"操作手册"，细节全部拆到资源文件。
- **能力边界重叠**。两个 skill 都声称处理"部署"，模型会犹豫或随机选。要么合并，要么在 description 里划清各自管辖的场景。
- **脚本依赖不声明**。skill 带了 Python 脚本却没写依赖，触发时环境缺包直接失败——这比不触发更糟，因为模型已经基于它规划了执行路径。
- **只加不减**。定期审视长期不被触发的 skill，删掉或改写 description。能力库和维护成本是同步增长的。

## 可复用建议

- description 套一个固定模板：**触发场景 + 输入特征 + 产出物**，三段写完不超过两行；
- 给 skills 目录加个简单 lint：检查 frontmatter 字段齐全、description 长度、正文是否引用了不存在的文件；
- 触发失误的 case 不要浪费，回写进 description，通常迭代两三轮就稳定了；
- 低频操作不值得做成 skill，普通文档引用即可，别为了机制而机制。

## 总结

Skills 机制的价值不在"能力多"，而在**上下文里永远只有当前需要的那部分**。按需加载换来的是三样东西：更准的工具选择、更低的固定 token 开销，以及能力在多个 agent 之间的可移植性——一个 skill 目录拷过去就能用。

建议从一个小而高频的流程开始提炼，跑通"提炼—收敛—验证—改写 description"这个闭环，再逐步迁移存量 prompt。一次性大迁移看起来爽，但排查触发问题时会很痛苦。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/66b193fee93aeb77.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/64dcf6d37e9c862d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/6fadebdabdfdfd89.png)

