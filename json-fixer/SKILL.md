---
name: json-fixer
description: 当用户或上游 Agent 需要将大模型的自由文本输出转换为严格 JSON、或修复已有 JSON 的语法错误、字段缺失时使用。自动提取有效 JSON 内容、修复断裂引号、补齐缺失字段，并严格校验输出格式。
whenToUse: 用户粘贴了一段包含 ```json 标记或多余文字的模型输出并要求“转成 JSON”；或上游工作流需要调用此技能来清洗 LLM 的原始返回。
disable-model-invocation: false
user-invocable: true
---

# JSON 强制格式化与容错清洗器

## 输入要求

- `raw_text`（String，必填）：需要清洗的原始文本。
- `expected_schema`（String，可选）：预期的 JSON Schema（JSON 字符串格式）。如果不提供，仅做语法修复和格式化，不做字段级校验。

## 执行步骤

### 1. 提取有效内容
如果 `raw_text` 包含 Markdown 代码围栏（如 ```json ... ```），剥掉围栏，只保留大括号 `{` 到匹配的 `}` 之间的内容。如果围栏不完整或缺失，从文本中定位第一个 `{` 和最后一个匹配的 `}`。

### 2. 语法修复
对提取出的内容执行以下修复：
- 将中文引号（“ ” ‘ ’）替换为英文双引号（"）。
- 修复断裂的引号（如 `"key: "value` 补全为 `"key": "value"`）。
- 移除 JSON 末尾的多余逗号。
- 如果内容被截断导致大括号不匹配，尝试补全缺失的 `}` 或 `]`。

### 3. Schema 校验（如果提供了 expected_schema）
将修复后的 JSON 与 `expected_schema` 进行比对：
- 检查必填字段是否全部存在。缺失的字段用 `null` 或空字符串补全，并在 `fixed_fields` 中记录。
- 检查字段类型是否匹配。类型不匹配时，尝试隐式转换（如将数字字符串转为数字），无法转换的标记为 `type_mismatch`。

### 4. 输出

严格输出以下 JSON 结构，不要添加任何解释性文字或 Markdown 围栏：

{
  "success": true,
  "cleaned_json": {},
  "fixed_fields": [],
  "errors": [],
  "raw_size": 0,
  "cleaned_size": 0
}

字段说明：
- `success`（Boolean）：是否成功产出合法 JSON。
- `cleaned_json`（Object）：修复后的 JSON 对象。如果修复失败，返回空对象。
- `fixed_fields`（Array）：被修复或补全的字段路径列表，如 `["$.price", "$.tags[0]"]`。
- `errors`（Array）：无法修复的错误描述。`success: false` 时必填。
- `raw_size`（Integer）：原始文本字符数。
- `cleaned_size`（Integer）：清洗后 JSON 字符串字符数。
