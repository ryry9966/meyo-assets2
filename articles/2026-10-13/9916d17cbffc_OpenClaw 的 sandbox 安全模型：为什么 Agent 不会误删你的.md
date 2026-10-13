---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 41388
source: 综合讨论
publishedAt: 2026-10-13
---

# OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件

## 背景

OpenClaw 这类常驻本地的 Agent 网关，核心能力就是让模型调用工具：读写文件、跑 shell、装依赖、操作 workspace。能力越全，"它哪天会不会把我的目录删了"这个担心就越常见。这篇帖拆一下 sandbox 的实际分层，以及为什么在正常配置下，误删是"可回滚事件"而不是"灾难事件"。

## 问题：两难

把 shell 直接交给模型，效率最高但风险不可控；完全不给权限，Agent 又基本没法干活。OpenClaw 的思路不是二选一，而是分层收紧爆炸半径（blast radius）：

1. **工作区即边界。** 文件类工具（read / write / edit）的路径解析锚定在 workspace 根目录内。越界路径、以及指向外部的 symlink，都会被路径解析层拒绝，模型看不到不代表它绕得过去。
2. **命令策略。** shell 工具有独立的 policy：denylist 拦截 `rm -rf`、`sudo`、`mkfs` 这类高危模式，可配置为"直接拒绝"或"升级为人工确认（ask）"。
3. **系统级隔离。** 开启 sandbox 后，执行发生在 Docker 容器里：非 root 用户运行，只 bind-mount workspace 和必要的临时目录。容器里删得再狠，宿主机受影响的也只有挂载进去的那部分。
4. **审批与审计。** 需要越权的操作走 approval 流程，且每次工具调用都有 session 日志，事后可回放。

一个关键认知：**Agent 在 workspace 内部的删除是允许的**。sandbox 的目标不是让它什么都不做，而是保证最坏情况收敛为"workspace 回滚一次"。

## 建议配置步骤

1. gateway 配置里确认 sandbox 处于 docker 模式，workspace 指向**专用目录**，不要指向 `$HOME`（具体字段名以你所用版本文档为准）。
2. 命令策略至少启用 denylist：不可恢复的操作设为 deny，可恢复的设为 ask。
3. workspace 用 git 管理，提交粒度按任务走。执行前有快照，回滚就是一条命令的事。
4. 用金丝雀验证：在 workspace 外放一个 canary 文件，让 Agent 尝试删除，确认被拦截；再回看日志，确认实际走的是哪一层防御。

## 踩坑点

- **Docker 不可用时的静默退化。** 容器起不来时，部分环境会降级为非隔离执行。改完配置务必跑一次金丝雀用例，确认实际生效的隔离层，别只盯着配置文件看。
- **图省事把 `$HOME` 挂进容器**，等于亲手拆掉第 3 层。
- **elevated / full-access 开关用完不关**，是绝大多数事故的前置条件。
- **变量展开类命令**：`rm -rf "$DIR"` 在变量为空时会退化成危险形态，这类模式要在 denylist 里显式覆盖。
- **凭据不要放 workspace。** 删不删得掉别人的文件是一回事，Agent 和容器读得到你的私钥、`.env` 是另一回事。

## 可复用建议

- 一个 Agent 一个 workspace，权限粒度对齐任务粒度，别共用。
- 把"执行前 git commit"写进任务模板，把安全从"指望模型自觉"变成"流程约束"。
- 定期回放 session 日志，比事后猜"它到底干了什么"便宜得多。
- 升级 OpenClaw 后重跑一遍金丝雀用例，把安全验证纳入常规回归。

## 总结

OpenClaw 不是靠"模型足够聪明所以不会删错"来兜底的，而是反过来假设模型一定会犯错，然后用工作区边界、命令策略、容器隔离、审批审计四层，把最坏结果压到可回滚。信任要建立在验证之上：金丝雀测试 + git 快照 + 日志回放，这套组合可以直接抄走。有踩坑经历欢迎在回帖里补充。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/c26770bc1cd6e9c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/762e1f6016dc222c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/2a33a5f302c3fb7e.png)

