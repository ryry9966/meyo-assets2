---
title: LLM 输出解析的防御性编程：一套处理混合 JSON 格式的分层管线
feedId: 42961
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

写 OpenClaw 插件或自定义技能时，经常需要模型返回结构化数据：任务清单、意图判断、对 MCP 工具输出的二次加工。提示词里写一句「只输出 JSON」，实际拿到的回复五花八门——有的包在 ```json 围栏里，有的套 `<answer>`、`<output>` 标签，有的前后带解说词，偶尔直接是一段裸 JSON。换模型、调温度、改提示词，格式分布还会漂移。

## 问题

最朴素的写法是 `JSON.parse(reply)`，挂了再用正则抓代码围栏。demo 阶段够用，跑真实流量后失败率通常在 5%–15%，且是偶发失败，难复现。更隐蔽的是：解析成功不等于数据正确，模型可能照着提示词里的示例回填了占位符，直接把假数据传给下游工具。

## 做法：按成本从低到高分层

1. **快路径**：去掉 BOM 和零宽字符，trim 后直接 parse。
2. **标签层**：依次尝试 `<json>` / `<answer>` / `<output>` / `<result>`，取内部内容再 parse。
3. **围栏层**：匹配所有代码围栏，倒序逐个尝试，取第一个能通过后续校验的（模型习惯把最终答案放最后一个块里）。
4. **括号扫描层**：定位 `{` 或 `[`，做字符串感知的括号配平扫描（处理引号与转义），失败再从最后一个 `{` 反向找。
5. **容错修复层**：去尾逗号、去注释、替换全角引号、补右括号——必须打 warning，不允许静默。
6. **校验层**：parse 成功后过 zod/ajv schema，不过则视为失败，走重问或降级逻辑。

```ts
export function extractJson(raw: string): unknown {
  const cleaned = raw.replace(/^\uFEFF/, "").trim();
  try { return JSON.parse(cleaned); } catch { /* 进入下一层 */ }

  for (const tag of ["json", "answer", "output", "result"]) {
    const m = cleaned.match(new RegExp(`<${tag}>([\\s\\S]*?)</${tag}>`));
    if (m) { const v = tryParse(m[1]); if (v !== undefined) return v; }
  }

  const fences = [...cleaned.matchAll(/```(?:json)?\s*([\s\S]*?)```/g)];
  for (const f of fences.reverse()) {
    const v = tryParse(f[1]);
    if (v !== undefined) return v;
  }

  for (const i of [cleaned.indexOf("{"), cleaned.indexOf("[")].filter(i => i >= 0)) {
    const slice = balancedScan(cleaned, i); // 字符串感知的配平扫描
    if (slice) { const v = tryParse(slice); if (v !== undefined) return v; }
  }

  const repaired = repair(cleaned); // 尾逗号/注释/全角引号/补括号
  if (repaired) { logger.warn("json repaired, prompt may be drifting"); return repaired; }
  throw new ParseError(raw.slice(0, 200));
}
```

## 踩坑点

- `/\{.*\}/` 不带 `s` 标志匹配不了多行，带了又容易横跨多个对象。宁可配平扫描，别用贪心正则。
- 括号计数如果不感知字符串，遇到内容本身含 `{` 的文本就会切错位置，这种内容在实际业务里非常常见。
- 多围栏时取第一个，可能抓到模型复述提示词里的示例 JSON。
- 修复层要打点计数。如果配平扫描和修复的命中率持续上升，说明提示词在漂移——该改 prompt，而不是继续加固解析器。
- 容错别做过头：连尾逗号都默默修，线上就永远发现不了模型输出在退步。

## 可复用建议

- 收敛成一个 `extractJson` 工具函数放进插件公共库，不要每个 skill 各抄一份正则。
- 失败时把原始输出截断落日志，攒一份「脏输出语料」当测试 fixtures，回归价值比任何新写的单测都高。
- 运行时若支持 JSON Schema 约束生成，优先用原生结构化输出；解析管线是安全网，不是主机制。
- 提示词里给格式示例要克制且一致：示例用围栏，模型就爱用围栏；示例用标签，模型就爱用标签。别让 prompt 和解析器互相打架。

## 总结

分层提取的本质是一条按成本排序的兜底链：直解 → 标签 → 围栏 → 配平 → 修复 → 校验。每一层都应可观测，命中率反过来指导提示词迭代。防御性解析的目标不是「什么都能解析」，而是把格式不确定性变成可测量、可回归的工程指标。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/5624c69832141574.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/81d68e7e3e2afcdb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/7495e9c04617ee4a.png)

