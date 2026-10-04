---
title: 给 LLM 输出解析上一道保险：JSON 混合格式的分层防御处理
feedId: 40377
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，让 LLM 返回 JSON 是最常见也最脆弱的一环。我们通常在 system prompt 里写"只输出 JSON"，但真实运行中，模型输出往往是混合格式：有时带 ```json 代码围栏，有时前后附带解释文字，有时 JSON 里混着中文标点。手动演示没问题，跑进自动化流水线就频繁崩。

## 问题

典型脏输出有这几类：

1. **围栏包裹**：```json\n{...}\n```
2. **自然语言混入**："好的，以下是结果：{...} 希望有帮助"
3. **标点漂移**：中文引号 ""、全角冒号、尾逗号
4. **截断**：max_tokens 打满，JSON 只输出一半
5. **双重编码**：JSON 字符串里又嵌了一层 JSON 字符串

直接 `json.loads()` 失败率不低，而解析一崩，上游整条 agent 链路就断。

## 做法

核心思路是**分层降级、逐层兜底**，而不是指望 prompt 一次解决：

1. **剥围栏**：用正则提取 ```...``` 之间的内容，没有围栏则用原文。
2. **定位边界**：找到第一个 `{` 或 `[`，再用括号配对（跳过字符串内部的括号、处理转义）找到匹配的收尾，截取中间片段。不要用贪婪正则匹配到最后一个 `}`，字符串里出现花括号时会截错。
3. **尝试解析**：标准 `json.loads` 先走；失败后做一次轻修复——中文引号/全角标点转 ASCII、去尾逗号、去 BOM——再试一次。也可以直接用 `json_repair` 这类库。
4. **校验结构**：用 pydantic 或 jsonschema 校验字段，缺字段给默认值或抛出明确异常。
5. **留证据**：无论成败，把原始输出完整写进 trace 日志。截断类问题只有对着原文才能定位。

骨架大致是：

```python
def parse_llm_json(raw: str):
    text = strip_code_fence(raw)
    frag = extract_balanced(text)
    for candidate in (frag, light_repair(frag)):
        try:
            return json.loads(candidate)
        except json.JSONDecodeError:
            continue
    raise ParseError(raw)  # 异常里带上原文
```

## 踩坑点

- 括号配对时没处理字符串内部的 `{ }` 和转义引号，截出来的片段必然是坏的
- 修复函数把字符串**值内部**的引号也替换了，JSON"修好了"，数据却被污染
- 对截断的 JSON 硬修，不如直接触发重试或拆小任务
- streaming 场景边收边 parse，正确做法是收完再走解析管线
- 过度信任 provider 的"结构化输出"开关，不同模型和渠道表现不一致，兜底层仍要保留

## 可复用建议

- 把 `parse_llm_json` 沉淀成公共 util，全项目只用一个入口，别到处复制粘贴正则
- 攒一个"脏输出语料库"做单元测试，每次遇到新的坏例就追加进去，守住回归
- 能用原生 function calling / 结构化输出就优先用，自研解析只做兜底
- 重试时把解析失败的原文和错误信息回传给模型要求纠正，通常一两轮就能过

## 总结

LLM 输出解析没有银弹。工程上靠谱的姿势是：不信任原始输出、分层降级、留全量日志、用测试语料守住回归。解析层稳了，上层的 agent 和自动化才敢真正放手跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/08f2bb80db0b1211.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/4cd486c46c552ded.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/279c88ff2cd20099.png)

