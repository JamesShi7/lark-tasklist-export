# lark-tasklist-export

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that exports [Feishu / Lark](https://www.feishu.cn/) tasklists into structured Markdown documents, with an optional interactive Dashboard.

## What It Does

Given a Feishu tasklist name or link, this skill:

1. **Locates** the tasklist via Feishu Task API
2. **Fetches** all uncompleted tasks (basic fields)
3. **Enriches** each task with custom field values (priority, project, group, etc.)
4. **Reverse-engineers** custom field option mappings (Feishu has no API for this — the skill infers GUID-to-name mappings from task context and asks you to confirm)
5. **Resolves** assignee/follower names via Contacts API
6. **Fetches** subtask details for tasks that have them
7. **Generates** a structured Markdown document
8. *(Optional)* **Generates** a self-contained HTML dashboard with board / grouped / table views, filters, and search

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [lark-cli](https://github.com/nicepkg/lark-cli) configured with user authentication
- Required scopes: `task:task:read`, `task:tasklist:read`, `contact:user.base:readonly`

## Installation

Copy this folder into your Claude Code skills directory:

```bash
cp -r lark-tasklist-export ~/.claude/skills/
```

## Usage

In Claude Code, just say:

> "帮我导出飞书任务清单 XXX"
> "把我的飞书任务清单整理成文档"
> "导出任务列表为 markdown"

Or provide a tasklist link directly.

After generating the Markdown, you'll be asked whether you also want an interactive Dashboard HTML.

## Output Example

**Markdown** — A structured document with:
- Stats overview (due today/tomorrow, high priority count, etc.)
- Task table sorted by due date, with custom field columns
- Subtask details with completion status
- Appendix with field definitions and team member list

**Dashboard** — A single HTML file you can open in any browser:
- Three views: Kanban board, grouped by project, full table
- Filter by priority / project / group
- Search tasks
- Subtask expansion

## File Structure

```
lark-tasklist-export/
├── SKILL.md                          # Core workflow (8 steps)
├── README.md                         # This file
└── references/
    ├── output-template.md            # Markdown output template
    ├── dashboard-template.md         # Dashboard generation guide
    └── dashboard-template.html       # Dashboard style/layout template
```

## How It Works

Feishu's Task API returns custom field values as opaque GUIDs rather than human-readable names, and there's no API to retrieve the option definitions. This skill works around that limitation by:

1. Collecting all unique GUID values across all tasks
2. Inferring names from task context (titles, tags like `【Legal】` or `【HR】`)
3. Presenting the inferred mapping as a table for you to confirm or correct
4. Proceeding only after you've verified the mappings

This approach works for any tasklist — no hardcoded mappings.

## License

MIT
