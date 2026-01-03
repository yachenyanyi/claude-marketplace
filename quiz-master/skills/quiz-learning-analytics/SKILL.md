---
name: quiz-learning-analytics
description: 当需要分析学习数据、计算题目优先级、识别薄弱知识点、生成学习报告或推荐学习策略时使用此技能
version: 1.0.0
---

# Quiz Learning Analytics

此技能提供学习数据分析、统计计算和智能推荐的算法和指导。当处理学习进度追踪、薄弱点分析或智能题目推荐时使用。

## 触发场景

此技能在以下场景中自动激活：

- 生成学习统计报告
- 分析薄弱知识点
- 计算题目优先级权重
- 推荐下次练习题目
- 评估学习进展趋势
- 预测学习效果

- 用户查看学习进度
- 用户询问薄弱点
- 系统需要智能推荐题目
- 生成个性化学习计划
- 分析学习习惯和模式

## 核心数据结构

### 学习日志格式

每次答题会话记录为：

```
[{timestamp}] Session Start - Bank: {bank}, Mode: {mode}
[{timestamp}] Question {id}: "{question}" - {Correct|Incorrect} ({time}s)
  - User Answer: {user_answer}
  - Correct Answer: {correct_answer}
[{timestamp}] Question {id}: "{question}" - {Correct|Incorrect} ({time}s)
  - User Answer: {user_answer}
  - Correct Answer: {correct_answer}
...
[{timestamp}] Session End - Total: {total}, Correct: {correct}, Accuracy: {rate}%
```

### 题目统计对象

```python
@dataclass
class QuestionStats:
    id: str                          # 题目ID
    bank: str                        # 所属题库
    attempts: int = 0                # 答题次数
    correct: int = 0                 # 正确次数
    incorrect: int = 0               # 错误次数
    skipped: int = 0                 # 跳过次数
    total_time: float = 0.0          # 总用时（秒）
    avg_time: float = 0.0            # 平均用时
    first_attempt: datetime = None   # 首次答题时间
    last_attempt: datetime = None    # 最后答题时间
    streak_incorrect: int = 0        # 连续错误次数
    streak_correct: int = 0          # 连续正确次数
    difficulty: str = "medium"       # 难度

    @property
    def accuracy(self) -> float:
        """正确率"""
        if self.attempts == 0:
            return 0.0
        return (self.correct / self.attempts) * 100

    @property
    def days_since_last_attempt(self) -> int:
        """距上次答题天数"""
        if self.last_attempt is None:
            return 9999
        return (datetime.now() - self.last_attempt).days
```

### 知识点统计对象

```python
@dataclass
class TagStats:
    tag: str                         # 知识点标签
    total_questions: int = 0         # 该标签题目总数
    attempts: int = 0                # 已答题数
    correct: int = 0                 # 正确数
    incorrect: int = 0               # 错误数
    accuracy: float = 0.0            # 正确率
    mastery_level: str = "unknown"   # 掌握程度

    def update_mastery(self):
        """更新掌握程度"""
        if self.attempts == 0:
            self.mastery_level = "unknown"
        elif self.accuracy >= 90:
            self.mastery_level = "mastered"      # 掌握
        elif self.accuracy >= 70:
            self.mastery_level = "familiar"      # 熟悉
        elif self.accuracy >= 50:
            self.mastery_level = "struggling"    # 生疏
        else:
            self.mastery_level = "weak"          # 薄弱
```

## 智能推荐算法

### 题目优先级计算

**权重公式**:

```python
def calculate_question_priority(stats: QuestionStats) -> float:
    """
    计算题目优先级权重

    权重越高，越应该优先练习
    """
    # 错误权重：每次错误加 2 分
    error_weight = stats.incorrect * 2.0

    # 时间权重：距上次答题每天加 0.5 分
    time_weight = stats.days_since_last_attempt * 0.5

    # 连续错误奖励：连续错误每次加 1.5 分
    streak_weight = stats.streak_incorrect * 1.5

    # 难度调整：困难题目额外加 0.5 分
    difficulty_weight = {
        "easy": 0.0,
        "medium": 0.3,
        "hard": 0.5
    }.get(stats.difficulty, 0.3)

    # 正确率惩罚：正确率高的降低优先级
    if stats.attempts >= 3:
        accuracy_penalty = (stats.accuracy / 100) * 2.0
    else:
        accuracy_penalty = 0.0

    # 综合权重
    priority = (
        error_weight +
        time_weight +
        streak_weight +
        difficulty_weight -
        accuracy_penalty
    )

    return max(0.0, priority)  # 确保非负
```

