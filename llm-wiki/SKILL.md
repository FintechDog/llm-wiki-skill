---
name: llm-wiki
description: >-
  构建并维护 Karpathy 风格的 LLM 知识库——一个自编译的 Obsidian Markdown Wiki，
  Agent 在其中摄入原始来源、编译交叉链接的概念/实体/摘要页面、基于语料回答问题、
  检查图谱健康状态，并审计来自 Obsidian 或本地 Web 查看器的人类反馈。
  适用于：(1) 为任何研究主题搭建新知识库，(2) 将文章/论文/PDF/网页摄入 raw/，
  (3) 基于现有原始材料编译或重构 Wiki 文章，(4) 回答 Wiki 问题并归档持久化答案，
  (5) 运行 lint 检查死链/孤立页面/覆盖缺口/审计状态，
  (6) 处理 audit/ 目录中的人类反馈并应用修正。
  不适用于通用笔记、日志或非 Wiki 用途的 Obsidian。
---

# LLM Wiki — Karpathy 知识库模式

> **实验性技能——持续迭代中。**
> 作者：Lewis Liu (lylewis@outlook.com) · 灵感来源于 [Karpathy 的 llm-wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## 核心理念

与 RAG（每次查询都重新检索原始文档）不同，LLM 将原始来源**编译**为持久化、交叉链接的 Wiki。每次摄入、查询、lint 和审计都会让 Wiki 更加丰富。知识不断积累——人类通过结构化的反馈渠道保持参与，而不是让临时修正消失在聊天记录中。

- **你负责**：提供原始材料、提出好问题、把控方向、对 AI 出错的地方提交反馈。
- **LLM 负责**：所有写作、交叉引用、归档、记录工作，以及根据你的反馈采取行动。

Wiki 是一个活的文档，包含**五项操作**——`compile`（编译）、`ingest`（摄入）、`query`（查询）、`lint`（检查）、`audit`（审计）。每次会话开始时都要读取 `CLAUDE.md` 和 `wiki/index.md`。

## 目录结构

```
<wiki-root>/
├── CLAUDE.md          ← 模式文件：范围、规范、当前文章列表、知识缺口
├── log/               ← 每日操作日志（每天一个文件）
│   ├── 20260409.md
│   └── 20260410.md
├── audit/             ← 人类反馈收件箱（每条评论一个文件）
│   ├── 20260409-143022-claude-code-size.md
│   └── resolved/      ← 已处理的反馈，附带解决说明归档
├── raw/               ← 不可变的源文档（LLM 只读，不写）
│   ├── articles/
│   ├── papers/
│   ├── notes/
│   └── refs/          ← 大型二进制文件的指针文件，存放在 raw/ 之外
├── wiki/              ← LLM 生成的知识（LLM 写入，你阅读）
│   ├── index.md       ← 主目录——每个页面，按分类组织
│   ├── concepts/      ← 概念/主题页面（超过 1200 词时拆分子文件夹）
│   ├── entities/      ← 人物、工具、论文、组织
│   └── summaries/     ← 每个来源的摘要页面
└── outputs/
    └── queries/       ← 查询答案（将持久化的答案提升到 wiki/）
```

`CLAUDE.md` 是**模式文件**——最重要的配置文件。它告诉 LLM Wiki 的范围、命名规范、当前文章列表、待解决问题和研究空白。阅读 `references/schema-guide.md` 了解如何编写它。每次会话开始时都要读取。

## 核心原则

以下所有内容都受四条规则约束。如果未来的指令与其中一条冲突，先向用户标记再行动。

### 1. 分而治之

单个概念页面**绝不**应试图端到端地覆盖复杂主题。目标：**每页 400–1200 词**。当主题超过这个范围时：

- 创建子文件夹：`wiki/concepts/<topic>/`
- 在 `wiki/concepts/<topic>/index.md` 放置简短索引页——定义、子页面列表、一句话摘要
- 将每个方面放在独立文件中：`wiki/concepts/<topic>/<aspect>.md`
- 在 `wiki/index.md` 中通过缩进列表展示层级

示例布局（来自真实 Wiki）：
```
wiki/tech/claude-code/
├── index.md                         （概览 + 子页面链接）
├── Claude_Code_Architecture.md
├── Claude_Code_Agent_Framework.md
├── Claude_Code_Bridge_System.md
├── Claude_Code_Query_Engine.md
├── Claude_Code_Skills_Plugins.md
├── Claude_Code_State_Management.md
└── Claude_Code_Tool_System.md
```

一个涵盖所有七个方面的臃肿文件将难以阅读且无法链接。七个专注的文件 + 一个索引页提供了导航、选择性阅读、清晰的反向链接和小型审计目标。

### 2. 图表用 Mermaid，公式用 KaTeX

- **任何流程、序列、层级或状态图**必须用 Mermaid 编写——绝不用 ASCII 艺术。ASCII 框容易过时且无法标注。
  ````
  ```mermaid
  flowchart LR
      A[raw/article.md] --> B[summary]
      B --> C[concept page]
      C --> D[index.md]
  ```
  ````
- **任何公式**必须用 KaTeX 编写：行内 `$f(x) = \sum_i w_i x_i$` 或块级 `$$...$$`。

两者都能在 Web 查看器中渲染（服务端 KaTeX，客户端 Mermaid），也能在 Obsidian 默认设置中渲染。

### 3. 原始文件策略

小型文本来源（md、txt、小型 pdf、小型图片）→ 复制到 `raw/<subfolder>/`。

大型二进制文件（视频、模型权重、安装程序、数据集、大于 10 MB 的大型 PDF）→ **不要复制**。改为：

- 在 `raw/refs/<slug>.md` 创建指针文件：
  ```yaml
  ---
  kind: ref
  external_path: /Volumes/external/models/llama-3-70b/
  size: ~140 GB
  ---
  ```
  后面附上简短描述，说明它是什么以及为什么对本 Wiki 重要。
- Wiki 页面像引用其他来源一样引用 `[[raw/refs/<slug>]]`。

这样让 Wiki 仓库对 Git 友好且便于移植。

### 4. 审计是人类反馈的入口

Wiki 由 AI 编写，有时会出错。原始来源由人类编写，彼此之间可能矛盾。`audit/` 目录让人类修正两者，而不会让修正消失在聊天记录中。

- 人类通过 Obsidian 插件或 Web 查看器提交反馈。每个反馈是 `audit/` 中的一个文件，包含 YAML 前置元数据（锚点、目标、严重程度）和 Markdown 正文。
- AI **必须**定期运行 `audit` 操作——绝不能静默忽略 `audit/*.md` 文件。
- 当反馈被应用后，文件移动到 `audit/resolved/`，附加 `# Resolution`（解决）部分，并在 `log/YYYYMMDD.md` 中记录日志条目。

完整文件格式和处理流程见 `references/audit-guide.md`。

---

## 五项操作

对 Wiki 的每个操作都属于这五种之一。每个操作都会追加条目到当天的日志文件（`log/YYYYMMDD.md`）。

### 1. `compile`（编译）

从现有 `raw/` 材料中（重新）组织 Wiki 内容——包括拆分超大页面、合并近似重复页面、重建 `index.md`。

**何时运行**：大批量摄入后、现有页面超过 1200 词时、`index.md` 不再反映实际情况时，或用户说"整理 Wiki"时。

**步骤**：
1. 读取 `CLAUDE.md`、`wiki/index.md` 和目标子树中的每个文件。
2. 对于每个超过约 1200 词的页面：规划拆分为 `concepts/<topic>/`，包含索引 + 子页面。写入前与用户确认方案。
3. 对于每对近似重复的页面：提议合并。确认后重写。
4. 重新生成 `wiki/index.md`，使每个页面恰好出现一次。
5. 日志：`## [HH:MM] compile | <你做了什么——涉及的文件、拆分、合并>`

### 2. `ingest`（摄入）

添加新来源。**一个来源通常涉及 5–15 个 Wiki 页面。**

**步骤**：
1. 将来源保存到正确的子文件夹：
   - 网络文章 → `raw/articles/<slug>.md`
   - 论文 → `raw/papers/<slug>.md`（大型 PDF 的提取文本）
   - 笔记 → `raw/notes/<slug>.md`
   - 大型二进制 → `raw/refs/<slug>.md` 指针文件（见原始文件策略）
2. 完整阅读来源。
3. 创建 `wiki/summaries/<slug>.md`（200–400 词——关键要点，不是重写；见 `references/article-guide.md`）。
4. 在 `wiki/concepts/` 中创建或更新相关概念页面。遵守分而治之原则：如果概念页面将超过 1200 词，拆分而不是硬塞。
5. 在 `wiki/entities/` 中为引用的新人物/工具/论文/组织创建或更新实体页面。
6. 更新 `wiki/index.md`，使新页面出现在正确的分类下。
7. 日志：`## [HH:MM] ingest | <slug> — <一句话描述>（涉及 N 个页面）`

### 3. `query`（查询）

**基于 Wiki 内容**回答问题，而非依赖通用知识。

**步骤**：
1. 读取 `wiki/index.md`。按分类扫描相关页面。
2. 完整读取识别出的页面；跟随一层 Wiki 链接。
3. 如果 Wiki 材料不足，如实说明并建议下一步摄入什么，而不是编造答案。
4. 综合答案，用 `[[页面名称]]` 行内引用页面。
5. 保存到 `outputs/queries/<YYYY-MM-DD>-<问题-slug>.md`。
6. 如果答案是持久化的（比较、分析或新综合）→ 将清理后的版本提升到 `wiki/concepts/`，添加到 `index.md`。
7. 日志：`## [HH:MM] query | <问题-slug>`（如果提升了，另起一行 `## [HH:MM] promote | ...`）。

### 4. `lint`（检查）

健康检查。运行：

```bash
python3 scripts/lint_wiki.py <wiki-root>
```

脚本报告：
- **死 Wiki 链接**——`[[Target]]` 但 `Target.md` 不存在
- **孤立页面**——没有入站 Wiki 链接的页面
- **缺失的索引条目**——未列在 `wiki/index.md` 中的页面
- **频繁链接的缺失页面**——`[[X]]` 被引用 3 次以上但没有对应页面
- **log/ 状态**——`log/` 中的散乱文件或错误文件名
- **audit/ 状态**——`audit/*.md` 中格式错误的 YAML 前置元数据
- **审计目标解析**——每个开放审计的 `target` 文件必须存在

对每个问题，提出修复方案，与用户确认后应用。日志：`## [HH:MM] lint | 发现 <N> 个问题，修复 <M> 个`。

### 5. `audit`（审计）

处理来自 `audit/` 的人类反馈。

**步骤**：
1. 运行 `python3 scripts/audit_review.py <wiki-root> --open` 获取分组列表。
2. 对每个开放审计，读取文件。使用 `anchor_before` / `anchor_text` / `anchor_after` 窗口定位目标文件中的精确范围（行号可能已偏移）。
3. 决定操作：
   - **接受**：将修正应用到目标文件。
   - **部分接受**：应用合理的部分，其余在解决说明中备注。
   - **拒绝**：在解决说明中解释原因——反馈可能基于对范围的误读或矛盾的来源。
   - **推迟**：添加到 `CLAUDE.md` 的"待研究问题"中，并将审计保留在原处并附注释。
4. 对于已应用的审计，在审计文件末尾附加 `# Resolution`（解决）部分：
   ```markdown
   # Resolution

   2026-04-10 · accepted.
   Fixed the file count (was "~1,900", corrected to "~1,800" per commit abc123).
   Updated: tech/Claude_Code.md lines 47–48.
   ```
5. 将文件从 `audit/` 移动到 `audit/resolved/`。文件名不变。
6. 每个已解决的审计记录日志：
   ```
   ## [HH:MM] audit | resolved 20260409-143022-a1b2 — <一句话描述>
   ```
7. 绝不删除审计文件。被拒绝的也要移动到 `resolved/`，在解决部分写明拒绝理由——这是有价值的历史记录。

完整审计文件格式见 `references/audit-guide.md`。

---

## 工具

| 工具 | 用途 |
|------|------|
| [Obsidian](https://obsidian.md) | 浏览 Wiki 的 IDE；图谱视图显示连接关系 |
| **`plugins/obsidian-audit/`** | Obsidian 插件——选中文本 → 添加反馈 → 写入 `audit/` |
| **`web/`** | 本地 Node.js 服务器——预览 Wiki，渲染 Mermaid/数学公式；选中 → 反馈 → `audit/` |
| `scripts/scaffold.py` | 引导创建新的 Wiki 目录树 |
| `scripts/lint_wiki.py` | 七步健康检查 |
| `scripts/audit_review.py` | 按目标文件分组开放/已解决的审计 |
| [qmd](https://github.com/tobi/qmd) | 可选的本地语义搜索（超过 100 页时有用） |

Obsidian 插件和 Web 查看器都以**相同格式**和**相同锚点算法**写入审计文件，因此从任一位置提交的反馈都可以在任一位置解决。

## 创建新 Wiki

```bash
python3 scripts/scaffold.py <wiki-root> "<主题标题>"
```

创建完整的目录树（包括 `log/<today>.md`、`audit/`、`audit/resolved/`），基于新模板生成空白的 `CLAUDE.md`，以及带有推荐分类布局的空白 `wiki/index.md`。

搭建完成后：
1. 填写 `CLAUDE.md`——定义范围、命名规范、初始研究问题。
2. 开始摄入来源。
3. 提出问题以积累 `outputs/queries/`；提升持久化的答案。
4. 定期运行 `lint`。
5. 有新反馈积累时运行 `audit`。

## `wiki/index.md` 格式

LLM 在每次编译时重建 `index.md`，并在每次摄入时更新它。格式：

```markdown
# Index — <主题>

> 一句话描述 Wiki 的范围。

## 🔖 导航
- [[#概念]] · [[#实体]] · [[#摘要]] · [[#待解决问题]]

## 概念
### <分类 A>
- [[concepts/Foo]] — 一句话摘要
- [[concepts/Bar/index|Bar]] — （文件夹拆分）一句话摘要
    - [[concepts/Bar/aspect-1]] — ...
    - [[concepts/Bar/aspect-2]] — ...

### <分类 B>
- ...

## 实体
- [[entities/Andrej Karpathy]] — AI 研究者，llm-wiki 模式作者

## 摘要（按时间顺序）
- 2026-04-09 — [[summaries/llm-wiki-gist]] — Karpathy 的原始 Gist

## 待解决问题
- Q1: ...
```

规则：
- 每个 Wiki 页面必须在 `index.md` 中恰好出现一次。`lint` 强制执行。
- 文件夹拆分概念通过缩进列表展示层级。
- `index.md` + `CLAUDE.md` 共同构成 AI 在会话开始时读取的内容。

## `log/` 格式

完整细节见 `references/log-guide.md`。最低要求：

- 每天一个文件：`log/YYYYMMDD.md`
- H1 = 日期；每个条目用 H2：`## [HH:MM] <操作> | <一句话描述>`
- 操作类型：`compile`、`ingest`、`query`、`lint`、`audit`、`promote`、`split`、`scaffold`

快速跨历史搜索：`grep -rh "^## \[" log/ | tail -20`。

## 使用场景

- **深度研究**——数周内阅读某主题的论文/文章；Wiki 随你的理解不断演进，审计追踪防止 AI 错误静默累积
- **个人 Wiki**——日志条目、笔记、想法汇编成个人百科全书；稍后评论不同意的内容，AI 会修正
- **团队知识库**——由 Slack 线程、会议笔记、文档驱动；团队成员通过 Web 查看器提交修正
- **阅读伴侣**——边读边归档每章内容；读完后形成丰富的伴侣 Wiki

## 参考资料

- `references/schema-guide.md`——如何编写 `CLAUDE.md`
- `references/article-guide.md`——如何编写好的 Wiki 文章（长度、Wiki 链接、Mermaid、数学、分而治之）
- `references/log-guide.md`——`log/` 文件夹规范
- `references/audit-guide.md`——审计文件格式、锚点策略、处理流程
- `references/tooling-tips.md`——Obsidian 设置、Web Clipper、qmd、插件 + Web 安装
