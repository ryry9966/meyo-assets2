---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38538
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 Agent 拥有 `exec` 这类直接执行 shell 的工具，再叠加 MCP、插件扩展后，能力边界基本等于“一个能敲命令的远程同事”。社区里最常被问到的问题之一是：让 Agent 帮忙清理目录，它会不会顺手把别的东西也删了？

## 问题：prompt 约束不构成安全边界

LLM 的行为是概率性的。在系统提示里写“请小心，不要删重要文件”，本质上是在用愿望做工程。任何一次工具调用的参数偏差、一次对上下文的误读，都可能变成不可逆的 `rm -rf`。OpenClaw 的设计前提很明确：**不假设模型不做错，而是假设它会做错，然后让错误出不了边界。**

## 做法：四层收敛，逐级兜底

**1. 容器隔离。** 开启 sandbox 后，Agent 的命令在 Docker 容器内执行，host 的 shell 对它不存在。容器里跑 `rm -rf /`，删的也只是容器自己的文件系统层。

**2. 最小挂载。** 默认只把 workspace 目录以 bind mount 挂进容器，host 其余路径在容器内不可见。没有挂载就没有攻击面——这是整条链路里最便宜也最有效的一层。

**3. 工具策略 + 审批。** tool policy 可以对 `exec` 做白名单/黑名单，未放行的危险命令走人工审批，确认后才执行。

**4. 快照兜底。** workspace 本身是读写挂载，sandbox 不防“workspace 之内”的删除。真正的兜底是把它纳入 git，任何删除都可回滚。

最小配置大致长这样（字段名随版本可能有出入，以仓库 docs 为准）：

```yaml
agents:
  defaults:
    sandbox:
      mode: all      # off | non-main | all
      scope: agent   # 每个 agent 独立容器
```

建议按以下顺序落地：

1. 配置 sandbox mode 与 scope，先在非主力 agent 上验证；
2. 检查挂载列表，确保没有 home、`/`、docker.sock 之类的宽路径；
3. 给 `exec` 配置审批和白名单；
4. workspace 内 `git init`，加一个定时 commit；
5. 做破坏性演练：让 Agent 分别执行 `rm -rf /tmp/test` 和 `rm -rf ~/test`，前者应在容器内生效，后者应直接报路径不存在。演练结果截图存档，作为上线依据。

## 踩坑点

- **workspace 内的删除是真实生效的。** 很多人在这里翻车：开了 sandbox 就以为万无一失，结果 Agent 在 workspace 里做“清理”，git 还没初始化。先上 git，再放开清理类任务。
- **挂载越宽，隔离越假。** 为了省事把整个 home 挂进容器，sandbox 就退化成了一个昂贵的 tmpfs。
- **容器内用 root 跑。** 写出的文件在 host 侧变成 root 属主，之后人肉维护 workspace 会频繁报权限错误。用非 root 用户并处理好 uid 映射。
- **审批疲劳。** 审批提示太频繁，人会变成无脑点“允许”，等于没有审批。宁可花时间收紧白名单，把审批留给真正高危的操作。
- **一次性验证不等于永久安全。** 升级模型、更新插件后，重跑一遍破坏性演练。

## 可复用建议

- 把「sandbox 全量开启 + workspace git 化」当作默认基线，而不是可选项；
- 每个 Agent 独立 workspace、独立容器 scope，避免互相污染；
- 把破坏性演练写进部署 checklist，任何改动后重放。

## 总结

Agent 安全的核心不是预测模型行为，而是收缩它的可达范围。OpenClaw 的 sandbox 用「容器边界 + 最小挂载 + 审批 + 快照」四层结构，把“误删文件”从不可逆事故，降级为一次可回滚的 workspace 内事件。边界画对了，才谈得上放心放手让 Agent 干活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/473fab796ec3401a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/c2e9f44d02b5f345.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/bb5cb962e858eb10.png)

