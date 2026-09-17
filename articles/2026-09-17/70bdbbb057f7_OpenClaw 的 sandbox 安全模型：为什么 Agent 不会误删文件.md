---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37965
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

让一个有 shell 权限的 Agent 直接跑在开发机上，最大的心理障碍不是它答得对不对，而是它会不会在某次“帮我清理临时文件”里把别的东西删掉。OpenClaw 的 sandbox 安全模型，就是为了把这类风险压到工程上可接受的水平。

## 问题的本质

误删文件几乎从不是“Agent 想删”，而是三类机械性错误：路径基准不一致（相对路径和 Agent 理解的根目录对不上）、通配符过宽（`rm -rf $TMP/*` 里变量恰好为空）、上下文错位（把会话早前的路径当成当前路径）。这类错误靠 prompt 约束不可靠——模型再稳也有低概率失误。所以 OpenClaw 的思路是：**不赌模型，赌架构**。即使模型错了，错误也要被隔离层挡住。

## 分层做法

防护由四层叠加，任何一层失守，下一层兜底：

1. **文件系统边界**。Agent 的文件类工具默认被限定在 workspace 目录，执行前做 canonical path 校验，解析后的绝对路径一旦越出 workspace 根，直接拒绝并记录。`~/.ssh`、`/etc` 这类路径天然不可达。
2. **进程沙箱**。开启 sandbox 后，Agent 跑在独立容器里：非 root 用户、workspace 以 bind mount 挂入、其余文件系统只读或不挂载。就算命令越界，容器内也看不到宿主机的真实数据。
3. **工具策略**。每个工具可配 allowlist / denylist。读写文件默认放行（限 workspace 内），但 `exec` 命中危险模式（递归删除、覆盖式重定向）会进入审批队列，需人工确认或被策略拦截。
4. **快照回滚**。workspace 定期做 git commit。即便前三层都漏了，`git reset` 也能把损失收敛到两次快照之间的增量。

典型配置大致是：

```yaml
sandbox:
  mode: "all"
  docker:
    bindMounts: ["/home/me/openclaw-workspace:/workspace"]
tools:
  policy:
    fs:
      root: "/home/me/openclaw-workspace"
    exec:
      approval: "dangerous-only"
```

## 踩坑点

- **把 workspace 挂到 home**。图省事把 `/home/me` 整个挂进去，边界形同虚设。workspace 应是专用目录，项目用完即拷。
- **symlink 逃逸**。校验层会跟随并拒绝越界软链，但如果你在 workspace 里手工建了指向外部的链接，自建挂载时要自己再核一遍。
- **Docker socket 挂载**。为“方便调试”把 `/var/run/docker.sock` 挂进沙箱，等于把宿主机 root 交出去，最常见的自我击穿。
- **审批全放行**。`approval` 设成 always-allow 跑一周后没人再看提示，防线退化成装饰。宁可被打断，也别全开。

## 可复用建议

- workspace 永远用专用目录，不放 home，不放含密钥的路径
- 宿主机上有价值的目录默认对沙箱不可见，确需访问时显式只读挂载
- exec 审批保留人工确认环节，dangerous-only 是比较稳的默认值
- 快照周期按容忍度定：能接受丢多少改动，就多久 commit 一次
- 定期做一次越界测试：让 Agent 尝试读 workspace 外的文件，确认返回的是拒绝而非内容

## 总结

“Agent 不会误删文件”不是因为它聪明，而是因为即使它犯傻，能碰到的文件系统就那么大，能执行的命令要过策略，执行完还有快照可退。OpenClaw 的 sandbox 模型，本质是把安全从“信任模型的输出”转移到“约束模型的运行环境”——前者无法验证，后者可以。模型能力会继续涨，但边界不该跟着涨。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/58bb6259ac885fd4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/e96d677fbf7e646c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/93601dce47c22974.png)

