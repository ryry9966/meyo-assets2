---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38790
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 agent 默认带文件读写和 shell 执行工具，跑自动化任务很方便，但也带来一个绕不开的问题：LLM 的输出本质上是不受信输入。它可能因为上下文歧义、路径拼接错误，或者干脆是幻觉，生成一条破坏性指令。所以 sandbox 的设计前提不是“信任模型”，而是“不信任模型，但让工作照常进行”。

## 问题

社区里被问得最多的一句是：让 Agent 直接操作文件系统，凭什么相信它不会一条 `rm -rf` 把项目目录清掉？

答案很简单：不凭信任，凭架构。模型删不了文件，是因为它在结构上就删不了。

## 做法：四层防线

OpenClaw 的 sandbox 是分层设计，核心原则是**所有强制点都在模型够不到的地方执行**。

**1. Workspace 路径围栏。** 所有文件工具被约束在 workspace root 内。每次路径操作先做 realpath 规范化（解符号链接、解 `..`），再判断是否越界。越界请求直接在 gateway 侧拒绝，不会进入模型上下文。

**2. 工具策略层。** gateway 对 shell 工具做的是命令解析而非字符串匹配：拆出命令树后，按 token 匹配 allowlist/denylist。`rm`、`dd`、`mkfs` 这类默认进高危名单。

**3. 审批门。** 写操作分两档：workspace 内的普通写直接放行；命中高危 pattern 的动作进入人工确认队列。审批逻辑在 gateway，模型无法给自己盖章。

**4. 快照回滚。** 批量写之前自动打 git checkpoint，误删可一条命令恢复。这是兜底，不是主防线。

配置示意：

```yaml
sandbox:
  root: ~/openclaw-workspace
  resolve_symlinks: true
  shell_policy:
    require_approval: [rm, dd, mkfs, "chmod -R"]
  snapshot: git
```

## 踩坑点

1. **符号链接逃逸。** 如果围栏只做字符串前缀匹配，agent 造一个指向 workspace 外的 symlink 就绕过去了。必须 realpath 之后再判界，且在文件打开前判定，防 TOCTOU。
2. **shell 包装绕过。** 策略拦了 `rm`，模型换成 `bash -c "rm ..."` 或 `find -delete`。denylist 要覆盖间接执行入口，更好的做法是收紧成 allowlist。
3. **把 Docker socket 挂进 sandbox。** 等于把宿主机 root 交出去，容器隔离瞬间失效。这类挂载应默认禁止。
4. **MCP 工具绕过主通道。** 有些 MCP server 自带文件工具，不走 gateway 策略层。每个 MCP server 必须声明 scopes，由 gateway 统一裁决。
5. **只在 happy path 测试。** “帮我清理一下目录”这类模糊指令最容易触发大范围删除，安全策略要用对抗性 prompt 专门测。

## 可复用建议

- 默认拒绝、显式放行。allowlist 比 denylist 可维护得多。
- 强制点放在模型读不到也改不到的层（gateway/宿主机），永远不要靠模型“自觉”。
- 用校验用户 HTTP 输入的心态校验模型输出。
- 快照要便宜到可以无脑打。恢复成本低，人才敢真正放权。
- 所有写操作留审计日志，事后能回答“它到底动了什么”。

## 总结

Agent 不误删文件，不是因为模型聪明，而是因为架构上它删不了：路径围栏限制范围，工具策略限制手段，审批门延迟高危动作，快照负责兜底。模型只负责提议，gateway 负责裁决。把安全从 prompt 里搬出来、放进基础设施，这才是 sandbox 模型的真正含义。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/cea78ffce2f19e64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/14e732e531567868.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/751d5f4bd29bb655.png)