**使用示例**:

```python
# 统计所有题目
all_stats = get_all_question_stats()

# 计算优先级
for stats in all_stats:
    stats.priority = calculate_question_priority(stats)

# 按优先级排序
sorted_questions = sorted(all_stats, key=lambda x: x.priority, reverse=True)

# 选择 Top N
recommended = sorted_questions[:10]
```

### 知识点优先级计算

```python
def calculate_tag_priority(tag_stats: TagStats) -> float:
    """
    计算知识点优先级

    考虑因素：
    - 正确率（越低越优先）
    - 答题数量（太少的不准确）
    - 错误集中度
    """
    # 基础优先级：正确率越低越高
    base_priority = (100 - tag_stats.accuracy) / 100

    # 答题数调整：至少3题才准确
    if tag_stats.attempts < 3:
        sample_penalty = 0.3
    else:
        sample_penalty = 0.0

    # 错误集中度：最近错误占比较高则提升优先级
    recent_errors = get_recent_error_count(tag_stats.tag, days=7)
    concentration_bonus = min(recent_errors / tag_stats.attempts, 0.5)

    priority = base_priority - sample_penalty + concentration_bonus

    return max(0.0, min(1.0, priority))
```

### 遗忘曲线应用

基于艾宾浩斯遗忘曲线：

```python
def should_review_question(stats: QuestionStats) -> bool:
    """
    判断是否应该复习该题目
    """
    days_since = stats.days_since_last_attempt

    # 根据掌握程度决定复习间隔
    if stats.accuracy >= 90:
        # 已掌握：30天后复习
        return days_since >= 30
    elif stats.accuracy >= 70:
        # 熟悉：7天后复习
        return days_since >= 7
    elif stats.accuracy >= 50:
        # 生疏：3天后复习
        return days_since >= 3
    else:
        # 薄弱：1天后复习
        return days_since >= 1

def get_review_questions(all_stats: List[QuestionStats], count: int) -> List[QuestionStats]:
    """
    获取需要复习的题目
    """
    due_for_review = [s for s in all_stats if should_review_question(s)]

    # 按优先级排序
    due_for_review.sort(key=lambda x: calculate_question_priority(x), reverse=True)

    return due_for_review[:count]
```

## 统计分析

### 基本统计指标

```python
def calculate_basic_stats(all_stats: List[QuestionStats]) -> dict:
    """计算基本统计数据"""
    total_questions = len(all_stats)
    attempted = [s for s in all_stats if s.attempts > 0]

    if not attempted:
        return {
            "total_questions": total_questions,
            "attempted": 0,
            "unattempted": total_questions,
            "overall_accuracy": 0.0,
            "avg_time": 0.0
        }

    total_attempts = sum(s.attempts for s in attempted)
    total_correct = sum(s.correct for s in attempted)
    total_time = sum(s.total_time for s in attempted)

    return {
        "total_questions": total_questions,
        "attempted": len(attempted),
        "unattempted": total_questions - len(attempted),
        "overall_accuracy": (total_correct / total_attempts) * 100 if total_attempts > 0 else 0.0,
        "avg_time": total_time / total_attempts if total_attempts > 0 else 0.0,
        "total_time_minutes": total_time / 60
    }
```

### 知识点分布分析

```python
def analyze_tag_distribution(all_stats: List[QuestionStats], bank_data: dict) -> List[TagStats]:
    """分析知识点分布"""
    tag_stats_map = {}

    for stats in all_stats:
        # 从题库获取题目标签
        question = find_question_by_id(bank_data, stats.id)
        if not question:
            continue

        for tag in question.get("tags", []):
            if tag not in tag_stats_map:
                tag_stats_map[tag] = TagStats(tag=tag)

            tag_stats_map[tag].total_questions += 1
            if stats.attempts > 0:
                tag_stats_map[tag].attempts += stats.attempts
                tag_stats_map[tag].correct += stats.correct
                tag_stats_map[tag].incorrect += stats.incorrect

    # 计算正确率和掌握程度
    for tag_stats in tag_stats_map.values():
        if tag_stats.attempts > 0:
            tag_stats.accuracy = (tag_stats.correct / tag_stats.attempts) * 100
        tag_stats.update_mastery()

    # 按优先级排序
    sorted_tags = sorted(tag_stats_map.values(), key=lambda x: calculate_tag_priority(x), reverse=True)

    return sorted_tags
```

### 时间趋势分析

