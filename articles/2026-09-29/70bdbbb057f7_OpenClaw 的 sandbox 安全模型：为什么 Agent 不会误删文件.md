---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39445
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 是常驻本机的 Agent 网关，模型默认能用 exec、read、write、edit 这些工具。但模型输出本质是概率生成：长上下文里拼错路径、误判当前目录、被工具返回内容里的注入指令带偏，这些都真实发生过。文件删除不可逆，所以这个问题的答案不能是"相信模型"，而必须是结构性防御。

## 问题

拆开看有三类真实风险：

1. **语义错误**：模型把 `$HOME` 当成 workspace 根，`rm -rf "$TARGET"` 里 TARGET 是个空变量；
2. **注入路径**：网页正文、第三方 MCP 的返回值里夹带指令，诱导 agent 执行破坏性命令；
3. **权限扩散**：群聊 session、个人 session、插件共享宿主机权限，一处越界处处越界。

## OpenClaw 的分层模型

我的理解是四层，各管一段：

**第 1 层：workspace 边界 + 文件工具路径校验。** read/write/edit 在执行前把目标路径解析成绝对路径，与 workspace root 比对，越界直接拒绝。注意：这只约束文件工具，shell 不在此列。

**第 2 层：Docker sandbox。** 在 openclaw.json 里把 `agents.defaults.sandbox.mode` 设为 `all`（或 `non-main`），exec 就在容器里跑，workspace 以 bind mount 挂入。容器里看不到宿主机其余文件系统，shell 里的 rm 物理上够不到挂载范围之外。

**第 3 层：exec 审批与白名单。** 对未命中白名单的高危命令，网关把审批请求推回聊天渠道，人点头才放行。

**第 4 层：会话隔离与快照。** session 级 sandbox 会重建容器并从 workspace 模板拷贝，做坏了丢弃容器重来；重要目录任务前再做一次快照兜底。

落地顺序建议：先切 `all` 模式 → 只挂载 workspace → 容器内非 root 运行 → 配置审批白名单 → 开快照。

## 踩坑点

- `non-main` 模式下主 session 不隔离，别想当然以为全隔离了；
- bind mount 图省事挂 `$HOME`，等于没隔离；
- 容器里挂 `docker.sock` 是经典自爆操作，一条命令就能逃逸；
- sandbox 默认不限容器出网，`curl` 外传数据这条路是开的，要单独配网络策略；
- 第三方 MCP 工具不走文件工具的路径校验，接入前按最小权限单独审；
- 容器内 root 写 bind mount 会把宿主机文件 owner 改乱，UID 要对齐。

## 可复用建议

- **纵深防御**：路径校验、sandbox、审批、快照四层都当"可能失效"来设计，任何单层不单独兜底；
- **最小 workspace**：目录里只放任务需要的文件，边界越小越安全；
- **留证据**：保留 exec 日志和审批记录，事后能复盘到具体哪条命令；
- **定期演练**：故意让 agent 执行越界删除，验证隔离是否真的生效。

## 总结

"Agent 不会误删文件"，不是因为模型足够聪明，而是破坏性操作的可达性被结构性收窄了：文件工具被路径校验圈在 workspace 内，shell 被 sandbox 圈在容器内，残余风险由人工审批和快照兜底。安全是系统属性，不是模型属性。具体配置项以你所用的 OpenClaw 版本文档为准。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/a6113dfe18dbba46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0b1129bdb6fc4bfa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b2b3562ff31aadcb.png)

