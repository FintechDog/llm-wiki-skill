# llm-wiki

**一个用于构建 Karpathy 风格 LLM 知识库的 OpenClaw / Codex Agent 技能。**

> 实验性技能——将持续迭代。
> 请通过 GitHub Issues 提交你的反馈。

灵感来源于 [Andrej Karpathy 的 llm-wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 以及社区在此基础上所做的工作。

## 这是什么

与 RAG（每次查询都重新检索原始文档）不同，该模式让 LLM 将原始来源**编译**为持久化、交叉链接的 Markdown Wiki。每次 `compile`（编译）、`ingest`（摄入）、`query`（查询）、`lint`（检查）和 `audit`（审计）都会让 Wiki 更加丰富。知识随时间不断积累。

- 你负责：提供原始材料、提出好问题、把控方向、对 AI 出错的地方提交反馈。
- LLM 负责：所有写作、交叉引用、归档、记录工作，以及根据你的反馈采取行动。

本仓库中的技能附带两个配套工具：

- **`plugins/obsidian-audit/`** — Obsidian 插件：在任何页面中选中文本，留下带严重程度的评论，评论会以带锚点的 Markdown 文件形式写入 `audit/`。
- **`web/`** — 本地 Node.js 预览服务器：渲染 Wiki（支持 Mermaid、KaTeX 和 Wiki 链接），允许你在浏览器中选中文本并提交反馈，同时显示每个页面的开放审计。

两个工具共享同一个 TypeScript 库（`audit-shared/`），因此从 Obsidian 和 Web 查看器写入的审计文件格式完全一致。

## 安装

```bash
# 将技能复制到你的 Agent 技能目录
cp -r llm-wiki/ ~/.claude/skills/llm-wiki/
# 或用于 Codex
cp -r llm-wiki/ ~/.codex/skills/llm-wiki/
```

然后在你的 Agent 配置中引用它，或者直接将 `llm-wiki/SKILL.md` 粘贴到你的 Agent 上下文中。

## 快速开始

```bash
# 1. 搭建新的 Wiki
python3 llm-wiki/scripts/scaffold.py ~/my-wiki "My Research Topic"

# 2. 添加来源
cp my-article.md ~/my-wiki/raw/articles/

# 3. 告诉你的 Agent："ingest raw/articles/my-article.md"

# 4. 提出问题："Wiki 关于 X 说了什么？"

# 5. 定期运行 lint
python3 llm-wiki/scripts/lint_wiki.py ~/my-wiki

# 6. 从 Web 查看器或 Obsidian 插件提交评论，然后处理它
python3 llm-wiki/scripts/audit_review.py ~/my-wiki --open
# 然后告诉你的 Agent："audit: process the open comments"
```

## 仓库内容

```
llm-wiki-skill/
├── llm-wiki/                    ← 技能
│   ├── SKILL.md                 ← 主技能文件（由 Agent 读取）
│   ├── references/
│   │   ├── schema-guide.md      ← CLAUDE.md 模式模板
│   │   ├── article-guide.md     ← 文章编写（分而治之、Mermaid、KaTeX）
│   │   ├── log-guide.md         ← log/ 文件夹规范
│   │   ├── audit-guide.md       ← 审计文件格式 + 处理流程
│   │   └── tooling-tips.md      ← Obsidian、qmd、插件 + Web
│   └── scripts/
│       ├── scaffold.py          ← 引导创建新的 Wiki 目录
│       ├── lint_wiki.py         ← 七步健康检查（链接、审计、日志状态）
│       └── audit_review.py      ← 按目标分组开放/已解决的审计
├── audit-shared/                ← 共享 TypeScript 库
│   └── src/{schema,anchor,id,serialize,index}.ts
├── plugins/obsidian-audit/      ← Obsidian 插件 — 从 Vault 提交审计
└── web/                         ← 本地 Node.js 预览 + 反馈服务器
    ├── server/                  ← Express + markdown-it + KaTeX + Wiki 链接
    └── client/                  ← 原生 TS SPA，支持 Mermaid + 选择弹出框
```

## 运行 Web 查看器

```bash
# 一次性设置（构建 audit-shared，安装依赖，打包客户端）
cd audit-shared && npm install && npm run build && cd ..
cd web && npm install && npm run build && cd ..

# 针对某个 Wiki 启动服务器
cd web
npm start -- --wiki "/path/to/your/wiki-root" --port 4175
# 打开 http://127.0.0.1:4175
```

## 构建 Obsidian 插件

```bash
cd audit-shared && npm install && npm run build && cd ..
cd plugins/obsidian-audit
npm install
npm run build
npm run link -- "/path/to/your/Obsidian vault"
# 在 Obsidian → 设置 → 社区插件中启用 'LLM Wiki Audit'。
```

## 使用场景

- **深度研究**——数周内阅读某主题的论文/文章
- **个人 Wiki**——Farzapedia 风格：将日志条目汇编成个人百科全书
- **团队知识库**——由 Slack 线程、会议笔记、文档驱动
- **阅读伴侣**——阅读书籍时构建丰富的伴侣 Wiki

## 相关工作

- [Karpathy 的原始 Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [pedronauck/skills karpathy-kb](https://github.com/pedronauck/skills/tree/main/skills/karpathy-kb) — 完整的 Obsidian Vault 集成
- [Astro-Han/karpathy-llm-wiki](https://github.com/Astro-Han/karpathy-llm-wiki) — 示例实现
- [qmd](https://github.com/tobi/qmd) — Markdown Wiki 的语义搜索

## 许可证

MIT
