---
name: quiz-ai-extractor
description: 当需要从非结构化文档（Word、PDF、扫描件等）中智能提取题目时使用此技能，利用 AI 理解文档内容并将其转换为标准题库格式
version: 1.0.0
---

# Quiz AI Extractor

此技能提供使用 AI 智能提取题目的指导和最佳实践。当处理非结构化或复杂格式的文档时使用。

## 触发场景

此技能在以下场景中自动激活：

- 用户从 Word 文档（.docx）导入题库
- 用户从 PDF 文档导入题库
- 文档格式不规范或结构复杂
- 需要理解上下文才能提取题目
- 文档包含多种格式混合
- 文档是扫描版图片

- 用户询问如何从 Word/PDF 提取题目
- AI 提取结果不准确需要优化
- 用户想提高提取准确率
- 用户需要处理特殊格式的文档

## AI 提取原理

### 为什么需要 AI？

非结构化文档的特点：
- 格式不统一（题目、选项、答案格式多样）
- 布局复杂（分栏、表格、图片混排）
- 需要理解语义（区分题目和解析、识别正确答案）
- 包含隐含信息（题号、难度标记）

AI 的优势：
- 语义理解：理解题目内容
- 模式识别：识别题目结构
- 容错能力：处理格式错误
- 上下文推理：推断缺失信息

### 提取流程

```
文档 → 文本提取 → AI 分析 → 结构化数据 → 标准化 → 验证 → JSON
```

**步骤说明**:
1. **文本提取**: 使用脚本提取纯文本（PDF/Word）
2. **AI 分析**: 使用 Claude 理解内容并提取题目
3. **结构化**: 组织为标准 JSON 格式
4. **验证**: 检查完整性和正确性
5. **人工确认**: 预览前5题让用户确认

## 文本提取

### Word 文档（.docx）

使用 `python-docx` 库：

```python
from docx import Document

def extract_text_from_docx(file_path):
    """提取 Word 文档文本"""
    doc = Document(file_path)
    text_content = []

    # 提取段落
    for para in doc.paragraphs:
        if para.text.strip():
            text_content.append(para.text)

    # 提取表格
    for table in doc.tables:
        for row in table.rows:
            row_text = [cell.text for cell in row.cells]
            text_content.append(' | '.join(row_text))

    return '\n'.join(text_content)
```

**注意事项**:
- 表格内容需要特殊处理
- 图片中的文字无法提取（需要 OCR）
- 样式信息会丢失

### PDF 文档（.pdf）

使用 `pdfplumber` 库：

```python
import pdfplumber

def extract_text_from_pdf(file_path):
    """提取 PDF 文本内容"""
    text_content = []

    with pdfplumber.open(file_path) as pdf:
        for page in pdf.pages:
            text = page.extract_text()
            if text:
                text_content.append(text)

    # 提取表格
        for page in pdf.pages:
            tables = page.extract_tables()
            for table in tables:
                for row in table:
                    text_content.append(' | '.join(row))

    return '\n'.join(text_content)
```

**注意事项**:
- 多栏布局可能混乱
- 扫描版 PDF 需要 OCR
- 表格需要单独提取

### OCR 处理（扫描版文档）

使用 `pytesseract` + `Pillow`:

```python
from PIL import Image
import pytesseract

def ocr_image(image_path):
    """OCR 提取图片文字"""
    image = Image.open(image_path)

    # 提取文字
    text = pytesseract.image_to_string(
        image,
        lang='chi_sim+eng',  # 中英文混合
        config='--psm 6'     # 假设单列文本
    )

    return text
```

**注意事项**:
- OCR 准确率受图片质量影响
- 需要先转换 PDF 页面为图片
- 复杂布局识别效果较差

## AI 提取提示工程

### 基础提示模板

