---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 40853
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

在 OpenClaw 插件和 MCP 工具里，让模型"输出一段 JSON"是最常用的集成方式：技能路由、参数抽取、结构化摘要，都靠它。prompt 里写得再清楚——"只输出 JSON，不要任何多余内容"——真实返回依然五花八门，尤其在多模型混用、调高 temperature、或切换供应商之后。

## 问题

我在一个技能路由插件里实测收到过的格式，至少有这些：

1. 标准 ` ```json ` 围栏；
2. 裸 JSON 直接吐出，前后无任何包裹；
3. 自定义标签包裹：`<result>{...}</result>`；
4. 推理模型先来一段 `<think>…</think>`，JSON 藏在后面；
5. JSON 前有"好的，以下是结果："这类客套话，后面还跟一句总结；
6. 围栏语言标错（```JSONC、```JSON5，甚至裸 ```）；
7. 伪 JSON：尾逗号、单引号、全角冒号和引号；
8. 输出被 max_tokens 截断，JSON 只剩半截。

裸 `JSON.parse` 的存活率大概六成，剩下四次失败全发生在深夜的自动化任务里。这不是模型问题，是工程问题。

## 做法：分层候选 + 逐层放宽

核心思路：不要假设单一格式，而是把"可能合法的片段"全部抽出来当候选，从最严格的解析开始试，逐层放宽。示意代码（TypeScript，`jsonrepair` 来自同名 npm 包）：

````ts
function parseLlmJson(raw: string): unknown {
  let s = raw.replace(/<think>[\s\S]*?(?:<\/think>|$)/g, ""); // 1. 剥思维链
  const fences = [...s.matchAll(/```(?:json[cs5]?|JSON)?\s*([\s\S]*?)```/g)].map(m => m[1]);
  const tag = s.match(/<(?:result|json|output)>([\s\S]*?)<\/(?:result|json|output)>/);
  const candidates = [tag?.[1], ...fences.reverse(), s].filter(Boolean);
  for (const c of candidates) {
    for (const seg of [c.trim(), braceExtract(c)]) {       // 2. 含裸括号提取
      if (!seg) continue;
      try { return JSON.parse(seg); } catch {}             // 3. 严格解析
      try { return JSON.parse(jsonrepair(seg)); } catch {} // 4. 宽松修复
    }
  }
  throw new Error("unparseable llm output");
}
````

四个要点：

- **候选顺序敏感**：先标签、再围栏、再原文；围栏取倒序，因为多数模型把最终结果放在最后——按你的实测分布调整。先严格 `JSON.parse`，失败才走 `jsonrepair`。宽松层成功率最高，但它会"修"出意料之外的结构，必须放在最后兜底。
- **`braceExtract` 是括号配对提取**（从第一个 `{` 或 `[` 找配对），必须跳过字符串字面量内部的 `{}` 和转义符，朴素计数会被 JSON 里的代码片段打穿。
- **解析成功后立刻过 schema 校验**（Zod / pydantic 都行），这一步和解析同等重要。
- **记录命中了哪一层**。围栏层命中率从 80% 掉到 30%，通常意味着 prompt 被改过或者换了模型。

## 踩坑点

- JSON 字段值里嵌了 ``` 围栏（比如存代码片段），围栏正则会提前截断。所以围栏只是候选之一，靠多候选互救，别当唯一来源。
- `<think>` 要处理未闭合的情况：输出被截断时思维链没有闭合标签，正则剥不掉，得加"无闭合就从头剥到尾"的分支（示例里用 `(?:<\/think>|$)` 兜住）。
- `jsonrepair` 不是银弹：单引号修复可能破坏内容里本来就含单引号的字符串。修复 ≠ 正确，schema 校验才是闸门。
- 截断的输出 repair 也救不回来。先看 finish_reason 和长度异常，直接重试或调大预算，别浪费在修复上。
- 模型偶尔输出两个 JSON 块（一个示例 + 一个结果），多候选会都试到，但要靠 schema 校验决定留下哪个。
- 全角标点归一化只处理 `，：“”` 这类标点，别全局替换误伤正文。

## 可复用建议

- 把解析器抽成独立 util，线上收集的真实坏样本做成 fixture 回归测试，接入新模型时跑一遍。
- 上游优先用 tool call / structured output（OpenClaw 里支持就用），防御解析只做兜底，不要本末倒置。
- 自动化任务把 raw output 完整落日志（脱敏后），否则连坏样本都收集不到。
- temperature 0~0.3 能显著减少格式发散，结构化抽取任务别吝啬。

## 总结

LLM 输出解析的稳定性不靠 prompt 祈祷，靠四件事：分层候选、严格优先、修复后校验、失败可观测。它不性感，但决定了你的插件是"偶尔抽风"还是"值得信任"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/96c2e7e7002e112e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/797f3217be0f1f79.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/43ee65674e65abd7.png)

