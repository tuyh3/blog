---
title: "如何创建并使用Skills"
date: "2026-05-13T06:32:38+05:00"
draft: false
summary: "读 Anthropic 官方文章后整理的 skill 创建笔记：5 步流程、SKILL.md 三要素，以及从 DOCX skill 学到的 menu approach。"
topics: ["ai"]
---

Skill 不是普通 prompt，也不是 CLAUDE.md；它是一套可复用的“专项工作能力包”，用来教 Claude 在特定任务/领域里按固定流程工作。

**Skill 定义**：用于特定任务或领域的自定义指令，可以扩展 Claude 的能力；通过 `SKILL.md` 文件，你是在教 Claude 如何更有效地处理某类具体场景。它的价值在于沉淀组织知识、标准化输出、处理复杂多步骤流程，避免你每次重复解释。

## 如何创建一个优秀的 Skill

**建议按 5 步来创建 skill：**先明确核心需求，再写 skill 名称，再写 description（描述），再写主 instructions（操作指南），最后上传/安装。遵循这种结构化的方法，培养能够更可靠地触发的技能。

`SKILL.md` 里真正影响 Claude 是否触发这个 skill 的，是 **name 和 description**；instructions 主要决定触发后怎么执行。

**skill 不是越大越好**。避免把无关内容都塞进上下文，应该用menu approach（菜单式组织法 / 菜单式 skill 结构）：主 `SKILL.md` 只写入口和选项，细节拆到其他文件，让 Claude 根据任务只读取需要的部分。

### 1. 明确核心需求

写 skill 前要先问清楚：这个 skill 解决什么问题？什么时候触发？成功标准是什么？边界和限制是什么？

举例：“从 PDF 提取金融数据并格式化成 CSV”比“帮我处理金融相关事情”更好，因为前者有明确输入、操作和输出。

这一步非常关键。

不要写这种 skill：

```
ai-development-helper
帮助我开发 AI 项目
```

太泛了，容易乱触发，也不具体。

应该写：

```
specs-first-development
当开始实现新功能前，先创建极简 spec，明确目标、范围、主流程、输入输出、验收标准和非目标。
```

### 2. 写 name

skill 需要三个核心组件：`name`、`description`、`instructions`。name 应该短、清晰、描述性强，推荐小写加连字符，比如 `pdf-editor`、`brand-guidelines`。

你的 skill 命名也应该这样：

```
specs-first-development
human-readable-design
terminology-translator
rag-quality-check
security-review
feature-debrief
codex-review-prep
```

不要写：

```
my-helper
dev-helper
ai-workflow
```

这种太泛。

### 3. 写 description

这是文章最重要的一段。它说 **description 决定 skill 什么时候被激活，是最关键的组件**；好的 description 要包括具体功能、清晰的触发条件、相关的背景（上下文）和边界。

描述不够清晰：

```
This skill helps with PDFs and documents.
这个skill用于帮助处理 PDF 和文档
```

应该写成：

```
description: >
  Security review workflow for code changes, pull requests, and pre-release checks.
  Use when asked to review code for authentication, authorization, input validation,
  file upload risks, path traversal, secret leakage, SQL/command injection,
  dependency risk, and unsafe logging. Not for general code style review,
  architecture review, or feature implementation.
  
  代码变更、拉取请求及发布前检查的安全审查流程。
  用于审查身份验证、授权、输入验证、文件上传风险、路径遍历、密钥泄露、SQL/命令注入、依赖项风险以及不安全日志记录等问题。不适用于常规代码风格审查、架构审查或功能实现审查。
```

这段里有四类信息：

```
它能做什么
什么时候触发
检查哪些内容
什么时候不该触发
```

### 4. 写主 instructions

 instructions 要清晰、易于阅读且便于操作。请使用 Markdown 标题、项目符号列表列出选项，并使用代码块提供示例。

结构清晰，层级分明：概述、前提条件、执行步骤、示例、错误处理和局限性。将复杂的流程分解为具有明确输入和输出的独立阶段。

