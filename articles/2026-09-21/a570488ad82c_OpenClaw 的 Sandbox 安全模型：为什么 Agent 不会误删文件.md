---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38394
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

OpenClaw 的 agent 默认带 exec 工具，能直接跑 shell。第一次让模型自由执行命令的人，几乎都想过同一个问题：它要是哪天"抽风"执行了 `rm -rf`，我去哪哭？这篇帖子拆一下 OpenClaw 的 sandbox 模型：为什么在配置正确的部署里，误删文件这件事的爆炸半径是可预期的。

## 问题：破坏力来自哪里

LLM 的错误是概率性的：路径幻觉、参数错位、把"清理临时文件"理解成"清理这个目录"。所以任何只靠提示词的约束（"你永远不要删除文件"）都是软护栏，不是边界。真正的边界只取决于一个问题：**agent 进程能 touch 到哪些文件**。

另外注意一个容易混淆的点：dmPolicy / pairing 这类配置控制的是"谁能触发 agent"，防的是外人；而 sandbox 防的是"agent 自己犯蠢"。本文聚焦后者。

## 做法：把删除圈进一个可丢弃的盒子

OpenClaw 的思路是三层叠加：进程级隔离 + 工作区挂载 + 工具策略。

1. **开 sandbox**。配置示意（字段名以你手上版本的文档为准）：

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "non-main" }  // 非 main agent 一律进容器
    }
  }
}
```

2. **只挂 workspace**。agent 在容器里能看到的宿主机文件，等于你显式 bind 的目录。默认只有 workspace，读写都发生在里面。
3. **容器内非 root 运行**。对系统路径没有写权限，`rm /etc/xxx` 直接 permission denied，而不是删完才报错。
4. **工具策略兜底**。高危工具/命令走 deny 或审批（elevated approvals），不指望模型自觉。
5. **验证边界，而不是相信配置**。在 agent 会话里让它执行：

```bash
pwd                  # 应显示容器内路径
ls /                 # 容器文件系统，不是你的笔记本
rm /etc/hostname     # 应失败
```

失败才是正确结果。这一步别省。

## 踩坑点

- **默认 non-main ≠ 全沙箱**。main agent 跑在宿主机上，很多人配完就以为全家都进容器了。
- **图省事把 `~` bind 进容器**。隔离直接归零，sandbox 变成一个昂贵的 shell。
- **挂了 docker.sock 或给了 root**。等于把钥匙插在门上。
- **为了 `npm install` 全量放开网络**。正确做法是域名 allowlist，而不是"先通了再说"。
- **多个 agent 共用同一 workspace**。你以为互不干扰，实际它们在同一条起跑线上互删。

## 可复用建议

- **最小爆炸半径**：agent 可写的最大目录，应当等于你"愿意随时重建"的目录。
- 把 agent 当"会打字的外包新人"发权限：账号、目录、网络都按最坏情况设。
- 删除类操作引导到 workspace 内的专用子目录，容器一销毁即清零。
- **定期演练**：主动让它执行破坏性命令，验证边界是否还在，比读十遍文档管用。
- exec 日志留痕，事后能回放"它到底干了什么"。

## 总结

OpenClaw 的 sandbox 模型不是让模型变聪明，而是把失败的代价压缩成"删一个容器、重建一个 workspace"。提示词负责降低出错概率，沙箱负责限制出错后果——前者不可靠，后者是确定性的。两个都配好，你才敢真的放手让它跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/ca6dff14fbd06326.png)

