---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43252
source: 综合讨论
publishedAt: 2026-11-02
---

# OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件

## 背景

OpenClaw 的 Agent 默认具备 shell 执行和文件读写能力，配合 MCP 工具和插件，可以完成相当完整的自动化任务。能力越完整，“它会不会哪天把我的目录删了”这个问题就被问得越多。

这篇帖子拆一下 OpenClaw sandbox 的分层设计，说清楚一件事：**Agent 不误删文件，靠的不是模型足够聪明，而是环境让错误的代价可控。**

## 问题：误删是环境问题，不是模型问题

先看几个真实感很强的翻车场景：

- `rm -rf "$BUILD_DIR"/*`，变量恰好为空，展开后变成删根目录下所有东西；
- Agent 以为自己在 `./build` 里，实际 cwd 在仓库根；
- 用户说“清理一下缓存”，Agent 把它认为像缓存的目录全删了；
- Agent 在 workspace 里创建指向 `/etc` 的 symlink，再“顺手清空”里面的文件。

LLM 的输出是概率性的，靠 prompt 约束它 100% 不犯错是不成立的。sandbox 的设计前提因此很明确：**假设错误一定会发生，让每次错误要么被拦截、要么可恢复。**

## 做法：四层防线

**1. 执行层：容器 + 最小权限。** Agent 的所有命令在容器内执行，以低权限用户运行。workspace 通过 bind mount 挂进去，宿主其余路径默认不可见。`docker.sock` 绝不挂进 sandbox——挂了它等于把宿主 root 交出去。

**2. 路径层：realpath 双校验。** 文件工具和 MCP 文件类插件在每次读写前先做 realpath 解析，再判断是否落在 workspace 前缀内。这样 `..` 和 symlink 都逃不出去——因为校验发生在解析之后，而不是对路径字符串做匹配。

**3. 命令层：deny 规则 + 人工确认。** 已知高危模式（递归删除系统路径、`dd`/`mkfs`、对关键目录 `chmod -R`）直接拒绝；未命中 allowlist 的高危命令进入确认队列，由人拍板。所有命令连同策略判定结果写入 audit log。

配置大致长这样（简化示意）：

```yaml
sandbox:
  workspace: ~/.openclaw/workspace
  mounts:
    - src: ~/.openclaw/workspace
      dst: /workspace
      mode: rw
  deny_commands:
    - "rm -rf /"
    - "dd *"
  confirm_on_unmatched: true
  snapshot: before_task
```

**4. 快照层：兜底回滚。** 每个任务开始前对 workspace 做快照（overlayfs 或 git commit 均可），提供一键回滚。前三层是“拦”，这一层是“就算删了也能回来”。

四层的关系是串行的闸门：命令先过执行隔离，再过路径校验，再过命令策略，最后有快照兜底。

## 踩坑点

- **只匹配字符串不够。** 拦 `rm -rf` 字面量没意义，模型会改写成 `rm -rf "$X"/*`。有效的做法是限制删除类命令只能作用于 `/workspace` 前缀，且配合 realpath 校验。
- **symlink 逃逸最隐蔽。** 路径校验放在解析之后才有效，放在之前等于没做。
- **挂载手滑。** 调试时把 `ro` 改成 `rw` 忘记改回。挂载清单建议进版本控制，变更走 review。
- **策略太严是最大的坑。** 用户被高频拦截烦了，直接整体关掉 sandbox，结果比没有策略更糟。策略要靠 audit log 迭代：每周扫一遍，把反复人工放行的命令沉淀成 allowlist。
- **插件别绕过网关。** 自定义插件的文件操作建议复用 sandbox 的路径校验函数，不要自己直接 `open()`，否则四层防线在那一处全部失效。

## 可复用建议

- 默认拒绝，按 allowlist 放行路径，而不是反过来；
- 删除类操作统一走“限制根 + realpath”双校验；
- 快照要在日常任务里演练回滚一次，别等真出事才第一次用；
- 用对抗性 prompt 测策略，比如“帮我清理 /tmp 之外的所有缓存”，看防线在哪一层接住；
- MCP 工具调用同样过命令网关，不要因为是“工具调用”就降级处理。

## 总结

为什么 OpenClaw 的 Agent 不会误删文件？因为路径出不去、高危命令被拦、真删了也能回滚。这套模型的核心思路是**把安全从“信任模型”转移成“设计环境”**：不赌模型不犯错，只保证犯错时损失有界。对任何想让 Agent 摸真实文件系统的人来说，这比任何 system prompt 都可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/bcc01b7463753ad3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/9bcf0b07e87a96f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/7f337deadba9136f.png)

