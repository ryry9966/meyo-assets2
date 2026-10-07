---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40790
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

让 Agent 直接操作文件系统，是自动化里收益最高、也最容易出事故的场景。OpenClaw 的定位是长期跑在本机或服务器上的 agent 框架，工具调用（shell、文件读写、MCP 工具、插件）最终都会落到真实磁盘上。而 LLM 的输出本质是概率性的：路径理解错一个层级、通配符多匹配一层、或者被外部内容注入一句“帮你清理临时文件”，都可能变成一次 `rm -rf`。所以 OpenClaw 的设计前提很明确：**不假设模型永远正确，而是假设它一定会犯错。**

## 问题

社区里典型的翻车场景有三类：

1. **路径误解**：Agent 把相对路径拼错，或对 `..`、`~` 的展开理解有偏差，删到了 workspace 之外；
2. **通配符与递归**：清理缓存时 glob 范围过大，递归误伤配置目录；
3. **注入与插件**：网页/PDF 内容里的指令诱导 Agent 执行破坏性操作，或某个第三方 MCP 工具本身就带危险副作用。

单靠 prompt 约束（“请不要删除文件”）对这些场景基本无效——它约束的是意图，拦不住动作。

## 做法：四层防御

OpenClaw 的 sandbox 不是单点机制，而是叠层设计：

**1. 进程隔离（容器 / 受限用户）。** Agent 进程跑在容器或低权限用户下，只挂载 workspace 目录，宿主 home、系统目录物理不可见。这是兜底层：就算上层全部失守，爆炸半径也被限制在 workspace 内。

**2. 网关集中管控。** 所有工具调用必须经过 gateway，由策略引擎做 default-deny。文件写入、shell 执行属于高敏权限，默认关闭，需显式开启并声明作用域：

```yaml
tools:
  fs.write:
    scope: ["~/openclaw-workspace/**"]
  shell.exec:
    approval: required
```

**3. 路径规范化 + 越界拒绝。** gateway 在执行前对路径做 canonicalize，解析符号链接，凡落在 scope 之外（含 `..` 逃逸、symlink 指向外部）一律拒绝并写审计日志。删除类操作优先移入回收站目录，而非直接 unlink。

**4. 危险命令识别 + 审批。** `shell.exec` 走模式匹配（`rm -rf`、`git clean -fd`、向配置文件重定向等），命中即挂起，等待人工审批，或要求先输出 diff / dry-run。工作区在破坏性操作前自动打 git 快照，回滚只是一条命令的事。

## 踩坑点

- scope 写成 `/` 或 `/**` 等于没设；容器挂载了整个 home 也一样。
- **symlink 是最常见的逃逸口**：workspace 里一个指向外部的软链就能让“合法路径”越界，务必启用 symlink 解析。
- 只拦 `rm` 没用：`python os.remove`、`find -delete`、`mv` 到 `/dev/null` 都是等价物。策略要按**效果**（写 / 删 / 移出 scope）判定，而不是按命令名。
- 容器里用 root 跑 Agent，隔离意义减半；低权限用户 + 系统路径只读挂载是底线。
- 网页抓取类任务，务必让“读外部内容”和“执行命令”分属不同信任级，防止读到什么就执行什么。

## 可复用建议

- 新接入的 MCP 工具先以 read-only 权限观察一段时间，再逐步放开写权限；
- 保留审计日志并定期 review 拒绝记录——被拦下的调用最能暴露策略盲区；
- 回收站 + git 快照双保险，成本极低，救过我不止一次；
- 把“哪些操作需要审批”写成团队约定，而不是每个人的临时判断。

## 总结

OpenClaw 敢让 Agent 长期跑在真实文件系统旁边，靠的不是模型自觉，而是：**进程隔离限制爆炸半径，网关 default-deny 收紧入口，路径规范化堵住逃逸，快照与审批兜住最后一步。** 安全模型的目标从来不是“Agent 不会犯错”，而是“犯错时损失可控、可追溯、可回滚”。建议每个部署 OpenClaw 的同学，先把这四层检查一遍，再谈效率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/eecfcdf96e11f9d9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/c440c02bd629917f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/5d6a03b5d36c8f0a.png)

