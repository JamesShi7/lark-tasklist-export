---
name: lark-tasklist-export
description: 从飞书任务清单（tasklist）中提取所有未完成任务及其自定义字段，生成结构化的 Markdown 文档。当用户要求导出/提取/读取飞书任务清单、把飞书任务做成文档、导出任务列表为 markdown 时使用。也适用于用户说"帮我看看任务清单里有什么"、"把清单里的任务整理成文档"等场景。即使用户没有明确说"导出"或"markdown"，只要涉及从飞书任务清单中批量提取任务信息并生成文档，就应触发此 skill。
---

# 飞书任务清单导出 Skill

从飞书任务清单中提取完整的未完成任务数据（含自定义字段、子任务、负责人姓名），生成一份结构化的 Markdown 文档，并可选生成一个可视化 Dashboard 网页。

## 前置条件

- 已完成 `lark-cli config init`（见 `../lark-shared/SKILL.md`）
- 以 user 身份执行（`--as user`），因为需要访问用户的任务清单和通讯录
- 需要 scope：`task:task:read`、`task:tasklist:read`、`contact:user.base:readonly`

## 核心工作流

```
1. 定位清单 → 2. 拉取任务列表 → 3. 逐条获取自定义字段 → 4. 反推字段选项映射 → 5. 解析人名 → 6. 获取子任务 → 7. 生成 Markdown → 8.（可选）生成 Dashboard
```

下面逐步说明。

### Step 1: 定位清单

用户可能给出清单名称或清单链接。

```bash
# 按名称搜索
lark-cli task +tasklist-search --query "清单名称" --as user

# 搜索结果中取 guid 字段
```

如果用户直接给了 `?guid=xxx` 链接，直接提取 guid。

### Step 2: 拉取所有未完成任务

这一步只需要拿基础字段（标题、截止时间、成员、子任务数）。自定义字段在 Step 3 单独获取。

```bash
# 首页（不加 page_token）
lark-cli task tasklists tasks --as user \
  --params '{"tasklist_guid":"<GUID>","page_size":50}'
```

**关键点**：
- `tasklists.tasks` 返回的是摘要，不含 `custom_fields`，这是正常的
- `completed_at == "0"` 表示未完成
- 必须串行翻页：检查 `has_more` 和 `page_token`，有则继续
- `page_size` 最大 50
- 此 API 禁止并发

收集所有 `completed_at == "0"` 的任务的 `guid`、`summary`、`due`、`members`、`subtask_count`。

### Step 3: 逐条获取自定义字段

对 Step 2 收集的每个任务 guid，调用详情 API 获取 `custom_fields`：

```bash
lark-cli task tasks get --as user \
  --params '{"task_guid":"<GUID>"}'
```

**关键点**：
- 响应中的 `custom_fields` 数组包含每个自定义字段的 `name`、`type`、`single_select_value`（或其他类型的值）
- `single_select_value` 返回的是 option 的 GUID，不是文本名称
- 必须串行调用，每个请求之间 sleep 0.2 秒，避免触发限流
- 这一步是整个流程中最耗时的（N 个任务 = N 次 API 调用）

### Step 4: 反推自定义字段选项映射

飞书目前没有 API 直接获取自定义字段的选项定义。必须从实际数据中反推，然后让用户确认。

**具体做法**：

1. 按 `custom_fields[].name` 分组——不同清单的自定义字段名称、数量、类型完全不同，不要假设任何固定字段名
2. 对每个字段，列出所有出现过的 `single_select_value` GUID
3. 对每个 GUID，列出使用了该选项的任务标题（帮助推断含义）
4. 根据任务标题中的【标签】、关键词等上下文，为每个 GUID 推断一个名称
5. 将推断结果整理成表格，**呈现给用户确认**，格式如下：

