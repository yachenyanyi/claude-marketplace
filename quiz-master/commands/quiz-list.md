---
name: quiz-list
description: 列出和管理所有已导入的题库，支持删除、重命名、合并等操作
allowed-tools:
  - Read
  - Write
  - Bash
argument-hint: [--action <list|delete|rename|merge>] [--bank <name>]
---

# Quiz Master - List and Manage Question Banks

你正在帮助用户管理已导入的题库。

## 任务步骤

### 1. 列出所有题库（默认操作）

使用 Bash 工具扫描题库目录：
```bash
ls -lh ~/.quiz-master/banks/*.json 2>/dev/null
```

如果目录不存在或为空：
```
📁 题库目录为空

还没有导入任何题库。

导入题库：
  /quiz-import <file-path>

支持格式：JSON, CSV, TXT, MD, Excel, Word, PDF
```

#### 显示题库列表

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 已导入的题库
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. english-vocab
   📝 题目数：150
   🏷️  标签：英语,词汇,水果
   📅 导入时间：2025-01-03
   📊 完成度：45% (68/150 题)
   ✅ 正确率：82%
   📂 文件：english-vocab.json

2. math-problems
   📝 题目数：200
   🏷️  标签：数学,代数,几何
   📅 导入时间：2025-01-02
   📊 完成度：23% (46/200 题)
   ✅ 正确率：71%
   📂 文件：math-problems.json

...

总计：{total_banks} 个题库，{total_questions} 道题目
```

#### 简洁模式

```
题库列表：
  english-vocab    150题  82%✓
  math-problems    200题  71%○
  history-quiz      80题  65%△
```

### 2. 查看题库详情

使用 `--bank <name>` 查看特定题库的详细信息：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 题库详情：english-vocab
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

基本信息：
  名称：english-vocab
  描述：常用英语词汇练习
  题目数：150
  标签：英语,词汇,水果,动物
  导入时间：2025-01-03 14:30:00
  原始格式：JSON

题目分布：
  选择题：100 (66.7%)
  填空题：30 (20.0%)
  问答题：20 (13.3%)

难度分布：
  简单：50 (33.3%)
  中等：80 (53.3%)
  困难：20 (13.3%)

学习进度：
  已答题：68/150 (45.3%)
  正确：56 (82.4%)
  错误：12 (17.6%)
  平均用时：3.2 秒/题

知识点掌握度：
  水果：90% ✓掌握
  动物：85% ○熟悉
  颜色：65% △生疏
  职业：40% ✗薄弱

管理操作：
  /quiz-list --action delete --bank english-vocab     删除题库
  /quiz-list --action rename --bank english-vocab     重命名题库
  /quiz-start english-vocab                          开始答题
```

### 3. 删除题库

使用 `--action delete --bank <name>` 删除题库：

#### 确认提示

```
⚠️  确认删除题库：{bank-name}

此操作将：
  - 删除题库文件：~/.quiz-master/banks/{bank-name}.json
  - 保留学习日志（答题记录不会丢失）

是否确认删除？(yes/no)
输入 'yes' 确认删除，或输入 'cancel' 取消
```

#### 删除成功

```
✅ 题库已删除

已删除：{bank-name} ({questions} 题)
学习日志已保留

如需重新导入，使用：
  /quiz-import <file-path>
```

#### 删除失败

```
❌ 删除失败

可能原因：
  - 题库文件不存在
  - 文件权限不足
  - 题库名称错误

可用题库：
  - {bank1}
  - {bank2}
```

### 4. 重命名题库

使用 `--action rename --bank <name>` 重命名题库：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✏️  重命名题库：{old-name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

请输入新名称（使用 kebab-case，如：english-words-level-1）:
```

#### 验证新名称

检查新名称是否：
- 遵循 kebab-case 命名规范
- 不与现有题库重名
- 不包含特殊字符

#### 重命名成功

```
✅ 题库已重命名

{old-name} → {new-name}

题库文件已更新
学习日志自动关联到新名称

开始答题：
  /quiz-start {new-name}
```

### 5. 合并题库

使用 `--action merge` 合并多个题库：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔀 合并题库
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

选择要合并的源题库（可多选）：
  [ ] english-vocab-fruit (50 题)
  [ ] english-vocab-animal (60 题)
  [ ] english-vocab-color (40 题)

目标题库名称：english-vocab-merged

确认合并？(yes/no)
```

#### 合并过程

1. 读取所有源题库
2. 去重（基于题目 ID 或内容相似度）
3. 合并标签
4. 生成新题库文件
5. 保留原题库（默认）或删除

#### 合并结果

```
✅ 题库合并完成

源题库：3 个
总题目：150 题
去重后：142 题
新题库：english-vocab-merged (142 题)

已保留原题库
如需删除源题库，使用：
  /quiz-list --action delete --bank <name>
```

### 6. 题库搜索

支持搜索功能：
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 搜索题库
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

搜索关键词：{keyword}

找到 {count} 个匹配的题库：
  - english-{keyword} (描述包含关键词)
  - {keyword}-practice (名称包含关键词)
  - advanced-questions (标签包含关键词)
```

### 7. 题库统计汇总

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 题库统计汇总
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

总题库数：{total_banks}
总题目数：{total_questions}
平均每库：{avg_questions} 题

最大题库：{largest_bank} ({count} 题)
最小题库：{smallest_bank} ({count} 题)

完成度分布：
  - 100% 完成：{count} 个
  - 50-99% 完成：{count} 个
  - <50% 完成：{count} 个

标签分布：
  - {tag1}: {count} 题
  - {tag2}: {count} 题
  ...
```

## 使用示例

**列出所有题库**:
```bash
/quiz-list
```

**查看题库详情**:
```bash
/quiz-list --bank english-vocab
```

**删除题库**:
```bash
/quiz-list --action delete --bank old-bank
```

**重命名题库**:
```bash
/quiz-list --action rename --bank english-vocab
```

**合并题库**:
```bash
/quiz-list --action merge
```

**搜索题库**:
```bash
/quiz-list --search math
```

## 参数说明

- **无参数**: 列出所有题库
- `--bank <name>`: 查看特定题库详情
- `--action list`: 列出题库（默认）
- `--action delete`: 删除题库
- `--action rename`: 重命名题库
- `--action merge`: 合并多个题库
- `--search <keyword>`: 搜索题库
- `--format simple`: 简洁格式
- `--format detail`: 详细格式（默认）

## 批量操作

支持批量删除和导出：
```bash
# 批量导出题库为 JSON
/quiz-list --export-all --format json

# 批量删除多个题库
/quiz-list --action delete --bank bank1,bank2,bank3
```

## 错误处理

**题库不存在**:
```
❌ 找不到题库：{bank-name}

可用题库：
  - {bank1}
  - {bank2}

查看所有题库：/quiz-list
```

**权限不足**:
```
❌ 权限不足

无法删除题库文件
请检查文件权限或使用管理员权限
```

**合并冲突**:
```
⚠️  合并冲突

发现 {count} 个重复题目
是否自动去重？(yes/no)
```

## 相关命令

- `/quiz-import`: 导入新题库
- `/quiz-start`: 使用题库开始答题
- `/quiz-status`: 查看题库学习进度

## 注意事项

1. 删除题库不会删除学习日志
2. 重命名题库会自动更新日志关联
3. 合并题库时建议先备份
4. 大题库（>500题）合并可能需要较长时间
