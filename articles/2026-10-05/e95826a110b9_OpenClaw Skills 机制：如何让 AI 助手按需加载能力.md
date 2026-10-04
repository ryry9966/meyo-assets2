---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 40506
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

把 Agent 从"能聊天"推进到"能干活"，通常分三步：接模型、接工具（MCP / CLI / 浏览器自动化）、然后往 system prompt 里不断塞使用说明。第三步最容易失控。接入的东西一多——十来个 MCP server、二十几个 CLI 子命令——prompt 里全是文档，token 成本线性上涨，模型选错工具的概率反而上升。

OpenClaw 的 Skills 机制就是冲着这个问题来的：能力按需加载。会话启动时，只有每个 Skill 的 name 和一句话 description 进入上下文；正文（SKILL.md 的 Markdown 部分）只在模型判断相关时才注入。本质上是上下文经济学：全量注入是浪费加干扰，完全不注入则模型不知道能力存在，Skills 走的是中间的渐进式披露路线。

## 做法

**1. 最小结构。** 一个 Skill 就是一个目录，核心是 SKILL.md：YAML frontmatter 写 name、description，需要的话加 requires 声明依赖的环境变量或二进制；正文写使用步骤。放进 workspace 的 skills 目录（或 `~/.openclaw/skills`），重启 gateway 或热加载后生效。

**2. description 是路由信号，不是简介。** 模型靠这句话决定要不要加载，写法要像路由规则："当用户需要 X 时使用；Y 场景不要用。"写成"这是一个用于……的工具"，基本等于没写。

**3. 正文克制。** SKILL.md 正文加载后同样占上下文，写长了等于把 prompt 膨胀换了个地方。步骤、命令、边界条件写清楚即可；大段参考资料放独立文件，正文里引用，让模型按需再读。

**4. 验证。** 准备几个"应该触发"和"不应该触发"的测试问法，跑一遍看 debug 日志里 skill 是否被注入。description 的措辞通常要迭代两三轮才稳定。

## 踩坑点

- description 太模糊 → 永远不触发；太宽泛 → 每轮都触发，等于白加载还污染上下文。
- 正文超长，一次加载吃掉省下来的 token，得不偿失。
- 没写 requires：触发了却在运行时才报缺环境变量或缺二进制，排查很绕。
- 改了 SKILL.md 以为立即生效，某些配置下其实要重启 gateway，以日志为准。
- 在 SKILL.md 里放密钥——它会被注入上下文，敏感信息一律走环境变量。

## 可复用建议

- 把 SKILL.md 当代码管：进 git、走 review、留变更记录，避免文档和工具版本漂移。
- 一个 Skill 只做一件事，多用途就拆。
- 每个 Skill 配三五个回归测试问法，改完 description 就跑一遍。
- 定期看触发率：长期不触发的重写或下线，频繁误触发的说明路由词有问题。
- 优先引用脚本和文件，而不是把内容内联进正文。

## 总结

Skills 机制的收益不来自"有这个功能"，而来自 description 的质量和 Skill 划分的纪律。它本质是 prompt 工程与软件工程的交叉活：description 是接口文档，正文是实现，触发行为是测试要覆盖的对象。按这个标准维护，Agent 的上下文会干净很多，工具选择准确率的提升是顺带的收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/f29089f5a7ca3435.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/9b0ac8b71a81a958.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/dcd2bbb2f7ed0ef4.png)

