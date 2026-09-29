---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39544
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 这类本地 Agent 网关最大的特点是：它不只会聊天，还能通过 exec 工具在你机器上直接跑 shell。第一次部署完，很多人的反应都一样——这不等于给一个"语言能力很强但确实会犯错"的东西发了把能碰真文件的钥匙？

## 问题

指望模型自律是不成立的。在 system prompt 里写"请小心操作文件"，属于建议，不是约束；prompt 注入、长上下文遗忘、模型抽风，任何一种都可能让"小心"失效。所以工程上真正要回答的问题是：**当 Agent 一定会执行那条最坏的命令时，损失半径（blast radius）被控制在哪一层？**

## OpenClaw 的做法：分层设防

sandbox 模型的核心思路不是"让模型更聪明"，而是默认模型会犯错，用环境把错误代价压到可恢复，大致四层：

1. **工作区约定**。Agent 默认被引导在 workspace 目录内读写，这是 prompt 层的第一道软约束。
2. **Docker 沙箱**。开启 sandbox 后，exec 在容器内执行，workspace 以 bind mount 挂入容器。`rm -rf` 删的是容器里的路径；没挂载的宿主目录，Agent 根本"看不见"。
3. **工具策略**。tool policy 可按 agent 收紧 exec：白名单命令、拒绝 `rm`/`dd`/`mkfs` 这类高危调用。策略在工具真正执行前拦截，不依赖模型自觉。
4. **确认与回滚**。高风险动作可配置为需要人工确认；workspace 用 git 管理，真删错了也能 restore。

配置示例（字段名以你那版文档为准）：

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": { "mode": "all", "scope": "agent" }
    }
  }
}
```

验证方式很朴素：在容器内外各建一个 `sandbox-test` 目录，让 Agent 跑一句 `rm -rf ./sandbox-test`，看它删掉的是哪一层。五分钟就能确认你的配置是否真的生效。

## 踩坑点

- **默认是 `non-main`**：主 Agent 默认不进沙箱，直跑宿主机。很多人以为"装了 Docker 就安全"，主 Agent 其实还在裸奔，要显式设成 `all`。
- **挂载过大**：把 `$HOME` 整个挂进容器，沙箱形同虚设。只挂 workspace，需要什么补什么。
- **别把 docker.sock 挂进容器**：等于把宿主 root 交给 Agent，前面三层全白做。
- **沙箱不防外传**：文件系统隔离挡不住 prompt 注入后的数据外泄，容器出网策略要单独收紧。
- **无 Docker 环境**：防护只剩策略 + 确认两层，要有意识地补上确认流程和更细的命令白名单。

## 可复用建议

- 用"最坏情况演练"评估配置：假设模型 100% 会执行最坏指令，逐层检查后果落点。
- **最小挂载、最小权限、可回滚**——这三件事的优先级高于任何 prompt 措辞。
- 保留 exec 审计日志，事后要能回答"它到底执行了什么"。
- 把沙箱当默认配置，而不是"以后再加的增强项"。

## 总结

Agent 不误删文件，不是因为模型可靠，而是最坏操作被关在容器里、被策略拦在工具前、被 git 兜了底。OpenClaw 的安全模型本质是承认模型会犯错，然后用环境把犯错的代价降到可恢复。信任可以给，边界要先建。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/dceeb59e59dd306c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/af6a474a423d079a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/d972ca1b3143344f.png)

