---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38479
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

OpenClaw 的 Agent 默认带执行能力：文件读写、shell、MCP 插件调用。社区里被问得最多的一句话是："让它帮我整理目录，会不会把整个 `~/Documents` 删了？"这个问题的答案不在 prompt 里，而在 sandbox 的执行层设计。

## 问题

LLM 的输出天然不可靠：路径幻觉、相对/绝对路径混用、把"清理临时文件"理解成无差别删除。靠 system prompt 写"请小心操作"没有任何强制力——模型可能听，也可能不听。所以 OpenClaw 的思路是：**不让危险路径有机会到达执行点**，而不是指望模型自觉。

## 做法

OpenClaw 的 sandbox 大致分四层：

1. **Workspace Root（根边界）**：Agent 能触达的文件系统被限定在 workspace 目录内。所有工具调用里的路径先做 canonicalize（解析 `..`、软链接），解析结果必须落在 root 内，否则直接拒绝，根本不会进入执行环节。
2. **能力分级**：读操作默认放行；写和删走 capability token，每个 MCP server 启动时声明 scopes（`fs:read` / `fs:write` / `fs:delete`）。删除是独立 scope，默认不授予。
3. **变更前置**：删除/覆盖前先生成 plan（源路径、目标、diff），按策略分流——白名单目录内自动通过，root 外或不可逆操作必须人工 confirm；同时自动打一个 git snapshot 作回滚点。
4. **网关统一校验**：即使插件自己实现了 delete，请求也要过网关做同样的路径校验，插件无法绕过。

最小配置示例：

```yaml
sandbox:
  root: ~/openclaw-workspace
  allow: [fs:read, fs:write]
  require_confirm: [fs:delete]
  snapshot: git
```

## 踩坑点

- **软链接逃逸**：workspace 里 `ln -s` 到 home 目录，如果先校验再 resolve 就会被穿透。务必 resolve 完再判前缀。
- **路径规范化差异**：macOS 文件系统大小写不敏感，`Docs/` 和 `docs/` 的判定可能不一致；尾随斜杠也要统一处理。
- **插件直连系统 API**：走网关的 fs 调用没问题，但插件自己 spawn 子进程执行 shell 就绕过了校验。建议把插件进程放进容器跑。
- **snapshot 未启用**：目录不是 git 仓库时回滚策略失效，这种情况下删除类操作应强制升级为人工 confirm。

## 可复用建议

- 默认 deny，按需开 scope，别图省事直接给 `fs:delete`。
- 删除永远走四步：plan → confirm → snapshot → execute。
- 校验逻辑放执行层，不要放在 prompt 里。
- 定期跑红队用例：故意让 Agent 尝试删除 workspace 外的文件，验证边界是否真的拦得住。

## 总结

"Agent 不会误删"不是因为模型乖，而是系统在兜底。OpenClaw 的做法是把安全从 prompt 下沉到执行层：根边界圈住活动范围，能力分级限制危险操作，变更前置提供反悔机会，网关校验保证插件也守规矩。模型可以犯错，但错误路径到不了不该到的地方——这才是自动化敢放开手脚跑的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/176cc3942c32997f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/fa29f1339913b592.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/9730ff9f7c863318.png)

