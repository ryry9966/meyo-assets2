---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39048
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

Agent 拿到文件和 shell 工具之后，社区里被问得最多的问题就是：“它会不会手滑把我的目录删了？”这个担心完全合理。LLM 的输出是概率性的，路径拼接错一级、通配符写得过宽、把 `rm -rf` 当成清理手段，都是真实发生过的故障模式。OpenClaw 的答案不是在 system prompt 里写一句“请小心操作”，而是把危险操作在架构层面挡住。安全约束放在 prompt 里，等于把防线寄托在模型“今天状态好”上，这在工程上不可接受。

## 做法：四层防线叠加

**1. 工作区隔离。** Agent 的文件工具默认只挂载 workspace root，对应宿主机上一个独立目录；进程层面再套容器/命名空间，路径逃逸基本封死。Agent 视角里的 `/`，其实是宿主机的 `~/.openclaw/workspace`。

**2. 能力分级。** 每个工具（包括 MCP 接入的第三方工具）声明 danger 等级：`read` 免审，`write` 限定在工作区内，`destructive`（删除、覆盖、改权限）默认 deny，必须在配置里显式放开。

**3. 软删除 + 快照。** 所有删除操作统一进 workspace 内的 `.trash` 目录，执行前对受影响路径做快照，出问题可回滚。

**4. 确认门。** `destructive` 操作先产出 plan，包含“受影响路径清单”，再交给 human-in-the-loop 或路径白名单规则审批，通过后才真正执行。

简化后的配置大致长这样：

```yaml
sandbox:
  root: ~/.openclaw/workspace
  network: deny
tools:
  fs.delete:
    level: destructive
    require: confirm     # confirm | allowlist | deny
    allowlist:
      - ${root}/tmp/**
```

## 踩坑点

- **把宿主目录直接挂进 workspace。** 这等于亲手拆掉第一层防线，快照也救不回 sandbox 外的东西。需要共享数据就走只读挂载。
- **allowlist 写得太宽。** 比如直接给 `${root}/**`，confirm 机制形同虚设，和没配一样。
- **子 Agent 继承了父级能力，却没继承审批链。** 危险调用从子 Agent 那里绕过了确认门。spawn 时记得显式收窄能力，而不是默认全量继承。
- **软删除目录本身被 Agent“清理”了。** 这是典型的自我拆台，`.trash` 必须对 Agent 设为不可写。

## 可复用建议

- 权限默认 deny、按需开放，宁可多一次确认，不要少一道闸。
- 危险操作永远先出计划再执行，plan 必须带受影响路径清单，事后可审计。
- 快照要放在 sandbox 外或只读存储里，否则“回滚手段”和“被保护对象”处在同一个信任域，一起翻车。
- MCP 工具不要豁免：第三方工具同样声明 danger 等级，别让它成为绕过审批的后门。

## 总结

OpenClaw 防误删靠的不是模型自觉，而是隔离、分级、软删除、确认门四层叠加：隔离决定“能碰到什么”，分级决定“能做什么”，软删除保证“做了能撤”，确认门保证“危险的事有人点头”。prompt 只是最后一层补充，不是第一道防线。把这套模型套到自己的 Agent 项目里，误删这类事故基本可以从“会不会发生”变成“理论上才可能发生”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/dfe0f305dafa649a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/63e707bda1dd7bb1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/d19ee5a15f89023e.png)

