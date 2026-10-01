---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40008
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的 agent 默认带 exec 工具，能直接跑 shell。这意味着模型某次生成 `rm -rf` 的概率并不为零——这不是能力问题，是概率问题。社区里最常被问的一句是：让 agent 自动整理目录，真不怕它把东西删了？

## 问题

风险来自三层叠加：

1. LLM 会幻觉路径：把 `~/work` 和容器里的 `/workspace` 混为一谈；
2. 工具调用没有天然权限边界：exec 继承 gateway 进程的用户身份；
3. 自动化场景无人值守：误操作发生时，没人去按那个"拒绝"。

单靠 prompt 约束（"请不要删文件"）不构成安全边界，这点社区已有共识。

## 做法：分层，不赌模型

OpenClaw sandbox 的核心思路，是让"误删"在物理上不可达，而不是指望模型自觉：

1. **文件系统边界**。sandbox 开启后，exec 在 Docker 容器内执行，宿主机文件系统对容器不可见，只有 workspace 以 bind mount 进入。agent 就算执行 `rm -rf /`，删的也只是容器层。
2. **模式分级**。sandbox mode 支持 off / non-main / main / all，默认 non-main：子 agent 全部进沙箱，主 agent 留在宿主机保留灵活性。做无人值守自动化，建议直接 all。
3. **最小挂载**。容器里默认没有你的 `.ssh`、`.aws`、浏览器配置。确实要给 agent 的数据，单独挂，尽量 read-only。
4. **工具门禁**。配合 tool policy 和 exec 审批，把高危命令模式挡在执行之前，而不是事后翻日志。

最小配置示例（字段名以你所用版本文档为准）：

```json5
{
  sandbox: { mode: "all" },
  agents: {
    defaults: { workspace: "/home/me/openclaw-workspace" }
  }
}
```

## 踩坑点

- **误以为所有 agent 都在沙箱里**。默认是 non-main，主 agent 的 exec 直接落在宿主机。配置完先让它 `ls /` 报告看到了什么，再决定信不信。
- **图省事把整个 home 挂进容器**。等价于没沙箱，还附赠把密钥喂给被注入页面的风险。
- **给容器挂 docker.sock**。等于把宿主机 root 交出去，任何一次注入都能兑现。
- **嫌 build 慢于是关沙箱**。正确姿势是用 setupScript 固化镜像层，而不是拆墙。
- **验证方式不对**。在 workspace 外放一个 canary 文件，让 agent "清理目录"，看它够不够得着——比盯着配置看可信得多。

## 可复用建议

- 默认拒绝：能不挂载就不挂载，能 read-only 就 read-only。
- 每个 agent 独立 workspace，配合 git，即使误删也有回滚。
- gateway 用专用低权限账号跑，别用日常账号。
- 想清楚 sandbox 的定位：它控制的是爆炸半径，不是对抗已拿到 root 的攻击者。

## 总结

Agent 不删你的文件，不是因为模型足够聪明，而是宿主机的文件系统根本不在它的视野里。OpenClaw 的 sandbox 把"信任模型"变成了"信任边界"：容器是墙，挂载是门，审批是门锁。墙修好了，prompt 才轮到谈礼仪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/efc05c04c53b21b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/2f4c7e78ecf157ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/d2cfd43220680149.png)

