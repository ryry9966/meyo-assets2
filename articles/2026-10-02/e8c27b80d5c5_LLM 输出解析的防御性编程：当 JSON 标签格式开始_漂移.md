---
title: LLM 输出解析的防御性编程：当 JSON 标签格式开始"漂移
feedId: 40033
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

在 OpenClaw 的 skill 和插件开发里，让 LLM 返回结构化结果几乎绕不开：工具调用参数、任务拆解、信息抽取，通常约定模型把 JSON 包在固定标签里输出，比如 `<json>...</json>`。Demo 阶段一切正常，问题出现在长时间真实运行之后。

## 问题

同一个 prompt，模型的输出格式会漂移：

- 有时老老实实输出 `<json>{...}</json>`；
- 有时套一层 ```json 围栏；
- 有时标签前后多出一段"解释性文字"；
- 有时输出两个标签块，一个是草稿一个是正式版；
- 偶尔 JSON 没写完就被截断。

直接 `json.loads` 的成功率大概八成出头，剩下两成随机失败——这比稳定失败更烦，因为自动化流程里你不知道哪一单会断。

## 做法

思路是把解析做成多级流水线，任一级失败就降级到下一级：

1. **优先结构化输出。** 能用 function calling / JSON mode 就别靠自由文本加标签，标签方案只作兜底通道。
2. **多来源提取候选。** 依次收集：标签内容（正则忽略大小写、非贪婪）、代码围栏内容（` ```json ` 和裸 ` ``` ` 都认）、裸 JSON 兜底（从首个 `{` 做括号配平扫描）。
3. **逐个候选尝试解析。** 先标准 `json.loads`；失败进修复层：去 BOM/零宽字符、统一全角引号、去尾逗号，或上 json-repair 类库。
4. **Schema 校验。** 解析成功不等于数据可用，用 pydantic 再验一遍字段和类型。
5. **失败重试。** 把原始输出和报错拼回 prompt，要求"仅输出标签内容"，通常一次能救回；连败则降级到人工队列。

核心骨架十来行：

```python
CANDS = re.findall(r"<json>(.*?)</json>", text, re.S | re.I) \
      + re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
for raw in CANDS:
    for cand in (raw, repair(raw)):
        try:
            return validate(json.loads(cand))
        except Exception:
            continue
raise ParseError(raw_text=text)  # 进入重试/降级
```

## 踩坑点

- **贪婪正则吞块**：`<json>.*</json>` 会把两个标签块之间的内容全吃掉，必须非贪婪 `.*?` 且配 `re.S`。
- **只取第一个匹配**：模型偶尔先给草稿再给正式版，应收集全部候选逐个试，取第一个解析通过的，而非第一个出现的。
- **修复器过度修复**：json-repair 可能把截断数组硬修成"合法但语义错误"的对象，修复后必须过 schema，不能直接信任。
- **围栏里嵌围栏**：JSON 字段值里带代码块时，围栏正则会提前截断，这类要靠括号配平兜底。
- **没存原文**：只记解析结果不记 raw text，排障时基本抓瞎。原文务必落盘。

## 可复用建议

- 提取器独立成模块并配单测；用线上真实失败样本建回归用例库，每遇新格式补一条。
- 提示词和解析器双保险：prompt 写清"仅输出标签内容，不要附加解释"，但解析器按"它一定会违约"来设计。
- 把解析成功率做成监控指标，格式违约率突变往往对应模型版本或上下文长度变化，是不错的告警信号。

## 总结

防御性解析的核心是心态转变：LLM 输出和第三方 API 一样，是不可信输入。结构化输出是第一道防线，多级提取 + 宽松修复 + schema 校验 + 失败重试构成完整兜底链。写一次流水线的成本，远低于每次线上挂掉后手动救火。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/5134dfa9242f19a2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/682cb2f93e9ed596.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/9ff9f1f025edd6f9.png)

