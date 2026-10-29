---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42967
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

给 Agent 配上 shell 和文件工具后，所有人都会冒出同一个念头：它会不会把我的目录删了？OpenClaw 的回答很工程化：**假设模型会犯错，安全不靠 prompt，靠边界。** 它的安全模型由四层组成：路径根定、工具网关、执行沙箱、策略审批。误删文件之所以没发生，是因为错误路径上的每一步都有闸门。

## 问题

常见的做法是“信任模型 + 提醒它小心 + 事后恢复”。这条路线有几个结构性漏洞：

- 模型可能误解相对路径，把宿主机根目录当成 workspace；
- `rm -rf` 拼接变量出错时，一次 exec 就是不可逆操作；
- 提示词能被长上下文稀释，第 50 轮对话时它已经“忘了”那条安全说明。

所以需要的是结构性防线，而不是行为规劝。

## 做法

**1. 路径根定（path rooting）。** 所有文件工具以 workspace 为唯一解析根。每个进出的路径先做 realpath 规范化，再做越界校验：`../` 逃逸、symlink 指向外部，一律直接拒绝。拒绝是终态，不给模型“换个姿势重试”的空间。

**2. 默认可见域收敛。** Agent 会话启动在 workspace 内，不继承宿主机 home。没挂载的目录，对它来说不存在——读不到的东西谈不上误删。

**3. 执行沙箱。** exec、browser 等落盘操作跑在容器里，只 bind-mount workspace，不挂 docker.sock，网络默认拒绝或按域名放行。配置大致是这个形状（字段名以你所用版本为准）：

```json5
sandbox: {
  mode: "all",          // 工具执行全部进沙箱
  mounts: workspace,    // 只挂 workspace
  network: "deny",      // 断网，按需放行域名
}
```

即使删除真的发生，爆炸半径也等于一个可重建的 workspace。

**4. 工具策略 + 审批。** tool policy 分 allow / ask / deny 三档，删除类、系统变更类默认 ask，审批动作写入审计日志。高危命令在到达 shell 之前就被拦下。

## 踩坑点

- **symlink 是最常被绕过的通道。** 必须在 realpath 之后校验，在 resolve 之前校验等于没校验。
- **图方便挂了整个 home，或者挂了 docker.sock**，沙箱直接形同虚设。
- **审批弹窗顺手点了“总是允许”**，策略就退化成摆设。高危类建议强制逐次确认。
- **拒绝信息含糊。** “路径不存在”和“越界拒绝”必须区分开，否则模型会把拒绝当成异常去绕。
- **容器内 UID 映射不一致**，可能导致 workspace 内权限意外放宽，删起来更“顺”。

## 可复用建议

- 最小挂载原则：只给 workspace，宁可让 Agent 多要一次权限。
- 删除类操作收敛成一个受控工具：先产出 plan/diff，人工确认再执行；删除进 workspace 内回收站，不做直接 unlink。
- 定期做越界探针测试：故意让 Agent 访问 workspace 外的路径，验证拒绝行为和日志是否如预期。
- 审计日志持久化。事后要能回答“它到底执行了什么”，而不是靠回忆复盘。

## 总结

Agent 不误删文件，不是因为模型聪明，而是因为**聪明被放在了错误的路径上，而每一步都有闸门**：路径根定挡住越界，沙箱限制爆炸半径，策略与审批拦住高危动作。prompt 是体验，边界才是安全。这套思路不限于 OpenClaw，任何带工具执行能力的 Agent 系统都值得照此自查一遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/3dce78a479370f30.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/8e6e2360057a2242.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/aeffc2e2286eea0b.png)

