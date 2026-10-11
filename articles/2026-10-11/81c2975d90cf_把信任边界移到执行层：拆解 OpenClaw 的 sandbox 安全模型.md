---
title: 把信任边界移到执行层：拆解 OpenClaw 的 sandbox 安全模型
feedId: 41207
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

给 Agent 接上 shell 和文件工具之后，最怕的场景很具体：一句“帮我清理一下临时文件”，模型把路径拼错、把通配符展开到意料之外的范围，一条 `rm -rf` 就落在了错误的目录上。LLM 的可靠性模型和文件系统并不匹配——它会自信地犯错，而且没有“手抖”之前的犹豫。

## 问题

不少框架的防御停留在 prompt 层：系统提示里写一句“不要删除文件”。这是软约束，会被注入改写、会被长上下文挤掉、会被模型在多步推理中自己“合理化”掉。误删的根因不是模型“想删”，而是执行链路上没有一道硬闸门。OpenClaw 的设计前提是：**不指望模型不出错，而是让危险操作在执行层直接失败。**

## OpenClaw 的 sandbox 是怎么做的

四层防御，由外到内：

1. **工作区隔离（jail）**：Agent 的文件可见范围被限定在 workspace 根目录内。所有文件工具的路径参数先做 realpath 归一化，解析结果必须落在 jail 内，否则直接拒绝。`../` 穿越和符号链接逃逸都在这一层挡住。
2. **能力声明（scopes）**：每个 MCP server 和插件在清单里声明能力（如 `fs:read`、`fs:write`、`shell:exec`）。未声明的能力在握手阶段就不下发对应工具，模型连“看见”的机会都没有。
3. **破坏性命令识别**：对 shell 工具的命令做解析，命中 `rm -rf`、`find -delete`、`git clean`、重定向覆盖等模式后按策略处理：人工确认、转入回收站（先 move 到 `.trash`，由用户定期清理）、或直接拒绝。
4. **快照回滚**：执行破坏性操作前自动对 workspace 打快照，出问题可整体回滚。

配置上大致四步：

```yaml
sandbox:
  enabled: true
  root: ./workspace
  destructive_policy: trash   # confirm | trash | deny
  snapshot: true
```

再按需收紧各 MCP server 的 scopes、打开 audit log，即可上线。

## 踩坑点

- **符号链接逃逸**：workspace 里放了一个指向 `~/` 的软链，realpath 之后就越界了。新版在创建 symlink 时会校验目标，旧版本要自己盯着。
- **shell 旁路**：文件工具管住了，但 Agent 用 shell 的 `mv`/`cp` 绕过路径校验。结论：shell 必须和文件工具跑在同一套 jail 里（容器或命名空间级隔离），只靠应用层路径校验不够。
- **TOCTOU**：路径校验和实际执行之间有时间窗，并发场景下 symlink 可能被替换。根治方案是 `openat` + `O_NOFOLLOW` 这类原子操作，避免先 check 后 use。
- **通配符滥用**：allowlist 写 `**` 等于没写。
- **Windows 路径**：反斜杠、大小写不敏感、盘符，路径归一化逻辑要单独测。
- **“临时关一下沙箱”**：为了调试方便关掉，然后忘了开回来。建议 sandbox 开关绑定 profile，而不是做成全局开关。

## 可复用建议

1. 最小权限：探索型 Agent 只给 `fs:read`，写操作收敛到单独的执行 Agent。
2. 永远不要给 Agent 真正的 `rm` 能力，破坏性操作一律走 trash + 快照。
3. 定期做 red-team 测试：故意让模型删 workspace 外的文件、删不存在的目录、通过软链越界，验证拦截是否真的生效。
4. audit log 只追加、落盘保存，出事故能完整复盘。

## 总结

Agent 安全的核心不是让模型更聪明，而是把信任边界从 prompt 层移到执行层。OpenClaw 的 sandbox 可以概括成一句话：模型可以犯错，但文件系统不允许它犯“删错文件”这种不可逆的错。防御做在执行层，才有资格谈“放心让 Agent 干活”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/b442417a0cecd025.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/9f77cc26398a4cce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/e8abccdc83963062.png)

