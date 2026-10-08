---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40917
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

让 Agent 直接操作文件系统，最常被问到的就是“它会不会执行 `rm -rf`”。OpenClaw 的设计前提恰好相反：**假设模型一定会犯错**——会幻觉出错误路径、会误解指令、会在长任务里丢失上下文。所以它的安全模型不建立在“提示词里写了别删文件”之上，而建立在“即使模型犯错，破坏也被限制在最小范围”这个假设上。

## 问题

提示词不是安全边界。System prompt 里的“禁止删除文件”对模型是软约束：它可能被间接注入绕过，可能被长上下文稀释，也可能只是因为路径解析错了就执行危险操作。真正需要的，是在工具调用落到操作系统**之前**，有一层与模型无关的、确定性的拦截。

## 做法：四层防线

1. **工作区隔离（文件系统层）**：Agent 的所有文件操作被限制在 workspace root 内。路径在写入前先做 canonicalize（resolve 真实路径、展开 `..`），超出 root 的写入直接拒绝，而不是靠提示词劝阻。
2. **进程沙箱（执行层）**：shell 类工具不在宿主机直接跑，而是落入沙箱（容器/namespace + 降权用户 + 系统调用过滤）。即使 Agent 通过 bash 绕过了文件工具的检查，`rm` 也只能碰到沙箱内挂载的内容。
3. **能力权限（工具层）**：每个插件/技能声明所需能力，默认 deny。删除、网络写、git push 等不可逆操作要么关闭，要么要求显式确认。策略写在配置里，可 review、可版本化。
4. **审计与 dry-run（观测层）**：所有工具调用（参数、返回值、拒绝原因）落日志；破坏性操作走 plan-then-execute，先输出将要做的事，确认后再执行。

配置上大致四步：声明 workspace root → 开启 sandbox → 按需放行能力 → 跑一轮破坏性用例验证。示意如下（字段以所用版本文档为准）：

```yaml
workspace:
  root: ~/projects/demo
sandbox:
  enabled: true
  network: deny
permissions:
  fs.write: workspace
  fs.delete: confirm
  shell: sandboxed
```

## 踩坑点

- **symlink 逃逸**：Agent 在工作区内创建指向外部的软链接再写入。路径检查若不先 resolve 真实路径，隔离形同虚设。核心做了 canonicalize，但自己写插件时很容易漏掉这一步。
- **命令绕过**：只给文件工具加检查，又同时给 Agent 一个全权限 shell，等于没做。校验必须放在离资源最近的一层（进程/内核），而不是某个工具的封装里。
- **确认疲劳**：人机确认弹得太频繁，用户会无脑点同意。建议把确认只留给真正不可逆的操作，其余用 dry-run 替代。
- **容器里跑 root**：uid 隔离在 root 下失效，沙箱进程必须降权运行。
- **权限一紧就全放开**：默认 deny 初期会打断工作流。正确做法是按报错逐条放行，而不是一把梭全开。

## 可复用建议

- 把“模型不犯错”从安全假设里删掉，所有防线按“模型会犯错”设计。
- 校验放在离资源最近的层；提示词约束只当 UX，不当边界。
- 破坏性操作默认 plan-then-execute + 审计日志，出问题可回放定位。
- 新插件上线前，用对抗用例测一遍沙箱：`../` 穿越软链接、`rm -rf $UNSET_VAR`、超长路径、跨区 rename。

## 总结

OpenClaw 之所以“不会误删文件”，不是因为模型足够聪明，而是因为错误发生时还有四层确定性的网接着。Sandbox 的价值在于把模型的不确定性圈在一个可恢复、可审计的范围内——这比任何提示词都可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/c11934cbd3e15156.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/ba9210a16ed4f219.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/a90950a61ecd6ef9.png)

