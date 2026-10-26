---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42711
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

让 LLM Agent 直接跑 shell 命令，很多人第一反应是：它会不会哪天把我的 home 目录 `rm` 掉。OpenClaw 的定位是常驻本机的个人助理，exec、文件读写都是标配工具，这个问题绕不开。它的答案不是“相信模型”，而是把权限收敛进环境和策略里：模型再怎么幻觉，环境里根本没有能被删的东西。

## 问题

传统两种做法都有缺陷。一是干脆不给 exec 权限，自动化能力直接砍半，批量重命名、跑脚本全做不了；二是给完整主机权限，靠 system prompt 写一句“请小心”——这在长会话、多步推理里基本不设防。模型补全一个错误的绝对路径、一个没加引号的变量，`rm -rf "$DIR/"` 就可能变成灾难。结论很明确：安全必须来自环境约束，而不是模型自觉。

## 做法：OpenClaw 的四层防线

1. **workspace 锚定**。Agent 的工作目录默认是它的 workspace，文件工具的相对路径都在这里解析；对主机路径的访问走 hostsfs 这一层，可以配置路径前缀白名单，没列出来的目录对 agent 不可见。
2. **Docker sandbox**。`sandbox.mode` 支持 off / non-main / all：non-main 只隔离群组和定时任务会话，all 把主会话也放进容器；`scope` 决定容器粒度（按 agent / 按 session / 按 workspace）。容器内以普通用户运行，默认只挂载 workspace，网络可以关。
3. **工具策略与 exec 审批**。`tools.exec.host` 指定命令落在沙箱、gateway 还是本机；`security` 控制基线；safeBins 白名单放行低风险命令，其余触发 `ask`，在聊天渠道里弹人工确认。
4. **提权确认流**。白名单外的操作必须人工点确认，常用安全命令攒进 allowlist 后，日常几乎不被打断。

配置示例（键名以你所用版本文档为准）：

```json
{
  "agents": { "defaults": { "sandbox": {
    "mode": "non-main",
    "scope": "agent"
  }}},
  "tools": { "exec": {
    "host": "sandbox",
    "security": "default",
    "ask": "always"
  }}
}
```

**验证**：开会话让 agent 依次执行 `pwd`、`whoami`、`ls /`，再让它“清空 /home/me/notes”。前三个确认它真的在容器里、路径视角正确；最后一个如果被拒或报错，说明审批链路是通的。这套验证建议每次升级后重跑一遍。

## 踩坑点

- **mode 停在默认 non-main**：群聊有沙箱，主会话直连主机。很多人以为有沙箱，恰恰在私聊时翻车。
- **把整个 `$HOME` 挂进容器**：隔离形同虚设。需要主机文件就单独挂子目录，能只读就只读。
- **容器 root + 可写挂载**：等于没隔离。
- **allowlist 前缀匹配过宽**：`rm` 系列混了进去；白名单按最小集合维护。
- **容器内外路径不一致**：agent 拿到主机路径开始瞎猜，调试时先在会话里确认它视角下的工作目录。
- **镜像缺工具**：agent 反复尝试在线安装，和关网络的策略打架，按需自建镜像。

## 可复用建议

- 起步就用 non-main + scope=agent，不够再收紧到 all。
- 危险命令用 `ask` 而不是一刀切 `deny`，否则排障时你会亲手关掉防线。
- 把“清空 /home”这类越界请求做成固定测试用例，升级后回归。
- 备份 workspace，沙箱是防线之一，不是唯一防线。
- `openclaw.json` 里这段配置进 git，改动可审计。

## 总结

Agent 不误删文件，靠的不是模型听话，而是分层：路径锚定限制**可见范围**，容器限制**影响范围**，策略与审批限制**动作范围**。默认最小权限、显式提权、定期验证——这三件事做到，才敢让 Agent 常驻在你自己的机器上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/7d9dcd523605abc1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/b66c3c7c73bfd9b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/eb6b276c9cfd339d.png)

