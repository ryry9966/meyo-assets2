---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 40023
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的典型用法，是给 Agent 一套文件与执行能力：读写 workspace、跑 shell、外接 MCP server。模型本质是概率系统，会拼错路径，也会误解"帮我清理一下临时文件"的边界。所以"Agent 会不会把我的东西删了"是每个认真部署过的人都会问的问题。OpenClaw 的设计答案很明确：**不指望模型可靠，而是把可靠性做在边界上。**

## 问题

很多人第一次配置只做了两层：给个工作目录 + 提示词里写"别乱动别的文件"。这有两个漏洞：

1. **路径只是约定，不是强制。** 一条 `rm -rf ../` 或一次工具调用参数写错，就能越出约定目录。
2. **删文件不只有 delete 工具。** `find -delete`、`git clean -fd` 都能绕过对"删除工具"的过滤。

单靠提示词或单一工具过滤都不够，OpenClaw 用的是分层模型。

## 做法：五层防线

**第一层：workspace 收口。** 文件工具根路径强制绑定 workspace，所有传入路径先做规范化（含符号链接解析后的真实路径），再判断是否落在根内，越界直接拒绝，不进模型重试。

**第二层：工具策略。** 每个工具按 read / write / exec 分类，写操作默认限定在 workspace 内；策略里没明确放行的能力就等于没有。

**第三层：exec 审批。** shell 命令走独立审批通道，默认 ask；allowlist 按"命令 + 参数模式"精确匹配，而不是只看首 token。

**第四层：容器隔离。** 开启 sandbox 后，工具执行跑在 Docker 容器里：非 root 用户、只挂载 workspace、网络受限。即使前三层被绕过，爆炸半径也被压在挂载点内。

**第五层：可恢复。** 删除类操作封装成"移动到 workspace 内 `.trash` 目录"，配合快照可回滚。

配置示意（字段以你当前版本文档为准，核心思路是**默认拒绝、逐项放行**）：

```yaml
sandbox:
  mode: all
tools:
  policy:
    fs.write: workspace
    exec: ask
```

验证方式很简单——做一次红队自测，让 Agent 依次尝试：读 workspace 外的 `~/.ssh`、写 `../outside.txt`、执行 `rm -rf /`。三条都应被拦截，并在审计日志留下记录。

## 踩坑点

- **挂载过大。** 把整个 home 目录挂进容器，第四层形同虚设。挂载点 = 爆炸半径。
- **allowlist 写宽。** `rm *` 或按前缀放行，等于没写。
- **MCP server 绕过。** 第三方 MCP 工具若自带文件能力，必须纳入同一套 tool policy，否则 exec 审批管不住它。
- **子 Agent / 插件继承权限过宽。** 派生 sub-agent 若全量继承，等于把漏洞复制了一份。
- **符号链接逃逸。** workspace 里一个指向外部的软链，就可能让"根内路径"写到根外。
- **只在 happy path 验证。** 没做过"故意越界"的自测，就不算配好了。

## 可复用建议

1. 按项目拆 workspace，最小挂载，不要图省事共享大目录。
2. 危险命令宁可直接 deny，另封装安全版本（如 trash 工具）供 Agent 调用。
3. 审计日志保留并定期翻——异常工具调用往往是策略漏洞的信号。
4. 把"越界自测"做成可重复脚本，每次升级 OpenClaw 或新增插件后跑一遍。
5. 新接 MCP server 时先全量 deny，观察后再按需放行。

## 总结

OpenClaw 不承诺模型不犯错，它承诺的是：**模型犯的错到不了你的文件系统。** 路径收口解决"写错地方"，工具策略解决"用错能力"，exec 审批解决"绕过工具"，容器隔离兜底，trash + 快照接住最后一步。这套模型的价值不在任何单层，而在"任意一层失效时，下一层还在"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/c7f78f64d62aa8ce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/55aa6617dcb2a744.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/d4834d5e123862f3.png)

