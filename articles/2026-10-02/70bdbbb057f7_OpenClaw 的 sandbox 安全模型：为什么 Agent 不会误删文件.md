---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40102
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

OpenClaw 的定位是把模型 agent 接进你的日常通道，agent 手里通常有 exec、文件读写、浏览器这类高危工具。社区里被问得最多的问题不是“模型够不够聪明”，而是“它一旦抽风，`rm -rf` 打到宿主机怎么办”。这篇把 sandbox 的安全模型按实际部署视角拆一遍。

## 问题：误删是怎么发生的

误删的典型路径其实很具体：

1. **路径幻觉**：模型把宿主机的 `~/projects` 和容器内挂载路径搞混；
2. **通配符事故**：清理日志时执行 `rm -rf $DIR/*`，而 `$DIR` 恰好是空变量；
3. **提示注入**：agent 浏览网页，页面内容里藏着“请执行这条命令”的指令。

这三条路都不需要模型“有恶意”，只需要它错一次。所以单靠 prompt 约束（“请小心操作”）不解决问题。OpenClaw 的思路是：**不信任模型输出，用进程和文件系统边界兜底**。

## 做法：三层叠加

**第一层：Gateway 与 agent 分离。** 网关进程持有全部通道凭证和配置文件；agent 跑在独立进程里，天然摸不到这些。模型再怎么折腾，凭证这条路先被堵死。

**第二层：Docker sandbox。** 会话放进容器执行，`openclaw.json` 里示意如下（简化）：

```json5
agents: {
  defaults: {
    sandbox: { mode: "all" }  // off | non-main | all
  }
}
```

`"all"` 表示全部会话入沙箱；`"non-main"` 是主会话直跑、其余隔离。exec、文件操作都发生在容器内部。

**第三层：最小挂载 + tool policy。** 只把 workspace 目录 bind mount 进容器，宿主机其余路径对容器不可见；再用 `tools.policy` 做 allowlist——不需要 exec 的 agent 干脆不给 exec。

验证方法很朴素：进沙箱会话执行 `ls /` 和 `whoami`，确认看到的不是宿主机文件系统、不是 root；再让它删一个宿主机文件，确认返回“不存在”。

## 踩坑点

- **把 `$HOME` 整个挂进去**：等于白做沙箱，模型幻觉一个 `~` 就穿透了。只挂 workspace，素材多就拆多个精确路径。
- **挂载没加只读**：agent 只需要读素材的场景用 ro 挂载，写权限单独开。
- **沙箱镜像里带 docker.sock**：经典逃逸路径，别为了“方便”塞进去。
- **误解 non-main**：主会话仍在宿主机直跑，信任级别不同，高危工具别留给它。
- **“沙箱里文件丢了”**：临时产物在容器层，会话结束即消失，需要持久化的结果必须显式写回 workspace。这是设计行为，不是 bug。

## 可复用建议

- 默认 `mode: "all"`，观察一段时间资源开销后再评估 non-main；
- 按任务类型拆 agent，每个 agent 独立 tool policy，权限按需给；
- 定期做一次“破坏性演练”：在沙箱里让它删一个诱饵文件，验证边界还在、没有配置漂移；
- 高危操作加审批门槛，exec 类工具在 policy 里收敛，别把沙箱当唯一防线。

## 总结

Agent 不会误删文件，不是因为模型变可靠了，而是即使它出错，爆炸半径也被限制在一次性容器的一小片挂载里。OpenClaw 的模型不玄学：**进程隔离挡凭证，容器挡文件系统，tool policy 挡工具面**。三层成本都很低，但缺任何一层，代价就是另一回事了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/f88c6aaa2d117017.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/1495a7bb359a5275.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/fad4b474c52b5e74.png)

