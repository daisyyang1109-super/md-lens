# Brief · md-lens Multi-MD Library 管理

**Status**: AUTONOMOUS EXECUTION · no daisy confirm gate
**Created**: 2026-05-18
**Branch**: `feat/multi-md-library`(已 pulled,可直接 work)
**Mode**: 你自己 read · design · code · test · commit · push · 不需要等 daisy confirm

---

## 任务

给 md-lens 加 **multi-md library 管理功能**,加 `--vault <dir>` flag。

她痛点:`daisy-rpg/docs/` 等项目里 30+ 个 md 文件,**跨文件 navigation / 找 / 关联看不到 → 乱**。Obsidian 太重,要 md-lens 风格的 single-file 0-dep + multi-md vault 能力。

她现在睡了,**你 autonomous 跑完,明早她直接 review PR**。

---

## 项目背景

- **位置**:`~/projects/md-lens/`
- **核心文件**:`md-lens.py`(单文件 Python + 浏览器端,41KB,~1000+ 行)
- **哲学**:single-file · 0 dependency · clone & run
- **当前 branch**:`feat/multi-md-library`(已切换)
- **CHANGELOG**:`v0.1.1` 已 ship

---

## 详细 design spec(不需 daisy 确认,按此 implement)

### CLI

```bash
md-lens some.md                       # 单文件 mode(不变,backward compat)
md-lens --vault ~/daisy-rpg/docs      # 新:vault mode
md-lens --vault . --port 8080         # 可选 port
md-lens --install                     # 不变
```

`argparse`:
- Positional `path`(单文件 path,可选)
- `--vault <dir>`(vault 目录,跟 positional 互斥)
- `--port` 默认 8000

如果两者都给 → 报错"`--vault` 跟 positional path 互斥"。
如果都没给 → 报错"required: path or --vault"。

### Vault mode 启动流程

```
1. walk vault dir → collect 所有 .md 文件(skip hidden / .git / node_modules)
2. 启 HTTP server on port
3. 渲染 vault HTML(left sidebar + right content panel)
4. SPA 模式:点 file → fetch /api/file?path=X → JS swap content
5. 打开浏览器 → http://localhost:PORT/
```

### HTTP endpoints(vault mode)

- `GET /` → vault HTML(sidebar + empty content)
- `GET /api/files` → JSON 列表所有 .md(相对 path + 文件大小 + mtime)
- `GET /api/file?path=X` → 单个 md 的 rendered HTML + TOC + metadata(`{html, toc, title, raw_size}`)
- `GET /api/search?q=X` → JSON 全文 hits(`[{file, line, snippet, count}]`)
- `POST /api/save?path=X` → save edited md(body 是 raw md)
- 单文件 mode 不破坏:`GET /` 仍然返回单文件 HTML(根据启动模式 branch)

### Sidebar UI

```
┌─ md-lens vault ──────────────────┐
│ 🔍 [search box]                   │
│                                   │
│ ▼ docs/                           │
│   ▼ design/                       │
│     • CHARACTER.md                │
│     • LORE.md ←(highlighted current)│
│   ▼ briefs/                       │
│     • brief-1.md                  │
│ ▼ archive/                        │
│   ...                             │
│                                   │
│ ── footer ──                      │
│ 30 files · 2.3 MB · ~5400 lines  │
└───────────────────────────────────┘
```

实现细节:
- 左侧 fixed sidebar 宽 240px,右侧 content panel 自适应
- 文件树:嵌套目录 collapsible(localStorage 记 expand 状态)
- Current file:背景色高亮 + sticky scroll into view
- Search box:**前端 JS in-memory search**(启动时 server preload 所有 md → 通过 `/api/files-content` 一次性 ship 给 client,client cache)。这样 search 是 instant local。
- Search hit 列表:点 → swap right panel + scroll 到 line
- Footer:静态显示 vault 总数 / 大小

### Right content panel

复用单文件 mode 现有渲染逻辑:
- 上面 H1 title
- TOC sticky on right(已有)
- 主 content 渲染(已有)
- 批注 toolbar(已有)
- 编辑 mode(已有)

**改动**:URL hash 加 `#file=path/to.md`,新 file 时 update。Refresh page 时根据 hash 自动 load file。

### LocalStorage schema 变化

**Before**(single file):
```
mdlens:annotations:<filepath_hash>
mdlens:draft:<filepath_hash>
```

