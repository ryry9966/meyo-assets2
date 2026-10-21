---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 42227
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

OpenClaw 的 agent 通常以较高自主度运行：读工作区、调工具、执行命令，自动化任务经常在没人盯着的时候跑一整晚。社区里被问得最多的问题不是"它代码写得好不好"，而是——它会不会某天把 `rm -rf` 落在错误的目录上。

## 问题：靠 prompt 拦不住删除

一个常见误区是在 system prompt 里写"不要删除文件"。要明确：**prompt 是请求，不是约束**。在长上下文、工具输出误导，或者用户一句"帮我清理一下临时目录"的模糊指令下，模型完全可能生成破坏性命令。安全必须落在执行层，而不是生成层。

## OpenClaw 的做法：路径围栏 + 分层管控

核心思路是把"能不能删"从模型行为问题，转化成文件系统权限问题。

1. **工作区围栏**。每个 agent 会话绑定一个 workspace 根目录，exec 类工具的所有路径参数先做规范化（realpath），落在围栏外的请求直接拒绝，根本不进模型。symlink 逃逸也在这一步被 realpath 解析拦住。
2. **读写分离**。默认对项目代码只读挂载，单独划一块 scratch 目录可写。删除、移动、覆盖只可能发生在 scratch 里，源码区物理上不可写。
3. **危险操作审批**。工具调用经过 policy 层，删除类调用（`rm`、`git clean`、递归删除等）默认进入 approval 队列，gateway 通知你确认。批处理场景可以对特定目录签发长期授权，粒度到路径，而不是通配符。
4. **审计留痕**。每次 exec 记录命令、工作目录和退出码，事后能回放"它到底干了什么"。

验证方法很简单：在项目根目录放一个 canary 文件，让 agent"清理磁盘上所有临时文件"，然后观察拒绝路径和审计日志。

## 踩坑点

- **把整个 home 目录设成 workspace**。围栏的强度等于你给的边界大小，边界太大等于没有。
- **图省事关掉 sandbox，或把工具权限放开成通配符**，等于回到裸奔。
- **只过滤 `rm` 命令字符串**。`find -delete`、`git clean -fd`、Python 里的 `shutil.rmtree` 都绕得过去。OpenClaw 走的是路径权限而非命令模式匹配，自己写 wrapper 时别犯这个错。
- **在 scratch 目录里再挂载其他路径**，会把洞重新打开。
- **插件和 MCP server 自带 exec 能力时**，记住它们同样要走 policy 层，不要为了方便单独放行。

## 可复用建议

- 最小可写原则：agent 需要写什么，就只开什么。
- 高风险自动化前先打快照（git 状态干净 + scratch 可随时丢弃），把"误删"降级成"重跑一次"。
- 一次性清理任务用独立的 agent 配置，不给持久写权限。
- 定期翻审计日志，比事后排查省一个数量级的时间。

## 总结

OpenClaw 不依赖模型"自觉"，而是假设它一定会犯错：路径围栏限制爆炸半径，读写分离保护源码，审批层给人类留最后一道闸门。安全模型的意义不是让 agent 更聪明，而是让最坏情况可承受。想清楚这一点，你给它更大权限时才有底气。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/8e3c426d46b98da1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/e0d5b3cb6c8aa6b4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/589a296e3e22a59e.png)

