---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是全量挂载
feedId: 42659
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

过去几个月我把一个 OpenClaw agent 从“全能”改成了“按需”。起因很简单：接入的 MCP server 和自定义工具越挂越多，光工具 schema 就占了 1.2 万 token，模型选错工具的次数肉眼可见地上升。Skills 机制就是为这个问题准备的：把能力打包成独立的技能单元，平时只在上下文里留一行元数据，命中时才加载正文和脚本。

## 问题

全量挂载工具有三个代价：

1. **上下文预算**。每个 MCP 工具的参数描述都要常驻 system prompt，几十个工具轻松吃掉上万 token，留给任务推理的空间被压缩。
2. **选择准确率**。可选项越多，模型在相似工具之间犹豫的概率越高。我统计过一次，工具超过 40 个之后，误选率大约翻了一倍。
3. **维护成本**。提示词片段、脚本、说明文档散落各处，换模型或迁移部署就要重调一遍。

## 做法

skills 目录约定很简单，每个技能一个文件夹：

```
skills/
  pdf-extract/
    SKILL.md        # frontmatter + 正文
    scripts/
      extract.py
  cron-report/
    SKILL.md
```

关键在加载策略，分三层：

**第一层：元数据常驻。** agent 启动时只读每个 SKILL.md 的 frontmatter（name + description），几十个技能也就几百 token。description 要写成路由规则，包含触发词和排除项：

```yaml
---
name: pdf-extract
description: 从 PDF 提取表格和文本，适用于发票、年报、合同。扫描件请改用 ocr-skill。
---
```

**第二层：正文按需注入。** 模型判断任务匹配后，用 read 工具加载 SKILL.md 正文——操作步骤、边界条件、脚本调用方式。没命中就永远不进上下文。

**第三层：确定性部分下沉到脚本。** 提取、转换、格式化这类不需要“智能”的步骤写成脚本，SKILL.md 只讲清楚“什么时候调、传什么参数”。模型负责路由和异常处理，不做逐步推理。

## 踩坑点

- **description 太泛**。我最初写的是“处理文档类任务”，结果什么任务都往里撞。改成列触发词 + 排除项之后才稳定。
- **正文一次全量注入**。有人把 3000 字的 SOP 全塞进 SKILL.md，等于没做懒加载。长文档应拆成正文 + 附录文件，用到哪步读哪步。
- **环境依赖不声明**。脚本依赖 Python 3.10 和外网访问，没写进 SKILL.md，换台机器就报错，模型还得兜圈子排查。
- **隐式互相引用**。skill A 正文里写“参考 skill B”，删了 B 之后 A 半瘫且不报错。要么复制必要内容，要么在 frontmatter 里显式声明依赖。

## 可复用建议

1. 从**高频、流程固定**的任务开始做 skill，别一上来包装所有能力。
2. 把 description 当路由规则维护：写触发词、写排除项，定期按触发日志修订。
3. 能脚本化的绝不留给模型自由发挥——脚本是确定性的，模型推理不是。
4. 每个技能单一职责，宁可多建几个小的，不要做成大杂烩。
5. 留一份触发日志（哪个任务命中了哪个 skill），这是调优 description 最直接的依据。

## 总结

Skills 机制本质是一笔“上下文经济学”交易：用一层间接寻址（元数据 → 按需加载正文），换回上下文预算、工具选择准确率和可维护性。它不神秘，就是把插件系统里“注册 + 懒加载”的老思路搬到了 agent 的提示词层。我的体感数据：迁移后常驻上下文从 1.2 万 token 降到 900 左右，误选工具的情况基本消失。如果你的 agent 也在被工具列表撑爆，值得花一个下午试试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/4f126f914daada1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/f61c93f2baee7474.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/139c28a891c6726d.png)