```python
def analyze_time_trends(logs: List[str], days: int = 7) -> dict:
    """分析最近N天的学习趋势"""
    from datetime import datetime, timedelta

    cutoff_date = datetime.now() - timedelta(days=days)
    daily_data = {}

    # 解析日志
    for log_entry in parse_logs(logs):
        if log_entry.timestamp < cutoff_date:
            continue

        date_str = log_entry.timestamp.strftime("%Y-%m-%d")
        if date_str not in daily_data:
            daily_data[date_str] = {
                "questions": 0,
                "correct": 0,
                "total_attempts": 0,
                "time": 0.0
            }

        daily_data[date_str]["questions"] += 1
        daily_data[date_str]["total_attempts"] += 1
        if log_entry.is_correct:
            daily_data[date_str]["correct"] += 1
        daily_data[date_str]["time"] += log_entry.time_taken

    # 计算每日正确率
    for date_data in daily_data.values():
        if date_data["total_attempts"] > 0:
            date_data["accuracy"] = (date_data["correct"] / date_data["total_attempts"]) * 100
        else:
            date_data["accuracy"] = 0.0

    return daily_data
```

### 学习习惯分析

```python
def analyze_study_habits(all_stats: List[QuestionStats], logs: List[str]) -> dict:
    """分析学习习惯"""
    # 学习频率
    sessions = parse_sessions(logs)
    study_dates = [s.timestamp.date() for s in sessions]

    if study_dates:
        unique_dates = len(set(study_dates))
        date_range = (max(study_dates) - min(study_dates)).days + 1
        avg_daily_questions = len(all_stats) / max(unique_dates, 1)

        # 最佳时段
        hour_counts = {}
        for session in sessions:
            hour = session.timestamp.hour
            hour_counts[hour] = hour_counts.get(hour, 0) + 1

        best_hour = max(hour_counts.items(), key=lambda x: x[1])[0] if hour_counts else None
    else:
        avg_daily_questions = 0
        best_hour = None

    # 答题速度
    avg_time = sum(s.avg_time for s in all_stats if s.attempts > 0) / len(all_stats) if all_stats else 0

    # 学习连续性
    streak = calculate_study_streak(study_dates)

    return {
        "avg_daily_questions": avg_daily_questions,
        "best_hour": best_hour,
        "avg_time_per_question": avg_time,
        "current_streak": streak,
        "total_study_days": unique_dates if study_dates else 0
    }
```

## 薄弱点识别

### 薄弱知识点识别

```python
def identify_weak_tags(tag_stats: List[TagStats], top_n: int = 10) -> List[dict]:
    """识别Top N薄弱知识点"""
    weak_tags = []

    for tag_stat in tag_stats:
        # 筛选条件
        if tag_stat.attempts >= 3 and tag_stat.accuracy < 70:
            weak_tags.append({
                "tag": tag_stat.tag,
                "accuracy": tag_stat.accuracy,
                "attempts": tag_stat.attempts,
                "mastery_level": tag_stat.mastery_level,
                "priority_score": calculate_tag_priority(tag_stat),
                "questions": get_questions_by_tag(tag_stat.tag)
            })

    # 按优先级排序
    weak_tags.sort(key=lambda x: x["priority_score"], reverse=True)

    return weak_tags[:top_n]
```

### 薄弱题目识别

```python
def identify_weak_questions(all_stats: List[QuestionStats], threshold: int = 3) -> List[dict]:
    """识别需要重点练习的题目"""
    weak_questions = []

    for stats in all_stats:
        # 薄弱题目：错误次数>=阈值 且 正确率<70%
        if stats.incorrect >= threshold and stats.accuracy < 70:
            weak_questions.append({
                "id": stats.id,
                "bank": stats.bank,
                "attempts": stats.attempts,
                "incorrect": stats.incorrect,
                "accuracy": stats.accuracy,
                "days_since_last": stats.days_since_last_attempt,
                "priority_score": calculate_question_priority(stats),
                "question_text": get_question_text(stats.id)
            })

    # 按优先级排序
    weak_questions.sort(key=lambda x: x["priority_score"], reverse=True)

    return weak_questions
```

## 学习报告生成

### 生成详细报告

```python
def generate_learning_report(user_data: dict) -> dict:
    """生成完整学习报告"""
    all_stats = get_all_question_stats()
    tag_stats = analyze_tag_distribution(all_stats, user_data["banks"])
    weak_tags = identify_weak_tags(tag_stats, top_n=10)
    weak_questions = identify_weak_questions(all_stats)
    basic_stats = calculate_basic_stats(all_stats)
    habits = analyze_study_habits(all_stats, user_data["logs"])
    trends = analyze_time_trends(user_data["logs"], days=7)

    return {
        "generated_at": datetime.now().isoformat(),
        "basic_stats": basic_stats,
        "tag_mastery": [
            {
                "tag": t.tag,
                "accuracy": t.accuracy,
                "mastery_level": t.mastery_level,
                "questions": t.total_questions,
                "attempted": t.attempts
            }
            for t in tag_stats
        ],
        "weak_points": {
            "tags": weak_tags,
            "questions": weak_questions[:20]
        },
        "study_habits": habits,
        "recent_trends": trends,
        "recommendations": generate_recommendations(weak_tags, weak_questions, habits)
    }
```