提供具体示例以展示正确用法。明确说明该技能无法执行的操作，以防止误用并管理用户预期。SKILL.md文件还可以包含其他参考文件和资源，以便更清晰地说明技能触发时您希望代理执行的操作。 

```
## Overview
## When to use
## Inputs
## Workflow
## Output format
## Quality gates
## Limitations
## Examples
```

### 5. 上传/安装 skill

根据你所在 Claude 平台上的技能类型，以下是如何上传技能以供使用： 

**Claude Code**：项目根目录下创建 skills/ 目录，并在其中添加包含 SKILL.md 文件的技能文件夹。安装插件时，Claude 会自动发现并使用这些文件。示例结构：

```
my-project/
├── skills/
│   └── my-skill/
│       └── SKILL.md
```

**Claude app**：前往**“设置”**并添加您的自定义技能。自定义技能需要启用代码执行功能的 Pro、Max、Team 或 Enterprise 套餐。在此处上传的技能仅供每个用户使用，不会在组织范围内共享，管理员也无法集中管理。

[**Claude Developer Platform**](https://www.claude.com/platform/api)：通过技能API（/v1/skills接口）上传技能。使用带有所需beta版请求头的POST请求：

```
curl -X POST "https://api.anthropic.com/v1/skills" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: skills-2025-10-02" \
  -F "display_title=My Skill Name" \
  -F "files[]=@my-skill/SKILL.md;filename=my-skill/SKILL.md"
```

### 6. 测试与验证 Skill

skill 上线前要用真实场景测试；测试矩阵至少包括三类：正常操作、边界情况、超出范围的请求；并测试触发：显式请求和自然语言请求是否会触发；相似但不该触发的请求是否不会触发。

这个点很重要。

你写完一个 skill，不是“能读就行”，而是要测：

**正常触发**

```
请用 security-review 检查这次 diff
```

**应该触发**

```
这次改动有没有安全风险？
```

**不该触发**

```
帮我优化代码可读性
```

不应该触发 security-review，而应该走 code-review 或普通能力。

### 7. 根据使用情况迭代

密切关注你的技能在实际应用中的表现。如果触发条件不一致，请改进描述。如果输出结果出现意外变化，请明确说明。与提示一样，优秀的技能也是在实践中不断完善的。

## 优秀skills的最佳实践和经验学习

### DOCX skill

claude的DOCX skill 强在：它不是只说“会处理 docx”，而是有清晰的 workflow decision tree（决策树）、用渐进披露保持主文件精简、用好坏示例说明复杂模式。 不同任务走不同流程，比如阅读、创建、编辑、redlining，并且复杂操作会要求读取额外参考文件。

我们可以直接借鉴这个思想。

比如你的 rag-quality-check skill 也应该有决策树：

```
如果问题是 parse 质量差 → 检查文本完整性、阅读顺序、表格、页码
如果问题是 chunk 质量差 → 检查孤儿 chunk、title_path、大小、表格/列表切碎
如果问题是检索不准 → 检查 dense、BM25、RRF、rerank、阈值
如果问题是回答乱编 → 检查证据、阈值、prompt、引用
```

DOCX skill 中这类引用：

```
Read [`docx-js.md`](docx-js.md)
Read [`ooxml.md`](ooxml.md)
Use "Redlining workflow"
```

它们不是凭空变出来的，而是 **skill 包里的附加文件**。

也就是说，一个 skill 不一定只有一个 `SKILL.md`，它可以是一个目录，里面放：

```
skills/
  docx/
    SKILL.md
    docx-js.md
    ooxml.md
    ooxml/
      scripts/
        unpack.py
        pack.py
```

`SKILL.md` 是入口文件；`ooxml.md`、`docx-js.md`、workflow 文档是这个 skill 的“子说明书 / 参考文件 / 工具说明”。

#### AI 是怎么通过 skill 找到它们的

核心机制是：**Claude 先看到 `SKILL.md`，再根据 `SKILL.md` 里的相对路径去读取附加文件。**

如果一个 skill 涵盖多个流程，`SKILL.md` 应该说明有哪些内容可用，并用相对路径引用独立文件；Claude 会只读取和当前任务相关的那个文件，而不是把所有文件都读进来。

所以流程是：

```
用户任务
  ↓
Claude 判断是否需要某个 skill
  ↓
加载该 skill 的 SKILL.md
  ↓
SKILL.md 里写：
  - 创建 docx 先读 docx-js.md
  - 编辑 docx 先读 ooxml.md
  - redlining workflow 读 ooxml.md 的相关部分
  ↓
Claude 按相对路径读取对应文件
  ↓
根据文件里的说明执行任务
```

#### “workflow 文档”放在哪里

有些 workflow 是直接写在 `SKILL.md` 里的，比如：

```
Redlining workflow for document review
Tracked changes workflow
```

也可以拆成独立文件，比如：

```
redlining-workflow.md
basic-editing-workflow.md
text-extraction.md
```

没有强制规定必须叫 `workflow.md`。它强调的是原则：**把内容拆成合理块，让 Claude 根据任务选择需要的文件。**

所以你看到的“workflow 文档”，可能有两种形式：

1. 直接写在 `SKILL.md` 的某个章节里
2. 拆到同目录下的独立 markdown 文件里

```
skills/docx/
  SKILL.md
  workflows/
    redlining.md
    text-extraction.md
    basic-ooxml-editing.md
```

只要 `SKILL.md` 里引用了它：

```
For redlining, read `workflows/redlining.md`.
```

Claude 就知道要去读它。

#### 那 Claude 是“自动扫描所有文件”吗？

不是你想象的那种“把整个 skill 目录全部读完”。

更准确地说：

1. Claude Code / Claude Skills 系统先发现有哪些 skill。
2. 用 skill 的 `name` 和 `description` 判断当前任务是否相关。
3. 相关时，Claude 读取该 skill 的 `SKILL.md`。
4. 如果 `SKILL.md` 里写了相对路径，例如 `docx-js.md`、`ooxml.md`，Claude 会在需要时继续读取这些文件。

skill 触发主要受 `name` 和 `description` 影响；`SKILL.md` 可以包含额外参考文件和 assets，用来补充说明。

所以不是：

```
一触发 skill → 读完整个目录
```

而是：

```
一触发 skill → 先读 SKILL.md → 再按里面的菜单读相关文件
```



## 附录：DOCX skill 原文（SKILL.md 全文）

以下为 Anthropic 官方 DOCX skill 的 `SKILL.md` 全文，作为前面 menu approach 与工作流决策树讨论的对照参考。原文较长，可选阅读。

````
---
name: docx
description: "Comprehensive document creation, editing, and analysis with support for tracked changes, comments, formatting preservation, and text extraction. When Claude needs to work with professional documents (.docx files) for: (1) Creating new documents, (2) Modifying or editing content, (3) Working with tracked changes, (4) Adding comments, or any other document tasks"
license: Proprietary. LICENSE.txt has complete terms
---

# DOCX creation, editing, and analysis

## Overview

A user may ask you to create, edit, or analyze the contents of a .docx file. A .docx file is essentially a ZIP archive containing XML files and other resources that you can read or edit. You have different tools and workflows available for different tasks.

## Workflow Decision Tree

### Reading/Analyzing Content
Use "Text extraction" or "Raw XML access" sections below

### Creating New Document
Use "Creating a new Word document" workflow

### Editing Existing Document
- **Your own document + simple changes**
  Use "Basic OOXML editing" workflow

- **Someone else's document**
  Use **"Redlining workflow"** (recommended default)

- **Legal, academic, business, or government docs**
  Use **"Redlining workflow"** (required)

## Reading and analyzing content

### Text extraction
If you just need to read the text contents of a document, you should convert the document to markdown using pandoc. Pandoc provides excellent support for preserving document structure and can show tracked changes:

```bash
# Convert document to markdown with tracked changes
pandoc --track-changes=all path-to-file.docx -o output.md
# Options: --track-changes=accept/reject/all
```

### Raw XML access
You need raw XML access for: comments, complex formatting, document structure, embedded media, and metadata. For any of these features, you'll need to unpack a document and read its raw XML contents.

#### Unpacking a file
`python ooxml/scripts/unpack.py <office_file> <output_directory>`

#### Key file structures
* `word/document.xml` - Main document contents
* `word/comments.xml` - Comments referenced in document.xml
* `word/media/` - Embedded images and media files
* Tracked changes use `<w:ins>` (insertions) and `<w:del>` (deletions) tags

## Creating a new Word document

When creating a new Word document from scratch, use **docx-js**, which allows you to create Word documents using JavaScript/TypeScript.

### Workflow
1. **MANDATORY - READ ENTIRE FILE**: Read [`docx-js.md`](docx-js.md) (~500 lines) completely from start to finish. **NEVER set any range limits when reading this file.** Read the full file content for detailed syntax, critical formatting rules, and best practices before proceeding with document creation.
2. Create a JavaScript/TypeScript file using Document, Paragraph, TextRun components (You can assume all dependencies are installed, but if not, refer to the dependencies section below)
3. Export as .docx using Packer.toBuffer()

## Editing an existing Word document

When editing an existing Word document, use the **Document library** (a Python library for OOXML manipulation). The library automatically handles infrastructure setup and provides methods for document manipulation. For complex scenarios, you can access the underlying DOM directly through the library.

### Workflow
1. **MANDATORY - READ ENTIRE FILE**: Read [`ooxml.md`](ooxml.md) (~600 lines) completely from start to finish. **NEVER set any range limits when reading this file.** Read the full file content for the Document library API and XML patterns for directly editing document files.
2. Unpack the document: `python ooxml/scripts/unpack.py <office_file> <output_directory>`
3. Create and run a Python script using the Document library (see "Document Library" section in ooxml.md)
4. Pack the final document: `python ooxml/scripts/pack.py <input_directory> <office_file>`

The Document library provides both high-level methods for common operations and direct DOM access for complex scenarios.

## Redlining workflow for document review

This workflow allows you to plan comprehensive tracked changes using markdown before implementing them in OOXML. **CRITICAL**: For complete tracked changes, you must implement ALL changes systematically.

**Batching Strategy**: Group related changes into batches of 3-10 changes. This makes debugging manageable while maintaining efficiency. Test each batch before moving to the next.

**Principle: Minimal, Precise Edits**
When implementing tracked changes, only mark text that actually changes. Repeating unchanged text makes edits harder to review and appears unprofessional. Break replacements into: [unchanged text] + [deletion] + [insertion] + [unchanged text]. Preserve the original run's RSID for unchanged text by extracting the `<w:r>` element from the original and reusing it.

Example - Changing "30 days" to "60 days" in a sentence:
```python
# BAD - Replaces entire sentence
'<w:del><w:r><w:delText>The term is 30 days.</w:delText></w:r></w:del><w:ins><w:r><w:t>The term is 60 days.</w:t></w:r></w:ins>'

# GOOD - Only marks what changed, preserves original <w:r> for unchanged text
'<w:r w:rsidR="00AB12CD"><w:t>The term is </w:t></w:r><w:del><w:r><w:delText>30</w:delText></w:r></w:del><w:ins><w:r><w:t>60</w:t></w:r></w:ins><w:r w:rsidR="00AB12CD"><w:t> days.</w:t></w:r>'
```

### Tracked changes workflow

1. **Get markdown representation**: Convert document to markdown with tracked changes preserved:
   ```bash
   pandoc --track-changes=all path-to-file.docx -o current.md
   ```

2. **Identify and group changes**: Review the document and identify ALL changes needed, organizing them into logical batches:

   **Location methods** (for finding changes in XML):
   - Section/heading numbers (e.g., "Section 3.2", "Article IV")
   - Paragraph identifiers if numbered
   - Grep patterns with unique surrounding text
   - Document structure (e.g., "first paragraph", "signature block")
   - **DO NOT use markdown line numbers** - they don't map to XML structure

   **Batch organization** (group 3-10 related changes per batch):
   - By section: "Batch 1: Section 2 amendments", "Batch 2: Section 5 updates"
   - By type: "Batch 1: Date corrections", "Batch 2: Party name changes"
   - By complexity: Start with simple text replacements, then tackle complex structural changes
   - Sequential: "Batch 1: Pages 1-3", "Batch 2: Pages 4-6"

3. **Read documentation and unpack**:
   - **MANDATORY - READ ENTIRE FILE**: Read [`ooxml.md`](ooxml.md) (~600 lines) completely from start to finish. **NEVER set any range limits when reading this file.** Pay special attention to the "Document Library" and "Tracked Change Patterns" sections.
   - **Unpack the document**: `python ooxml/scripts/unpack.py <file.docx> <dir>`
   - **Note the suggested RSID**: The unpack script will suggest an RSID to use for your tracked changes. Copy this RSID for use in step 4b.

4. **Implement changes in batches**: Group changes logically (by section, by type, or by proximity) and implement them together in a single script. This approach:
   - Makes debugging easier (smaller batch = easier to isolate errors)
   - Allows incremental progress
   - Maintains efficiency (batch size of 3-10 changes works well)

   **Suggested batch groupings:**
   - By document section (e.g., "Section 3 changes", "Definitions", "Termination clause")
   - By change type (e.g., "Date changes", "Party name updates", "Legal term replacements")
   - By proximity (e.g., "Changes on pages 1-3", "Changes in first half of document")

   For each batch of related changes:

   **a. Map text to XML**: Grep for text in `word/document.xml` to verify how text is split across `<w:r>` elements.

   **b. Create and run script**: Use `get_node` to find nodes, implement changes, then `doc.save()`. See **"Document Library"** section in ooxml.md for patterns.

   **Note**: Always grep `word/document.xml` immediately before writing a script to get current line numbers and verify text content. Line numbers change after each script run.

5. **Pack the document**: After all batches are complete, convert the unpacked directory back to .docx:
   ```bash
   python ooxml/scripts/pack.py unpacked reviewed-document.docx
   ```

6. **Final verification**: Do a comprehensive check of the complete document:
   - Convert final document to markdown:
     ```bash
     pandoc --track-changes=all reviewed-document.docx -o verification.md
     ```
   - Verify ALL changes were applied correctly:
     ```bash
     grep "original phrase" verification.md  # Should NOT find it
     grep "replacement phrase" verification.md  # Should find it
     ```
   - Check that no unintended changes were introduced


## Converting Documents to Images

To visually analyze Word documents, convert them to images using a two-step process:

1. **Convert DOCX to PDF**:
   ```bash
   soffice --headless --convert-to pdf document.docx
   ```

2. **Convert PDF pages to JPEG images**:
   ```bash
   pdftoppm -jpeg -r 150 document.pdf page
   ```
   This creates files like `page-1.jpg`, `page-2.jpg`, etc.

Options:
- `-r 150`: Sets resolution to 150 DPI (adjust for quality/size balance)
- `-jpeg`: Output JPEG format (use `-png` for PNG if preferred)
- `-f N`: First page to convert (e.g., `-f 2` starts from page 2)
- `-l N`: Last page to convert (e.g., `-l 5` stops at page 5)
- `page`: Prefix for output files

Example for specific range:
```bash
pdftoppm -jpeg -r 150 -f 2 -l 5 document.pdf page  # Converts only pages 2-5
```

## Code Style Guidelines
**IMPORTANT**: When generating code for DOCX operations:
- Write concise code
- Avoid verbose variable names and redundant operations
- Avoid unnecessary print statements

## Dependencies

Required dependencies (install if not available):

- **pandoc**: `sudo apt-get install pandoc` (for text extraction)
- **docx**: `npm install -g docx` (for creating new documents)
- **LibreOffice**: `sudo apt-get install libreoffice` (for PDF conversion)
- **Poppler**: `sudo apt-get install poppler-utils` (for pdftoppm to convert PDF to images)
- **defusedxml**: `pip install defusedxml` (for secure XML parsing)
````

## 参考

- [How to create skills: key steps, limitations and examples](https://claude.com/blog/how-to-create-skills-key-steps-limitations-and-examples)
