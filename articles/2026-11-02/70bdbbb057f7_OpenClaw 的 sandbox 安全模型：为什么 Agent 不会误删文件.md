---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43238
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 的 Agent 默认带 exec、文件读写这类工具能力，能直接跑 shell 命令。这意味着一次幻觉、一次路径解析错误，理论上就可能执行 `rm -rf`。社区里“敢不敢让 Agent 碰宿主机目录”是高频问题。OpenClaw 的答案不是靠提示词约束，而是把工具执行放进 sandbox——安全靠运行时架构保证，不靠模型自觉。

## 风险从哪来

拆开看是三类：

1. **模型侧**：上下文混淆导致路径写错，比如把用户目录当成临时目录；“清理一下”被理解成删除。
2. **工具侧**：MCP server 或插件自身权限过宽，Agent 借道执行了越权操作。
3. **配置侧**：bind mount 挂得太宽、容器以 root 运行，等于没隔离。

## OpenClaw 的分层做法

**第一层：进程隔离。** 开启 sandbox 后，exec 和编辑类工具默认在 Docker 容器内执行，Agent 看到的“机器”只是容器。宿主机上除了挂载进来的 workspace，其他路径它根本“看不见”——不是没权限删，是不存在。

```json
{
  "agents": {
    "defaults": {
      "sandbox": { "mode": "all" }
    }
  }
}
```

**第二层：文件系统收敛。** workspace 以 bind mount 挂进容器固定工作目录，读写被约束在这棵目录树内。只挂需要的目录，其余用只读挂载或干脆不挂。

**第三层：工具策略。** 通过工具 profile / 白名单收紧能力面。日常编码任务给 exec + 文件工具就够，browser、node 之类按需开。工具集在 gateway 侧统一裁决，不依赖模型“答应得好好的”。

**第四层：高风险动作审批。** 触达容器外的操作走审批通道，人确认才执行。

结果：即使模型真的生成了 `rm -rf /`，作用域也只是容器内的临时文件系统，删完重建即可，宿主机无感。

## 踩坑点

- **挂载即信任**：把整个 `$HOME` 挂进 sandbox，隔离名存实亡。
- **root 容器**：镜像里以 root 跑 Agent，逃逸成本变低，建议建低权限用户。
- **MCP 是旁路**：sandbox 管的是 OpenClaw 工具执行，外部 MCP server 若自己持有宿主机权限，就绕过了。给它单独跑容器或限制权限。
- **静默降级**：Docker 不可用时 sandbox 可能没真正生效，启动后确认容器确实起来了。
- **路径心智模型**：容器内路径 ≠ 宿主机路径，调试脚本前先确认自己站在哪一层。

## 可复用建议

1. 默认 `sandbox: all`，确有需要再降级，而不是反过来。
2. 挂载最小化 + 只读优先，把“可写”当成显式授权。
3. workspace 目录纳入 git 或定时快照——sandbox 是防线，不是备份。
4. 把插件和 MCP 当第三方代码对待，做权限审查。
5. 定期核对 gateway 审计日志，确认工具调用范围与预期一致。

## 总结

“Agent 不会误删文件”不是因为模型聪明，而是因为它物理上够不到。OpenClaw 的 sandbox 模型把安全从“提示词承诺”变成“运行时事实”：进程隔离限定爆炸半径，文件系统收敛限定可见范围，工具策略限定能力面，审批通道兜底高风险动作。四层叠起来，误删从概率问题变成架构上不可达。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/1912008b11267a70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/396b5b439debd611.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/10258816b09a1a0f.png)

