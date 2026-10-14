---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41532
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

用 Agent 做自动化的人，多少都想过这个问题：模型一旦理解偏差，加上 shell 通配符的放大效应，`rm -rf` 就是灾难。早期很多框架的做法是在 prompt 里写“请小心操作”，这不叫安全设计，叫许愿。

OpenClaw 的思路不同：它不指望模型自觉，而是假设模型一定会犯错的**前提下**做结构隔离。这里以我本地跑通的版本为例，具体配置字段名以你所用版本文档为准。

## OpenClaw 的三层防线

**第一层：文件系统边界。** Agent 的工具调用（exec、read、write）默认运行在 sandbox 容器内，只有 workspace 目录被挂载进去，宿主机的其余路径对它不可见。Agent 执行 `rm -rf ~`，删的是容器内不存在的路径或容器自己的可写层，碰不到宿主机。

**第二层：工具策略。** exec 这类高危工具可以按 agent 配置为需要人工审批，或直接 deny。典型做法是：主会话的 agent 保持半自动，后台定时任务的 agent 全部走审批。

**第三层：版本控制兜底。** workspace 用 git 管理，每次任务前自动 commit。就算容器内的文件被搞坏，回滚是一条命令的事。

## 实操步骤

1. 确认 sandbox 开启，模式设为覆盖所有 agent（至少覆盖非主会话 agent）；
2. workspace 指向独立目录，**不要指向 HOME 或含敏感数据的目录**；
3. 对 exec 工具启用审批策略，范围先收紧再逐步放开；
4. workspace 初始化 git，任务前后各一次 commit；
5. 做一次“捣乱测试”：直接让 agent 执行 `rm -rf ~`，观察实际效果。我测试时它只返回了路径不存在的错误——这正是边界生效的证明。

```json
{
  "sandbox": { "mode": "all" },
  "tools": { "exec": { "policy": "require-approval" } }
}
```

## 踩坑点

- **软链穿透**：把 workspace 里的子目录软链到家目录，容器内的路径边界等于白设。挂载前用 `ls -l` 检查一遍。
- **sandbox 关了还以为开着**：调试时临时关掉 sandbox，忘了恢复，这是最常见的真实事故来源。建议改完配置就跑一次捣乱测试。
- **挂载 Docker socket 给容器**：等于把宿主机 root 交出去，任何边界都失效。
- **审批疲劳**：如果所有操作都要批，人很快就会无脑点“同意”。审批只留给真正高危的工具，低危操作放自动。
- **网络出口**：容器内删除风险被隔离了，但数据外带风险还在，敏感 workspace 建议直接切断容器网络。

## 可复用建议

- 最小暴露面：Agent 只挂载它完成任务必需的目录，多余的一个字节都别给；
- 按**工具粒度**控权，而不是相信模型的口头承诺；
- git commit 粒度要小，大粒度的快照在回滚时等于没有；
- 把“越狱演练”纳入例行维护，每次升级配置后跑一遍。

## 总结

Agent 不会误删文件，不是因为它被说服了，而是因为它被关住了。OpenClaw 的安全模型本质是一句话：**容器隔离定边界，工具审批设闸门，版本控制做兜底**——三条腿都到位，模型的偶发幻觉就只是日志里的一行错误，而不是你硬盘上的一声巨响。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/93e67bb3c9c09d62.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/d8067ecfbc026ae4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/9a528b8c8acb9de5.png)

