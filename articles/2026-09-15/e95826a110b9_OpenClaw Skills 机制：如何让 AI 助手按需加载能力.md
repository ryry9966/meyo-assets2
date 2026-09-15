---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37683
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景：能力清单 vs 上下文预算

Agent 的能力规模和上下文预算是一对矛盾。功能少，助手鸡肋；把所有操作文档塞进 system prompt，token 消耗高，还会稀释注意力——模型在一堆用不上的指令里挑重点，命中率反而下降。

OpenClaw 的 Skills 机制就是针对这个问题：能力以独立目录（skill）存放，常驻上下文的只有一行「名字 + 描述」，正文按需加载。官方说法是 progressive disclosure，工程上理解成惰性加载即可。它和 MCP 是互补关系：MCP 扩展可调用的接口，skill 扩展的是模型「怎么做事」的操作知识。

## 机制拆解

加载分两段：

1. **元数据阶段**：会话建立时，网关扫描所有可用 skill，把 `name` 和 `description` 注入 system prompt，每个 skill 只占几十 token。
2. **触发阶段**：用户请求匹配某条描述时，agent 自主调用 read 读取对应 `SKILL.md` 全文，再按其中步骤执行。

关键在于第二步是模型自己决策的，所以 description 的质量直接决定命中率。

## 实操步骤

以一个「整理截图并归档」的 skill 为例：

1. 在工作区建目录：`<workspace>/skills/screenshot-organizer/`；
2. 写 `SKILL.md`，frontmatter 至少包含：

```yaml
---
name: screenshot-organizer
description: 当用户要求整理、重命名或归档屏幕截图时使用。触发词：截图、桌面太乱、归档图片。
---
```

3. 正文写具体步骤：扫描哪个目录、按什么规则重命名、调用哪个脚本、失败时如何兜底；
4. 重开会话，用 `openclaw skills list` 确认已识别，再用自然语言触发一次，观察 agent 是否先读了 SKILL.md；
5. 有环境依赖就加门控：

```yaml
metadata:
  requires:
    bins: [imagemagick]
```

二进制缺失时 skill 会被标记为不可用，agent 不会盲目调用然后报错。

## 踩坑记录

- **description 写成功能说明书**。「本技能可以处理图片」这种写法基本不会命中。要写触发场景：「当用户说桌面截图太乱、要求按日期归档时使用」。
- **SKILL.md 太长**。超过两三百行，一次读入就是上下文冲击。把重操作拆成脚本文件放在 skill 目录里，正文只写调用方式和边界条件。
- **frontmatter 格式错误**。缩进或冒号写错，skill 会静默失效。排查第一步永远是跑 `skills list` 看它还在不在。
- **多处同名**。bundled、managed、workspace 都能放 skill，同名时按优先级覆盖。改了文件没生效，先用 list 确认实际加载的是哪一份。
- **长会话缓存**。SKILL.md 被读过之后，本次会话内通常不会重读。改完内容请开新会话验证，别在旧会话里反复调。
- **漏写 requires**。依赖 CLI 工具却不声明，agent 会在缺工具的环境里硬调，白费一轮交互。

## 可复用建议

- description 套模板：「当用户要做 X / 提到 Y 时使用。触发词：A、B。」
- 每个 skill 只做一件事，正文控制在一两百行，超了就拆子流程；
- 脚本和模板放进 skill 目录，正文用相对路径引用，方便整目录迁移、分享；
- 新 skill 上线前用干净会话测三类输入：标准触发、近义改写、无关请求（确认不误触发）；
- 高频操作沉淀成 skill，比往 AGENTS.md 里堆规则更省上下文。

## 总结

Skills 机制的本质是上下文经济学：把「知道自己会什么」（常驻、极小）和「具体怎么做」（按需、完整）拆开。写好一个 skill 的成本不在代码，而在 description 的触发设计和正文的克制程度。清单化、小步验证，这套机制就能稳定吃进日常自动化流程里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/43c5b8edde2f41b0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/7162adb2c471923f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/2cf57481a3102920.png)

