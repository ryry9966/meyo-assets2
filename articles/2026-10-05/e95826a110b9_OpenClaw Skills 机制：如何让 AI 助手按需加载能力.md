---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40511
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

Agent 玩久了一个问题绕不开：能力越堆越多，system prompt 越来越长。MCP server 一多，工具 schema 全量注入，几十个工具常驻上下文——token 成本上去了，模型注意力反而被稀释，该用的没用上，不该碰的偶尔误触。

OpenClaw 的解法是 Skills：把"能力"拆成一个一个 Markdown 包，运行时按需加载。它遵循的是 Agent Skills 的通用格式，但落地非常轻：一个文件夹 + 一个 SKILL.md，不需要写代码。

## 机制核心：渐进式加载

OpenClaw 在会话启动时，只把每个 skill 的 name 和 description（各一行）注入 system prompt，相当于给模型一份"菜单"。真正的操作说明在 SKILL.md 正文里，只有当模型判断当前任务匹配某个 skill 时，才会用 read 工具把正文读进来。

常驻成本是 O(技能数) 而不是 O(全部指令长度)。装 30 个 skill，平时多花的可能只有几百 token。

## 动手做：三步创建一个 skill

**1. 建目录写文件**

在 `~/.openclaw/skills/`（全局）或 workspace 的 `skills/` 目录下：

```
skills/
  daily-report/
    SKILL.md
```

SKILL.md 最低要求就是 frontmatter + 正文：

```markdown
---
name: daily-report
description: 当用户要求生成当日工作日报、汇总今日会话并推送时使用。
---

# 日报生成流程

1. 读取今天的会话摘要，按项目分组
2. 按模板组织：进展 / 风险 / 明日计划
3. 先给用户确认，再执行推送
```

**2. 验证加载**

```bash
openclaw skills list
```

确认新 skill 在列表中、description 显示正常。

**3. 实测触发**

开一个新会话，用自然语言描述任务，观察它是否主动读取 SKILL.md。也可以直接说"按 daily-report 的流程走"强制触发。

不想手写可以直接从 ClawHub 装现成的：`npx clawhub@latest install <skill-name>`。

## 踩坑点

- **description 含糊 → 永远不触发。**"处理报告类任务"这种写法模型猜不出意图。要写清"什么时候用我"：触发场景 + 输入特征 + 期望产出。
- **description 太宽 → 过度触发。**写成"辅助编码"，几乎每次对话都会加载，白花 token。
- **正文太长。**正文是按需加载，但一旦命中就是全量进上下文。长参考资料拆成独立文件，正文里只写"需要时读取 xxx"。
- **依赖的 CLI 没装。**skill 里写"运行 foo --json"，宿主机却没有 foo，到执行时才报错。建议正文开头列出前置依赖，或用 frontmatter 的 requires 字段声明 bins。
- **改完不生效。**skill 列表是会话启动时注入的，改完要开新会话，别在旧会话里反复测试怀疑人生。
- **沙箱/Docker 环境**注意 workspace 挂载路径与权限，全局目录和 workspace 目录的优先级别搞混。

## 可复用建议

1. **沉淀流程，而不是堆记忆。**凡是"每次都要口头教一遍"的操作，都值得做成 skill。memory 管"是什么"，skill 管"怎么做"。
2. **Skill 与 MCP 分工。**MCP 提供工具接口（API、数据库），skill 承载流程与判断逻辑。很多场景一个 skill + bash 调 CLI 就够了，不必强上 MCP server。
3. **skills 目录放进 git。**SKILL.md 本质是文档，天然适合版本化、review 和跨设备同步。
4. **定期清理。**`openclaw skills list` 里半年没触发过的 skill，要么修 description，要么删掉。

## 总结

Skills 的价值不在"多"，而在"省"：平时只占一行描述，用时才展开完整指令。一句话 description 写得好不好，比十页正文更能决定这个 skill 有没有用。建议从把一个重复性日常流程抽成 SKILL.md 开始，跑通"触发 → 加载 → 执行"链路，再逐步扩充自己的能力库。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/70119d67637f7914.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/1d09c84a6e2fbf69.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/1761d58a579b96f8.png)

