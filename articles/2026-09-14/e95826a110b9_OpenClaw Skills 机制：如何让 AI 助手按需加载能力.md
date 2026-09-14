---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37566
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 框架发展到一定规模都会遇到同一个问题：能力越接越多，上下文越来越胖。早期挂一两个 MCP server，工具定义几百 token 无伤大雅；等 filesystem、browser、数据库、内部 API 十几个 server 全接上，光 tool schema 就吃掉几万 token，模型选工具的准确率肉眼可见地下滑。Skills 机制就是针对这个问题设计的：把"能力"从常驻改成按需加载。

## 问题

传统做法是把所有工具说明和领域文档全部塞进 system prompt，代价有三个：token 成本线性上涨，长会话尤其明显；工具太多导致模型选择困难，误调用、幻觉调用增多；很多流程性知识（比如团队的部署流程）根本不适合写成工具，只能写成文档，而文档全文注入同样烧上下文。

Skills 的思路是分级披露：agent 启动时只注入每个 skill 的一行描述，正文和资源文件等模型判断任务匹配时再读。

## 做法

一个最小可用的 skill 就是一个目录加一个 SKILL.md：

```
skills/
  deploy-check/
    SKILL.md
    scripts/
      precheck.sh
```

SKILL.md 分两部分：

```markdown
---
name: deploy-check
description: 当需要上线前检查服务状态、确认回滚方案时使用。
  仅适用于生产部署，不适用于本地调试。
---

## 步骤
1. 运行 scripts/precheck.sh，解读输出
2. 确认灰度比例配置后再继续
```

frontmatter 的 `name` 和 `description` 在启动时注入上下文；正文是触发后才读取的操作手册；`scripts/` 是确定性执行逻辑，模型只负责调用和解读结果。

实践中三步走：

1. **把重复操作沉淀成 skill**。凡是你会反复口头指导 AI 的流程，都值得写。
2. **description 写"何时用"，而不是"是什么"**。模型靠这句话决定是否加载，必须包含触发场景和排除场景。
3. **确定性逻辑下沉到脚本**。能写成 shell/python 的别让模型自由发挥，脚本输出比模型生成更可复现。

## 踩坑点

- **description 太泛**。写过"日常开发辅助"这种描述，结果几乎每次会话都被加载，等于白做。好的描述要能明确回答"什么任务不该用它"。
- **正文写成论文**。SKILL.md 超过两三千字，模型读一次的代价就抵消了按需加载的收益。长背景材料拆成独立文件，正文只留引用路径。
- **脚本环境依赖没写清**。precheck.sh 在我机器上能跑，分享给同事后因 Python 版本不一致直接失败。脚本开头注释写清依赖和版本。
- **改了不生效**。部分实现里 skill 元数据在会话启动时缓存，改完 SKILL.md 要新开会话才能验证，别在旧会话里反复调试。
- **多个 skill 描述重叠**。两个 skill 都声称处理"部署"，模型会在两者之间摇摆。要么合并，要么在描述里划清边界。

## 可复用建议

- 用 git 管理 skills 目录，演进历史本身就是团队知识库；
- description 用统一模板：触发场景 + 排除场景 + 输出说明；
- 一个 skill 只解决一类问题，宁可多建也不要做"大而全"的万能 skill；
- 验证过的 skill 收进共享仓库，新人接入 agent 时直接挂载，比口头传授流程靠谱得多。

## 总结

Skills 机制的本质是两件事：上下文预算管理，以及程序性知识的外置。它不替代 MCP——MCP 提供能力接口，Skill 提供"什么时候用、怎么用"的使用手册。两者配合，agent 才能在能力持续增长的同时保持上下文干净。建议从团队里最高频的一个重复流程开始写第一个 skill，跑通加载、执行、沉淀的闭环后再扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/5d51ac002aad44cd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/28175534f1446e04.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/d228cbad531354fa.png)

