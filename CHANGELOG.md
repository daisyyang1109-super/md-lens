# Changelog

## [0.1.1] - 2026-05-15 · 内置 --install 一键装

### 新增
- **`python3 md-lens.py --install`** — 自动 cp 到 `~/.claude/md-lens/` 标准位置,提示 alias 配置
- 之后任意位置 `md-lens some-doc.md` 即可

### 改进
- README 优先推荐 `--install` 方式

## [0.1.0] - 2026-05-15 · Initial release

### 主要功能

- **Markdown 渲染**(`md-lens.py`)
  - 单文件 Python + 浏览器端 marked.js 渲染
  - 左侧 TOC 浮动 sidebar + IntersectionObserver scroll spy
  - 阅读优化样式(字体 / 间距 / 表格 / 代码块 / 引用)
  - Mermaid 流程图渲染

- **批注系统**(4 类)
  - 好 / 不好 / 建议 / 疑问
  - 上下文锚定 + nth-match,避免短词误匹配
  - 侧边面板按类型分组 + 定位 / 编辑 / 删除
  - localStorage 持久化
  - 导出 JSON(给 AI 改文档)或 Markdown 摘要

- **编辑模式**
  - contenteditable + 浮动格式工具栏
  - H2 / H3 / P / Bold / Italic / Code / 引用 / 列表 / 链接
  - TOC 实时跟随编辑(debounce 400ms)
  - localStorage 草稿自动保存(防丢失)
  - 跨 tab 冲突警告

- **导出**
  - 导出修改后的 .md(turndown.js 反向转换)
  - 导出批注 JSON
  - 复制批注 Markdown 摘要

### 设计原则

- **零依赖**:Python 标准库 + CDN 加载 marked.js,clone 完即跑
- **单文件可移植**:875 行 Python
- **localStorage 优先**:无 server,数据用户自己控制
