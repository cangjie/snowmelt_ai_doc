---
name: end-work
description: Capture work session summary and project state changes before ending a session on Snowmeet AI. Use this at the end of your work day or when wrapping up a major task — triggers on "end work", "done for today", "wrapping up", "save progress", "update context", "that's all for now", or any indication that a substantial work session is complete. This ensures the project documentation stays current and future sessions have accurate context, and archives the session transcript to `snowmeet_ai_doc/sessions/`.
---

# End Work — Capture Session Summary, State, and Transcript

Closing a work session on an evolving project requires documenting what changed. This skill helps you capture session progress, update the project context, and archive the conversation transcript so the next session picks up with accurate information.

**项目上下文文件位置**：`snowmeet_ai_doc/CLAUDE.md`（位于仓库根目录下的 `snowmeet_ai_doc/` 目录，例如 `/Users/cangjie/source/snowmeet/snowmeet_ai/snowmeet_ai_doc/CLAUDE.md`）。所有读写都必须落到这里。

**聊天记录归档位置**：`snowmeet_ai_doc/sessions/YYYY-MM-DD_{topic}.md`。当前 working dir 下一定能找到 `snowmeet_ai_doc/sessions/` 目录（首次触发时若不存在则 `mkdir -p` 创建）。

## Process

1. **Summarize** what was accomplished:
   - What files were created/modified?
   - What new functionality is working?
   - What blockers or learnings emerged?
   - What's ready for the next session?

2. **Identify** what needs updating in CLAUDE.md:
   - If you completed a step in the current iteration, note which tasks moved from 🚧 to ✅
   - If you discovered new gotchas, add them to "Known issues"
   - If the "Next steps" list changed, highlight what's new
   - If you added new key files, they should be listed
   - Add a new dated entry to the dev log with today's work
   - Update the "当前状态" date stamp if it shifted (e.g. 2026-05-12 下午 → 2026-05-14 凌晨)

3. **Prepare** the changes:
   - Draft the exact updates to CLAUDE.md (formatted as markdown)
   - Draft the transcript archive markdown (see template below)
   - Show the user what will be added
   - Ask for confirmation before updating

4. **Finalize** (after user approval):
   - Update CLAUDE.md with the new entries
   - Write the transcript archive to `snowmeet_ai_doc/sessions/YYYY-MM-DD_{topic}.md`
   - Create a brief handoff note summarizing what's ready vs. what's blocked
   - Confirm all changes are saved

## Transcript archive format

Filename: `snowmeet_ai_doc/sessions/{YYYY-MM-DD}_{short-topic-slug}.md`
- 日期用 session 起始当天（跨夜也按起始日）
- topic-slug 用英文小写 + 连字符，3-5 词概括主题（如 `rent_order_diff_and_skill`、`payment_identity_plan`、`auth_middleware_rewrite`）
- 若已存在同名文件，加 `-2` / `-3` 后缀

模板：

```markdown
# {YYYY-MM-DD} {简短标题}：{一句话概述}

按时间线/主题整理。{背景一两句，说明这场会话是接续什么任务、改动落在哪个目录}。

## 1. {主题一标题}

### 1.1 {子节}

- 关键发现/操作 1
- 关键发现/操作 2
- ...

### 1.2 {子节}

- ...

## 2. {主题二标题}

...

## 关键改动文件

| 文件 | 改动 |
|---|---|
| `path/to/file.py` | 简述 |
| ... | ... |

## 学到的小知识

1. **{要点}**：详细解释
2. ...
```

写法原则：
- 详细到能复现 + 理解前因后果，重要 SQL/字段差异/路径要保留
- 不要事无巨细贴所有 tool call，按"做了什么、为什么、结果如何"组织
- 用户提出的核心问题/决策必须保留原话或贴近原意
- 每条 bullet 30 字内为佳；长论述用 sub-bullet 或独立段落
- 涉及代码引用用 markdown 链接到相对路径

## Why this matters

CLAUDE.md 是项目状态的单一来源；sessions/ 是工作过程的详细备查。前者让下一次会话能立即接上，后者让"那天为什么这条数据找不出来 / 减免怎么定义的"这种细节问题翻一份 markdown 即可，不用爬聊天记录。

## Output format

Present three sections:
- **What changed** — 2–3 bullet points on accomplishments and blockers
- **Suggested updates to CLAUDE.md** — The exact text to add/modify, ready to copy in
- **Transcript archive path** — `snowmeet_ai_doc/sessions/YYYY-MM-DD_{topic}.md`（draft 内容随后展示或直接写）

## Example trigger phrases

- "end work"
- "done for today"
- "wrapping up"
- "let's save progress"
- "update the context"
- "save my work"
- "今天到这"
- "收尾"
