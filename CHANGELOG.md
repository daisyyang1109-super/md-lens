# Changelog

## [0.2.0] - 2026-05-18 · --vault flag · multi-md library 管理

### 新增

- **`--vault <dir>` mode** — 一个目录下全部 .md 一站式管理,起本地 HTTP server(stdlib `http.server`,仍 0 依赖)
- **Sidebar 文件树**:嵌套目录 collapsible,目录展开状态记 localStorage
- **In-memory 全文搜索**:启动时一次 fetch 全部 md content 到前端,后续搜索 instant(< 100ms),命中高亮
- **SPA 文件切换**:点 file → fetch → marked 渲染 → URL hash 同步(`#file=path/to.md`),刷新页面恢复当前文件
- **per-file 批注**:每个 .md 独立 localStorage key,key 格式 `mdlens:vault:<hash>:annotations:<rel_path>`
- **保存回 vault**:编辑后点「💾 保存到 vault」直接写回原 .md 文件(POST `/api/save`,server 端有 path traversal 防护)

### CLI

```
md-lens some.md                  # 单文件 mode(不变,backward compat)
md-lens --vault ~/projects/docs  # 新:vault mode
md-lens --vault . --port 8080    # 可选端口(默认 8000)
md-lens --install                # 不变
```

`--vault` 跟 positional path 互斥;给一个或都不给会报错。

### HTTP endpoints(vault mode)

| endpoint | 说明 |
|---|---|
| `GET /` | vault SPA HTML(sidebar + content panel) |
| `GET /api/files` | JSON list: `{files: [{path, size, mtime}], total_size, count}` |
| `GET /api/file?path=X` | 单文件 raw md + title(从 H1 提取) |
| `GET /api/files-content` | 全部 md content 一次性 ship,供前端 in-memory search |
| `POST /api/save?path=X` | 保存 raw md(body 是 markdown 文本) |

### 设计原则保持

- **零依赖**:vault 用 Python stdlib `http.server` + `urllib.parse`,前端仍 CDN marked.js + turndown.js
- **单文件可移植**:仍是一个 `md-lens.py`(0.2.0 加到 1304 行,395 净 add)
- **localStorage scope**:vault hash + relative path 双重 namespace,不同 vault / 不同文件 互不污染
- **单文件 mode 100% backward compat**:`md-lens some.md` 输出 .html 行为完全不变

### 性能(实测 vault 89 files / 1.7 MB)

- `/api/files`: < 50ms
- `/api/file` (52 KB md): < 25ms
- 启动到首屏可交互: < 1s
- 前端 search 输入 → results: < 100ms(in-memory)

### 已知限制

- vault > 1000 file 时 `/api/files-content` 一次性 ship 可能慢(future PR 改 lazy + indexed search)
- save 是 overwrite 直接覆盖,无版本控制(用户应 git 管理 vault)
- 目录展开状态 search 重渲染后未精确恢复(localStorage 仍记,下次启动可读)

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
