---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42343
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 的 Agent 拿到的是"能动手"的工具集：exec、文件读写，还有各类插件和 MCP 工具。第一次跑自动化的人几乎都有同一个担忧：模型抽风时来一句 `rm -rf` 怎么办？

先给结论：OpenClaw 不是靠"模型足够聪明所以不会删错"来兜底的，而是靠一套分层边界。模型一定会犯错，边界的作用是让每次错误的代价足够小。

## 风险到底在哪

实际场景里，误删误写主要来自四类：

1. **路径幻觉**：模型拼出一个不存在的相对路径，落到错误目录；
2. **cwd 漂移**：脚本在容器里的工作目录和宿主机不一致，`rm -rf ./build` 删的不是你以为的那个 build；
3. **命令拼接**：一条看似无害的命令后面接了管道或 `&&`，副作用超出预期；
4. **权限过宽的插件**：第三方工具直接携带 host 级执行能力，绕过你精心配置的策略。

## 分层做法

**第一层：workspace 绑定。** 文件类工具默认锚定在 agent 的 workspace 目录内，出界路径要么被拒绝，要么走提权审批。Agent 的"整个世界"就是那个目录。

**第二层：容器沙箱。** 对需要跑 exec 的 agent，打开 sandbox（`mode: "all"`），每个 agent 独立 scope。命令在一次性容器里执行，只挂载 workspace，非 root 运行，网络按需收紧。就算模型真的执行了 `rm -rf /`，删的也只是容器层。

**第三层：工具策略。** exec 支持 allow/deny 规则，`rm -rf`、`dd`、`mkfs` 这类高危模式直接拦截或转人工确认；提权执行默认要求确认。

**第四层：可回滚。** workspace 用 git 仓库化，配合定时快照。真出事，恢复成本是分钟级，而不是"重做一周"。

一个最小配置骨架（字段名以你所用版本的 docs 为准）：

```json
{
  "agents": {
    "defaults": {
      "sandbox": { "mode": "all", "scope": "agent" },
      "workspace": "~/openclaw-workspace"
    }
  }
}
```

## 踩坑点

- **把 `$HOME` 挂进沙箱**：等于没隔离。只挂 workspace，需要什么数据显式挂什么。
- **`mode: "non-main"` 常态化**：主 agent 裸奔跑任务，演示可以，生产不行。
- **审批疲劳**：确认弹窗一多就无脑点同意，人审这道门形同虚设。宁可收紧 deny 规则，减少弹窗数量。
- **容器 uid 映射**：容器内以 root 写出的文件，回到宿主机属主变成 root，后续 git 操作报错。启动时固定运行用户。
- **插件自带工具**：接入新 MCP/插件后，先看它暴露了哪些 tools，别默认它们遵守你的 exec 策略。

## 可复用建议

1. 新 agent 默认"最小权限三件套"：sandbox all + agent 独立 scope + workspace git 化。
2. deny 列表按团队自己的历史事故来维护，不要照抄别人的模板。
3. 定期回放 exec 日志，看 agent 实际执行了什么，比事后追责有用得多。
4. 高危操作（批量删除、批量写入、对外请求）一律走人工确认，不要为效率开例外。

## 总结

"Agent 不会误删文件"的准确表述是：**即使它试图误删，也会被 workspace 边界拦住、被容器限制住、被 deny 规则挡下、被快照兜底**。这套安全模型的价值不在于让模型变可靠，而在于让单点失败变得便宜。四层配齐之后，你才能真正放心地把自动化任务交给它。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/8435e1911eecf7f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/d6e2619ec3cbfceb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/ef2dcb805dd8628f.png)