```
请确认以下自定义字段映射（我根据任务内容推断的）：

## {字段名1}
| GUID | 推断名称 | 依据任务 |
|------|---------|---------|
| abc1... | {推断名称} | {使用了此选项的任务标题} |
| abc2... | {推断名称} | {使用了此选项的任务标题} |

## {字段名2}
| GUID | 推断名称 | 依据任务 |
|------|---------|---------|
| def1... | {推断名称} | {使用了此选项的任务标题} |
| ... | ... | ... |

如有错误请指出，确认无误后我继续生成文档。
```

6. 如果用户提供了截图，结合截图中的信息来推断
7. 对于无法推断的 GUID，标记为 `未知选项(GUID前8位)`，让用户提供名称

**重要**：每个清单的自定义字段完全不同，不能跨清单复用映射。每次处理新清单都必须重新推断和确认。

### Step 5: 解析负责人姓名

从 Step 2 的 `members` 中收集**所有**出现的 `open_id`（包括 `role == "assignee"` 和 `role == "follower"`），然后解析姓名：

```bash
# 自己
lark-cli contact +get-user --as user

# 其他人
lark-cli contact +get-user --user-id "<open_id>" --as user
```

提取返回的 `data.user.name` 字段。串行调用，每个间隔 0.2 秒。

注意：只收集 assignee 会导致遗漏（如 follower 人员），务必收集所有成员。

### Step 6: 获取子任务

对 `subtask_count > 0` 的任务，获取子任务列表：

```bash
lark-cli task subtasks list --as user \
  --params '{"task_guid":"<PARENT_GUID>"}'
```

每个子任务提取：`summary`（标题）、`completed_at`（是否完成）、`members`（负责人）。

### Step 7: 生成 Markdown

输出模板参考 [`references/output-template.md`](references/output-template.md)，包含：

1. **文档头部**：清单名称、导出时间、任务总数
2. **统计概览**：按截止时间和第一个自定义字段的简要统计
3. **任务表格**：按截止时间排序（今天 → 明天 → 未来 → 无截止时间），列为：
   - 序号、任务标题、各自定义字段列、截止时间、子任务数、负责人
   - 自定义字段列的名称和数量因清单而异，按实际推断结果动态生成
4. **子任务详情**：对有子任务的任务，在表格后列出子任务清单（含完成状态和负责人）
5. **附录**：
   - 各自定义字段的选项说明（如用户有提供含义）
   - 团队成员 open_id ↔ 姓名对照表

将生成的 Markdown 文件保存到用户当前工作目录下，文件名格式：`<清单名称>-未完成任务.md`

### Step 8: 可选生成 Dashboard 网页

Markdown 生成完成后，**询问用户**是否需要生成一个可视化的 Dashboard 网页：

> "Markdown 文档已生成。需要我额外生成一个可视化的 Dashboard 网页吗？可以在浏览器里直接打开，支持看板/分组/表格三种视图，还能筛选和搜索。"

如果用户同意，按照 [`references/dashboard-template.md`](references/dashboard-template.md) 生成 HTML 文件。

关键点：
- Dashboard 是一个**自包含的 HTML 文件**，所有数据内嵌为 JS 对象，无需服务器
- 任务数据从已提取的数据直接生成（不再调 API）
- 文件名格式：`task-dashboard.html`，保存在与 Markdown 相同的目录
- 样式和布局参考模板，但自定义字段名称、选项颜色等根据实际数据动态调整

如果用户拒绝，仅返回 Markdown 文件即可。

## 注意事项

- 所有 list 类 API（`tasklists.tasks`、`subtasks.list`）必须串行调用
- 批量获取任务详情时，每个请求间隔 0.2-0.3 秒
- 截止时间从毫秒时间戳转为本地时区（UTC+8）可读日期
- 当天显示"今天"，次日显示"明天"
- 飞书任务清单的自定义字段因清单而异，不同清单的字段和选项完全不同，不能跨清单复用映射
- 如果用户提到业务规则（如"某个字段只适用于特定项目"），在生成的文档中注明
- Dashboard 网页中的颜色方案：按第一个自定义字段分配不同颜色，提供默认配色方案
