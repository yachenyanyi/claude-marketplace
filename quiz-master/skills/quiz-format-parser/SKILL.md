---
name: quiz-format-parser
description: 当用户需要解析、验证或转换题库文件格式时使用此技能，支持 JSON/CSV/TXT/MD/Excel/Word/PDF 等多种格式的识别、解析和标准化转换
version: 1.0.0
---

# Quiz Format Parser

此技能提供题库文件格式的解析、验证和转换指导。当处理题库导入、格式识别或数据转换任务时使用。

## 触发场景

此技能在以下场景中自动激活：

- 用户尝试导入题库文件（JSON/CSV/TXT/MD/Excel/Word/PDF）
- 需要验证题库格式的正确性
- 需要将一种格式转换为另一种格式
- 需要修复格式错误的题库文件
- 需要理解不同格式的结构规范

- 用户询问题库文件格式要求
- 用户遇到导入失败或格式错误
- 用户想自定义题库格式
- 用户需要批量转换多个题库文件

## 核心概念

### 标准题库数据结构

所有题库格式最终都应转换为以下标准 JSON 结构：

```json
{
  "name": "题库名称",
  "description": "题库描述",
  "tags": ["标签1", "标签2"],
  "questions": [
    {
      "id": "unique-id",
      "type": "choice|fillblank|essay",
      "question": "题目内容",
      "options": ["选项A", "选项B", "选项C", "选项D"],
      "answer": "正确答案",
      "explanation": "解析说明（可选）",
      "tags": ["知识点1"],
      "difficulty": "easy|medium|hard",
      "metadata": {}
    }
  ]
}
```

### 字段说明

**题库级别**:
- `name`: 题库名称（必需，kebab-case）
- `description`: 题库描述（可选）
- `tags`: 题库标签（可选，字符串数组）
- `questions`: 题目数组（必需）

**题目级别**:
- `id`: 唯一标识符（必需，全局唯一）
- `type`: 题目类型（必需）
  - `choice`: 选择题（单选/多选）
  - `fillblank`: 填空题
  - `essay`: 问答题
- `question`: 题目内容（必需）
- `options`: 选项数组（choice 类型必需，至少2个）
- `answer`: 正确答案（必需）
- `explanation`: 解析说明（可选）
- `tags`: 题目标签（可选，字符串数组）
- `difficulty`: 难度等级（可选，默认 "medium"）
- `metadata`: 额外元数据（可选，自由格式对象）

## 格式解析规则

### JSON 格式

**识别特征**:
- 文件扩展名 `.json`
- 内容以 `{` 开头
- 包含 `"name"` 和 `"questions"` 字段

**验证规则**:
```python
def validate_json_quiz(data):
    # 必需字段
    assert "name" in data, "缺少 name 字段"
    assert "questions" in data, "缺少 questions 字段"
    assert isinstance(data["questions"], list), "questions 必须是数组"

    # 题目验证
    for idx, q in enumerate(data["questions"]):
        assert "id" in q, f"题目 {idx+1} 缺少 id"
        assert "type" in q, f"题目 {idx+1} 缺少 type"
        assert "question" in q, f"题目 {idx+1} 缺少 question"
        assert "answer" in q, f"题目 {idx+1} 缺少 answer"

        # 类型特定验证
        if q["type"] == "choice":
            assert "options" in q, f"题目 {idx+1} 是选择题但缺少 options"
            assert len(q["options"]) >= 2, f"题目 {idx+1} 选项不足2个"
```

**转换目标**: JSON 是标准格式，无需转换

### CSV 格式

**识别特征**:
- 文件扩展名 `.csv`
- 第一行包含表头
- 逗号分隔的列

**标准表头**:
```csv
id,type,question,options,answer,explanation,tags,difficulty
q1,choice,apple的意思?,香蕉,苹果,橙子,苹果,apple是苹果,水果,easy
```

**解析规则**:
1. 第一行必须是表头
2. 支持中英文列名（如 `question` 或 `问题`）
3. `options` 和 `tags` 列用分隔符分割（默认逗号，可配置）
4. 空字段视为可选字段

**转换示例**:
```python
import csv

def csv_to_json(csv_file):
    reader = csv.DictReader(csv_file)
    questions = []

    for row in reader:
        question = {
            "id": row["id"],
            "type": row["type"],
            "question": row["question"],
            "answer": row["answer"]
        }

        # 可选字段
        if row["options"]:
            question["options"] = row["options"].split(",")
        if row["explanation"]:
            question["explanation"] = row["explanation"]
        if row["tags"]:
            question["tags"] = row["tags"].split(",")
        if row["difficulty"]:
            question["difficulty"] = row["difficulty"]

        questions.append(question)

    return {"name": "csv-bank", "questions": questions}
```

### TXT 格式

**识别特征**:
- 文件扩展名 `.txt`
- 使用分隔符分割题目（如 `---` 或空行）
- 使用前缀标识字段（如 `Q:`, `A:`）

**标准格式**:
```
Q: apple 的中文意思是？
A: 苹果
Options: 香蕉,苹果,橙子,葡萄
Explanation: apple 是苹果的英文单词
Tags: 水果,英语
Difficulty: easy
Type: choice

---
Q: banana 的中文意思是？
A: 香蕉
Options: 香蕉,苹果,橙子,葡萄
Explanation: banana 是香蕉的英文单词
Tags: 水果,英语
Difficulty: easy
Type: choice
```

**解析规则**:
1. 按 `---` 分割题目
2. 每题内部按行解析字段
3. 字段格式：`Key: Value`
4. 支持的字段前缀：
   - `Q:` 或 `Question:` → question
   - `A:` 或 `Answer:` → answer
   - `Options:` → options（逗号分隔）
   - `Explanation:` → explanation
   - `Tags:` → tags（逗号分隔）
   - `Difficulty:` → difficulty
   - `Type:` → type

