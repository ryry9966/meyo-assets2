---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的实践
feedId: 40701
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的 agent 默认带一套固定的系统提示和工具列表。能力一多，上下文很快被塞满：每个能力的说明、参数、注意事项全部常驻 prompt，实际用到的可能不到十分之一。接了十几个 MCP server 之后这个问题更明显——光工具 schema 就能吃掉几千 token。

Skills 机制用**渐进式披露**（progressive disclosure）解决这个问题：系统提示里只注入技能的名称和一句话描述，任务命中时才读取完整说明文件。

## 问题

常见做法只有两种：全量注入（token 浪费，且多个能力的说明互相干扰）或手动切换（人肉路由，不现实）。我们需要的是 agent 根据任务自己判断加载哪个技能，并且加载动作本身足够便宜。

## 做法

**1. 技能就是文件夹。** 在 workspace 的 `skills/` 下建目录，放一个 `SKILL.md`：

```markdown
---
name: video-clip
description: 用 ffmpeg 按时间点截取视频片段并压缩。当用户要求剪辑、截取、压缩视频时使用。
---
## 用法
1. 运行 scripts/cut.sh <输入> <起点> <时长>
2. 输出统一放 output/ 目录
```

**2. frontmatter 是关键。** `name` + `description` 常驻系统提示，正文按需加载。description 写"何时使用"，不是"这是什么"。

**3. 声明依赖。** 元数据里可写 `requires.env`（如 `GITHUB_TOKEN`）、`requires.bin`（如 `ffmpeg`），网关启动时会检测缺失项并在状态里标红。

**4. 细节下沉到脚本。** 技能可以带脚本，agent 通过 exec 工具调用，长参数说明写在脚本注释里。

**5. 验证。** 用 `openclaw skills list` 确认技能被识别；问 agent "你现在有哪些能力"，它应列出摘要而非全文；再给一个匹配任务，观察是否读取 `SKILL.md`。

## 踩坑点

- **description 写成功能清单**：模型匹配的是触发条件，写得不像使用场景就永远不会被触发。
- **反向问题**：描述太宽泛会误触发，不相关任务也把技能读进来，白耗 token。
- **依赖没配**：技能列表显示正常，一跑就报错。先看网关的 requires 检测结果，别靠跑失败来发现。
- **正文写成百科全书**：几千字的 SKILL.md 一次读入，等于把按需加载变回全量加载。正文控制在几百字。
- **同名冲突**：workspace 和 bundled 目录下同名技能存在覆盖关系，排查时先确认实际加载路径。
- **改完不重启**：已有会话可能还在用旧的技能快照，新开会话才生效。

## 可复用建议

- 一个技能只做一类事，宁可拆细，不要做"万能技能"。
- description 套模板：**动作 + 适用场景 + 关键词**，提高模型命中率。
- 技能目录进 git，随 workspace 版本化；通用技能发到 ClawHub 或用 git 仓库共享给团队。
- 高频任务优先做成技能，而不是往系统提示里堆规则——这是最直接的上下文节省手段。

## 总结

Skills 机制的本质，是把"能力说明"从常驻上下文变成可检索资源，代价只是每个技能一条描述的 token。对于接了大量 MCP 工具或跑长会话的场景，这是少数立竿见影的优化点。建议从把现有系统提示里的操作手册拆成两三个技能开始，实测前后 token 消耗差异，再决定铺开节奏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/078fbbd97e091041.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/9e916d9a4b9bed57.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/a8eb38a9889587a2.png)

