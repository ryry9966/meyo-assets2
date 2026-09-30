---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的分层处理实践
feedId: 39858
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

在 OpenClaw 里写自动化流程（skill、插件、MCP 工具编排）时，经常需要让 LLM 返回结构化结果，再交给下游代码消费：路由决策、参数抽取、任务拆分都是典型场景。提示词里写一句"只输出 JSON"，看起来是约定，实际上是概率性的——换模型版本、调温度、加长上下文之后，输出格式总会漂移。

## 问题

实际遇到的输出形态至少有这么几类：

- 包在 ` ```json ` 围栏里的；有时 JSON 字符串值里还嵌着反引号，形成"围栏套围栏"；
- 包在 `<json>…</json>` 或 `<output>…</output>` 标签里的；
- 裸 JSON，前后还带一句"以下是解析结果："；
- 被 max_tokens 截断的半截 JSON。

只在提示词层面死磕格式，迟早会在生产上炸一次。解析层必须自己做防御。

## 做法

思路是把解析做成一个分层管道，按成本从低到高逐级兜底：

```ts
function extractJSON(raw: string): ParseResult {
  const candidates = [
    raw,                                   // 1. 直接解析
    ...fencedBlocks(raw),                  // 2. 抠 ``` 围栏，取最长一块
    ...taggedBlocks(raw),                  // 3. 抠 <json>/<output> 标签
    ...balancedScans(raw),                 // 4. 括号配平扫描
  ];
  for (const c of candidates) {
    try { return { ok: true, data: JSON.parse(c), strategy: '...' }; }
    catch { /* 记录失败层级，继续 */ }
  }
  // 5. 轻量修复：去尾逗号、全角引号替换，再试一轮
  return { ok: false, raw };
}
```

四个要点：

1. **候选按序尝试**，谁先解析成功用谁，同时记录 `strategy` 字段，方便后续统计。
2. **括号配平扫描必须字符串感知**——维护 `inString` 和 `escape` 两个状态位，否则字符串里的 `{` 会把扫描带偏。
3. **解析成功不等于结束**，还要过 schema 校验（ajv/zod），类型不对、必填缺失同样算失败。
4. **失败后不要盲目重试**，把原始输出和具体报错拼进下一轮提示词让模型自我修正，一般一轮就够。

## 踩坑点

- `/\{.*\}/s` 贪婪正则会跨对象匹配；`/\{.*?\}/` 非贪婪会截断嵌套对象。两种都出过事故，最后还是老老实实写配平扫描。
- 修复要克制：单引号转双引号要看上下文，值里本来就带单引号时，激进修复会改语义。
- 明显截断的输出不要修，直接触发续写或缩短请求，修半截 JSON 纯属浪费时间。
- 中文上下文里常见全角引号和零宽字符，预处理统一清一遍。
- `temperature=0` 不保证格式稳定，模型一升级照样漂移，别把宝押在这上面。

## 可复用建议

- 解析逻辑收敛到一个工具模块，禁止在业务代码里散落正则。
- 返回结构统一为 `{ ok, data, strategy, raw }`，`raw` 一定要留着，排查全靠它。
- 建一个"坏输出语料库"：线上每出一次解析失败，就把样本收进单元测试。这个语料库比任何格式文档都管用。
- 定期统计各 strategy 的命中率。如果大部分流量靠"修复"兜住，说明提示词该改了，而不是解析该加强了。

## 总结

LLM 的输出本质上是不可信输入。提示词里的格式约定只是第一道防线，真正稳的是：分层兜底解析 + schema 校验 + 带报错回传的一次重试。这套东西一百行以内能写完，但能让自动化流程的可用性实实在在上一个台阶。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/f8f3b1d4d064b486.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/0fef6e3fc0855c71.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/92f517ff40365379.png)

