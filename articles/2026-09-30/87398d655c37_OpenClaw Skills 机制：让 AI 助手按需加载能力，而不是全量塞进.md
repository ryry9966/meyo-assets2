---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是全量塞进 Prompt
feedId: 39662
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 的 agent 形态是"一个大脑 + 若干外部能力"。早期常见的做法是把所有操作指引、命令模板、流程说明全堆进 system prompt 或全局记忆，结果上下文动辄上万 token。Skills 机制的核心是**渐进式披露（progressive disclosure）**：会话启动时只注入每个技能的 `name` + `description`（一个技能几十 token），模型判断当前任务相关时，才去读取完整的 `SKILL.md` 正文和附带脚本。

## 问题

全量注入有三类具体痛点：

1. **成本与干扰**：无关指引长期占用上下文，稀释模型注意力，反而容易跑偏；
2. **知识与工具耦合**：MCP 适合暴露工具接口，但"什么时候用、怎么用"的流程知识塞进 MCP 既别扭又费 token；
3. **环境差异**：不同机器装的工具不同，写死的 prompt 到新环境一半是空话。

## 做法

Skill 就是一个文件夹。最小结构：

```text
skills/
  video-clip/
    SKILL.md
    scripts/
      cut.sh
```

`SKILL.md` 前置元数据：

```yaml
---
name: video-clip
description: 用 ffmpeg 截取并压制视频片段。当用户提到"剪一段"、"截取片段"时使用。
metadata:
  requires:
    bins: [ffmpeg]
---
```

正文写流程：参数约定、常用命令模板、失败兜底。三个关键点：

- **description 决定触发**。模型只凭它决定是否展开整个技能，写"何时用"比写"是什么"更重要；
- **requires 做环境门控**。目标机器没有 ffmpeg，该技能直接从候选列表剪掉，不会出现"念了半步发现工具不存在"的尴尬；
- 改动后需要重启会话（或重载技能列表）才生效，当前会话已注入的内容不会自动刷新。

验证方式：开新会话问一个贴近场景的问题，观察是否引用技能里的命令，或直接查看启动日志中注入的技能清单。

## 踩坑点

- **description 太泛**（如"帮助用户处理各种任务"）→ 永远被加载，等于没做按需；太窄 → 该触发时不触发。用"用户说什么样的话时用"来措辞。
- **把秘密写进 SKILL.md**：技能内容会进入上下文，API key、密码绝不能放，用环境变量引用。
- **正文越写越长**：`SKILL.md` 本体控制在几百行内，长参考资料拆成 references 文件，正文里只写"何时读哪个文件"。
- 脚本忘记 `chmod +x`，模型调用时直接 permission denied。
- workspace 目录与全局目录存在同名技能时会冲突或覆盖，注意优先级。

## 可复用建议

- 把高频重复的"口述流程"沉淀成技能：部署检查、日志排查、某类格式转换，都是好候选；
- 粒度对齐**一次任务**而不是**一个领域**——领域太大会退化成另一个全量 prompt；
- 与 MCP 分工：MCP 管工具接口，Skills 管使用策略与流程知识，是叠加关系不是替代关系；
- 团队共用时把 skills 目录进 git，review 技能文件比 review 口头约定靠谱得多。

## 总结

Skills 本质上是把 prompt 工程文件化、按需化：用一两行 description 换取"任务相关时才展开"的完整知识，再用环境元数据把纸上能力和真实环境对齐。写好一个技能大约十几分钟，换来的是更干净的上下文和更可控的行为——对长期运行的 agent 来说，这笔账很划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/133dd3186d21e9c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/4a280c2136d9a134.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/2aee4fadcd200efd.png)

