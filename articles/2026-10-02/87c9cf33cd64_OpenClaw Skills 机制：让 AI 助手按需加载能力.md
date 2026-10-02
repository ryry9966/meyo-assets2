---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 40097
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

OpenClaw 里扩展 agent 能力有几条路：MCP 负责接外部工具，插件负责改运行时行为，而 Skills 沉淀的是"操作知识"——一组结构化的 Markdown 指令，外加可选的脚本和模板。前两者解决"能不能做"，Skills 解决"遇到这类任务该怎么做"。

## 问题：能力装多了，上下文先扛不住

最直觉的做法是把所有能力说明书都塞进系统提示词。三五条还好，攒到几十条之后：会话启动变慢，每一轮对话都背着几万 token 的"死重"，模型注意力被稀释，反而更容易漏步骤。

Skills 的核心设计就是冲着这个来的：**渐进式披露（progressive disclosure）**，加载分三层：

1. 启动时只注入每个 skill 的 `name` + `description`，几十个 skill 也就几百 token；
2. 模型判断当前任务命中某条 description，才读取完整的 `SKILL.md`；
3. SKILL.md 里引用的脚本、模板、参考文档，按需再读。

换句话说，**description 就是路由器**，它写得准不准，直接决定 skill 会不会被真正用起来。

## 做法：写一个最小可用 Skill

目录结构：

```
skills/
└── weekly-report/
    ├── SKILL.md
    └── scripts/
        └── collect_stats.py
```

`SKILL.md` 示例：

```markdown
---
name: weekly-report
description: 汇总近 7 天 git 提交、看板与值班日志，生成周报草稿。当用户提到"周报"、"本周总结"时使用。
---

# 周报生成

1. 运行 `python3 scripts/collect_stats.py --since 7d` 获取结构化摘要。
2. 按团队进展 / 风险 / 下周计划三段整理，不得虚构脚本输出之外的数据。
3. 依赖环境变量 `GIT_TOKEN`，缺失时提示用户配置，不要静默降级。

详细排版模板见 templates.md，仅当用户指定格式时再读取。
```

步骤上：把目录放进 workspace 的 `skills/` 下，重启会话（或触发热加载）后，先让 agent 列出当前可用 skills 做验证；然后不喊 skill 名字，用自然语言描述任务，看它能否自己命中。

## 踩坑记录

1. **description 写得太泛**。"帮你处理数据"这种描述等于没有——要么什么任务都命中，要么永远不命中。改成"当用户要清洗 CSV、去重、处理缺失值时使用"，把触发场景和关键词写实。
2. **SKILL.md 写成万字长文**。它是提示词不是文档，超过一两百行就该拆：主文件只留流程骨架，细节下沉到被引用的文件里。
3. **脚本里写死绝对路径**。换台机器就挂，skill 内一律用相对 skill 根目录的相对路径。
4. **密钥写进 SKILL.md 并提交 git**。改用环境变量，在 frontmatter 里声明所需 env，缺失时明确报错。
5. **两个 skill 触发范围重叠**。模型会随机选一个，行为时好时坏。拆分时按任务边界切，不要按工具切。

## 可复用建议

- **脚本优先**：确定性逻辑交给 bash/python，提示词只做编排和兜底判断；
- 每个 skill 只做一件事，宁可多写几个小的，不要堆一个大而全的；
- 用 git 管理 skills 目录，团队共享时它就是一套可 code review 的 SOP；
- 验证触发方式永远是"自然语言描述任务"，而不是复述 skill 名。

## 总结

Skills 不是又一层插件系统，而是把操作知识做成可懒加载的模块。写好一条具体的 description，把重活交给脚本，把细节推迟到真正需要的那一刻——上下文省下来了，行为反而更稳。建议从一个你每周都在手动重复的流程开始写第一个 skill，收益最快。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/5429014d92631335.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/0be5b17752af2dc0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/31b657ce3877e491.png)

