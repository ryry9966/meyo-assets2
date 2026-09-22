---
title: OpenClaw sandbox 安全模型拆解：为什么 Agent 删不掉你的文件
feedId: 38486
source: 综合讨论
publishedAt: 2026-09-22
---

# 背景

给 Agent 接上 shell 和文件读写工具之后，大家最先问的不是"它能做什么"，而是"它会不会把我的目录删了"。这不是杞人忧天：模型会拼错路径、把 glob 写宽、在多步任务里搞错工作目录。OpenClaw 的基本立场是——**不依赖模型"懂事"，把安全边界放在工具层**。

# 问题

Prompt 是软约束。系统提示里写"请不要删除文件"没有任何强制力：模型可能被注入、可能推理出错、可能被一段恶意文档带偏。任何依赖 LLM 自觉的安全设计，最终都会在生产环境失效。

# 做法：四层防线

OpenClaw 的 sandbox 在每次工具调用上依次过四层：

1. **路径围栏**：所有文件工具调用先做 realpath 解析，解析后落在 workspace root 之外的路径直接拒绝。符号链接按解析后的真实路径判断，不看表面前缀。
2. **权限分级**：读操作自动放行；写按目录 allowlist；删除、覆盖、shell 执行归为 destructive 级，默认拦截，需要显式确认或预授权。
3. **命令分析**：shell 工具不是透传，先过一层解析，识别 `rm -rf`、`find -delete`、重定向覆盖、空变量展开的 glob 等模式，命中即拦。
4. **快照回滚**：批量写/删前对目标子树做快照，存放在 workspace 外的备份目录，Agent 自己够不到。

一份最小配置：

```yaml
sandbox:
  root: ~/openclaw-workspace
  write_allowlist: [workspace/**, /tmp/openclaw/**]
  destructive: confirm        # confirm | allow | deny
  shell_deny_patterns: ["rm -rf", "find * -delete"]
  snapshot:
    enabled: true
    dir: /var/lib/openclaw/snapshots
```

# 踩坑点

- **符号链接逃逸**：早期版本只做字符串前缀匹配，workspace 里一个指向 `/etc` 的软链就绕过了围栏。必须用 realpath 之后的路径做判断。
- **相对路径基准漂移**：Agent 中途 `cd` 之后，相对路径语义全变。路径校验要针对每一次调用，而不是会话开始时的目录。
- **快照目录放 workspace 里等于没做**：Agent 一条 `rm` 就把备份端了。快照必须在围栏之外，且对该工具不可见。
- **第三方 MCP 工具旁路**：如果插件自带执行能力，会绕过文件策略。每个 MCP 工具要单独发 capability，按最小权限给。
- **拒绝列表永远补不全**：别指望穷举所有危险命令，优先 allowlist 思路。

# 可复用建议

- 把 prompt 当不可信输入处理，策略只放工具边界，模型说了不算。
- 每个破坏性操作都要可逆：先快照，删除先进回收目录而非直接 unlink。
- 所有工具调用落审计日志，参数、路径、命中策略一并记录，事后能还原现场。
- 在 CI 里用一组对抗性 prompt（诱导删除、路径穿越、软链逃逸）做回归测试，sandbox 改动必须过这套用例。

# 总结

Agent 不会误删文件，不是因为模型足够聪明，而是因为**最坏情况发生时，边界不依赖模型**。路径围栏、权限分级、命令分析、快照回滚，任何一层单独拿出来都不够，叠在一起才是 sandbox。如果你在自己的工作流里接了新的文件类工具，先问一句：它走的是同一套边界吗？

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/3ae2dce2dc2d5586.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/aa4dd04037d56c9d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/813953cf9a4e847f.png)

