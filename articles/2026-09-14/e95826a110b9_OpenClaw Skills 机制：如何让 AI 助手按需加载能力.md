---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37519
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 里的 agent 是长期驻留的：系统提示、工具清单、记忆文件常驻上下文。能力越堆越多，最直接的后果是 token 固定成本上升，以及无关指令对当前任务的干扰。Skills 机制就是针对这个问题：技能以目录形式存在，启动时只在系统提示里登记一行 `name + description`，完整的 `SKILL.md` 正文由模型判断相关时再读取——典型的渐进式披露。

## 问题

我之前的做法是把常用流程（部署检查、日志排查、数据备份）直接写进 `AGENTS.md`，累积到两百多行。后果有三个：每次会话固定开销大；做无关任务时这些内容也会占据注意力；改一处要通读全文件。需要一种"只放目录、按需取正文"的组织方式。

## 做法

1. **建技能目录**。放在 `~/.openclaw/skills/<name>/SKILL.md`（或工作区 `skills/` 下），一个技能一个目录。
2. **写 frontmatter**，重点是 description 和 requires 门控：

```markdown
---
name: db-snapshot
description: 当用户要求备份、恢复或迁移 PostgreSQL 数据库时使用
metadata:
  openclaw:
    requires:
      bins: [pg_dump]
      env: [DATABASE_URL]
---

## 步骤
1. 先跑 pg_dump --version 确认可用；
2. 导出到带时间戳的文件名；
3. 用 pg_restore --list 做校验后再报告完成。

## 失败处理
导出失败不要重试超过一次，把 stderr 原样带回给用户。
```

3. **重启或热重载 gateway**，用 `/skills` 查看状态：`loaded`、`gated (missing bins)` 等一目了然。
4. **验证触发**：换几种口语化说法问 agent，观察日志里是否出现对技能正文的读取。触发不到，多半是 description 的问题。

## 踩坑点

- **description 决定一切**。写"数据库工具"这种泛描述基本不会命中；要写成"当用户要求备份/恢复 PostgreSQL 时"，把触发场景直接写进去。
- **`always: true` 别乱加**。它会让正文全量注入，等于退回老路，只留给每轮都必需的安全类技能。
- **requires 是硬门控**。缺二进制或缺环境变量时技能直接不可见，排障先看 gate 原因，别怀疑自己写错了格式。
- **正文避免机器特定内容**：绝对路径、内网地址不要写死，用环境变量或占位说明。
- **技能间互相引用要克制**，A 引 B、B 引 A 会造成加载噪音。

## 可复用建议

- 粒度标准：一个技能对应"一次对话内能走完的一类流程"，跨任务公共部分留在 `AGENTS.md`。
- 把自检写进正文：dry-run、`--check`、导出后的完整性校验，让 agent 自己闭环。
- skills 目录纳入 git，review 时重点看两行：description 和 requires。
- 定期看日志里技能被读取的频次，长期为零的技能要么重写 description，要么删掉。

## 总结

Skills 的本质是"一行索引 + 按需正文"，把上下文预算从"什么都带上"改成"用到才取"。实践中最重要的不是把文档写长，而是把 description 写准、把门控交给 requires、把验证步骤固化进正文——这三件事做完，能力管理就从"堆 prompt"变成了"管目录"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/4d4b5530579d7988.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/02b11035a5011a90.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/76e35d48e14e6016.png)

