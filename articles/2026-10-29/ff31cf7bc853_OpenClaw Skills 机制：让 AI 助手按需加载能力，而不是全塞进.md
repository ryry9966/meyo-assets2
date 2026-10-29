---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是全塞进 prompt
feedId: 42945
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

用 OpenClaw 跑日常自动化一段时间后，最直观的感受是"能力清单"会失控：今天加一条截图脚本说明，明天加一段日历操作，全往系统 prompt 和 workspace 常驻文档里塞。三个月后上下文里躺着几十条指引，每周真正用到的不到五条。

Skills 机制就是针对这个问题设计的：**元信息常驻，正文按需加载**。系统 prompt 只注入每个 skill 的名字加一句 description，agent 判断当前任务匹配时，才去读完整的 SKILL.md。本质是渐进式披露，把"检索哪个能力"这件事交给模型自己。

## 直接堆 prompt 的三个问题

1. 常驻内容吃掉上下文预算，留给任务本身和对话历史的空间变少；
2. 无关指令互相干扰，堆得越多遵循越差；
3. 声明与实现脱节——模型自称会，真执行时命令参数写错。

## 做法

**1. 建目录。** 用户级技能放 workspace 的 `skills/` 下，一个技能一个文件夹，核心是 `SKILL.md`；安装目录自带的 bundled skills 就是现成模板。

**2. 写 frontmatter，重点打磨 description。** description 是 agent 决定"要不要加载"的唯一依据，要写成检索查询的样子，包含用户真实会说的话：

```markdown
---
name: log-digest
description: 汇总本地日志中的报错与异常。当用户提到"日志摘要""报错汇总""昨晚挂没挂"时使用。
metadata:
  openclaw:
    requires:
      bins: [jq]
    env:
      LOG_DIR:
        required: true
---

## 何时使用 / 步骤 / 失败怎么办
（正文：调用命令、参数示例、输出判读、常见失败分支）
```

字段 schema 以官方 skills 文档为准，但 description 决定触发率，值得反复改。

**3. 过程脚本化。** 确定性步骤（拉日志、过滤、拼格式）写成同目录脚本，SKILL.md 里只写何时用、怎么调、怎么判失败。模型执行脚本比现场推理命令可靠得多。

**4. 校验与实测。** `openclaw skills list` 看每个技能是否 eligible——缺二进制或缺 env 会直接标出来；`openclaw skills info <name>` 看加载详情。最后用一句口语化的真实请求测一遍，确认 agent 在对的时机读了 SKILL.md。

## 踩坑点

- **description 太泛**："处理文件"这类描述永远不会触发，或者什么都往它身上引。写清场景和触发词。
- **requires 未满足**：机器上没装依赖二进制，技能整体被判 ineligible，agent 根本看不见，很容易误判成 prompt 问题。
- **SKILL.md 太长**：加载一次吃掉大块上下文，等于把堆 prompt 的问题换了个位置。正文控制在几百行内，背景故事删掉。
- **密钥写进 SKILL.md**：技能内容会进上下文，凭据一律走环境变量。
- **职责重叠**：两个技能描述相近会触发打架，一个技能一个明确场景。

## 可复用建议

- 把 description 当搜索引擎的 query 来优化，用真实失败案例反向补触发词；
- 每次改动后跑一轮真实任务回归，别只看 `skills list` 全绿；
- 对比 MCP：不需要常驻进程和状态的轻能力，用 skills（说明书 + 脚本）足够；需要长连接、复杂状态的服务再上 MCP，两者不冲突。

## 总结

Skills 的价值不在"能装更多插件"，而在于把能力的声明和实现分开：声明常驻、实现按需。一个 markdown 文件就能起步，但 description 和失败分支写得清不清楚，直接决定它是自动化利器还是摆设。建议从"你现在最常重复向助手解释的那件事"开始，把它抽成第一个 skill。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/17ac26dc090ca37d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/47072bed592eb77b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/3a0db901c09a4c27.png)

