---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40531
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

Agent 用久了都会遇到同一个问题：能力越多，上下文越脏。工具说明、调用示例、团队约定全部塞进 system prompt，对话还没开始几千 token 就没了。MCP 接入的 server 一多更明显——十几个服务、上百个 tool 描述，模型选错工具的概率肉眼可见地上升。

OpenClaw 的 Skills 机制就是针对这个问题设计的：把"能力"从常驻上下文里拆出来，按需加载。

## 机制怎么运作

一个 Skill 就是一个目录，核心文件是 SKILL.md：

```
my-skill/
├── SKILL.md          # 元数据 + 使用说明
├── scripts/
│   └── convert.py    # 可执行脚本
└── references/
    └── api-notes.md  # 补充资料
```

SKILL.md 头部是 YAML frontmatter，`name` 和 `description` 两个字段最关键。加载分三层：

1. **元数据层（常驻）**：会话启动时只注入所有 skill 的 name + description，每个几十 token；
2. **正文层（触发加载）**：模型判断某个 skill 与当前任务相关时，才读取完整 SKILL.md；
3. **资源层（引用加载）**：SKILL.md 里提到的脚本和文档，用到才读；脚本甚至可以只执行、不读入。

这就是渐进式披露（progressive disclosure）。上下文占用从"全量常驻"变成"按路径展开"，能力数量和上下文成本就此解耦。

## 实际操作步骤

1. 建目录、写 SKILL.md，description 用"什么时候用我"的句式，写清触发条件和关键词；
2. 稳定的多步流程写成 scripts/ 下的脚本，让模型执行而不是逐步复述；
3. 长文档拆到 references/，正文里只留路径和阅读时机；
4. 重启会话，用一个触发测试集（5~10 条该触发/不该触发的指令）验证行为。

## 踩坑点

- **description 太泛或太含糊**。太泛会永远被触发，太含糊则永远不触发。写完拿真实任务跑一轮，对着加载日志改，比凭感觉调快得多；
- **SKILL.md 越写越长**。正文超过几百行，等于把 prompt 膨胀问题换个地方复现。我的经验值是单文件控制在 500 行以内，超了就往 references/ 拆；
- **脚本路径假设错误**。脚本默认 CWD、硬编码绝对路径是最常见的翻车点，脚本内部一律用相对 skill 根目录的路径；
- **skill 与 MCP 工具职责重叠**。同一能力既有 skill 又有 MCP tool，模型会随机二选一，行为不可复现。建议定一条边界：有状态、需鉴权的走 MCP，纯流程知识走 skill；
- **skill 数量本身失控**。元数据虽小，两百个描述加起来也不少，还会稀释注意力，定期归档不用的。

## 可复用建议

- 把 skill 当代码管：进 git、走 review、带版本，每次改动有 diff 可查；
- description 是这套机制里性价比最高的几十个 token，值得反复打磨；
- 通用流程沉淀成 skill，团队私有约定也做成 skill，新人上手成本明显下降；
- 排障顺序：先确认三层加载是否发生——没触发查 description，触发了出错查脚本和路径。

## 总结

Skills 机制的本质不是新功能，而是一种上下文纪律：启动时只承诺"我有什么"，用到时才展示"怎么做"。对维护多 agent、多工具链的团队来说，它把扩展能力从"改 prompt"变成"加目录"，可测试、可版本化，是少数工程收益能立刻体现在 token 账单和稳定性上的实践。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/a954b9a615a9ffdc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/e03b4e37e312a8a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/2d12023a280a22fd.png)

