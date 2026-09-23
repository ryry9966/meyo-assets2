---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 38638
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

让 Agent 拿到 shell 和文件工具之后，社区里被问得最多的问题就是：它会不会哪天手滑，把我整个目录删了。这个担心是合理的——模型在长上下文里确实会拼错路径、误解析相对目录，而 `rm -rf` 没有后悔药。OpenClaw 的思路不是指望模型永远不犯错，而是把"犯错能造成的破坏"压缩到可接受范围。

## 问题：误删的本质是执行面太大

单看一次删除操作，风险来自三层叠加：

1. **模型层**：路径解析错误、通配符展开范围比预期大；
2. **工具层**：exec 工具默认能力过宽，等于把宿主机 shell 交给一个概率系统；
3. **权限层**：进程以什么用户运行、能看到哪些路径。

三层全放开，误删只是时间问题。

## 做法：四层防线

**1. 文件系统隔离。** 启用 Docker sandbox 后，每个 agent 会话跑在独立容器里，宿主机文件系统对它不可见，只有 workspace 以 bind mount 进容器。容器内以非 root 用户运行，越出 workspace 的写操作天然失败。

**2. 工具路径收口。** 内置文件工具（read/write/edit/remove）统一走路径解析器：先 resolve，再校验是否落在 workspace 根内，越界直接拒绝。对 workspace 内的 symlink 保持警惕——不过即便符号链接指向容器的 `/etc`，也拿不到宿主机的 `/etc`，因为宿主文件根本没挂进来。

**3. 命令层拦截。** exec 类命令按策略门控：`rm -rf`、`find -delete`、`dd`、对挂载点外的写操作等模式默认进 approval 队列或直接 deny。方向是 deny-by-default，不是 allow-by-default 加黑名单。

**4. 作用域划分。** sandbox 按 agent 隔离，A 的 workspace 对 B 不可见，自动化任务的误伤面进一步收窄。

配置大致长这样（字段名以你的版本为准）：

```json
{
  "agents": { "defaults": { "sandbox": { "mode": "all" } } },
  "tools": { "exec": { "approval": "dangerous-only" } }
}
```

## 踩坑点

- **把整个 home 挂进 sandbox 图方便**，等于亲手拆掉第一层防线，这是最常见的自废武功。
- **sandbox ≠ 备份**。workspace 是 rw 挂载，容器内的删除对宿主机同样生效，只是爆炸半径被限定在 workspace 内。
- **挂载 docker socket 进容器**约等于交出宿主机控制权，绝对不要。
- **approval 疲劳**：弹三次确认之后就无脑放行，门控形同虚设。宁可把白名单配细，也不要训练自己秒按。
- 有人图快关掉 sandbox，建议只在一次性、无敏感数据的任务里这么做，用完即恢复。

## 可复用建议

- workspace 纳入 git 管理，删除可回滚，这是性价比最高的兜底；
- 只需要读的目录（参考文档、代码库）用 ro 挂载；
- 不需要联网的任务收紧网络出口，降低"删完再掩盖"的操作空间；
- 定期翻 exec 审计日志，看 agent 实际执行了什么，比事后追责有用得多。

## 总结

OpenClaw 不承诺模型零失误，而是让失误的代价可控：看不到的删不了，看得见的删了也能回滚。这套安全模型的价值不在于某一层有多强，而在于四层叠加之后，单点失效不再等于事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/0b37f6f16b2fe483.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/304dcf7f3ad826e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/498f6fd0f9586ef7.png)