```
请从以下文档内容中提取所有题目，转换为 JSON 格式。

文档内容：
{document_content}

输出格式要求：
{
  "name": "题库名称（从文档标题推断）",
  "description": "题库描述",
  "tags": ["标签1", "标签2"],
  "questions": [
    {
      "id": "唯一ID（自动生成，如 q1, q2, q3）",
      "type": "choice|fillblank|essay",
      "question": "题目内容（完整准确）",
      "options": ["选项A", "选项B", "选项C", "选项D"],
      "answer": "正确答案",
      "explanation": "解析说明（如果有）",
      "tags": ["知识点标签"],
      "difficulty": "easy|medium|hard"
    }
  ]
}

提取要求：
1. 提取所有题目，不要遗漏
2. 准确识别题目类型（选择题、填空题、问答题）
3. 保留选项顺序和正确答案
4. 提取解析和标签（如果有）
5. 推断难度等级（基于题目复杂度）
6. 只输出 JSON，不要包含其他文字

请开始提取：
```

### 优化提示模板

**针对选择题**:
```
文档包含选择题。请注意：
1. 选项可能是 A. B. C. D. 或 1. 2. 3. 4.
2. 正确答案可能标记为 ✓、粗体、或单独说明
3. 仔细区分题目和选项
```

**针对判断题**:
```
文档包含判断题。请注意：
1. 判断题应归类为 choice 类型
2. options 固定为 ["正确", "错误"] 或 ["对", "错"]
3. 根据答案或标记判断正确选项
```

**针对填空题**:
```
文档包含填空题。请注意：
1. 填空题类型为 fillblank
2. 答案可能有多变体（同义词），都用数组表示
3. 去除括号、下划线等填空标记
```

**针对混合格式**:
```
文档包含多种题型混合。请根据题目特征自动分类：
- 有选项 → choice
- 有填空标记 → fillblank
- 其他 → essay

保持题目顺序，不要重排。
```

## 提高提取准确率

### 文档预处理

**清理格式**:
```python
def clean_document_text(raw_text):
    """清理文档文本"""
    # 移除页眉页脚
    text = re.sub(r'第\s*\d+\s*页', '', text)

    # 统一换行符
    text = text.replace('\r\n', '\n')

    # 移除多余空格
    text = re.sub(r' +', ' ', text)

    # 保留题目结构标记
    text = text.replace('．', '.')  # 统一点号
    text = text.replace('（', '(')
    text = text.replace('）', ')')

    return text
```

**分段处理**:
```python
def split_into_sections(text):
    """将文档分割为题目段落"""
    # 按题号分割（1.、2. 或 一、二、）
    sections = re.split(r'(?:^|\n)\s*(?:\d+\.|[一二三四五六七八九十]+[、.])', text)

    return [s.strip() for s in sections if s.strip()]
```

### 分步提取策略

**策略1: 先识别结构，再提取内容**
```
第1步：识别文档中有多少题，分别是什么类型
第2步：逐题提取详细信息
第3步：整合为 JSON
```

**策略2: 先提取关键词，再补全细节**
```
第1步：提取所有题目文本
第2步：提取所有选项和答案
第3步：配对并验证
```

**策略3: 先提取选择题，再处理其他题型**
```
第1步：提取所有选择题（有选项的）
第2步：提取填空题（有填空标记的）
第3步：剩余的作为问答题
```

### 验证和修正

**自动验证**:
```python
def validate_extracted_quiz(quiz_data):
    """验证提取的题库"""
    issues = []

    for idx, q in enumerate(quiz_data.get("questions", [])):
        # 检查必需字段
        if not q.get("question"):
            issues.append(f"题目 {idx+1}: 缺少 question")

        if not q.get("answer"):
            issues.append(f"题目 {idx+1}: 缺少 answer")

        # 检查选择题选项
        if q.get("type") == "choice":
            if not q.get("options") or len(q["options"]) < 2:
                issues.append(f"题目 {idx+1}: 选择题选项不足")

        # 检查答案在选项中
        if q.get("type") == "choice" and q.get("options"):
            if q["answer"] not in q["options"]:
                issues.append(f"题目 {idx+1}: 答案不在选项中")

    return issues
```

**人工修正建议**:
```
⚠️  发现 {issues_count} 个问题

问题列表：
1. 题目 3: 缺少 answer
   - 建议：手动补充答案或删除该题

2. 题目 7: 答案 "B" 不在选项中
   - 选项：["A. 苹果", "B. 香蕉", "C. 橙子"]
   - 建议：检查答案是否应该是 "B. 香蕉"

是否继续导入？(y/n)
y - 跳过问题题目，导入其他
n - 取消，手动修正后重试
```

