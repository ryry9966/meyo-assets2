---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41728
source: 综合讨论
publishedAt: 2026-10-16
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景

做 Agent 常见的一步错棋：把所有能力说明全部塞进系统提示词。三五个工具时没问题，攒到二三十个，每次会话开头就固定背上几万 token——大部分能力一整周用不上一次，却始终在线。OpenClaw 的 Skills 机制把这个问题变成了文件组织问题：能力写成独立文件夹，按需加载。

## 机制拆解

一个技能就是一个目录：

```
my-skill/
├── SKILL.md        # frontmatter(name, description) + 正文指令
├── scripts/        # 可选，可执行脚本
└── references/     # 可选，详细文档
```

核心是渐进式加载，分三层：

1. 会话启动时，只注入所有技能的 name 和 description，每个几十 token，几十个技能开销也可控；
2. 模型判断当前任务匹配某个 description，主动 read 对应的 SKILL.md 拿到完整指令；
3. SKILL.md 里引用的脚本和文档，只在真正执行那一步才被读取。

也就是说，上下文里常驻的只是"目录页"，正文和附件全部懒加载。

## 上手步骤

1. 在技能目录（`~/.openclaw/skills` 或 workspace 下的 skills 文件夹）新建子目录；
2. 写 SKILL.md：frontmatter 填 name 和 description，正文写操作步骤、约束和边界情况；
3. 有确定性流程就丢进 scripts/，让技能调用脚本，而不是用自然语言复述算法；
4. 重启会话（或触发重新扫描），用真实任务问一遍，观察它是否读了这个技能、读的时机对不对。

## 踩坑点

- **description 写成功能简介而不是触发条件。**"用来处理文档"召不回，"当用户要求把网页转成 PDF 并发送邮件时使用"才召回。把它当检索 query 写，塞进用户真实会说的话。
- **description 太宽泛。**一个叫"处理文件"的技能会抢走本该属于其他技能的任务。
- **职责重叠的技能并存**，模型的选择接近抽签，先合并再拆分。
- **关键约束只在聊天里说过一次**，没写进 SKILL.md——下次会话等于没说。
- **脚本依赖没装、路径写死**，技能在被调用的那一刻才暴露问题。
- **SKILL.md 本身写了几千行**，加载一次就把省下来的上下文吃回去了。

## 可复用建议

- 一个技能只做一件事，内部再按 references/ 分层；正文控制在几百行内，长细节外链；
- 确定性步骤一律脚本化，自然语言只负责"何时用、注意什么"；
- 不想从零写，可以先从 ClawHub 拉几个现成技能读源码，看别人怎么组织 description，再魔改成自己的；
- 每次改动后用真实对话回归一遍，盯日志确认"召回 → 读取 → 执行"链路完整，别只看"看起来能跑"。

## 总结

Skills 的本质是上下文的懒加载：把"提示词工程"降维成"文件怎么摆"。写得好的技能库，上下文占用低、触发准、可版本管理、可分享；写得差的，不过是把 prompt 拆成了很多个文件。差别就在 description 的写法和分层粒度上——这两处值得反复迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/73e2880677ea74e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/e4920011c9ca79e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/f492a8cee89e318c.png)