### 生成个性化建议

```python
def generate_recommendations(weak_tags: List, weak_questions: List, habits: dict) -> List[str]:
    """生成个性化学习建议"""
    recommendations = []

    # 基于薄弱知识点
    if weak_tags:
        top_weak = weak_tags[0]
        recommendations.append(
            f"重点练习「{top_weak['tag']}」知识点，正确率仅 {top_weak['accuracy']:.1f}%"
        )

    # 基于学习频率
    if habits["avg_daily_questions"] < 10:
        recommendations.append(
            f"建议每天练习至少10题，当前平均每天 {habits['avg_daily_questions']:.1f} 题"
        )

    # 基于答题速度
    if habits["avg_time_per_question"] > 10:
        recommendations.append(
            f"答题速度偏慢（平均 {habits['avg_time_per_question']:.1f} 秒/题），建议加强基础知识练习"
        )

    # 基于学习连续性
    if habits["current_streak"] >= 7:
        recommendations.append(
            f"太棒了！已连续学习 {habits['current_streak']} 天，继续保持"
        )

    return recommendations
```

## 数据可视化

### 文本图表生成

```python
def generate_text_chart(data: dict, chart_type: str = "bar") -> str:
    """生成文本图表"""
    if chart_type == "bar":
        return generate_bar_chart(data)
    elif chart_type == "line":
        return generate_line_chart(data)
    elif chart_type == "progress":
        return generate_progress_bar(data)

def generate_progress_bar(value: int, total: int, width: int = 20) -> str:
    """生成进度条"""
    filled = int((value / total) * width)
    bar = "█" * filled + "░" * (width - filled)
    return f"{bar} {value}/{total} ({value/total*100:.1f}%)"

def generate_mastery_matrix(tag_stats: List[TagStats]) -> str:
    """生成知识点掌握度矩阵"""
    lines = ["知识点          题目数  正确率  掌握度", "─" * 50]

    for tag_stat in tag_stats[:15]:
        mastery_symbol = {
            "mastered": "✓掌握",
            "familiar": "○熟悉",
            "struggling": "△生疏",
            "weak": "✗薄弱",
            "unknown": "?未知"
        }.get(tag_stat.mastery_level, "?未知")

        lines.append(
            f"{tag_stat.tag:<15} {tag_stat.total_questions:>3}  "
            f"{tag_stat.accuracy:>6.1f}%  {mastery_symbol}"
        )

    return "\n".join(lines)
```

## 性能优化

### 日志解析优化

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def parse_log_file_cached(file_path: str) -> List[LogEntry]:
    """缓存解析结果"""
    return parse_log_file(file_path)

def get_all_stats_fast() -> dict:
    """快速获取所有统计（使用缓存）"""
    log_files = glob.glob("~/.quiz-master/logs/*.log")

    all_entries = []
    for log_file in log_files:
        all_entries.extend(parse_log_file_cached(log_file))

    return aggregate_stats(all_entries)
```

### 增量更新

```python
last_update_time = None

def get_stats_incremental() -> dict:
    """增量更新统计（只处理新日志）"""
    global last_update_time

    new_logs = get_logs_after(last_update_time)

    if new_logs:
        update_stats(new_logs)
        last_update_time = datetime.now()

    return get_current_stats()
```

## 最佳实践

1. **数据准确性**:
   - 验证日志完整性
   - 处理异常数据
   - 定期备份数据

2. **统计有效性**:
   - 确保样本量足够（至少3题）
   - 考虑时间因素
   - 综合多个指标

3. **推荐合理性**:
   - 平衡新旧题目
   - 避免过度重复
   - 考虑用户疲劳

4. **报告可读性**:
   - 使用可视化图表
   - 突出关键信息
   - 提供行动建议

5. **性能考虑**:
   - 缓存计算结果
   - 增量更新数据
   - 延迟加载大文件

## 参考资源

查看 `references/` 目录获取：
- 算法详细说明
- 统计公式推导
- 可视化示例代码
- 性能优化技巧

相关技能：
- `quiz-format-parser`: 题库数据结构
