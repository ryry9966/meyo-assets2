---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37675
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw 的核心工作方式，是给 LLM Agent 一组工具：文件读写、shell 执行、MCP 工具调用。Agent 的规划能力再强，最终落地都是这些工具调用。只要它能执行 `rm`，"误删"就不是一个假设性问题，而是一个概率问题。

## 问题：为什么 prompt 约束不够

在系统提示里写"不要删除文件"，属于软约束。实践中常见的失效方式：

- 执行"清理临时文件"类任务时，模型对边界判断出错；
- 组合命令引入副作用，比如 `find ... -delete`、`rsync --delete`；
- 被间接注入的内容诱导，执行了删除指令。

所以 OpenClaw 的设计思路不是"劝住模型"，而是让删除操作**要么出不了作用域，要么可以撤销**。

## 做法：四层防御

**1. 文件系统作用域（workspace root）**
Agent 的文件工具默认以 workspace 为唯一可写根。所有路径先做 realpath 解析，再强制前缀校验，symlink 和 `..` 逃逸在这一层被拦掉。workspace 外一律 deny。

**2. 容器级 sandbox**
exec 工具跑在容器里：宿主机只挂载 workspace（必要时附加只读挂载），以非 root 用户运行，网络默认隔离。就算命令层被绕过，容器里也"没有东西可删"。

**3. 危险命令审批门**
`rm -rf`、`sudo`、`dd`、覆盖式写入等模式命中后，进入人工确认队列；无人值守模式下直接拒绝。注意：过滤针对的是解析后的完整命令，而不是信任模型返回的单条字符串。

**4. 删除先行快照**
文件删除默认移入 workspace 内的 trash 目录，定时任务对 workspace 做快照。误删的恢复成本从"不可能"降为"一条命令"。

## 踩坑点

- **symlink 逃逸**：workspace 里一个指向宿主 home 的软链接，能让前缀校验整体失效。必须先 resolve 再校验，并默认禁止 workspace 内出现指向外部的 symlink。
- **`bash -c` 组合命令**：单命令黑名单拦不住 `"rm -rf /tmp/x && rm -rf ~"` 这种拼接。过滤要么下沉到容器/文件系统层，要么对整段脚本做解析后再判断。
- **容器里跑 root + 挂载过宽**：sandbox 配置里最常见的事故。为了图省事挂载整个 home，容器形同虚设。
- **Agent 的"清理冲动"**：磁盘或上下文有压力时，模型可能主动提出清理。给它的临时目录要和资产目录物理分开，别给它"帮忙"的机会。

## 可复用建议

1. 默认 deny、白名单放行，别反过来；
2. 安全边界放在系统能力层（容器、挂载、运行用户），工具过滤层只作为补充；
3. 每一层都假设上一层会被绕过；
4. 审批门的决策日志落盘，出事要能复盘到具体那次工具调用；
5. 定期演练：在测试环境故意让 Agent 执行危险命令，验证整条拦截链路没有静默失效。

## 总结

"Agent 不会误删文件"，不是因为模型足够聪明，而是因为最坏情况下，它的删除操作出不了 sandbox，出得去的也能恢复。把安全放进环境而不是 prompt，是这套 sandbox 模型最值得借鉴的一点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/8b34cba1ed531aca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/1218c6729af9b542.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/9ab2332338cf2469.png)

