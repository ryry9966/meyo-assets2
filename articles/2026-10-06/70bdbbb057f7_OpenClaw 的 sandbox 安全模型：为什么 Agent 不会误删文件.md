---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40642
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

让 Agent 直接操作文件系统后，大家担心的往往不是它写不出代码，而是某次规划错误后执行了一条 `rm -rf`。在提示词里写“删除前请谨慎”属于君子协定：模型一旦在长上下文中拼错路径，或被间接注入的指令带偏，这类约定没有任何强制力。OpenClaw 的思路是反过来的——不指望 Agent 不犯错，而是假设它一定会犯错，然后在执行层把错误代价压到可恢复。

## 四层防线

**1. 路径作用域（path scoping）**。Agent 启动时绑定一个 workspace root，所有文件工具（read/write/delete/move）在沙箱层先做路径规范化再校验：绝对路径越界、`../` 回溯逃逸、symlink 指向 root 之外，一律拒绝。Agent 世界观里的“文件系统”就只有这棵目录树，越界不是被警告，而是工具直接报错。

**2. 能力门控（capability gate）**。工具注册时声明能力级别，`fs:read`、`fs:write`、`fs:delete` 是三档。`fs:delete` 默认关闭，需显式开启；开启后单批删除超过阈值（如 10 个文件，或命中关键目录名）会触发人工确认。MCP 接入的第三方工具走同一套声明，没声明删除能力就调不了删除操作。

**3. 计划-执行分离（dry-run）**。破坏性操作默认先产出变更清单（哪些路径、什么动作），沙箱校验清单合法后才执行。自动化流水线可设 `dry_run: true`，让 Agent 每轮只输出计划，由人或 CI 审核后放行。

**4. 回收站 + 审计日志**。即便删除被放行，文件也是移入 workspace 内的 `.trash/`（保留 N 天），而非直接 unlink。被拒绝和被放行的操作都落入 audit log，事后可以回放“Agent 到底试图干了什么”。

## 实操步骤

```yaml
sandbox:
  workspace_root: ~/projects/demo   # 显式指定，别用默认当前目录
  mode: strict                      # 越界直接报错，不降级为只读
  capabilities: [fs:read, fs:write] # 按任务裁剪，此例不含删除
  dry_run: true
  trash: { enabled: true, retain_days: 7, exclude: [node_modules, dist] }
```

1. 按任务最小化授权：只读检索不给 `fs:write`，重构任务给写但不开删。
2. 自动化场景全程 dry-run，人工确认只卡在删除类操作上。
3. 上线前做对抗演练：明确要求 Agent “删除 workspace 外的某个文件”，确认沙箱拒绝且留下审计记录。

## 踩坑点

- **shell 工具是最大的后门**。沙箱管住了文件工具，但如果给了无限制的 shell，一条 `rm` 就绕过全部防线。要么把 shell 跑进容器并限制可写挂载，要么在工具白名单里禁掉 shell 的文件删除类用法。
- **symlink 逃逸**。workspace 里指向外部的软链接若不做 realpath 解析就可能被利用，升级后确认沙箱对 symlink 做了规范化。
- **第三方 MCP 工具越权声明**。接入新 MCP server 时看一眼能力清单，声明了 `fs:delete` 却用不上的，直接裁掉。
- **调试时开的 auto-approve 忘了关**。这是最常见的事故来源，建议把确认开关做成 profile，调试与生产物理隔离。

## 可复用建议

- 安全边界放在执行层而不是提示层：提示词只降低犯错概率，不负责兜底。
- 给“删除”单独建模，它和“写”不是同一种风险，值得独立的确认流程。
- 新工具来源一律按最小能力接入，遇到限制再逐步放开。
- 定期看 audit log，被拒绝操作的列表就是最好的安全体检报告。

## 总结

“Agent 不会误删文件”不是因为模型足够聪明，而是因为沙箱让它物理上删不到。OpenClaw 的安全模型本质是分层减损：作用域限定爆炸半径，能力门控抬高破坏门槛，dry-run 提供人审窗口，回收站保证可恢复，审计日志保证可追责。把这套配置跑通之后，才能放心把 Agent 接进日常文件操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/7f43cddafdc6ba87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/badf4fdee66f0777.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/a8da98d762077014.png)

