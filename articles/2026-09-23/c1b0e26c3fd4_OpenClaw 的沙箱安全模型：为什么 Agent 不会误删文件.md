---
title: OpenClaw 的沙箱安全模型：为什么 Agent 不会误删文件
feedId: 38606
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

很多人第一次让 OpenClaw 接管本地环境时，担心的不是它做不了事，而是它"太能做"——一条失控的 `rm -rf` 或一次递归覆盖，就可能清空几个月的工作区。OpenClaw 的安全设计并不指望模型"自觉"，而是默认模型一定会犯错，用沙箱和策略把错误限制在可恢复的范围内。

## 问题

Agent 的高风险动作主要是三类：删除或覆盖文件、执行任意 shell 命令、访问工作区之外的路径。单靠 prompt 约束（"请不要删文件"）在长上下文和工具链组合下完全不可靠。真正要解决的问题是：如何在不牺牲 Agent 能力的前提下，把破坏半径压到最小？

## 做法：三层防线

OpenClaw 的沙箱模型本质上由三层组成，各管一件事：

**1. 工作区边界。** 文件类工具默认以 workspace 为根路径解析，工作区之外的内容对普通工具调用基本不可见。这是第一层"限可见性"。

**2. 容器沙箱。** 开启 sandbox 后，exec 工具产生的 shell 命令全部在 Docker 容器内执行，宿主机文件系统默认不挂载，只有 workspace 通过 bind mount 进入容器。就算命令"发疯"，它能破坏的也只有这份挂载。这是"限爆炸半径"。

**3. 执行审批。** safe 模式下，allowlist 之外的命令需要人工确认，deny 规则可以直接拦截 `rm -rf` 这类危险模式。这是"限动作本身"。

配置的大致形态：

```yaml
agents:
  defaults:
    sandbox:
      mode: all        # 非 main agent 全部进容器
    tools:
      exec:
        approvals: safe
```

然后在 docker setup 中只声明 workspace 挂载和少量显式需要的 extra mounts。

验证方式很简单：让 agent 执行 `ls /`、尝试删除工作区外的一个测试路径，确认被拦截；再故意删工作区内的文件，确认影响范围只在挂载卷内。策略写完必须实测，不能只看配置。

## 踩坑点

- **图方便把 `$HOME` 挂进容器**，沙箱等于白开，容器内的进程一样能递归删宿主数据。
- **审批设成"全部放行"后通知轰炸**，人开始无脑确认，审批层形同虚设。宁可把 allowlist 写精确。
- **符号链接逃逸**：workspace 里一个指向外部目录的 symlink 会让边界失效，定期清理或校验 realPath。
- **gateway 以 root 跑 + sandbox 关闭**的组合，测试时没事，出事时致命。
- 群聊消息可能路由到没进沙箱的 main agent，主号上别挂高危技能。

## 可复用建议

1. 原则：**默认拒绝，显式放行**。挂载、命令、网络能力全部按需开启。
2. 高危改动前先 git 提交或快照 workspace，把"可恢复"作为最后兜底。
3. 用一个一次性的 scratch agent 复演破坏性场景，验证策略真的生效。
4. 审批和执行日志要留存，出问题时能回放 agent 到底执行了什么。

## 总结

OpenClaw 不承诺模型永不犯错，而是让每一个错误都撞在墙上，而不是撞在你的数据上。工作区边界限可见性，容器沙箱限爆炸半径，执行审批限动作本身——三层各司其职，缺任何一层，都不要上线跑无人值守的自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/a22048b1f56e6c89.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/0ce0b0d898e4dfc7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/54e15840c35e57be.png)

