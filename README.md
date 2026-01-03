# Claude Code Marketplace - 个人插件市场

这是我的个人 Claude Code 插件市场，收录了我开发和维护的 Claude Code 插件。

## 📦 已收录插件

### Quiz Master

**智能刷题学习插件** - 帮助你高效学习和掌握知识

- **功能特性**:
  - 📚 支持多种题库格式 (JSON/CSV/TXT/MD/Excel/Word/PDF)
  - 🎯 AI 智能提取题目
  - 🧠 基于遗忘曲线的智能推荐
  - 📊 学习数据分析和薄弱点识别
  - ⚡ 交互式答题和即时反馈

- **命令**:
  - `/quiz-import` - 导入题库
  - `/quiz-start` - 开始答题
  - `/quiz-status` - 查看进度
  - `/quiz-analyze` - 分析薄弱点
  - `/quiz-list` - 管理题库

- **仓库**: https://github.com/yachenyanyi/claude-tools

## 🚀 如何使用

### 添加这个市场

在 Claude Code 中：

1. 打开插件管理界面
2. 选择 "Add Marketplace"
3. 输入: `https://github.com/yachenyanyi/claude-marketplace.git`

### 安装插件

添加市场后，你可以：

1. 在插件列表中浏览可用插件
2. 点击安装需要的插件
3. 插件会自动安装到本地

## 🛠️ 贡献指南

### 添加新插件

如果你想在这个市场添加新插件：

1. Fork 这个仓库
2. 在 `marketplace.json` 的 `plugins` 数组中添加插件信息
3. 提交 Pull Request

### 插件信息格式

```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "插件描述",
  "homepage": "https://github.com/username/plugin",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/plugin.git"
  },
  "keywords": ["keyword1", "keyword2"],
  "features": ["特性1", "特性2"],
  "commands": [...],
  "skills": [...],
  "agents": [...],
  "install": {
    "type": "git",
    "url": "插件仓库地址"
  }
}
```

## 📝 许可证

MIT License

## 🔗 相关链接

- [Claude Code 官方文档](https://docs.claude.com)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)

---

**维护者**: yachenyanyi
