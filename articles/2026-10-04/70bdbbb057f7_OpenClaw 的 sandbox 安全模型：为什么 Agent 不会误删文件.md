---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40337
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

给 Agent 配上 shell 执行能力，本质上等于把你的磁盘交给一个概率性输出文本的模型。OpenClaw 的思路是不赌模型“懂事”：exec 和文件工具默认运行在沙箱容器里，宿主机上只暴露一个 gateway 入口。理解这套模型，才知道哪些操作是安全的、哪些是你在配置里亲手挖的坑。

## 问题

LLM 生成的命令无法保证正确：`rm -rf "$DIR"/` 里变量为空、glob 展开到错误路径、插件带了未预期的副作用——任何一种都可能变成一次事故。如果 Agent 直接跑在宿主机用户态，边界就只剩模型的“自觉”，这在工程上不可接受。

## 做法：四层防线

1. **进程隔离**：exec 默认在 Docker 容器内执行，与宿主机文件系统天然隔离。
2. **文件系统边界**：只有 workspace 目录被挂载进容器，其余路径不可见。文件工具写入前做 realpath 解析，跨出 workspace 的路径（包括 symlink 逃逸）直接拒绝。
3. **工具门控**：工具按 allowlist 逐个开启；exec 可配命令白名单，白名单外的命令进入人工审批。
4. **审计与回滚**：gateway 记录每次命令与文件操作；workspace 纳入 git，误操作可 revert。

示意配置（以实际版本 schema 为准）：

```yaml
sandbox:
  mode: all                      # exec/文件工具默认进容器
  scope:
    - ~/.openclaw/workspace      # 唯一可写挂载
tools:
  exec:
    approval: missing            # 白名单外的命令需人工确认
```

## 踩坑点

- **图省事把整个 HOME 或 `/` 挂进容器**：路径边界检查形同虚设，等于亲手关掉沙箱。
- **MCP 工具绕过沙箱**：filesystem MCP 的根目录指向 `/`，Agent 会“换条路”写文件。MCP server 的作用域同样要收敛到 workspace。
- **macOS 上 Docker 慢就常驻关沙箱**：至少保留 exec 审批，退回人审模式，别裸奔。
- **沙箱内不是零损失**：`rm` 仍能删光 workspace，只是烧不到宿主机。所以 workspace 必须 git + 快照。
- **symlink**：workspace 里指向外部的软链靠 realpath 校验拦截，自行加挂载时别破坏这条路径校验链。

## 可复用建议

1. **默认拒绝，按需放行**：工具一个一个开，不做“先全开再收”。
2. **单一可写根**：插件、MCP、定时任务，所有组件只写 workspace。
3. **先审批后自动化**：新 Agent 前几天跑 approval 模式，看 exec 日志再逐步收紧。
4. **做破坏性演练**：定期用“清理这个目录”这类危险指令试探自己的配置，看它实际能碰到哪里。
5. **备份是最后防线**：git + 定期快照，永远不要把安全押在模型的一次输出上。

## 总结

OpenClaw 不假设模型不犯错，而是约束犯错的爆炸半径：容器隔离执行、路径边界校验、工具门控、人工审批、日志与回滚，层层递进。核心不是信任，而是约束。你在接入插件和 MCP 时守住同样的原则，才敢真正把 Agent 放进日常自动化里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/0be9be1a762b592f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/325ba4f0fa1b6a93.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/8a8d7342cac73501.png)

