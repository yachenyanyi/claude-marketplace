---
name: quiz-import
description: 导入题库文件到 Quiz Master 系统，自动识别 JSON/CSV/TXT/MD/Excel/Word/PDF 等格式
allowed-tools:
  - Read
  - Write
  - Bash
argument-hint: <file-path>
---

# Quiz Master - Import Question Bank

你正在帮助用户导入题库文件到 Quiz Master 学习系统。系统支持自动格式识别和 AI 智能提取。

## 任务步骤

### 1. 读取文件并自动识别格式

使用 Read 工具读取用户指定的文件路径，然后自动识别文件类型：

**按文件扩展名识别**:
- `.json` → JSON 格式
- `.csv` → CSV 格式
- `.txt` → TXT 格式
- `.md` → Markdown 格式
- `.xlsx`, `.xls` → Excel 格式
- `.docx`, `.doc` → Word 格式
- `.pdf` → PDF 格式

**扩展名不明确时**:
- 尝试读取文件前 1000 字节
- 检测文件特征（如 JSON 的 `{`，CSV 的 `,`，Markdown 的 `#`）
- 自动判断最可能的格式

### 2. 根据格式处理文件

#### 2.1 结构化格式（JSON/CSV/TXT）

**JSON 格式**:
- 验证 JSON 结构有效性
- 检查必需字段：`name`, `questions`
- 验证每个题目包含：`id`, `type`, `question`, `answer`

**CSV 格式**:
- 按逗号分隔解析
- 第一行应为表头：`id,type,question,options,answer,explanation,tags,difficulty`
- 验证每行的必需字段

**TXT 格式**:
- 按分隔符 `---` 分割多个题目
- 解析每个题目的字段（以 `Q:`, `A:`, `Tags:` 等开头）

#### 2.2 Markdown 格式

**识别模式**:
- 代码块格式（` ```json ` 或 ` ``` `）
- 列表格式（`1.`, `-`）
- Q&A 格式（`**Q:**`, `**问:**`）

**提取策略**:
- 使用 `quiz-format-parser` skill 的 Markdown 解析规则
- 提取问题、答案、选项、标签等字段
- 转换为标准 JSON 格式

**示例 Markdown 格式**:
```markdown
# 英语词汇题库

## 题目 1
**问题**: apple 的中文意思是？
A. 香蕉
B. 苹果
C. 橙子

**答案**: B
**解析**: apple 是苹果的英文单词
**标签**: 水果
**难度**: 简单
```

#### 2.3 Excel 格式 (.xlsx/.xls)

**处理流程**:
1. 使用 `C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/convert-excel.py` 脚本
2. 提取所有工作表
3. 识别题目表格（包含 "question", "问题", "answer", "答案" 等列）
4. 转换为标准 JSON 格式

**脚本调用**:
```bash
python C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/convert-excel.py "{file-path}"
```

#### 2.4 Word 格式 (.docx)

**处理流程**:
1. 使用 `C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/convert-docx.py` 脚本
2. 提取文本内容
3. **使用 AI 智能提取**（推荐）:
   - 将文档内容发送给 Claude
   - 使用结构化提示提取题目
   - 返回标准 JSON 格式

**AI 提取提示模板**:
```
请从以下文档内容中提取所有题目，转换为 JSON 格式。

文档内容：
{document_content}

输出格式要求：
{
  "name": "题库名称",
  "description": "题库描述",
  "tags": ["标签1", "标签2"],
  "questions": [
    {
      "id": "unique-id",
      "type": "choice|fillblank|essay",
      "question": "题目内容",
      "options": ["A选项", "B选项"],
      "answer": "正确答案",
      "explanation": "解析说明",
      "tags": ["知识点"],
      "difficulty": "easy|medium|hard"
    }
  ]
}

请只输出 JSON，不要包含其他文字。
```

#### 2.5 PDF 格式 (.pdf)

**处理流程**:
1. 使用 `C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/convert-pdf.py` 脚本
2. 提取 PDF 文本内容
3. **使用 AI 智能提取**（必须）:
   - PDF 是非结构化格式，需要 AI 理解
   - 同样使用上述 AI 提取提示模板

### 3. 预览确认（必需）

在正式导入前，显示前 5 题预览：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 题库导入预览
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

格式识别：{detected_format}
题库名称：{bank_name}
题目总数：{total_questions}
标签：{tags}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
前 5 题预览：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. [{type}] {question}
   答案：{answer}
   标签：{tags}

2. [{type}] {question}
   答案：{answer}
   标签：{tags}

...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

是否确认导入？(y/n/q)
y - 确认导入
n - 取消导入
q - 退出并手动修正格式
```

### 4. 验证题目数据

对每个题目进行验证：
- ✅ `id` 唯一性检查（如无 ID，自动生成）
- ✅ `type` 必须是：choice, fillblank, essay 之一
- ✅ `question` 和 `answer` 字段非空
- ✅ 如果是 choice 类型，`options` 必须提供且至少2个选项
- ⚠️ 可选字段：`explanation`, `tags`, `difficulty`

**自动修正**:
- 如果 `id` 重复或缺失，自动生成 UUID
- 如果 `type` 缺失，根据题目特征推断（有选项→choice，否则→essay）
- 如果 `difficulty` 缺失，默认为 "medium"

### 5. 创建数据目录

使用 Bash 工具创建必要的目录：
```bash
mkdir -p ~/.quiz-master/banks
mkdir -p ~/.quiz-master/logs
mkdir -p ~/.quiz-master/temp
```

### 6. 保存标准化题库

将解析后的题库保存为 JSON 格式：
- 文件路径：`~/.quiz-master/banks/{bank-name}.json`
- 文件名使用 kebab-case（例如：`english-vocab.json`）
- 包含所有验证通过的题目

### 7. 生成导入报告

向用户显示：
```
✅ 导入成功！

