---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 38618
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

跑 agent 时间长了都会遇到同一个问题：能力越堆越多，system prompt 越来越长。把所有工具说明、操作流程全量塞进上下文，代价是三重的——token 费用上涨、模型注意力被稀释、不该触发的流程反而被误触发。OpenClaw 的 Skills 机制针对的就是这个矛盾：采用渐进式披露（progressive disclosure）的思路，启动时只注入每个技能的名称和一句话描述，正文等到模型判断"用得上"时才读取。

## 问题

具体一点：假设你给助手配了工单分诊、日报生成、服务器巡检、天气查询等十几个流程。全量注入时，每次对话都背着几千字操作手册，模型经常在"该查工单"和"该巡检"之间选错；而如果干脆不注入，模型根本不知道自己有这些能力。这是一个典型的检索问题，不该靠堆上下文硬扛。

## 做法

Skills 在 OpenClaw 里就是一个文件夹约定。在 workspace 下建：

```
skills/
  incident-triage/
    SKILL.md
    scripts/triage.sh
    references/runbook.md
```

`SKILL.md` 分两部分。frontmatter 是给元数据解析器看的：

```yaml
---
name: incident-triage
description: 做告警的初步分诊；用户提到告警、故障、线上问题时使用
metadata:
  requires:
    bins: [kubectl]
---
```

正文是给模型看的，建议固定四段：何时使用、何时不用、输入输出、步骤与失败处理。

关键在于加载顺序：会话启动时，gateway 只把 name + description 拼进上下文；模型判断相关后，通过 skills 工具读取 `SKILL.md` 拿到完整正文；`references/` 下的长文档再按需读。三层结构，逐层加载。

落地步骤：

1. 挑一个高频、流程稳定的任务做成第一个 skill，别一上来就搬全部；
2. description 写触发条件，不写功能介绍（下面细说）；
3. 确定性操作落成 `scripts/` 里的脚本，`SKILL.md` 只写调用方式；
4. 重启 gateway，开新会话让 agent 列出可用技能，确认识别；
5. 跑三组测试：无关请求不触发、相关请求必触发、触发后一次做对。

## 踩坑点

- **description 写成产品文案**。"强大的工单分析工具"这种描述，模型无法判断何时该用。要写成"用户提到告警、故障、服务不可用时使用"——这段话是模型做加载决策的唯一依据。
- **正文太长**。`SKILL.md` 写到几千字，加载一次就把按需加载省下的 token 吐回去了。正文几百字封顶，细节进 `references/`。
- **requires 校验的是宿主环境**。bins 检查的是 PATH；如果实际执行在容器或远程机里，技能会被判定不可用而隐藏，排查时先看执行环境。
- **改了不生效**。元数据在会话启动时注入，改完 `SKILL.md` 必须开新会话验证，别在旧会话里反复调。
- **技能互相抢触发**。两个技能描述高度重叠时，模型的选择接近随机。主动错开措辞，或在正文里写清"何时不用本技能"。

## 可复用建议

把 skill 当"操作手册"而不是"功能插件"：一个 skill 对应一类可复现的任务，而不是一个 API 封装。description 用固定模板——"做 X；当用户提到 Y 时使用"。skills 目录进 git，改动可回滚、可分享给团队复用。能用脚本表达的就别用自然语言步骤：模型的自由发挥空间越小，输出越稳定。

## 总结

Skills 的本质，是把 prompt 工程从"维护一个越来越大的 system prompt"变成"维护一组按文件组织、按需检索的操作手册"。省 token 只是副产品，真正的收益是能力可测试、可版本化、可独立演进。建议从今天最常重复口头交代给助手的那个流程开始，把它做成你的第一个 skill。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/4d4309c6fcd6fa83.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/172430647ccec67e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/8766c7d2395972a4.png)

