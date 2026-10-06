---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40708
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 是常驻网关型的个人 Agent，模型本体只负责推理，真正“动手”的是工具调用：执行 shell 命令、读写文件、操作浏览器。问题在于，只要模型能执行 `exec`，那么 `rm -rf`、`mv` 覆盖、路径幻觉这些风险就始终存在。这不是“换个更强的模型”能解决的——错误删除是一个概率问题，跑得够久一定会撞上。

## 问题

早期不少人的部署方式是让 Agent 直接在宿主机上执行命令。真实出过的事故类型很集中：工作目录判断错误删掉了同级目录、把 home 下的配置文件当临时文件清理、变量没加引号导致通配符展开到意料之外的位置。这些不是提示词写得不够好，而是架构上没有兜底。

## 做法

OpenClaw 的思路是把“模型犯错”当成默认假设，用 sandbox 限制爆炸半径：

1. **容器隔离**：`exec` 类工具默认在 Docker 容器内执行，文件系统、进程树、网络与宿主机隔离。删文件删的是容器层，宿主机不受影响。
2. **最小挂载**：只把 agent 的 workspace 目录以 bind mount 挂进容器，home、SSH、docker socket 一概不进。scope 可按 agent 或按 session 隔离，避免多会话互相覆盖。
3. **权限收敛**：容器内进程以非 root 用户运行；可限制 egress 网络，装依赖走代理或预构建镜像。
4. **凭据分离**：API key、token 等由 gateway 进程持有，不注入容器环境变量，模型本身拿不到。
5. **宿主机操作走白名单**：确实需要宿主机能力的场景，走确认机制而非直接放开。

配置示意（字段名以你所用版本文档为准）：

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "scope": "agent",
        "workspaceAccess": "rw"
      }
    }
  }
}
```

## 踩坑点

- **挂载了整个 home 目录**：隔离形同虚设，容器里 `rm` 删的就是真实文件。
- **为了图方便挂了 docker.sock**：等于把宿主机 root 交给容器，比不挂 sandbox 还危险。
- **以为 sandbox 等于备份**：workspace 是读写挂载，容器内的删除对挂载卷是真删除。workspace 务必用 git 管理，配合自动 commit 做回滚线。
- **容器内没网，装依赖失败，于是关掉 sandbox**：这是最常见的倒退路径。正确做法是配置 egress 白名单，或把依赖打进镜像。
- **容器内外路径不一致**：软链接、大小写差异导致挂错目录，删错位置。挂载前先在容器里 `ls` 确认。

## 可复用建议

- 最小挂载原则：只挂 workspace，其余一律不进容器。
- 多 agent 场景用 `scope: agent` 独立隔离，别图省事用 shared。
- workspace 初始化时就 `git init`，每天定时 commit——这是最后一条防线。
- 危险命令靠确认网关拦截，不要在提示词里“恳求”模型别删。
- 保留 exec 审计日志，出事后能复盘到具体一次调用。

## 总结

OpenClaw 的 sandbox 模型核心不是“相信模型不会犯错”，而是“犯错时损失被限制在容器和 workspace 内”。分层防线依次是：容器隔离、挂载最小化、凭据分离、git 可回滚。把它当成默认架构而不是可选项，Agent 自动化才能真正放心跑起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/81f060f7f100102c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1d9098959ed0548a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1bde9fb643799131.png)

