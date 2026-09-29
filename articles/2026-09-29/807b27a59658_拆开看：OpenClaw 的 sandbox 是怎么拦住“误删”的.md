---
title: 拆开看：OpenClaw 的 sandbox 是怎么拦住“误删”的
feedId: 39599
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 agent 默认带一整套文件工具（read / write / patch / move / remove）和 shell 执行能力，再叠加 MCP server 暴露的第三方工具。能干活的前提是能碰真实文件系统，这也意味着一次幻觉、一次路径拼接错误，就可能变成一场 `rm -rf`。所以在 OpenClaw 里，sandbox 不是可选的加固项，而是默认层。

## 问题

只靠 system prompt 写一句“删除前请小心”是靠不住的——模型输出本质是概率性的，提示词约束不构成边界。OpenClaw 的思路是把安全从“请求模型自律”改成“运行时强制”：路径、工具能力、操作类型分别设闸，模型只能在这些闸门之间做选择，而不是被信任。

## 做法

整体是一个四层漏斗，请求必须逐层穿过：

1. **工作区监狱（jail）**：所有文件工具的路径参数先规范化（resolve symlink、展开 `..`），再检查是否落在 workspace root 之内。逃逸路径直接在工具层拒绝，不会进入模型上下文。
2. **能力分级**：配置里给每个工具标 capability：`read` / `write` / `destructive`。`remove`、`shell`、以及所有 MCP 工具默认按 `destructive` 处理，需要显式声明才可用。
3. **破坏性闸门**：destructive 工具执行前走三步——dry-run 计算影响面（涉及文件数、目录深度、是否在 git repo 外）→ 变更写入 journal → 命中阈值时挂起等人工 confirm。注意 confirm 的粒度是“这一次调用”，不是“本次会话”，防止一次授权后被无限复用。
4. **快照与回滚**：write / move / remove 落盘前，原文件先进带保留窗口的 trash 目录，配合 journal 可按会话回放撤销。

一个最小配置示意（字段名以你版本的 schema 为准）：

```json
{
  "workspace": { "root": "~/projects/demo", "followSymlinks": false },
  "tools": {
    "remove": { "capability": "destructive", "confirm": "always" },
    "shell":  { "capability": "destructive", "pathPolicy": "inherit-jail" }
  }
}
```

## 踩坑点

- **workspace 设成了 home 目录**，jail 形同虚设。范围越小越好，宁可麻烦也别图省事。
- **shell 工具没继承 jail**。文件工具拦得住 `rm`，拦不住 agent 用 shell 起子进程删东西，`pathPolicy: inherit-jail` 必须开，否则干脆禁 shell。
- **symlink 逃逸**。workspace 里一条指向 `~/` 的软链就够绕过路径检查，`followSymlinks: false` 不要关。
- **MCP 工具自带文件语义**。第三方 server 的 "cleanup" 类工具不在 OpenClaw 文件工具白名单里，接入时记得手动标 capability，别默认信任。
- **trash 目录**默认对模型不可见，防止 agent 读到 journal 后“自作聪明”清理现场，回滚就没了。

## 可复用建议

1. 每个会话一个最小 workspace，跑完即弃，不复用根目录。
2. 新插件或 MCP server 接入时，先全部按 `destructive` 跑一周，看 journal 再逐步降级。
3. 定期演练：故意诱导 agent 执行危险指令，确认闸门命中，把结果写进回归测试。
4. audit log 单独存储、和 workspace 不同盘，防“一锅端”。

## 总结

误删不是模型“太笨”，而是把不可靠的输出直接接到了高权限通道上。OpenClaw 的 sandbox 本质是四道串联的闸：路径收敛、能力分级、破坏性确认、可回滚。模型犯错是常态，架构的目标是让错误在到达文件系统之前就被消费掉。花五分钟配好这些闸门，比事后恢复数据一个下午划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ac6b43c5349bb17e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/abadfb14407bf989.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0f59cc547b678d88.png)