**After**(vault):
```
mdlens:vault:<vault_path_hash>:annotations:<relative_filepath>
mdlens:vault:<vault_path_hash>:draft:<relative_filepath>
mdlens:vault:<vault_path_hash>:expanded_dirs (JSON array)
```

**单文件 mode key 不变**(backward compat)。

### File walking & filtering

```python
def collect_md_files(vault_root: Path) -> list[Path]:
    skip_dirs = {'.git', 'node_modules', '.next', '__pycache__', '.venv', 'venv'}
    md_files = []
    for path in vault_root.rglob('*.md'):
        if any(part in skip_dirs for part in path.parts):
            continue
        if any(part.startswith('.') for part in path.parts):  # skip hidden
            continue
        md_files.append(path.relative_to(vault_root))
    return sorted(md_files)
```

### Search algorithm(client-side in-memory)

```javascript
// 启动时 ship 全部 .md 内容(简单粗暴,适合 < 1000 file vault)
// 大 vault > 1000 file 暂不考虑(future PR)
const SEARCH_INDEX = await fetch('/api/files-content').then(r => r.json())
// [{path, content, title}]

function search(query) {
  const lowerQ = query.toLowerCase()
  const hits = []
  for (const file of SEARCH_INDEX) {
    const lines = file.content.split('\n')
    for (let i = 0; i < lines.length; i++) {
      if (lines[i].toLowerCase().includes(lowerQ)) {
        hits.push({
          file: file.path,
          line: i + 1,
          snippet: lines[i].slice(0, 120),
          context: lines.slice(Math.max(0, i-1), i+2).join('\n')
        })
      }
    }
  }
  return hits.slice(0, 100)  // cap
}
```

性能:30 file × ~200 line each = 6000 search ops in memory · < 50ms · 完全够。

### Performance budget

- Vault startup(walk + read 30 files): < 500ms
- File switch(click → render): < 100ms
- Search input → results: < 100ms(local JS)
- HTML page size: < 5MB total(包括所有 md preloaded for search)

如果 vault > 100 file 性能崩,**记录在 CHANGELOG,future PR 优化**。

---

## 你的任务步骤(autonomous)

### Step 1 · Read full md-lens.py(~5 min)

```bash
cd ~/projects/md-lens
git branch --show-current   # 应该是 feat/multi-md-library
wc -l md-lens.py            # 行数 reference
```

完整 read md-lens.py 学:
- argparse 怎么 set up
- HTTP server 结构(`SimpleHTTPRequestHandler` 子类?)
- Markdown → HTML 渲染逻辑(用了什么 lib?Python markdown / mistune / 内置实现?)
- 单文件 inline HTML+CSS+JS 结构
- localStorage key 命名 convention
- TOC 生成

### Step 2 · Implement(autonomous,按上面 spec)

修改 `md-lens.py`:
1. argparse 加 `--vault` flag
2. main():分发 single-file mode vs vault mode
3. 加 `collect_md_files()` function
4. HTTP handler 加新 endpoints:`/api/files` / `/api/file` / `/api/files-content` / `/api/search` / `/api/save`
5. HTML template:加 vault layout(left sidebar)
6. JS:fetch + render file tree · click handler · search handler · URL hash sync
7. LocalStorage:按新 schema(vault-aware key)
8. Update CHANGELOG:`v0.2.0 - --vault flag for multi-md library mode`
9. Update README:加 vault 用法 section

**总改动 < 400 行 ideal**(400 是 hard cap)。

### Step 3 · Test

创建 3 个测试 vault + 跑:

```bash
# Test 1 · single-file mode 回归
echo "# Test\nhello" > /tmp/test.md
python3 md-lens.py /tmp/test.md
# 验证:浏览器开 → 渲染正常 → 跟以前一样

# Test 2 · 平铺 vault
mkdir -p /tmp/test-vault
echo "# File 1" > /tmp/test-vault/a.md
echo "# File 2\ncontent foo bar" > /tmp/test-vault/b.md
echo "# File 3\ndifferent content baz" > /tmp/test-vault/c.md
python3 md-lens.py --vault /tmp/test-vault
# 验证:sidebar 列 3 file · 点切换 · 搜 "foo" hit b.md

# Test 3 · 嵌套 vault(真 daisy-rpg/docs)
python3 md-lens.py --vault ~/Downloads/daisy-全部打包-20260430/个人项目/daisy新项目/daisy-rpg/docs
# 验证:30+ md tree 嵌套展开 · 切换 < 100ms · 搜索 work
```

