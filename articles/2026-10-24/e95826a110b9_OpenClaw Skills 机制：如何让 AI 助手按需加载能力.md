---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42468
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

Agent 用久了都会遇到同一个问题：能力越堆越多，系统提示词越来越胖。我早期把数据库查询、截图、部署脚本、周报模板全写进系统提示词，结果 10k+ token 常驻上下文，模型对工具的选择反而变迟钝了——指令太多，约等于没有指令。

OpenClaw 的 Skills 机制就是针对这个问题：把能力拆成独立的 Markdown 文件，平时只加载"目录"，用到时才读"正文"。

## 核心原理：渐进式披露

每个 Skill 是一个文件夹，里面一个 SKILL.md，frontmatter 声明 name 和 description：

```markdown
---
name: ffmpeg-cut
description: 裁剪/转码/压缩视频。当用户要求截取片段、转格式、压体积时使用。
metadata:
  requires:
    bins: [ffmpeg]
---

## 步骤
1. 先用 ffprobe 确认时长和编码
2. 无损裁剪优先 -ss/-to 组合
...
```

启动时 OpenClaw 只把所有 skill 的 name + description 注入提示词（几百 token）；agent 判断当前任务命中某个 skill，再用 read 工具展开正文。这就是渐进式披露：**索引常驻，正文按需**。

另外，`requires` 里的 bins/env 不满足时，对应 skill 会被直接过滤，根本不进索引——这是环境门控，而不是运行时报错。

## 实操步骤

1. 在 `~/.openclaw/skills/<skill-name>/` 下创建 SKILL.md；
2. frontmatter 只留 name、description、requires，别加花活；
3. description 按"做什么 + 什么时候用"写，塞进用户真实的说法；
4. 正文用祈使句写操作步骤，控制在 100 行以内；
5. 重启会话后做双向验证：问一个应该触发的，看 agent 是否 read 了对的文件；再问一个不该触发的，确认没有误伤。

## 踩坑点

- **description 写成功能介绍而非触发条件。**"视频处理工具集"基本不会命中；"当用户要求截取视频片段时使用"命中率高得多。
- **正文写成百科。**正文是给 agent 执行用的，不是给人看的文档，每次展开都是真金白银的 token。
- **把工具接线写进 skill。**Skill 适合"怎么做"的程序性知识；如果能力本质是一组 API 调用，该用 MCP server，别拿 skill 模拟工具。
- **依赖没装，skill 被静默过滤。**启动后看加载日志，确认 skill 真的进了索引，而不是以为它生效了。
- **一个 skill 塞多个能力。**description 关键词互相稀释，触发率一起掉。

## 可复用建议

- 一 skill 一件事：宁可五个小 skill，不要一个大杂烩。
- description 是写给模型的路由信号，统一用"当……时使用"句式。
- 正文里放检查点（先 ffprobe 再动手），比放原理讲解有用。
- 判断标准：agent 需要的是"步骤"就用 Skill，需要的是"调用"就用 MCP。
- 上线前拿 5~10 条真实问法回归一遍触发率，比反复改提示词靠谱。

## 总结

Skills 的本质是"索引 + 按需展开"：用一份轻量目录替换常驻的全量提示词，上下文占用降一个数量级，路由准确率反而上升。写 skill 时把 80% 的精力放在 description 上——它决定了这套机制是省 token，还是摆设。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/bb1efb27b01d4d64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/62b055d72debaddb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/7341c3cf7f6435e9.png)

