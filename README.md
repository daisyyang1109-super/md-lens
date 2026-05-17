# md-lens

![Python](https://img.shields.io/badge/Python-3.8%2B-blue) ![License](https://img.shields.io/badge/License-MIT-green) ![Status](https://img.shields.io/badge/Status-Alpha-orange) ![Zero Dependency](https://img.shields.io/badge/Deps-Zero-success)

> 给 markdown 文件做批注 + reviewer 工具。
> 阅读 → 选段批注(好/不好/建议/疑问)→ 编辑改 → 导出回 .md / JSON。
> 单文件 Python + 浏览器端,clone 完 0 依赖。

**适合场景**:

- 你用 AI 写了一篇文档,想就着原文逐段标"这里改 / 这里不对 / 这里问号",然后**结构化反馈给 AI 重新生成**
- 你要 review 同事 / 学生 / 投稿人写的 markdown,**做行间批注但又不想用 Word 那么重**
- 你在写长文,**想看着 TOC 自动跟随调结构,改完导出新 .md**

不是 Notion AI 那种"AI 一键改"工具,是**人批注 + 给 AI / 写作者结构化反馈**的工具。

---

## 关键特性

- **批注 4 类带 opinion**:好 / 不好 / 建议 / 疑问 — 不是通用高亮,是带语义标签的反馈
- **批注 → JSON 导出**:结构化喂回 AI,让 AI 知道你具体哪里满意哪里不行
- **编辑模式 + TOC 实时跟随**:改标题左侧目录自动更新,像 Word 但 markdown 原生
- **localStorage 持久化**:批注 / 草稿不丢
- **单文件 + 0 依赖**:Python 标准库 + CDN 加载 marked.js,clone 完即跑

---

## 截图

| 阅读模式 + TOC | 选段批注 |
|:---:|:---:|
| (screenshot placeholder) | (screenshot placeholder) |
| **编辑模式 + 格式工具栏** | **批注侧边栏 + 导出** |
| (screenshot placeholder) | (screenshot placeholder) |

---

## 快速开始

### 一键安装(推荐)

```bash
git clone https://github.com/daisyyang1109-super/md-lens.git ~/md-lens
python3 ~/md-lens/md-lens.py --install
```

自动把 md-lens.py cp 到 `~/.claude/md-lens/`,提示你加 alias:

```bash
echo "alias md-lens='python3 ~/.claude/md-lens/md-lens.py'" >> ~/.zshrc
source ~/.zshrc
```

之后任意位置都能跑:

```bash
md-lens some-doc.md                  # 渲染同名 .html 并打开
md-lens some-doc.md "标题"           # 自定义标题
md-lens some-doc.md '标题' out.html  # 指定输出路径
md-lens --vault ~/projects/docs      # vault mode:目录下全部 .md 一站式
md-lens --vault . --port 8080        # vault 可选 port(默认 8000)
```

### 或者直接跑(不装)

```bash
python3 ~/md-lens/md-lens.py path/to/your.md "页面标题" output.html
open output.html
```

之后:

- **阅读**:左侧 TOC,滚动自动高亮当前章节
- **批注**:选段后弹工具栏,选 4 类标记 + 加 comment
- **编辑**:右下角 "✏ 编辑" → 选段格式化(H2/H3/B/I/Code/引用/列表)→ TOC 跟着更新
- **导出**:右下角 "💾 导出 MD" → 下载你改完的 .md

---

## 典型 workflow

### Workflow A · review AI 写的初稿

```
1. AI 写初稿 → 保存 .md
2. md-lens 渲染 → 看 + 批注(好 / 不好 / 建议 / 疑问)
3. 导出批注 JSON → 复制给 AI
4. AI 改下一版 → 重新渲染
5. 直到满意 → 导出最终 .md
```

### Workflow B · review 别人写的文档

```
1. 拿到 .md(同事写的 / 学生交的 / 投稿)
2. md-lens 渲染 → 选段批注 + 加意见
3. 导出 Markdown 摘要 → 发回作者
4. 作者按摘要改 → 再来一轮
```

### Workflow C · 自己长文写作

```
1. 草稿 .md → md-lens 渲染
2. 阅读模式看整体结构(TOC + 内容)
3. 编辑模式改某段(选段 → H2 / B / Code 等)
4. 改完 → 导出新 .md → git commit
```

---

## 跟 mirror-viewer 的关系

md-lens 是 [mirror-viewer](https://github.com/daisyyang1109-super/mirror-viewer) 的姊妹项目,共同形成 **lens 系列**:

- **mirror-viewer**:Claude Code session 的 data lens(看 jsonl)
- **md-lens**:任意 markdown 的 reading + editing lens(看 / 改 / 批注 .md)

两个都是**"个人化定制 GUI"**形态的实验,单文件 + 0 依赖 + 跟 AI 协作 workflow 友好 是共同设计原则。

---

## Vault mode(0.2.0 新)

一个目录下 30+ md 文件,跨文件跳来跳去 / 搜不到 / 关联看不到 — 这种"轻量 Obsidian"场景:

```bash
md-lens --vault ~/projects/some-docs
# 浏览器自动开 → 左侧文件树 + 搜索框 + 内容面板
```

特性:

- **文件树 sidebar**:嵌套目录可折叠(展开状态记 localStorage)
- **全文搜索**:启动时 ship 全部 md 到前端,搜索是 in-memory(< 100ms)
- **SPA 切换**:点 file → swap 内容,URL hash 同步,刷新页面恢复当前文件
- **per-file 批注**:每个 .md 独立 localStorage,不串台
- **保存回 vault**:编辑 → 点「💾 保存到 vault」直接覆盖原 .md(server 端 path traversal 防护)
- **0 dependency**:Python stdlib `http.server` + 前端 CDN marked/turndown,clone 完就跑

**适合场景**:`daisy-rpg/docs/` 这种 30+ md 单项目文档库 · BUG-TRACKER + 设计文档 + brief 同时翻 · 不用切 IDE 就能改

不适合 1000+ md 的大 vault(future PR 优化 lazy load + indexed search)。

---

## Roadmap

- [x] 0.1.0:渲染 + TOC + 批注 4 类 + 编辑模式 + 导出 MD
- [x] 0.2.0:`--vault` 模式 · 文件树 + 全文搜索 + 保存回 vault
- [ ] 0.3.0:GitHub Gist 云同步(批注跨设备)
- [ ] 0.4.0:vault 索引优化 · 支持 1000+ file
- [ ] 0.5.0:选段一键出图(集成 mermaid + AI 出图)

---

## License

MIT
