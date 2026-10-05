---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40570
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

OpenClaw 这类 agent 最大的特点是“有手”：能执行 shell、能读写文件、能调 MCP 工具。能力越大，误操作面越大。社区里最常见的一类担心是：让 agent 清理目录、重构项目，它会不会一个路径解析错误就把不该删的东西干掉了？

OpenClaw 的回答很工程化：**默认模型不可信、工具调用不可信，安全靠运行时边界，不靠提示词里写“请小心”。**

## 问题拆开看

误删文件需要同时满足几个条件：agent 有执行能力、能碰到目标路径、命令没有二次确认、删了无法恢复。安全模型的价值是把这几个条件逐个打断，而不是祈祷模型那天状态好。

## OpenClaw 的分层做法

1. **运行时隔离**。开启 sandbox 后，agent 的 exec 和文件工具默认跑在 Docker 容器里，宿主机对它不可见。workspace 通过 bind mount 挂进去，挂多少、暴露多少。
2. **路径边界**。写操作被限制在挂载的 workspace 内，挂载点之外的路径对它等于不存在，软链接逃逸也由容器隔离顺带处理。
3. **工具策略**。tools 层支持 allow/deny，删除类高风险命令可以进 deny 名单，或降级为需要人工 approval。
4. **执行确认**。exec approvals 打开后，破坏性命令会先交给用户确认，agent 自己绕不过去。
5. **快照兜底**。workspace 写操作配合快照/回收策略，前面全漏了也还能回滚。

## 配置步骤（最小可用版）

1. sandbox 范围设为 `all`（至少 `non-main`，保护主会话）；
2. bind mount 只挂任务需要的目录，别图省事挂 home；
3. tools policy 收紧，删除类命令进 deny 或 approval；
4. 打开 exec approvals；
5. 演练一次：丢给 agent 一个“清理临时目录”的任务，翻日志看它的每一步工具调用，确认边界真的生效。具体字段名以你所用版本文档为准。

## 踩坑点

- **挂载太大**：把整个 home 挂进容器，沙箱只剩心理安慰。
- **容器内 root**：不降权 + 可写挂载，隔离强度大打折扣。
- **MCP/插件不走沙箱**：第三方 MCP server 如果跑在宿主机上，它的文件操作不经容器边界，要单独审权限。
- **提示词不是安全边界**：system prompt 写“不要删文件”只是建议，不是约束。
- **gateway 与 sandbox 权限差**：gateway 在宿主机、agent 在容器里，报权限错误时别下意识去提权，先想是不是本来就不该放行。

## 可复用建议

- 把“模型今天可能抽风”当成默认设计假设；
- 权限最小化，宁可 agent 抱怨，也不预先放开；
- 破坏性操作设三道闸：路径白名单、人工确认、可回滚；
- 定期做“红队演练”：给个模糊任务，观察 agent 撞到边界时行为是否可预期；
- 审计日志落盘，且 agent 自身没有写权限。

## 总结

Agent 不误删文件，靠的不是模型听话，而是让它“够不着”。OpenClaw 的 sandbox 模型本质是分层防御：容器隔离管“看得见什么”，工具策略管“能做什么”，approval 管“什么时候要人点头”，快照管“出了事怎么办”。任何一层单独看都不完美，叠起来才是真正的边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/75b48879388c5b78.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/45268b9680ec7373.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/f388a79fdb0044a0.png)

