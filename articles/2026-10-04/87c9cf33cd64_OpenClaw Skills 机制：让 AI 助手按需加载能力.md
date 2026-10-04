---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 40415
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景：上下文先被撑爆

跑 Agent 一段时间后，最先膨胀的往往不是代码，是上下文。MCP 挂了七八个 server，工具列表常驻 system prompt；再叠加各种“遇到 XX 情况应该 XX”的规则，一个会话还没干正事，配额先吃掉一大截。代价是三重的：token 成本变高、指令遵循变差、模型在几十个工具里挑错。

OpenClaw 的 Skills 机制针对的就是这个问题：把“低频但专业”的能力从常驻上下文里拆出去，放到磁盘上，模型判断需要时才加载。

## 解法：元数据常驻，正文按需

一个 Skill 就是一个文件夹，最低要求只有一个 `SKILL.md`：

```text
skills/log-triage/
├── SKILL.md
├── scripts/scan.sh
└── references/patterns.md
```

```markdown
---
name: log-triage
description: 当用户要求分析服务日志、定位报错原因或统计错误频率时使用
---

1. 先运行 scripts/scan.sh 获取错误摘要，不要通读原始日志
2. 命中 references/patterns.md 中的已知模式时，按对应预案处理
3. 输出结论前附上关键日志行原文
```

会话启动时，OpenClaw 只扫描 frontmatter，把 `name` 和 `description` 注入系统提示——每个技能几十个 token。正文的操作步骤、参考文档、脚本都不进上下文。只有当模型判断任务命中某个 description，才去读完整的 SKILL.md；脚本更进一步，直接在宿主机执行，只有输出结果进上下文，脚本本身永远不进。

本质上是把上下文当缓存用：元数据常驻，正文按需分页。

## 上手：写一个能被正确触发的 Skill

1. **description 是唯一的路由键。** 别写“日志分析助手”，要写触发条件：“当用户要求分析服务日志、定位报错、统计错误频率时使用”。模型只凭这句话决定是否加载，含糊的描述等于没写。
2. **正文写操作，不写知识。** SKILL.md 只放步骤、约束、命令，控制在几十行；参考资料拆到 references/，标注“仅在需要时读取”。
3. **确定性逻辑脚本化。** 扫描、统计、格式转换这类能脚本跑的，绝不让模型现场推理。模型只消费脚本输出，省 token 且稳定。
4. **验证触发。** 起新会话，先问一个无关问题确认技能没加载，再问一个命中场景的问题，观察它是否读取 SKILL.md。不验证的技能等于没上线。

## 踩坑点

- **描述重叠。** 两个技能的 description 都写“报错处理”，模型会犹豫或两个都加载。定期 grep 所有 description，保证触发边界互斥。
- **把 Skill 当 prompt 片段用。** 每次会话都必须生效的规则（安全约束、输出格式）应放常驻提示，不该埋进按需加载的技能里。
- **脚本依赖。** 脚本在宿主机执行，Python 版本、第三方包、相对路径都要部署时确认；建议脚本内部按技能根目录解析路径。
- **正文越写越长。** 技能会随时间膨胀，超过一屏就该继续往 references/ 拆。

## 可复用的分流原则

一个简单的判断标准：**高频且必须生效的 → 常驻提示；低频、步骤长、有脚本的 → Skill；需要和外部系统交互的 → MCP 工具。** 三者互补而非替代。另外把 skills/ 目录进 git，技能的演进历史和代码一样值得追溯。

## 总结

Skills 本质上是给 Agent 上下文加了一层懒加载：用几十个 token 的元数据换掉几千 token 的常驻指令，换来更干净的上下文和更稳的指令遵循。没有黑魔法，难点全在描述怎么写、正文怎么拆。把这套分流习惯建立起来，Agent 的能力数量增长时上下文占用依然克制——这才是它和“往 system prompt 里无限堆规则”真正拉开差距的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/9d969418a28abb18.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/ae67067126ca7a0b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/abded338ae14f0f7.png)

