---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是全部背着跑
feedId: 39788
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

给 Agent 扩能力，最直接的做法是把所有工具说明、操作手册都塞进 system prompt。能力少时没问题，一旦超过十几个，上下文迅速膨胀：token 成本上升，模型注意力被稀释，真正相关的指令反而被淹没。

OpenClaw 的 Skills 机制走的是另一条路：渐进式加载。会话启动时只注入每个 skill 的元信息（name + description），正文不进上下文；只有当模型判断当前任务命中某个 skill 时，才去读取完整内容。类比：目录常驻内存，条目按需取用。

## 问题

实践中反复出现三类故障：

1. 写了 skill 但从不触发——description 含糊，模型无从判断何时该用；
2. 乱触发——描述太宽，闲聊也把说明书拉进上下文；
3. 触发即爆炸——正文几千字，一次命中就吃掉半个窗口。

## 做法

一个 skill 就是一个目录，核心是 `SKILL.md`，放在 `~/.openclaw/workspace/skills/<名称>/`（工作区级）或 `~/.openclaw/skills/`（托管级）。最小结构：

```markdown
---
name: log-triage
description: 当用户要求排查服务报错、分析日志文件时使用。覆盖取样、错误聚类、时间线还原三步。
metadata:
  environment:
    bins: [rg]
    os: [darwin, linux]
---

# 日志排查
1. 用 rg 按 ERROR/WARN 取样最近 200 行
2. ...（正文控制在 50 行内）
3. 深度规则放 references/depth.md，需要时再读
```

三个要点：

1. **description 是路由契约**。写“什么情况下用”，不写“它是什么”。动词开头 + 明确触发条件，模型才能做出命中判断。
2. **用 metadata.environment 做门控**。依赖的 CLI 不存在、OS 不匹配时，skill 直接不进候选列表，而不是进入列表后执行失败。
3. **正文瘦身，细节外移**。SKILL.md 只留主干流程，长表格、长规则拆成同目录参考文件，让 agent 需要时再加载。

改完后重启会话，聊天里用 `/skill` 确认已注册，再问一句应触发的话，观察 agent 是否去读了 SKILL.md，链路即闭环。

## 踩坑点

- frontmatter 的 description 含冒号没加引号，加载静默失败，线索只在日志里；
- 两个 skill 描述高度重叠，路由摇摆不定——合并，或用条件切开；
- 把密钥写进 SKILL.md。它是纯文本文件，凭据请走环境变量；
- Windows 机器上跑了限定 darwin/linux 的 skill，没配 os 门控，每次都尝试、每次都失败；
- 堆了二十多个 skill 从不审计，描述全是“帮助用户处理各种任务”，等于没有路由。

## 可复用建议

- 一个 skill 只做一件事，宁要十个小的，不要一个万能的；
- skill 承载“流程知识”（做某件事的 playbook），MCP 工具承载“结构化接口”（调 API），两者互补而非互替；
- 每月审计：连续数周未触发的 skill，要么重写 description，要么删掉；
- 高频 skill 配上 emoji 图标，在 `/skill` 列表里一眼可辨。

## 总结

Skills 的写法门槛极低，一个 markdown 文件就够；真正的工程量在 description 的打磨和目录的纪律上。建立“目录常驻、内容按需”这个模型后，Agent 的能力可以持续累积而不拖累上下文——这是它比“什么都塞 prompt”更可扩展的根本原因。

---