📚 题库：{bank-name}
📝 题目数：{total} 题
🏷️  标签：{tags}
📂 保存位置：~/.quiz-master/banks/{bank-name}.json

原格式：{original_format}
✨ 已自动转换为标准 JSON 格式

题目类型分布：
  - 选择题：{choice_count}
  - 填空题：{fillblank_count}
  - 问答题：{essay_count}

难度分布：
  - 简单：{easy_count}
  - 中等：{medium_count}
  - 困难：{hard_count}

自动修正：
  - 生成的 ID：{generated_ids_count}
  - 推断的类型：{inferred_types_count}
  - 默认难度：{default_difficulty_count}
```

### 8. 错误处理

#### 8.1 文件读取失败

```
❌ 无法读取文件：{file-path}

可能原因：
  - 文件路径不正确
  - 文件不存在
  - 文件权限不足
  - 文件已损坏

建议：
  - 检查文件路径是否正确
  - 确认文件存在并可访问
  - 尝试使用绝对路径
```

#### 8.2 格式识别失败

```
❌ 无法识别文件格式

文件：{file-path}
扩展名：.{extension}

可能原因：
  - 文件格式不受支持
  - 文件内容为空或损坏
  - 扩展名与实际格式不符

建议：
  - 支持的格式：JSON, CSV, TXT, MD, Excel, Word, PDF
  - 尝试将文件转换为支持的格式
  - 手动指定格式：/quiz-import {file-path} --format {format}
```

#### 8.3 AI 提取失败（Word/PDF）

```
⚠️  AI 提取遇到问题

文件：{file-path}

可能原因：
  - 文档是扫描版图片（PDF）
  - 文档格式过于复杂
  - 文档内容不符合题目结构

建议：
  - 如果是扫描版 PDF，先使用 OCR 工具处理
  - 尝试将内容复制到 TXT/MD 文件
  - 手动调整为标准格式后再导入
```

#### 8.4 题目数据错误

```
⚠️  导入部分成功

总题数：{total}
成功：{success}
失败：{failed}

失败题目：
  - 第 {line} 行: {error_reason}

失败原因：
  - 缺少必需字段（question/answer）
  - 选项数量不足（choice 类型至少需要 2 个选项）
  - ID 重复且无法自动修正

选项：
  1. 只导入成功的 {success} 题
  2. 查看详细错误并手动修正
  3. 取消导入

请选择 (1/2/3):
```

### 9. 保存原始文件（可选）

询问用户是否保存原始导入文件：
```bash
# 保存到 ~/.quiz-master/temp/ 目录
cp "{original-file}" "~/.quiz-master/temp/{bank-name}.{ext}"
```

## 使用示例

**导入 JSON 题库**:
```bash
/quiz-import ./questions/english-vocab.json
```

**导入 Markdown 题库**:
```bash
/quiz-import ./study-notes/math-questions.md
```

**导入 Excel 题库**:
```bash
/quiz-import ./data/questions.xlsx
```

**导入 Word 题库（使用 AI 提取）**:
```bash
/quiz-import ./documents/history-quiz.docx
```

**导入 PDF 题库（使用 AI 提取）**:
```bash
/quiz-import ./exams/science-test.pdf
```

**手动指定格式**:
```bash
/quiz-import ./unknown-file.data --format txt
```

## 格式转换技术细节

### Excel 转换

脚本使用 `openpyxl` 或 `pandas` 库：
```python
import pandas as pd

def convert_excel_to_json(file_path):
    df = pd.read_excel(file_path)
    # 识别列名（中英文）
    # 转换为标准 JSON 格式
    return json_output
```

### Word 转换

脚本使用 `python-docx` 库提取文本：
```python
from docx import Document

def extract_text_from_docx(file_path):
    doc = Document(file_path)
    text = '\n'.join([para.text for para in doc.paragraphs])
    return text  # 然后用 AI 提取
```

### PDF 转换

脚本使用 `PyPDF2` 或 `pdfplumber` 库：
```python
import pdfplumber

def extract_text_from_pdf(file_path):
    with pdfplumber.open(file_path) as pdf:
        text = '\n'.join([page.extract_text() for page in pdf.pages])
    return text  # 然后用 AI 提取
```

## AI 提取最佳实践

1. **文档结构越清晰，提取效果越好**：
   - 使用标题区分题目
   - 使用列表或编号组织选项
   - 明确标注答案和解析

2. **提高提取准确性**：
   - 使用一致的格式（如所有题目都用 "Q:" 开头）
   - 避免过于复杂的嵌套结构
   - 提供示例题目

3. **验证 AI 提取结果**：
   - 仔细检查预览
   - 特别注意选择题的选项顺序
   - 确认答案匹配正确

## 注意事项

1. **文件编码**: 确保文本文件使用 UTF-8 编码
2. **AI 依赖**: Word/PDF 导入需要 Claude API 调用
3. **大文件处理**: 超过 1000 题的文件可能需要分批处理
4. **扫描版 PDF**: 需要先 OCR 处理，不支持图片直接提取
5. **ID 唯一性**: 自动生成的 ID 使用 UUID 格式

## 依赖脚本

创建以下转换脚本在 `scripts/` 目录：
- `convert-excel.py`: Excel 转 JSON
- `convert-docx.py`: Word 文本提取
- `convert-pdf.py`: PDF 文本提取

## 相关技能

- `quiz-format-parser`: 提供详细的格式解析指导和示例
- `quiz-ai-extractor`: AI 智能提取最佳实践
- `quiz-learning-analytics`: 了解题目数据结构

## 相关命令

- `/quiz-list`: 查看已导入的题库
- `/quiz-start`: 使用题库开始答题
