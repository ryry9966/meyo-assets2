---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41280
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

给 Agent 接上文件和 shell 工具之后，大家最担心的通常不是它写不出代码，而是某次理解偏差让它执行了一条 `rm -rf`。OpenClaw 在设计上默认假设模型会犯错，所以它的 sandbox 不靠 prompt 里写一句"请小心"，而是把约束下沉到工具层和执行层。最近我把这套模型完整跑了一遍，记录一下它的防御结构和实际效果。

## 问题：误删是怎么发生的

复盘常见的误删路径，几乎都不是模型"蓄意破坏"，而是三类工程问题：

1. **路径歧义**：相对路径、错误的 cwd、`..` 回退到预期之外；
2. **通配符放大**：`rm build/*` 的展开发生在 shell，模型看到的和实际执行的不是一个东西；
3. **权限过宽**：为了让任务跑通，直接放开整个 `sh` 工具，等于没有沙箱。

## 做法：四层防御

**第一层，工作区锚定。** 所有文件工具启动时绑定一个 workspace root，任何路径先做 canonicalize：解析符号链接、折叠 `..`，解析结果落在 root 之外的请求直接拒绝。这一步在工具入口完成，不依赖模型自觉。

**第二层，破坏性操作分级。** 工具层对操作分类：读、写、删是不同的 capability。删除、递归删除、覆盖移动会被标记为 destructive，默认策略是 dry-run 加人工确认，或直接 deny。配置里可按路径写 allow/deny 规则，比如只允许 `src/**` 可写，`**/.git/**` 一律只读。

**第三层，执行层隔离。** 模型通过 shell 调脚本时，脚本内部做了什么，工具层看不见。所以严格模式会把 shell 执行放进 OS 级沙箱（容器/命名空间），文件系统挂载限制到 workspace，系统目录只读挂载。这样即使脚本里有 `rm -rf /` 的语义，作用域也被物理限制住。

**第四层，审计与回滚。** 所有写/删操作进 journal，被删文件先移入 trash 目录而非直接 unlink，出问题可以从审计日志恢复。

配置上大致三步：开启 sandbox 并指定 workspace root；按项目写 path allow/deny；destructive 策略设为 confirm 或 dry-run。验证跑通后再逐步放宽到自动执行。

## 踩坑点

- **symlink 逃逸**：workspace 里链接到 home 目录，canonicalize 后路径就出去了。默认拒绝是正确行为，需要共享目录就显式声明额外挂载点。
- **cwd 混淆**：Agent 常假设 cwd 是项目根目录。建议每个 session 显式注入绝对路径约定，减少相对路径使用。
- **放开 `sh -c` 等于拆掉前三层**。需要 shell 的任务，优先拆成受控子工具，而不是给一个万能入口。
- **git clean、构建脚本这类"合法删除"** 会被 destructive 规则拦下，这是预期行为，按项目加白名单即可，别为省事全局关闭。

## 可复用建议

- 默认拒绝、白名单放行，不要反过来；
- 破坏性判定看最终 argv 和真实路径，不看模型输出的"意图"；
- 删除先进 trash，给系统留一条后悔路径；
- 用对抗性 prompt 做回归测试：让它删 workspace 外的文件、用通配符删、写脚本再执行，全部应该在沙箱层被拦截。

## 总结

OpenClaw 能避免误删，靠的不是模型更聪明，而是把"模型可能犯错"当成设计前提：路径入口归一化、操作分级、执行隔离、全程可回滚。任何要接文件系统的 Agent 项目，这套四层模型都值得直接复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/1bc99bdac5e3a423.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/c15f7834eb90880b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/6db1b1f2b44e8626.png)

