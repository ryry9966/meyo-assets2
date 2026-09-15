---
title: Agent 的 tools.md 实践：分层、生成、自检，管住本地配置与环境差异
feedId: 37682
source: 综合讨论
publishedAt: 2026-09-15
---

# 背景

Agent 落地到实际工作流后，最大的摩擦往往不是模型能力，而是"每个环境长得不一样"。同一套 agent 配置，在我的 macOS 上能直接调 ffmpeg，到同事的 Linux 服务器上路径就变了；MCP server 本机用 stdio 启动，到 CI 里得换方式。这些差异如果散落在 prompt、脚本和口头约定里，迟早出事。

tools.md 的思路是：给 agent 一份"本环境工具说明书"，写清当前机器上有什么工具、怎么调、缺什么。但多数人的用法是手写一份静态文件——这恰恰是新问题的开始。

# 问题

手写 tools.md 常见三类翻车：

1. **绝对路径写死**。`/Users/xxx/...` 一进版本库，换台机器就失效。
2. **文档腐化**。环境升级后没人更新，agent 按旧说明调用，失败后开始自由发挥，幻觉就是这么养出来的。
3. **秘密泄漏**。为了"一次配好"，有人把 API key 直接写进文件提交了。

根因是：把"环境事实"当成了"手工文档"。环境事实应该被探测出来，不是被写出来。

# 做法

我们目前的实践分三层，可直接复用：

**1. 分层加载：base + local**

- `tools.base.md`：进版本库。只写与机器无关的内容——项目用到哪些工具、各自的用途、需要哪些环境变量的**名字**（不是值）、统一调用约定。
- `tools.local.md`：gitignore。只写本机事实：实际路径、版本号、本机特有的启动参数。agent 先读 base，再用 local 覆盖。

**2. local 文件由脚本生成，不手写**

写一个十几行的探测脚本：遍历工具清单，逐个 `command -v` / `--version`，把结果渲染成 markdown。同事 clone 仓库后跑一次 `./scripts/gen-tools-md.sh`，local 文件就有了。手写内容只剩 base 层，腐化面积大幅缩小。

**3. 内置自检命令**

在 base 里写明验证方式："调用任何工具前先跑 `tools doctor`，任何一项 FAIL 就停下来报告，不要猜测替代方案。"这一条能拦住大部分幻觉调用。

# 踩坑点

- **别在 markdown 里写条件逻辑。**"如果是 Windows 就……"这种句子 agent 经常读岔。让生成脚本按平台直接输出对应内容，文件里只留事实。
- **控制长度。**tools.md 会进上下文，每个工具三五行足够：名称、调用方式、版本、已知坑。写成 wiki 反而稀释关键信息。
- **Windows 路径转义。**生成脚本里注意反斜杠和引号处理，否则 agent 拿到的命令直接不可执行。
- **CI 里也要跑。**加一步"重新生成 local 内容并与 base 做一致性 diff"，环境变量改名、工具被移除这类破坏性变更会在 CI 就炸出来，而不是等 agent 上线后才发现。

# 可复用建议

- 提交的是"生成器 + base"，不是完整配置；附一份 `tools.local.example.md` 作为格式样例。
- 环境变量一律只写名字，值放 .env 或 secret manager。
- 每个工具条目固定四个字段：用途、调用命令、版本探测、常见失败表现。格式统一后 agent 解析的稳定性明显提升。
- 把"工具不可用时怎么办"也写进 base（跳过 / 询问 / 降级），别让 agent 每次现场决策。

# 总结

tools.md 本质上是一份环境快照契约。正确姿势不是"认真手写"，而是"分层 + 生成 + 自检"：base 层管约定，local 层管事实，脚本负责探测，CI 负责防漂移。把环境差异从 prompt 里的隐患，变成版本库里可 review 的产物——这才是 agent 配置该有的工程化处理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/169da0c3b9f36ce3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/8e635b1b9f866d5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/fac05340efdfdbe3.png)

