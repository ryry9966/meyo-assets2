---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38602
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

让 Agent 直接操作本机文件，是自动化实践里最常见的诉求，也是最容易出事故的场景。OpenClaw 的设计前提很冷静：LLM 会出错、会幻觉路径、会把"清理临时文件"理解成"清理目录"。所以它没有把安全寄托在"模型够聪明"上，而是把 Agent 的每一次工具调用都当作不可信输入来处理。

## 问题：裸跑 shell 到底险在哪

- 模型对路径的把握并不可靠，相对路径、软链接、`~` 展开这类细节尤其容易翻车；
- 一条 `rm -rf` 的变量拼接失误，就足以删掉整个工作目录甚至家目录；
- Agent 执行是全自动的，人不在环里时没有刹车。

社区里"让 AI 清理磁盘结果删了项目"的故事并不夸张。这是工程问题，得用工程手段解决。

## OpenClaw 的做法：四层防线

核心思路是纵深防御，任何一层失效都还有下一层兜底。

**1. 文件系统隔离。** 开启 sandbox 后，exec 类工具默认在独立容器里执行，Agent 看到的根文件系统是沙箱自己的，宿主机只有 workspace 目录以受控方式挂载进去。即使 Agent 执行了灾难性命令，作用域也被限制在挂载范围内。

**2. 工具分级。** 工具策略分三档：`allow` 直接放行，`confirm` 弹给用户确认，`deny` 直接拒绝。默认读类操作放行、写和执行类需确认，未注册的工具按最严格一档处理。

**3. 路径规范化。** 所有文件操作先做 realpath 解析，落在 workspace 外的写入直接拒绝。这一步专门防软链接逃逸——在工作目录里建一个指向 `/etc` 的软链接，是经典攻击面。

**4. 危险模式拦截 + 快照。** 对命令做模式匹配，命中 `rm -rf`、`mkfs`、`dd` 等模式时强制升级为人工确认；同时每次破坏性操作前对 workspace 打快照，误删可以回滚。

示意配置（节选）：

```yaml
sandbox:
  enabled: true
  workspace: ./workspace   # 只挂这一个目录
tools:
  policy:
    "fs.read": allow
    "fs.write": confirm
    "exec": confirm
  denyPatterns:
    - "rm -rf"
    - "mkfs*"
    - "dd if=*"
```

## 实测步骤

1. 开启 sandbox，workspace 只挂最小必要目录；
2. 在 workspace 外放一个 canary 文件，提示词里故意让 Agent"清理所有 .log 文件"，观察它是否越界；
3. 在 workspace 内造一个指向外部的软链接，验证 realpath 拦截是否生效；
4. 回看工具调用审计日志，确认每一步都有记录、可追溯。

## 踩坑点

- **挂载过宽。** 图省事把整个 HOME 挂进去，隔离就形同虚设。workspace 越小越安全。
- **只信黑名单。** `denyPatterns` 用 base64、变量拼接就能绕过，它只是减速带，文件系统隔离才是主防线。
- **忘了插件和 MCP。** 第三方 MCP server 自带文件工具，不走 shell 黑名单，一样要纳入工具策略。装插件时按不可信代码对待。
- **沙箱里挂 Docker socket** 等于把 root 交出去，坚决不要。
- **gateway 自身用 root 跑**，等于沙箱白做。

## 可复用建议

- 权限最小化是默认项，不是可选项；
- 把 workspace 当契约：Agent 只拥有这个目录，其余一切默认不可见；
- 破坏性操作前必打快照，让"误删"从事故降级为可恢复事件；
- 定期跑一次红队提示词（故意诱导越界），验证防线还在。

## 总结

OpenClaw 之所以能放心让 Agent 动文件，不是因为模型可靠，而是因为默认假设它不可靠。隔离、分级、拦截、快照四层叠起来，单点失误不再等于事故。这套思路不绑定具体产品，任何让 LLM 接触文件系统的项目都值得照搬。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/9b520a9ae5f6eafc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/edccd5ec4054c60e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/7add11e65749f3d3.png)

