---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41938
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 的定位是“跑在你自己机器上的常驻 Agent”：它有 `exec` 可以执行 shell 命令，有 `read/write/edit` 可以直接改文件，还能通过 MCP 和插件不断扩权。权限给得越足，一个自然的问题就是——模型幻觉一下、被 prompt injection 一下，一条 `rm -rf` 落在家目录上怎么办？

## 问题本质：别靠模型自觉

很多人第一反应是在 system prompt 里写“不要删除文件”。这不可靠：模型会被注入、会算错相对路径、会在长上下文里“忘掉”约束。OpenClaw 的思路是把**模型承诺**和**运行时强制**分开——安全边界放在进程与文件系统层，模型层只负责把活干好，不负责兜底。

## 做法：三层约束

**第一层：workspace 根目录绑定。** 文件类工具的路径全部相对 workspace root 解析，越界路径（`../` 穿越或指向外部的绝对路径）在工具层直接被拒绝，请求根本不会落到磁盘。这是纯代码逻辑，不依赖模型“听话”。

**第二层：Docker sandbox。** 把 sandbox 模式打开后（示意，以你版本的实际 schema 为准）：

```jsonc
// ~/.openclaw/openclaw.json
{ "agents": { "defaults": { "sandbox": { "mode": "all" } } } }
```

`exec` 等工具会在一次性容器里执行，只把 workspace bind-mount 进去。此时即使模型真的生成了 `rm -rf /`，删的也是容器内的可写层，宿主机上除 workspace 挂载点之外毫发无损。想给 agent 看参考资料，用只读挂载（ro）。

**第三层：审批与命令策略。** 对 `exec` 这类“能往外走”的工具配置 allowlist + 审批：白名单内命令直接放行，模糊或高危命令弹确认，由你在聊天里点头。MCP 和插件引入的第三方工具也能套同一套 policy。

一句话总结：模型想删文件，要先过路径检查 → 再被容器圈住 → 最后可能还要你确认。

## 踩坑点

- **图省事把 `$HOME` 或 `/` 挂进容器**：sandbox 立刻失效，等于裸奔。挂载列表保持最小。
- **host 模式 + 日常用户跑 gateway**：此时 `exec` 就是你本人的身份，删文件没有任何缓冲。确需 host 模式，至少换专用低权限用户。
- **workspace 里放指向外部的 symlink**：文件工具对 symlink 的解析策略随版本有差异，别把它当安全边界，敏感目录不要用链接引进来。
- **给容器挂了 `docker.sock` 或用 `--privileged`**：等于亲手递出逃逸通道。插件“需要管理 Docker”时最容易犯。
- **第三方 MCP server 不在 sandbox 里**：它们是独立进程，默认按 gateway 用户权限跑。接入前先搞清楚它的工具会碰哪些路径。

## 可复用建议

1. 默认开 sandbox；放宽必须是显式决定，而不是默认配置。
2. 挂载最小化：workspace 读写，参考资料只读，其他一律不挂。
3. `exec` 走白名单 + 审批，宁可多点一次确认。
4. 做一次金丝雀测试：在 workspace 外放一个空文件，让 agent 去删它，确认请求被拒、宿主机文件完好，再投入生产。
5. 边界之外仍要有备份——纵深防御，最后一层永远是快照。

## 总结

OpenClaw 不会让 Agent 误删文件，不是因为承诺模型“很乖”，而是把删文件这条路切成了三段：路径检查挡住越界写，容器挡住失控命令，审批挡住高危动作。模型层负责能力，运行时层负责兜底。理解了这个分层，你在扩展插件、接入 MCP、调整挂载时，才知道每一步究竟在动哪一层的安全假设——这是比“记住几条配置”更可迁移的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/e7df69301c7b005d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/95851487fe1ec010.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/ba78c6909a02e843.png)

