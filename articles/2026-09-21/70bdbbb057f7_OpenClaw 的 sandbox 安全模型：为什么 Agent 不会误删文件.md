---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38341
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

只要给 Agent 配上 shell 和文件读写工具，它就同时拥有了“搞坏你环境”的能力。社区里常见的翻车不是模型不够聪明，而是我们给了它不该有的可见性和权限。OpenClaw 的设计立场是：**不指望模型永远不犯错，而是让它犯错时无路可走**。这篇帖子拆一下它的 sandbox 安全模型，解释“误删文件”为什么在实践中极少发生。

## 问题

复盘下来，Agent 误删基本来自五类：

1. **意图偏差**：把“清理临时文件”执行成递归删除；
2. **路径错误**：长会话中 cwd 漂移，相对路径指到别处；
3. **展开意外**：变量为空时 `rm -rf $DIR/*` 变成 `rm -rf /*`；
4. **注入**：MCP 工具返回内容里夹带“顺便删掉日志”之类的指令；
5. **越界**：workspace 里的 symlink 指向 `$HOME`。

这五类没有一类能靠 prompt 解决。

## OpenClaw 的分层做法

**第 0 层：文件系统沙箱。** Agent 进程只挂载 workspace 声明的目录，其余路径在它眼里不存在。执行前对路径做 canonicalize + realpath 校验，任何 symlink 解析到沙箱外直接拒绝。这是兜底层，完全不依赖模型行为。

**第 1 层：操作分级。** 工具按 read / write / delete 分级，删除和移动走 destructive 通道，与普通写分开鉴权。配置大致是：

```yaml
sandbox:
  root: ./workspace
  mounts: [./workspace, ./data]
permissions:
  read: allow
  write: allow
  delete: confirm      # 或 trash：移入回收目录
tools:
  deny: [network_write]
```

**第 2 层：计划-执行分离。** Agent 先产出结构化操作计划（路径、动作、理由），由独立的 executor 校验计划合法性后才落盘。删除类操作可开 dry-run，先看将影响哪些文件。

**第 3 层：作用域收紧。** 每个 session 维护独立的路径白名单和工具白名单，插件与 MCP server 按需授权，默认拒绝。

**审计日志**贯穿所有层：每次写/删除都记录计划 ID、路径、结果，可回放。

## 踩坑点

- **symlink 逃逸**：仓库里历史遗留的软链指向 home 目录，必须在 realpath 这层拦，而不是信工具描述；
- **白名单写成 `/**`**：等于关掉沙箱，社区里出过这种事故；
- **确认疲劳**：把 `delete: confirm` 顺手改成 always-allow，一周后翻车。建议改用 trash 模式，而不是放开确认；
- **cwd 漂移**：executor 每次以绝对路径执行，别依赖会话状态；
- **用 prompt 代替沙箱**：“请不要删除文件”这种约束，在注入面前没有意义。

## 可复用建议

1. 最小可见性优先于最小权限：先让危险路径“看不见”，再谈禁不禁；
2. 删除尽量改成“移入 trash 目录”，由定时任务统一清理，把不可逆操作变成可逆的；
3. 上线前做红队演练：故意让 Agent 执行危险指令，验证每一层真的拦得住；
4. 审计日志接告警，删除量异常时人工介入。

## 总结

Agent 不会误删文件，不是因为它足够聪明，而是因为即使它想删，第 0 层就无路可走，第 1、2 层还有确认和计划校验兜着。这套模型的核心是**不信任**：不信任模型、不信任工具描述、不信任单层防线。把上面的配置思路搬进你们自己的自动化流程，大部分“手滑”问题在架构层面就消失了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/9577e724502a9160.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/6ee91be60ccca23d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/accbcb14e340cbca.png)

