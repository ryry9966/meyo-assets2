---
title: 别直接 loads：LLM 输出里混合 JSON 格式的分层防御解析
feedId: 41004
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，LLM 输出几乎是唯一的数据入口。我们习惯在 system prompt 里写一句"只输出 JSON"，然后在代码里 `json.loads` 一把梭。测试时看起来没问题，一旦接上多模型、多版本、多 agent 串联的真实流水线，拿到的输出就会变成：裸 JSON、```json 围栏、`<json>` 标签包裹、前后带一句"以下是结果"、中文语境下的全角引号，甚至被 max_tokens 截断的半截 JSON。

## 问题

直接解析的失败率上线后开始飘，典型故障有两种：一次解析失败导致整条工具调用链中断；或者更隐蔽——随手一个正则把坏数据"修"成了格式合法但内容错误的数据，下游静默地用错了。

## 做法

思路是分层解析、逐级降级，并且记录每一层走了哪条路径：

1. **预处理**：去 BOM、去首尾空白。不要全局替换全角引号，中文正文里的 " " 可能是合法内容。
2. **直接 `json.loads`**：能走捷径就走捷径。
3. **围栏提取**：匹配 ```json 或 ```，注意模型可能在字符串里再嵌套一层围栏。
4. **标签提取**：匹配 `<json>...</json>` 之类，非贪婪 + DOTALL，命中多个时取第一个并打日志。
5. **括号扫描**：从第一个 `{` 或 `[` 开始做深度计数，跳过字符串字面量和转义，取平衡段。这是最稳的一层，不依赖任何约定格式。
6. **修复层（可选）**：容忍尾逗号等小毛病，但每个修复动作必须可观测。

```python
def extract_json(text: str):
    text = text.strip().lstrip("\ufeff")
    try:
        return json.loads(text), "direct"
    except json.JSONDecodeError:
        pass
    for name, pattern in (("fence", FENCE_RE), ("tag", TAG_RE)):
        if (m := pattern.search(text)) and (blob := m.group(1).strip()):
            try:
                return json.loads(blob), name
            except json.JSONDecodeError:
                continue
    if blob := scan_balanced(text):  # 感知字符串的深度计数
        return json.loads(blob), "scanner"
    raise JSONExtractError(raw=text)  # 永远带上原始报文
```

## 踩坑点

- **正则跨块误匹配**：`.*?` + DOTALL 在长输出上会吃进不该吃的范围；括号扫描如果不去感知字符串内部，markdown 正文里的花括号会让深度直接失衡。
- **全角引号替换**：把 " " 全局换成 " 会破坏含中文引号的字符串内容，只该修结构位，不动字符串内部。
- **截断不等于格式错误**：max_tokens 打满后的 JSON 永远不完整，"修复"出来的是错数据。应识别截断特征后重试或快速失败。
- **多个 JSON 块**：取第一个不一定是答案，策略要显式声明。
- **json_repair 类库**：可能静默改写数据，修复前后至少要有计数或 diff。

## 可复用建议

- 解析收敛到一个模块，所有工具调用走同一入口，不要各插件各写一套正则。
- 把解析路径（direct / fence / tag / scanner / repair）作为标签写进日志和指标，跑两周你就知道该在哪一层优化 prompt。
- 抽取成功不等于数据正确，解析后面必须接 schema 校验。
- 失败重试一次（更强约束的 prompt + 低温），仍失败则携带原始报文报错，禁止静默吞掉。
- 把线上真实坏样本沉淀成单测 fixture，回归测试比文档管用。

## 总结

解析层是你和一个不确定的对手之间的契约。防御性解析负责兜底，可观测性负责告诉你去哪修 prompt，两者缺一不可。格式约定值得做，但永远不要让正确性依赖它。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/270c86cc5d3ed7ce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/d254d2233d873e4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/42142dcf6a619ca9.png)