**3 case 都 pass 才能 commit**。

挂的话:debug · 修 · retry。最多 retry 5 次。5 次都挂,**stop · 写 BLOCKED.md 记 issue · 别 commit 半残品**。

### Step 4 · Commit + push

```bash
git add md-lens.py README.md CHANGELOG.md docs/briefs/multi-md-library-brief.md
git status
git diff --stat
git commit -m "feat: add --vault flag for multi-md library mode

- new --vault <dir> mode with file tree sidebar
- client-side in-memory search across all .md files
- per-file annotations isolated in localStorage (vault-aware key)
- single-file mode unchanged (backward compat)
- < 400 lines added · 0 new dependencies

Closes #N/A (no issue, daisy-driven)
"
git push -u origin feat/multi-md-library
```

### Step 5 · 写 SESSION-REPORT.md(明早 daisy 直接 read)

```bash
# 在 md-lens/docs/ 写
cat > docs/briefs/SESSION-REPORT-2026-05-18.md <<'EOF'
# md-lens multi-md vault · session report

## 做完了

- ✓ `--vault <dir>` flag
- ✓ file tree sidebar
- ✓ client-side search
- ✓ per-file annotations
- ✓ 3 case test pass
- ✓ commit · push 到 feat/multi-md-library

## Code change

- `md-lens.py`: +XXX lines / -YY lines
- `README.md`: 加 vault section
- `CHANGELOG.md`: 加 v0.2.0 entry

## Performance(verified)

- 30 file vault startup: XXX ms
- File switch: XXX ms
- Search: XXX ms

## Trade-off

- ...

## 你 review 时该看的 3 处

1. ...
2. ...
3. ...

## 下一步(待 daisy 决定)

- merge 到 main? → `git checkout main && git merge feat/multi-md-library`
- 重新 install:`python3 md-lens.py --install`
- 试试 daisy-rpg vault:`md-lens --vault ~/Downloads/.../daisy-rpg/docs`

EOF
```

报告里要有具体数字 + 具体 line ranges,**不要 generic 的"做完了"**。

### Step 6 · 如果 push 出问题

如果 git push 没权限 / network 问题 / 等等 —— **不要 force**。改成 `git push -u origin feat/multi-md-library --dry-run` 先看,然后报告问题在 SESSION-REPORT 里。daisy 明早处理。

---

## 严格约束

- ✗ **不引入任何第三方 dependency**(单文件 0-dep 灵魂)
- ✗ 不 break single-file mode(必须 backward compat,test 1 必过)
- ✗ 不超 400 行净改动
- ✗ 不直接 merge 到 main(留 branch)
- ✗ 不用 `pip install`(系统 Python 都能跑)
- ✗ 不写 frontend framework(vanilla JS)
- ✗ 不假装 test pass(test 真跑,不通过 stop)
- ✓ 中文注释关键函数(用户友好)
- ✓ Update CHANGELOG · README
- ✓ Performance budget 内
- ✓ Commit message 用 conventional commit format

---

## Done 标准

1. `feat/multi-md-library` branch · 已 push 到 origin
2. 3 个 test case 全 pass(报告里写具体 ms)
3. `docs/briefs/SESSION-REPORT-2026-05-18.md` 写完
4. README 加 `--vault` 用法
5. CHANGELOG 加 v0.2.0 entry
6. < 400 行 net change(`git diff --stat main..HEAD`)

---

## 关键 reference

- mirror-viewer:`github.com/daisyyang1109-super/mirror-viewer`(姊妹工具,**可以 read 它的 cross-session 搜索实现作 inspiration**,但**不要 copy code**)
- md-lens v0.1.1 已发布,你这次是 v0.2.0
- lens 系列定位:local-first · 0-dep · founder portfolio

---

## 失败模式 fallback

如果你 implement 到一半发现 my design spec 有 bug(e.g. localStorage schema 冲突 / endpoint 设计不合理) —— **不要硬撑**:

1. Stop coding
2. 写 `docs/briefs/BLOCKED-2026-05-18.md`:具体 blocker + 你怀疑的 root cause + 建议的修正
3. revert uncommitted changes(`git checkout md-lens.py`)
4. 不 commit · 不 push · daisy 明早决定

**Honest blocker > 假装跑完**。
