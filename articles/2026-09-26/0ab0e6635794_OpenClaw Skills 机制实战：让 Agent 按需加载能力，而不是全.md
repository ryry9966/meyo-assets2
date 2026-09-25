---
title: OpenClaw Skills 机制实战：让 Agent 按需加载能力，而不是全量塞进上下文
feedId: 39027
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 的 Skills 本质上是一种"能力包"机制：每个 skill 是一个目录，核心是 `SKILL.md`（YAML frontmatter + Markdown 正文）。运行时 Agent 只把所有 skill 的 `name` / `description` 元数据放进上下文，正文只在任务匹配时才被读取，也就是渐进式披露（progressive disclosure）。

这个设计决定了两件事：**description 是唯一的触发面，正文是按需加载的手册**。理解这一点，写 skill 的思路就完全不一样了。

## 问题

实践中常见两种反模式：

1. **全量注入**：把所有工具说明、操作流程塞进系统提示。token 成本高，且指令越长遵循越差。
2. **单个体积超大的 skill**：几十个流程写在一个文件里，触发粒度极粗，要么不触发，要么一触发就把一大坨无关内容拉进上下文。

两种做法都会让"能力"变成上下文的负担，而不是资产。

## 做法

**1. 目录结构。** skills 放在 workspace 的 `skills/` 下，一个功能一个目录，目录内 `SKILL.md` 加可选的脚本和参考文件：

```
skills/
  db-rollback/
    SKILL.md
    scripts/snapshot-list.sh
```

**2. 写好 frontmatter。** description 是唯一决定"会不会被加载"的字段，建议两句话结构——什么时候用 + 什么时候不要用：

```markdown
---
name: db-rollback
description: Use when restoring a database to a previous snapshot. Do not use for schema migrations.
---
```

**3. 正文写成操作手册，不是教程。** 模型本来就会做通用的事，你只需要提供项目特有的信息：具体命令、路径、命名约定。不要写"首先让我们了解什么是数据库回滚"。

**4. 二级懒加载。** 把长参考、详细步骤放进正文里引用的独立文件，正文控制在一两屏内。这样即使触发，也只是拉入一份提纲。

**5. 验证触发。** 用真实任务问 Agent，观察实际加载了哪个 skill。不触发就改 description 措辞，误触发就收紧条件。这一步不能省，写完不测的 skill 基本是死代码。

## 踩坑点

- **description 太泛** → 每个任务都命中，挤占其他 skill；**太窄** → 该用的时候永远轮不到它。
- **同名遮蔽**：用户级和项目级目录里出现同名 skill 时，实际生效的可能不是你以为的那个，排查时先查加载顺序。
- **把密钥写进 SKILL.md**——skill 正文会进上下文，等于把秘密广播给每一次会话。
- **数量失控**：skill 超过几十个后，元数据列表本身就变成噪音，需要定期裁剪归档。
- **正文过长**：一触发就拉入三千字，懒加载形同虚设。

## 可复用建议

- **一个 skill 只做一件事**，多用途 skill 迟早要拆。
- 可重复步骤**固化成脚本**，正文只写"运行 `./scripts/xxx.sh`"，比散文指令稳定得多，也更省 token。
- **用 git 管理 skills**，跟项目代码一起评审和回滚。
- 写个审计脚本，列出所有 skill 的名称、description 和正文字数，正文超过阈值（比如 300 行）就强制拆分。

## 总结

Skills 的价值在于把"能力"从常驻上下文变成按需检索的资源。把 description 当搜索键反复打磨，把正文当手册尽量精简，上下文的空间才能留给真正重要的东西。这套机制不难，难的是保持克制： Resist 把所有东西都做成 skill 的冲动，先问一句——这个能力真的需要 Agent 记住吗？

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/7a0fceb072698de2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/0f238eadc5900b48.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9258ad850db301d6.png)

