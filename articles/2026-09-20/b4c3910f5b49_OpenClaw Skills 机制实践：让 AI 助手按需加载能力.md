---
title: OpenClaw Skills 机制实践：让 AI 助手按需加载能力
feedId: 38235
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

OpenClaw 的能力模型很直接：system prompt 定义人格与规则，工具层负责执行。刚开始只接一两个 MCP server 时没什么问题，但当你陆续接上浏览器、定时任务、家庭自动化、自建脚本之后，一个矛盾就出现了——所有能力说明都堆在上下文里，token 占用线性上涨，模型选错工具的概率也明显上升。

Skills 机制就是针对这件事的：把能力做成独立目录，启动时只把每个 skill 的 `name` + `description` 注入上下文（各占一行），模型判断当前任务命中某个 skill 时，才去读取完整定义。也就是常说的渐进式加载（progressive disclosure）。

## 问题

举个真实场景。我让助手每周一生成一份服务器周报，流程涉及读日志、跑统计脚本、按固定模板汇总。之前是每次在对话里口头描述，一写就是三四十行，而且这些描述和日常聊天混在一起，助手偶尔会提前自作主张跑脚本。

核心诉求有三个：

1. **能力可沉淀**：写一次，之后所有会话可用；
2. **上下文要省**：用不到的能力不该常驻 prompt；
3. **触发要准**：该出手时出手，不该时不打扰。

## 做法与步骤

1. **确认 skills 目录**。OpenClaw 会扫描 workspace 下的 `skills/`（也可用全局 `~/.openclaw/skills/`），每个子目录是一个 skill。
2. **建目录，写 `SKILL.md`**。frontmatter 至少给 `name` 和 `description`。description 是路由的关键，建议用"当用户要求/提到 X 时使用"的句式，把触发场景写具体。比如周报 skill：*当用户要求生成服务器周报、或提到每周例行状态汇总时使用*。
3. **正文写操作步骤**：先跑哪个脚本、输出放哪、汇总格式、失败时如何兜底。确定性逻辑尽量放进 `scripts/` 子目录的脚本里，skill 正文只负责"何时调哪个"。
4. **重启 gateway**（或等它重扫），用一个贴近真实的指令做触发测试，比如"帮我把上周机器状态整理一下"，而不是直接念 skill 名。
5. **观察命中情况，回头改 description**。这个迭代通常要两三轮，别指望一次写对。

## 踩坑点

- **description 写太泛**。我第一版写"处理服务器相关任务"，结果聊天里随口提到服务器也触发了。改成明确动词 + 明确产物（生成周报）后才稳定。
- **一个 skill 背太多职责**。周报 skill 一度包含"重启服务"的指令，导致纯汇总场景它也想去执行运维动作。拆开后各自触发都准了。
- **脚本没有执行权限**。容器里忘了 `chmod +x`，agent 反复重试失败却不会主动告诉你原因，翻日志才发现 Permission denied。
- **改了 SKILL.md 没生效**。部分版本对已加载的 skill 不做热更新，改完保险起见重启一次。
- **从旧对话 copy 出来的重复定义**。命名冲突时加载顺序不可控，先合并再拆分。

## 可复用建议

- 判断标准：同一个流程你口头描述超过两次，就值得沉淀成 skill。
- 三层各司其职：**"何时用"写进 description，"怎么用"写进正文，"精确逻辑"写进脚本**。
- skills 目录直接进 git，改动可回溯，也方便团队共享。
- 保持克制：skill 数量超过二三十个时，description 本身也会成为上下文负担，先做合并。

## 总结

Skills 不是什么新范式，本质是把 system prompt 从"全家桶"改成"索引 + 按需取用"。它真正的价值在于三件事：上下文成本可控、能力可沉淀复用、触发边界可以被你显式设计。经验上，写好一行 description，比多堆十个 skill 更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/8248a60b3ac5cd55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/fbac719d539b1f22.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/1958ccfeff39846a.png)

