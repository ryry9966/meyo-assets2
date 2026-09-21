---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38355
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

OpenClaw 的 Skills 是一套"渐进式披露"的能力加载机制。它不把所有说明塞进上下文，而是只将每个技能的 `name` 和 `description` 注入系统提示；等 Agent 判断当前任务与某个技能相关时，才把 `SKILL.md` 正文加载进来。这意味着你可以同时挂几十个技能，而日常 token 开销几乎不变。

它和 MCP 是互补关系：MCP 负责接工具（进程、API、资源），Skills 负责沉淀流程性知识——"这类活该怎么做"。很多能力其实不需要一个常驻 MCP server，一份写清楚的 markdown 更省、更稳。

## 问题

实际使用中最常见的两个坑：

1. **全量加载不可行**。把所有操作手册写进系统提示，很快撑爆上下文，模型注意力也会被稀释。
2. **写了没人用**。技能描述含糊，Agent 匹配不到场景，技能等于不存在。

Skills 机制正是针对这两点的解法：描述常驻、正文按需、依赖做门禁。

## 做法与步骤

1. 建目录：`~/.openclaw/skills/<skill-name>/SKILL.md`（工作区技能）。
2. 写 frontmatter，重点在 `description`：

```markdown
---
name: log-triage
description: 排查网关或 agent 日志报错时使用，覆盖日志定位、常见错误模式与修复步骤
---
（正文：步骤、命令、注意事项）
```

3. 用 `requires` 做门禁。技能依赖某个二进制或环境变量时显式声明：

```yaml
metadata:
  requires:
    bins: ["gh"]
    env: ["GITHUB_TOKEN"]
```

不满足条件的运行环境里，该技能不会出现在候选列表，避免 Agent 拿着不可执行的步骤空转。

4. 正文按"什么时候用 + 怎么做"组织。长篇参考资料拆成同目录的独立文件，让 Agent 需要时再通过 exec 工具读取，而不是一次全塞进正文。
5. 验证：`openclaw skills list` 看加载状态，`openclaw skills info <name>` 看依赖与触发条件是否符合预期。

## 踩坑点

- **description 是唯一的触发入口**。写"工具集合"这种描述等于没写，要写成触发条件："当用户要求 X / 出现 Y 现象时使用"。
- **不要滥用 `always: true`**。强制常驻的技能一多，渐进式披露就退化成全量加载，机制白搭。
- **技能不是沙箱**。正文里的脚本通过 exec 工具执行，权限等同于宿主进程，不可信来源的技能别直接丢进 skills 目录。
- **跨机器同步前核对依赖**。正文里的命令要在目标机器跑得通，注意 `bins` 和 `os` 差异，Windows 与 Linux 的路径、包管理器都不同。

## 可复用建议

- 一个技能只解决一类问题，宁可拆成两个，也不要写"大杂烩"。
- 把 description 当作检索 query 来写，写完自问一句：Agent 在什么场景下会"想到"它？
- 技能目录进 Git，改动走 review，和代码同等对待；出问题时能回滚。
- 遇到"技能没触发"，先用 `openclaw skills info` 确认资格状态（依赖是否满足），再回头检查 description 措辞是否贴合真实场景。

## 总结

Skills 的本质，是把"何时需要什么知识"变成运行时决策：元数据常驻、正文按需、依赖做门禁。它不替代 MCP，而是让流程性知识以最低的上下文成本被准确调用。技能写得好不好，标准只有一条——Agent 在该用的时候能不能想到它，想到之后能不能照着跑通。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/9dd6620a16914e66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/cbf3fe5131b681f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/c96a7a63416fbdd1.png)

