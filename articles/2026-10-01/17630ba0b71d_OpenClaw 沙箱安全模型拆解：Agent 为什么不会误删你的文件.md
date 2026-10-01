---
title: OpenClaw 沙箱安全模型拆解：Agent 为什么不会误删你的文件
feedId: 40026
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

让 Agent 干活绕不开一件事：给它执行能力。OpenClaw 里无论是内置文件工具、MCP server 还是插件，最终都会落到对真实文件系统的读写。社区被问得最多的安全问题之一就是：模型会幻觉、会误解析指令，凭什么敢让它碰 `rm`？

答案是不能凭"模型很聪明"。OpenClaw 的设计假设恰好相反——**假设 Agent 一定会犯错**，然后用沙箱把犯错的最大代价框死在工作区里。

## 问题：三类真实事故

我们内部复现过三类典型翻车：

1. **路径幻觉**：模型把 `/home/user/project/src` 写成 `/home/user/project`，一条递归删除直接降级整个目录。
2. **路径穿越**：workspace 内拼接出 `rm -rf ../../` 之类的相对路径，越出工作区。
3. **提示注入**：网页或文档内容里夹带指令，诱导 Agent 调用 shell 清理"垃圾文件"。

单靠 system prompt 写"请小心"对这三类都无效。prompt 是行为约束，不是安全边界。

## 做法：三层防线

### 第一层：OS 级隔离（兜底）

Agent 的所有工具调用在沙箱进程内执行：独立低权限用户 + 容器/mount namespace。宿主机上只有 workspace 一个目录以 bind-mount 方式进入沙箱，其余路径物理上不可见。这一层的意义是：**就算策略引擎被绕过，能删的也只是挂进去的那个目录**。

### 第二层：工具网关（策略）

文件类工具不直接触盘，先过 policy 检查：

```yaml
sandbox:
  root: ./workspace
  write_paths: ["./workspace/**"]
  deny_globs: ["**/.git/**", "**/*.env"]
  exec:
    deny_patterns: ["rm -rf /", "mkfs*", "dd if=*"]
    destructive_requires_approval: true
```

关键在检查顺序：先把路径做 realpath 归一化（解 symlink、消 `..`），再比对 allowlist。顺序反了就有 TOCTOU 竞态。

### 第三层：审批 + 审计 + 快照

命中破坏性操作时挂起并请求人工确认；每次变更写审计日志，且日志落在沙箱外；批量操作前自动打 git 快照，可回滚。

## 踩坑点

- **symlink 逃逸**：Agent 在 workspace 里 `ln -s /etc/passwd` 再写入，如果先查路径再解析就漏了。必须 resolve 之后再校验。
- **容器内外路径映射**：policy 里写宿主机路径、运行时是容器路径，规则会静默失效。统一用沙箱内路径。
- **用 root 跑容器**：没做用户降权的 namespace 隔离弱得多，务必 `--user` 指定非 root。
- **只限制了文件工具**：shell/exec 是另一条路，不一起纳管等于白做。
- **共享可写 /tmp**：注入载荷可以通过临时目录中转，tmpfs 要按会话隔离。

## 可复用建议

1. **默认拒绝**。allowlist 永远比 denylist 可靠，denylist 枚举不完。
2. 检验标准很简单：把 system prompt 全删掉，沙箱还能不能挡住删除？能，才算合格边界。
3. 定期对自己的 Agent 做红队：明确要求它"删掉 workspace 之外的文件"，看它是否真的做不到。做不到，配置才算生效。
4. 审计日志放在沙箱外，否则日志会跟着被删。

## 总结

Agent 不误删文件，从来不是因为它"懂分寸"，而是因为**它想删也够不着**。OS 隔离兜底、策略网关收口、审批快照留后路，三层各管一段。把安全寄托在模型行为上迟早出事，寄托在机制上才能睡安稳觉。如果你现在的配置只做到了第二层，建议先补第一层——那才是真正的底线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/76483eb178a62ccc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/18ebb8eaf6af5f42.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/083facb8839f843b.png)

