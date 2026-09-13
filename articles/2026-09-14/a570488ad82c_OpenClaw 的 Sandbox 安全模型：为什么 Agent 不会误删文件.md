---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37452
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 接上 shell 和文件工具之后，能力就从“生成文本”升级为“改变操作系统状态”。一条幻觉出来的路径、一次被注入的指令，都足以让 `rm -rf` 打到错误目录。很多团队的第一反应是在 system prompt 里写“请小心，不要删除文件”——但这不是安全机制，只是请求。

## 问题：模型输出本质是不可信输入

Agent 的执行环境拥有接近人类运维的权限，而它的“决策”来自概率采样。任何只依赖“希望模型做对”的方案，时间拉长必然出事。一个合格的安全模型要回答的是：**即使模型输出完全错误，系统如何保证损害有上限？**

## OpenClaw 的做法：四层防御

1. **路径收敛层**：所有文件工具的路径先在工作区根目录内做规范化（realpath 解析、软链接展开、`..` 折叠），解析后若落在根外，直接拒绝。读工具默认只读。
2. **能力分层**：读 / 写 / 删除 / 执行是相互独立的权限，按需授予。删除默认不是真删——文件先移入带时间戳的 trash 目录，保留期过后才真正清理。
3. **命令拦截层**：对 shell 命令不做字符串黑名单（拦不住各种变形写法），优先走参数化 exec；必须透传时做命令结构解析，命中删除/覆盖模式即降级为需人工确认，并附 dry-run 结果。
4. **OS 隔离层**：Agent 主进程跑在容器/namespace 内，工作区以 bind mount 挂入，系统目录只读，网络按策略开关。前三层被绕过时，这一层兜底。

策略配置大致长这样（示意）：

```yaml
workspace:
  root: ./workspace
  resolve_symlinks: true
file_tools:
  read: allow
  write: allow
  delete:
    mode: trash        # trash | confirm | deny
    trash_ttl: 72h
  shell:
    mode: exec_strict
    confirm_patterns: ["rm", "unlink", "rmtree"]
```

## 踩坑点

- **软链接逃逸**：工作区里一个指向 `/etc` 的 symlink 就能让路径检查形同虚设。必须在 realpath 之后判定，并注意检查与打开之间的 TOCTOU 窗口。
- **命令黑名单不可靠**：`sh -c` 拼接出来的命令没法靠 grep 拦，能参数化就参数化。
- **MCP 工具绕过沙箱**：沙箱只约束 Agent 主进程。某个跑在沙箱外、持有宿主全权限的 MCP server 会让整套模型失效——每个工具要按最小权限分别沙箱化。
- **确认疲劳**：所有操作都弹确认，用户会变成无脑回车。只对不可逆操作设确认，并用 dry-run 明确列出“将影响哪些文件”。
- **monorepo / worktree 场景**下，“工作区根”和“仓库根”不一致，策略容易配错范围。

## 可复用建议

- **默认拒绝、显式允许**。策略写在配置里，可 review、可版本化，不要写在 prompt 里。
- 危险操作要“慢”（确认 + dry-run + 快照），安全操作要“快”。
- 把逃逸测试写进 CI：构造软链接、`..` 混淆、编码路径，专门跑一遍越界用例。
- 全量审计日志：哪个会话、什么上下文、动了哪个路径，事后必须能复盘。

## 总结

Agent 不会误删文件，不是因为模型聪明，而是因为错误的输出在结构上无法触达危险操作。提示词负责引导行为，沙箱负责承担失败。两层叠加，才敢放心把 Agent 接进真实文件系统。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0b7ad914d738fb9c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/be408f46c16151ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/2d5d1251ff476af7.png)

