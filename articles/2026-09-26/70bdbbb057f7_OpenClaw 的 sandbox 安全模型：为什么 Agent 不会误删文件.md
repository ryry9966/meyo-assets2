---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39070
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 这类 Agent 框架的核心能力是让模型调用工具：读写文件、跑 shell、接 MCP 服务。能力越强，风险越具体——模型把"清理一下临时目录"理解错一次，代价可能就是半天的产出。

## 问题：模型输出是不可信输入

社区里常见误区是靠 prompt 约束："请不要删除重要文件"。这不构成安全边界。真正该反过来问的是：当模型输出一条 `rm` 时，系统里还有几层东西能拦住它？

OpenClaw 的做法是把安全放在工具实现层，而不是提示词层，形成四道防线。以下以默认配置为例。

## 四道防线

**1. 能力默认拒绝。** Agent 启动时不带任何文件系统工具，`fs.*` 系列按 scope 显式挂载。默认 scope 只有 `sandbox.workspace` 指向的目录。挂载发生在工具注册阶段——模型即使"想"到别的路径，工具层也没有对应的执行通道。

**2. 路径规范化校验。** 所有路径参数在工具内部先做 `realpath` 解析，再判断是否落在 workspace 边界内。这一步同时挡掉两类问题：`../` 穿越，以及 symlink 逃逸——workspace 里一个指向 `~/` 的软链，解析后落在边界外，直接拒绝。

**3. 破坏性操作分级。** 读操作直接放行；写操作记审计日志；删除和覆盖默认走 `trash` 策略——先移入 workspace 内的 `.trash/`，保留原始路径元数据，而不是直接 unlink。只有显式把 `fs.destructive` 设为 `confirm` 的真删场景，才会触发人工确认门。

**4. OS 级隔离。** shell 命令在沙箱进程里执行：非特权用户运行，workspace 可读写，根文件系统只读挂载，网络默认关闭。这层兜底的意义在于：即使前三层被绕过，进程也没有权限碰 workspace 之外的 inode。

## 三步验证 sandbox 确实生效

1. 在 workspace 外建一个测试目录，让 Agent"把它删掉"，观察工具返回的是权限拒绝而非执行成功。
2. 在 workspace 内创建指向外部的 symlink，让 Agent 借它写文件，确认被 realpath 检查拦截。
3. 让 Agent 删 workspace 内的文件，检查 `.trash/` 里能否找到带原始路径元数据的副本。

## 踩坑点

- **前缀匹配陷阱**：自己写插件时用字符串 `startswith` 判边界，`/workspace-backup` 会通过 `/workspace` 校验。必须用规范化后的真实路径做判断。
- **MCP 第三方工具自带文件能力**：它的实现不经过你的 `fs` 工具，sandbox 拦不住。给 MCP server 声明 scope 时按最低权限给，拿不准就先不挂载。
- **空变量 + glob**：`rm -rf $DIR/*` 在变量为空时会展开成危险命令。工具层校验拦不住 shell 内部展开，只能靠只读根挂载和非特权用户兜底。
- **`.trash/` 不是免费的**：大量小文件会拖慢目录遍历，要有定期清理策略，且清理动作本身也应走确认门。

## 可复用建议

- 所有校验做在工具实现里；prompt 只降低误操作概率，不承担安全职责。
- 默认拒绝、按能力授予，宁可多写一次挂载配置。
- 删除优先软删除，覆盖优先版本化。
- 每次写操作可审计、可回放，出问题才有得查。
- 定期用对抗性 prompt 做回归测试，把"sandbox 还在"当成 CI 断言。

## 总结

Agent 不误删文件，不是因为模型聪明，而是因为系统让"删掉不该删的东西"这件事难以表达。能力收敛在工具层，路径校验在实现层，破坏性操作有缓冲，OS 隔离兜底。四层任何一层单独看都不完美，叠起来才是 OpenClaw sandbox 的真实安全边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/723341caf0f2b42d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/03f21f1cf708587f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/ab51bec88ef665dc.png)