## 常见问题处理

### Q: 题号不连续怎么办？

**A**: 重新编号或保留原号：

```python
def renumber_questions(questions):
    """重新编号题目"""
    for idx, q in enumerate(questions, start=1):
        # 保留原ID为metadata
        if "id" in q:
            q.setdefault("metadata", {})["original_id"] = q["id"]

        # 生成新ID
        q["id"] = f"q{idx}"

    return questions
```

### Q: 选项格式不一致怎么办？

**A**: 规范化选项格式：

```python
def normalize_options(options):
    """规范化选项"""
    normalized = []

    for opt in options:
        # 移除前缀 "A. ", "1、", etc.
        opt = re.sub(r'^[A-D]\.?\s*|^\d+[、.]\s*', '', opt)

        # 去除多余空格
        opt = opt.strip()

        if opt:
            normalized.append(opt)

    return normalized
```

### Q: 答案格式不明确怎么办？

**A**: 尝试多种解析方式：

```python
def parse_answer(answer_text, question_type, options=None):
    """解析答案"""
    if question_type == "choice":
        # 可能是 "A", "A. xxx", 或 "xxx"
        if answer_text.strip() in ["A", "B", "C", "D"]:
            # 如果有选项，返回完整选项
            if options:
                idx = ord(answer_text) - ord('A')
                return options[idx]
            return answer_text

        # 尝试匹配选项内容
        if options:
            for opt in options:
                if answer_text in opt or opt in answer_text:
                    return opt

        return answer_text

    return answer_text
```

### Q: 如何处理图片中的题目？

**A**: 使用 OCR + AI：

```python
def extract_quiz_from_images(image_paths):
    """从图片提取题库"""
    all_text = []

    for img_path in image_paths:
        # OCR 提取
        text = ocr_image(img_path)
        all_text.append(text)

    # 合并文本
    combined_text = '\n'.join(all_text)

    # AI 提取
    return ai_extract_questions(combined_text)
```

## 提取脚本

脚本位置：`C:\Users\Administrator\.claude\plugins\cache\claude-plugins-official\plugin-dev\4fee769bf4ec/scripts/`

### convert-docx.py

```python
#!/usr/bin/env python3
"""
Word 文档转题库 JSON
提取文本后使用 AI 解析
"""

from docx import Document
import json
import sys

def main(docx_path):
    # 提取文本
    doc = Document(docx_path)
    text = '\n'.join([p.text for p in doc.paragraphs if p.text.strip()])

    # 输出文本供 AI 处理
    print("=== 文档内容 ===")
    print(text)
    print("\n=== 请将以上内容发送给 Claude 进行提取 ===")

if __name__ == "__main__":
    main(sys.argv[1])
```

### convert-pdf.py

```python
#!/usr/bin/env python3
"""
PDF 文档转题库 JSON
提取文本后使用 AI 解析
"""

import pdfplumber
import sys

def main(pdf_path):
    text_content = []

    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            text = page.extract_text()
            if text:
                text_content.append(text)

    # 输出文本供 AI 处理
    print("=== 文档内容 ===")
    print('\n'.join(text_content))
    print("\n=== 请将以上内容发送给 Claude 进行提取 ===")

if __name__ == "__main__":
    main(sys.argv[1])
```

## 最佳实践

1. **文档准备**:
   - 使用清晰的标题和格式
   - 统一的题目编号
   - 明确标记答案和解析

2. **提示优化**:
   - 根据文档类型调整提示
   - 提供具体的提取要求
   - 包含示例格式

3. **验证确认**:
   - 始终预览前5题
   - 检查关键题目
   - 保留原始文档备查

4. **迭代改进**:
   - 记录提取失败案例
   - 优化提示模板
   - 建立常见问题库

5. **性能考虑**:
   - 大文档分批处理（每次5000字）
   - 缓存提取结果
   - 使用流式 API

## 参考资源

查看 `references/` 目录获取：
- 提示模板示例
- 文档格式指南
- 常见问题解决方案
- OCR 配置说明

相关技能：
- `quiz-format-parser`: 格式解析和验证
