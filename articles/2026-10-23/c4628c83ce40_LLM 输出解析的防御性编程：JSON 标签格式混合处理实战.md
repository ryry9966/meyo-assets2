---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 42386
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

在 OpenClaw 的插件和自动化流程里，让模型按约定输出 JSON 是高频场景：工具参数回填、MCP 结果整理、定时任务的摘要落库，都依赖一段"看起来能直接 `json.loads`"的文本。我们通常在 prompt 里写"请用 ```` ```json ```` 代码块输出"或"用 `<json>` 标签包裹"，但实测下来，同一个模型在不同温度、不同上下文长度、甚至不同会话里，给出的格式并不稳定。

## 问题

我把近两周插件日志里的解析失败样本拉出来粗看，分布大致是：

- 约四成是**裸 JSON**，没有任何包裹；
- 约三成是 ```` ```json ```` 围栏，但前面带一句"好的，以下是结果"；
- 其余是 `<json>` / `<JSON>` / `<output>` 标签混用、大小写不定、带属性变体；
- JSON 内部混着**全角引号、尾逗号、`//` 注释**；
- 少数样本输出两段 JSON，或代码围栏里再嵌围栏。

裸 parse 的结果就是：测试环境跑得通，上线后间歇性炸，而且炸的位置每次不一样。

## 做法

思路是**分层降级**，每层只做一件事，失败就交给下一层：

1. **提取层**：正则剥出候选文本，围栏、标签、裸文本都算候选，按优先级排队；
2. **归一化层**：全角引号转半角、去尾逗号、去行注释、清零宽字符与 BOM；
3. **解析层**：逐候选 `json.loads`，全失败再用 `ast.literal_eval` 兜底（能接住单引号字典）；
4. **验证层**：pydantic 校验字段与类型，不通过就把**具体报错**拼回 prompt 重试一次；
5. **留痕**：无论成败，原始文本都落日志。

核心逻辑压缩后二十行左右：

```python
import json, re, ast

CAND = [
    re.compile(r"```(?:json)?\s*(.+?)```", re.S | re.I),    # 非贪婪
    re.compile(r"```(?:json)?\s*(.+)\s*```", re.S | re.I),  # 贪婪，防嵌套
    re.compile(r"<(?:json|output|result)\b[^>]*>(.+?)</", re.S | re.I),
]

def parse_llm_json(text: str):
    candidates = [m.group(1) for p in CAND for m in p.finditer(text)]
    candidates.append(text)  # 裸 JSON 兜底
    for c in candidates:
        s = c.strip().strip("`").strip()
        s = (s.replace("\u201c", '"').replace("\u201d", '"')
              .replace("\u2018", "'").replace("\u2019", "'"))
        s = re.sub(r"^\s*(//|#).*?$", "", s, flags=re.M)
        s = re.sub(r",\s*([}\]])", r"\1", s)  # 尾逗号
        for fn in (json.loads, ast.literal_eval):
            try:
                return fn(s)
            except Exception:
                continue
    return None
```

## 踩坑点

- 正则务必 `re.I`，标签匹配要允许属性：`<json lang="zh">` 这种变体真实存在。
- 贪婪与非贪婪各有坑：内容嵌围栏时，非贪婪截断、贪婪吞后文。两种候选都生成，交给解析层裁决。
- 多 JSON 块取"第一个解析成功的"，别取最后一个——补充说明里常跟着一段示例。
- 归一化别贪：跑两轮还修不好就直接走重试，硬修出来的脏数据比解析失败更难排查。
- 重试必须带具体报错（哪个字符、哪一行），否则模型会原样再犯一遍。

## 可复用建议

- 把解析器做成独立中间件，统一返回 `{ok, data, raw, error}`，业务代码永远不直接碰原始文本。
- 真实失败样本沉淀成测试集，换模型、改 prompt 后跑一遍回归，比临时写单测值钱得多。
- 能用结构化输出 / JSON mode 就优先用，防御解析是兜底，不是第一选择。
- 温度压低，few-shot 里给一组"错误示例 + 正确示例"，能把格式漂移压下去一大半。

## 总结

和 LLM 打交道的解析代码，本质不是"解析"，而是**分层降级 + 全程留痕**。默认模型会犯错，把每一次犯错变成可复现的用例，系统才能从间歇性崩溃变成可预期运行。上面这段代码可以直接抄进插件里，欢迎在回帖里贴你踩到的坏样本，攒成社区的公共测试集。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/64065fd7cc20c9e5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/1e5a752f00b12b66.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/38915bcc0a839b2b.png)

