---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 42002
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

Agent 扩展能力最朴素的办法，是把所有操作说明塞进 system prompt。技能数量一上两位数，问题立刻暴露：上下文预算被静态说明吃掉大半，模型在高密度指令里也容易张冠李戴。OpenClaw 的 Skills 机制走的是渐进式披露（progressive disclosure）路线——平时只暴露目录，用到哪本翻哪本。

## 问题：能力说明与上下文预算的矛盾

静态注入的代价是三重的：token 成本恒定，不管这次对话用不用得上；指令密度越高，单条指令的遵循率越低；每次更新说明都要动 system prompt，改坏了影响全局。

## 机制拆解与做法

一个 Skill 就是一个文件夹，核心是 SKILL.md：

```text
my-skill/
├── SKILL.md        # frontmatter + 正文
├── scripts/
│   └── deploy.sh
└── references/
    └── api-notes.md
```

加载分三层：

1. **会话启动**：只有 frontmatter 里的 `name` 和 `description` 被注入上下文，单个 skill 大约几十 token；
2. **按需展开**：模型判断当前任务与某条 description 相关时，主动读取 SKILL.md 全文；
3. **资源懒加载**：正文引用的脚本和参考文档，只在真正执行时才被使用。

落地步骤：

1. 在 `~/.openclaw/skills/` 下建目录（项目级可放 workspace 的 skills 目录）；
2. 写 SKILL.md，frontmatter 至少包含 name 和 description；
3. description 写"什么时候用"，不写"这是什么"。比如"当用户要求部署到测试环境并需要回滚方案时使用"，而不是"一个部署工具"；
4. 正文控制在可快速通读的篇幅，细节下沉到 references/ 附属文件；
5. 重载会话后用一句自然语言验证触发，看日志确认是"读了全文"，而不是模型凭印象猜。

## 踩坑点

- **description 写太宽**：写成"处理文件相关任务"，几乎所有对话都会误触发。按触发条件写，宁窄勿宽。
- **SKILL.md 写成论文**：全文几千字，每次触发一次性吃掉大块上下文，等于把静态注入换了个地方。正文只放"做什么、按什么顺序"，"为什么、细节"全部下沉。
- **脚本环境没声明**：skill 脚本跑在宿主机，Python 版本、依赖包不写清楚，第一次执行就翻车。正文开头写死运行前提。
- **幂等性**：agent 有重试倾向，非幂等脚本（如重复创建资源）要么加存在性检查，要么先 dry-run。
- **权限边界**：脚本以 agent 权限执行，危险操作应在正文里明确要求先向用户确认，别指望模型自觉。

## 可复用建议

- 建一个 SKILL.md 模板仓库，frontmatter 字段、目录结构统一，新人照抄就能写；
- 高频、步骤固定的操作固化为脚本，skill 正文只负责"何时调用哪个脚本"；
- 把 skills 目录纳入 git，description 每次修改都留 diff，触发率异常时方便回溯；
- 定期清理：一个季度没被触发过的 skill，要么 description 有问题，要么它不该存在。

## 总结

Skills 机制的本质，是把"能力说明书"从上下文的常驻开销，变成按需检索的资料库。它不增加模型能力，但能显著提升上下文利用率和大技能集下的触发准确率。写 skill 的功夫，七分在 description，三分在正文的克制篇幅。先把你 system prompt 里最占地方的那段说明拆出来试一次，效果好不好，日志会告诉你。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/7bfff5356e885994.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/8f09809f09367c1c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/f2a69d53b4e3f41a.png)

