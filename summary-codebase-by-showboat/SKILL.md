---
name: summary_codebase_by_showboat
description: |
  Use showboat (uvx showboat) to produce a walkthrough.md that explains a codebase
  end-to-end in Chinese. Trigger: user asks for a "code walkthrough", "code tour",
  or "explain how this project works". Workflow: read all source files → plan a
  linear execution-flow outline → run `uvx showboat --help` → build walkthrough.md
  with `showboat note` for Chinese commentary and `showboat exec bash "sed …"` /
  `showboat exec bash "grep …"` for live code snippets. Final document is in Chinese
  and is verifiable via `showboat verify walkthrough.md`.
author: Claude Code
version: 1.0.0
date: 2026-05-16
---

# 用 Showboat 生成代码导读 (summary_codebase_by_showboat)

## Problem

需要对一个不熟悉的代码库给出详尽的中文代码导读，要求：
- 有真实代码片段（而非手写伪代码）
- 按执行流线性排列，读者可以按顺序跟读
- 解说与代码混排，最终产物是一份可验证的 Markdown 文档

## Context / Trigger Conditions

当用户说：
- "Read the source and plan a linear walkthrough"
- "解释这个项目是怎么运行的"
- "给我做一份代码导读"
- "用 showboat 生成 walkthrough"

或明确提到"final results using chinese"/"用中文写"时触发本流程。

## Solution

### 第一阶段：阅读代码库

**并行读取所有关键文件**，不要串行。一次 message 里同时调用多个 Read / Bash 工具：

```
# 先列出所有 .py 文件
find . -type f -name "*.py" | sort

# 同时读取：入口文件、数据层、核心逻辑、输出层
Read main.py / app.py / storage/db.py / matcher/*.py / reporter/*.py
```

读的时候脑子里勾勒执行流草图：
```
入口 → 数据层初始化 → 采集/爬取 → 核心处理 → 输出
```

### 第二阶段：学习 showboat

**每次使用前先运行一遍 help**，showboat 版本会更新，不要假设命令不变：

```bash
uvx showboat --help
```

核心命令速查（截至 showboat 0.6.x）：

| 命令 | 用途 |
|------|------|
| `uvx showboat init <file> <title>` | 新建文档，写入标题和时间戳 |
| `uvx showboat note <file> "text"` | 追加纯文字解说（支持 Markdown） |
| `uvx showboat exec <file> bash "cmd"` | 运行命令，把代码+输出一并追加到文档 |
| `uvx showboat pop <file>` | 删除最后一条记录（出错时用） |
| `uvx showboat verify <file>` | 重跑所有 exec 块，验证输出是否一致 |

### 第三阶段：规划导读大纲

按**执行流**（不是文件目录）排列章节，典型结构：

```
## 项目概览          ← 一张流程图 + 两句话
## 第一步：入口文件  ← main() / CLI 参数解析
## 第二步：数据层    ← 数据库初始化、Schema、关键查询
## 第三步：采集层    ← 每个爬虫/数据源，重点讲不同之处
## 第四步：核心处理  ← 业务逻辑核心（AI/算法/转换）
## 第五步：输出层    ← 报告生成、API 路由
## 第六步：Web 界面  ← 前后端交互（如有）
## 第七步：完整数据流 ← 端到端流程树状图
## 总结             ← 设计亮点 + 扩展注意事项
```

### 第四阶段：逐节构建 walkthrough.md

**初始化文档**：
```bash
uvx showboat init walkthrough.md "项目名 代码导读"
```

**每节的模式**（note → exec → note → exec ...）：

```bash
# 1. 先写解说
uvx showboat note walkthrough.md "## 第N步：XXX

解说文字，支持 Markdown 表格、有序列表、代码围栏等。"

# 2. 再插入代码片段
uvx showboat exec walkthrough.md bash "sed -n '行号1,行号2p' 文件路径"

# 或用 grep 展示跨文件规律
uvx showboat exec walkthrough.md bash "grep -n 'pattern' file1.py file2.py"
```

**选取代码片段的原则**：
- 每次 `sed` 只截取**一个完整函数或关键结构**（15–60 行为宜）
- 截取点要能说明解说中提到的设计决策，不要截无关代码
- 用行号精确截取（先 Read 文件确认行号，再写 sed 命令）

**解说语言规范（中文）**：
- 每节标题：`## 第N步：模块名 \`文件路径\``
- 核心术语后括号英文：`批量请求（batch request）`
- 设计要点用加粗 `**要点名称**`
- 注意事项用无序列表

### 第五阶段：收尾

**补充端到端数据流**（纯文字树状图，放在最后一个技术节之后）：

```
## 第七步：完整数据流

\`\`\`
python main.py
│
├── init_db()
├── run_scraping()
│   ├── scraper_A()
│   └── scraper_B()
├── save_signals()          # INSERT OR IGNORE
├── get_unmatched()
├── core_processing()       # 核心逻辑
└── generate_output()
\`\`\`
```

**补充总结节**：
```bash
uvx showboat note walkthrough.md "## 总结：设计亮点与注意事项

**亮点：**
- ...

**扩展时需注意的地方：**
- ..."
```

**用 grep 佐证总结中提到的跨文件问题**：
```bash
uvx showboat exec walkthrough.md bash "grep -n 'PATTERN' file1.py file2.py"
```

## Verification

```bash
# 重跑所有 exec 块，确认输出没有变化
uvx showboat verify walkthrough.md
```

输出 `All N code blocks verified` 则文档可信。

## Example

本 skill 的来源就是一次完整的实例：在 `socrates-finds-you` 项目中生成了
`walkthrough.md`，覆盖 `main.py → storage/db.py → scrapers/ → matcher/claude_match.py
→ reporter/daily_report.py → app.py`，共 7 节 + 总结，全程中文解说，
代码片段由 `sed` 实时截取自源文件。

## Notes

- `uvx showboat exec` 的 `bash` 子命令在当前工作目录执行，路径要与 `cwd` 一致
- `showboat note` 中的 Markdown 反引号需要转义（`\``）或改用单引号包裹参数
- 如果一条 `exec` 失败，立即 `uvx showboat pop walkthrough.md` 再修正命令
- `SERVICE_PRIORITY` 这类常量如果在多个文件中重复定义，是值得在总结节点名的典型问题
- 最终文档应该能让不熟悉这个代码库的人按顺序阅读后，理解整条数据流

## References

- showboat 官方 CLI 帮助：`uvx showboat --help`
- showboat PyPI 页面：https://pypi.org/project/showboat/
