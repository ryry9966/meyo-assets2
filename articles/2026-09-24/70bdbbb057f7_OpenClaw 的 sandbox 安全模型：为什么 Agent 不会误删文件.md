---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38759
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

在 OpenClaw 的自动化实践里，Agent 拿到 shell 和文件工具之后，大家担心的往往不是"它不会干活"，而是"它干活时不看路"——一条拼错路径的 `rm -rf` 或一次错误的覆盖写入，足以让工作区报废。OpenClaw 的 sandbox 安全模型就是围绕这个问题设计的：**不依赖模型自觉，而是让"删不掉"成为架构属性**。

## 问题

把 LLM 接上文件系统工具后，风险来自三个层面：

1. 模型输出本身有噪声，绝对路径和相对路径混用、目录名拼错都常见；
2. 一次任务可能经过主 Agent → MCP server → 插件 → 子 Agent 多跳，每一跳都是新的执行点；
3. 字符串级的命令过滤（黑名单拦 `rm`）极易被管道、变量展开、别名绕过。

所以"提示词里写清楚"或"正则拦危险命令"都不成立，防线必须下沉。

## 做法：三层拦截

**第一层：文件系统视图。** Agent 启动时只挂两个目录：源码目录以只读挂入，可写区仅 `workdir/workspace`。校验放在 mount namespace + Landlock/seccomp 层，而不是应用代码里拼路径判断。这样即使模型输出了 `../../etc/hosts`，写请求也到不了真实文件系统。

**第二层：能力范围声明。** 每个工具（内置 fs 工具、MCP server、插件）注册时声明 scope：可读路径、可写路径、允许的命令类别。MCP server 不继承主 Agent 的权限，每个 server 单独裁剪；能力令牌按会话签发，过期失效。

**第三层：破坏性操作网关。** 删除、覆盖、批量移动强制走 dry-run：先把变更落成 staging diff（删了哪些文件、动了哪些字节），再决定执行、转人工确认或拒绝。`rm` 不是被字符串匹配拦下，而是它的 syscall 路径根本不在可写视图的允许集里。

最小配置示例：

```yaml
sandbox:
  mounts:
    - { src: ./repo, mode: ro }
    - { src: ./workspace, mode: rw }
  gateway:
    destructive: dry-run   # preview | approve | deny
  audit: ./logs/audit.jsonl
```

## 踩坑点

- **符号链接逃逸**：workspace 里一个指向沙箱外的软链能让写请求穿透。所有路径必须 resolve 之后再校验，OpenClaw 默认拒绝指向沙箱外的 symlink。
- **只拦外层 Agent**：MCP server 是独立进程，自带 fs 工具就有自己的逃逸面，必须过同一套 scope 校验。
- **信任面过大**：图省事把整个 HOME 挂成可写，沙箱形同虚设。可写区越小越好，`node_modules`、venv 这类高频写入单独给挂载。
- **白名单逐步放宽**：为了"跑得通"一路放开权限，最后等于没沙箱。权限收紧的成本远低于事后清理。

## 可复用建议

1. 默认拒绝、按需放行，权限收敛到本次任务的最小集；
2. 校验尽量下沉到 OS 层，应用层路径判断只当第二道防线；
3. 一切破坏性操作先产出可审阅的 diff，再执行；
4. 审计日志逐条记录工具调用与沙箱裁决，出事能回放；
5. 把"Agent 误删"写进 CI 冒烟测试：故意构造越权写请求，断言被拒。

## 总结

OpenClaw 之所以"不会误删文件"，不是因为模型足够聪明，而是因为系统里不存在一条能让越权删除生效的路径：只读挂载挡住源头，能力裁剪限制每一跳，dry-run 网关兜住最后一步。好的 Agent 安全模型不试图预测模型行为，而是让危险行为无处落地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f171036487a51d46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/60e179f75cd0634a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9529f979801c3855.png)

