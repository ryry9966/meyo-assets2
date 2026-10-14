---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是把说明书全塞进上下文
feedId: 41526
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

用 OpenClaw 一段时间后，大部分人都会遇到同一个膨胀问题：接了聊天渠道，挂了几个 MCP server，又往 AGENTS.md 里堆各种操作说明——浏览器自动化、转码、报表流程、Home Assistant 控制。结果每次会话上下文都很大，模型对长尾指令的遵循越来越差，token 账单也难看。

Skills 机制就是针对这个问题的：把一项能力打包成一个目录，运行时只把「目录名 + 一句话描述」放进上下文，模型判断当前任务匹配后，才去读完整说明。本质是渐进式披露（progressive disclosure）。

## 问题

全量注入的方式有三个具体痛点：

1. **不管需不需要都占上下文**：本轮任务是转码，报表流程的十几条说明也照样在场。
2. **指令干扰**：说明越多，单条指令权重越低，模型开始选择性「忘事」。
3. **难维护**：能力混在一个大 prompt 里，改一处怕牵连全身。

## 做法

接入一个自定义 skill 只需要三步：

1. **建目录**：workspace 下 `./skills/<skill-name>/SKILL.md`（个人全局能力放 `~/.openclaw/skills/`）。
2. **写 frontmatter**，重点打磨 description：

```markdown
---
name: ffmpeg-batch-transcode
description: Batch transcode or compress video/audio with ffmpeg.
  Use when the user asks to compress, convert, or extract audio
  from media files.
---
（正文：步骤、参数约束、输出约定）
```

3. **正文只写干活的动作**：步骤、限制、示例命令。长参考资料放 `references/`，脚本放 `scripts/`，让模型按需再读，不要一次全塞。

然后新开会话测试：用 `/skills` 确认 skill 被发现；发一句触发性需求（「帮我把这几个 mp4 压一下」），观察它是否命中并加载。稳定后进 git，多机和团队直接复用。

## 踩坑点

- **description 写成标题式**（「视频转码工具」）而不是触发式。它是路由用的 prompt，要写清「什么时候用」，带具体关键词，这一行决定了 skill 会不会被选中。
- **name 和目录名不一致**，skill 会被直接跳过。
- **把所有细节塞进 SKILL.md 正文**：加载时照样撑大上下文，长内容务必拆到 `references/`。
- **两个 skill 描述高度重叠**，模型会选错。在 description 里做显式区分，比如按输入类型或场景划界。
- **别把 token、密钥写进 SKILL.md**，它是工作区里的明文文件。
- **改完不生效**：skill 在新会话才会重新扫描，别在旧会话里反复试。
- skill 里的脚本记得给执行权限，正文写明工作目录假设，否则模型经常在错误路径下执行。

## 可复用建议

- **一个 skill 只做一件事**。description 里出现「并且」，就该考虑拆成两个。
- 把 description 当路由 prompt 来写，值得花一半时间在那几十个字上。
- SKILL.md 控制在一两百行内，细节下沉到引用文件。
- 每个 skill 配一句固定的冒烟测试话术，改动后跑一遍再提交。

## 总结

Skills 的价值不在于多了一种插件格式，而在于把能力的选择权从启动时的全量注入，变成运行时的按需匹配。description 写得好，上下文就干净，模型表现也稳定。建议先把现有大 prompt 拆成两三个 skill 试跑验证，比一次性重构划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/f14e98ee59f546aa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/a1338921b425e40c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/bca13d3ed92e6448.png)