**转换示例**:
```python
def parse_txt_question(text):
    lines = text.strip().split('\n')
    question = {}

    for line in lines:
        if ':' not in line:
            continue

        key, value = line.split(':', 1)
        key = key.strip().lower()
        value = value.strip()

        if key in ['q', 'question']:
            question['question'] = value
        elif key in ['a', 'answer']:
            question['answer'] = value
        elif key == 'options':
            question['options'] = [v.strip() for v in value.split(',')]
        # ... 其他字段

    return question
```

### Markdown 格式

**识别特征**:
- 文件扩展名 `.md`
- 包含 Markdown 标记（`#`, `**`, `-` 等）

**支持格式**:

**格式1: 代码块**
```markdown
# 英语词汇题库

```json
{
  "questions": [
    {
      "id": "q1",
      "question": "apple 的意思？",
      "answer": "苹果"
    }
  ]
}
```

**格式2: 列表**
```markdown
# 题库

## 题目 1
**问题**: apple 的中文意思是？
**选项**:
- A. 香蕉
- B. 苹果
- C. 橙子

**答案**: B
**解析**: apple 是苹果
**标签**: 水果
**难度**: 简单

---
## 题目 2
...
```

**格式3: Q&A**
```markdown
# 题库

**Q1:** apple 的中文意思是？
A. 香蕉
B. 苹果
C. 橙子

**A:** B
**解析:** apple 是苹果
**标签:** 水果

---

**Q2:** ...
```

**解析规则**:
1. 提取标题作为题库名称
2. 按二级标题（`##`）或分隔符（`---`）分割题目
3. 解析加粗字段：`**字段名**: 值`
4. 解析列表作为选项

**转换示例**:
```python
import re

def parse_markdown_questions(md_text):
    # 提取题目块
    questions_blocks = re.split(r'##+\s*\d+\.?\s*题目|---', md_text)

    questions = []
    for idx, block in enumerate(questions_blocks):
        if not block.strip():
            continue

        q = {
            "id": f"q{idx+1}",
            "question": extract_field(block, ["问题", "Question", "Q"]),
            "answer": extract_field(block, ["答案", "Answer", "A"]),
            "options": extract_options(block),
            "explanation": extract_field(block, ["解析", "Explanation"]),
            "tags": extract_tags(block)
        }
        questions.append(q)

    return questions
```

## 格式转换流程

### 通用转换步骤

1. **识别格式**:
   - 检查文件扩展名
   - 读取文件前1000字节
   - 检测格式特征

2. **解析为中间结构**:
   - 使用格式特定的解析器
   - 提取题目数据
   - 验证必需字段

3. **验证和修正**:
   - 生成唯一 ID
   - 推断题目类型
   - 设置默认值

4. **转换为标准 JSON**:
   - 构建标准 JSON 对象
   - 保存为 `.json` 文件

### 转换脚本位置

所有转换脚本位于：`C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/`

可用脚本：
- `parse-json.py`: JSON 解析和验证
- `parse-csv.py`: CSV 转 JSON
- `parse-txt.py`: TXT 解析
- `parse-markdown.py`: Markdown 解析
- `convert-excel.py`: Excel 转 JSON
- `convert-docx.py`: Word 文本提取
- `convert-pdf.py`: PDF 文本提取

### 使用脚本

**从命令调用脚本**:
```bash
python C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/parse-csv.py input.csv
```

**从 Python 导入**:
```python
import sys
sys.path.append('C:\\Users\\Administrator\\.claude\\plugins\\cache\\claude-plugins-official\\plugin-dev\\4fee769bf4ec\\scripts')

from parse_csv import csv_to_json
data = csv_to_json('input.csv')
```

## 常见问题

### Q: 如何处理格式错误的题库？

**A**: 使用验证工具检测错误并自动修正：

```python
def fix_invalid_quiz(data):
    # 生成缺失的 ID
    for idx, q in enumerate(data.get("questions", [])):
        if "id" not in q:
            q["id"] = f"auto-{uuid.uuid4().hex[:8]}"

        # 推断类型
        if "type" not in q:
            if "options" in q:
                q["type"] = "choice"
            else:
                q["type"] = "essay"

        # 设置默认难度
        if "difficulty" not in q:
            q["difficulty"] = "medium"

    return data
```

### Q: 如何自定义字段分隔符？

**A**: 在解析时指定分隔符参数：

```python
def parse_txt_custom(text, field_delimiter=':', option_delimiter=','):
    # 自定义字段分隔符和选项分隔符
    pass
```

### Q: 如何处理大文件？

**A**: 使用流式解析和分批处理：

```python
def parse_large_csv(file_path, batch_size=1000):
    with open(file_path) as f:
        reader = csv.DictReader(f)
        batch = []

        for row in reader:
            batch.append(row)
            if len(batch) >= batch_size:
                yield process_batch(batch)
                batch = []

        if batch:
            yield process_batch(batch)
```

## 最佳实践

1. **优先使用结构化格式**: JSON > CSV > TXT > Markdown
2. **明确指定类型**: 不要依赖自动推断
3. **使用标准字段名**: 避免自定义字段
4. **提供解析示例**: 在 `examples/` 目录中提供各格式示例
5. **验证转换结果**: 转换后验证题目数量和内容
6. **保留原始文件**: 转换时备份原文件

## 参考资源

查看 `references/` 目录获取：
- JSON Schema 定义
- CSV 模板文件
- TXT 格式示例
- Markdown 格式示例
- 转换脚本文档

查看 `examples/` 目录获取：
- 真实题库示例
- 格式转换示例
- 错误修复示例
