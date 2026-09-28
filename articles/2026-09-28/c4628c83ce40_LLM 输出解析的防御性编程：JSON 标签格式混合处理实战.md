---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 39214
source: 综合讨论
publishedAt: 2026-09-28
---

# 背景

在 OpenClaw 的插件、skill 和自动化流程里，我们经常需要模型返回结构化 JSON：节点间传参、写回配置、触发下游动作。提示词里通常约定一种格式，比如用 `<result>` 标签包裹。但只要换模型、调温度、拉长上下文，输出格式就会漂移——而且往往在你最不希望它漂移的那次生产运行中。

# 问题

同一个提示词，一周内我收集到的真实输出至少有五种：

- 整齐的 `<result>{...}</result>`
- ` ```json ` 围栏块包裹
- 裸 JSON，但前面带一句「以下是解析结果：」
- JSON 后面跟着模型自己的补充说明
- 结构正确，但带尾逗号、全角引号、字符串里未转义的换行

任何一处直接 `JSON.parse` 都会抛异常。在 agent 链路里，一个解析失败会中断整条自动化任务，而报错往往只有一句 `Unexpected token`，很难定位。

# 做法：四层解析管线

核心思路是把解析收敛为独立模块，而不是散落各处的一行 parse。分四层，逐层降级：

**第一层：提取**。按优先级尝试：

1. 自定义标签 `<result>...</result>`（非贪婪匹配）
2. ` ```json ` 围栏块（明确策略：取第一个还是最长的一个）
3. 字符串感知的花括号配对：从第一个 `{` 扫到配对的 `}`，扫描时跳过字符串字面量与转义符

**第二层：修复**。提取或 parse 失败时，用 json-repair 类库处理尾逗号、单引号、智能引号、未转义换行。修复动作必须记日志，不能静默。

**第三层：校验**。用 zod（Python 侧用 pydantic）定义 schema，失败时输出具体路径，例如 `tools[2].name: expected string`。

**第四层：有界重试**。把校验错误拼回提示词重发一次，最多一两次；仍失败就走兜底分支，并保存原始输出。

简化的第一层示意（TypeScript）：

```ts
function extractJson(raw: string): string | null {
  const tag = raw.match(/<result>([\s\S]*?)<\/result>/);
  if (tag) return tag[1].trim();
  const fence = raw.match(/```(?:json)?\s*([\s\S]*?)```/);
  if (fence) return fence[1].trim();
  // 字符串感知的花括号配对
  const start = raw.indexOf("{");
  if (start === -1) return null;
  let depth = 0, inStr = false, esc = false;
  for (let i = start; i < raw.length; i++) {
    const c = raw[i];
    if (esc) { esc = false; continue; }
    if (c === "\\") { esc = true; continue; }
    if (c === '"') inStr = !inStr;
    if (inStr) continue;
    if (c === "{") depth++;
    if (c === "}" && --depth === 0) return raw.slice(start, i + 1);
  }
  return null;
}
```

# 踩坑点

- 围栏正则用贪婪匹配会吞掉后续代码块和说明文字，务必非贪婪，且想清楚「取第一个还是最长」。
- 朴素花括号计数会被字符串里的 `{` `}` 打崩，必须带字符串状态机。
- 字符串值本身包含 ` ``` ` 时（比如让模型总结代码），围栏正则会提前截断——所以标签提取要放在围栏之前。
- 全角引号 “ ” 在中文语料多的场景高频出现，部分 repair 库不覆盖，可自己加替换规则。
- 修复层太激进会把 `123` 修成 `"123"`，类型悄悄变化，下游照样出错。修复必须可观测。
- `max_tokens` 截断导致的残缺 JSON 不是格式问题，重试前先调大限额。
- 流式输出场景可引入 partial JSON 解析改善体验，但落库前仍要走完整管线。

# 可复用建议

1. 能用 function calling / 结构化输出就用，标签方案只做兜底，两者都做；MCP 工具返回的结构化结果同理，不要假设它永远干净。
2. 解析器全项目唯一入口，禁止各处裸 parse。
3. 记录 prompt 版本、模型名、原始输出三元组，攒一份真实失败样本集，做 golden test 回归。
4. 提示词里给一个完整的输入输出示例，比反复强调「只输出 JSON」有效得多。
5. schema 尽量扁平，嵌套越深，漂移概率越高。

# 总结

把模型输出当不可信输入处理：提取、修复、校验、有界重试，每一层可观测、可降级。防御性解析的目的不是把脏输出洗干净，而是让失败发生在你设计好的位置，并留下足够的证据去修上游的提示词。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/cdaf3dfdb40ff51d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/77e98924fa14bf13.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/a3347f7205c9ba67.png)

