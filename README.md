# 🔄 Weekly Azure DevOps Update

A Claude Code skill that automates weekly status updates for your Azure DevOps work items by combining activity from ADO with insights from the M365 ecosystem — emails, Teams chats, meetings, and files.

## ✨ What it does

- 📥 Fetches your work items (Key Results, User Stories, Tasks, etc.) from an ADO query
- 📊 Reads previous comments to establish a baseline
- 🔍 Searches your M365 data (emails, Teams chats, meetings, files) for recent activity related to each work item
- ✏️ Writes **incremental-only** updates — only what's new since the last update
- 📤 Posts the updates as comments directly on each work item in ADO

## ⚙️ How it works

The skill picks up from where it left off. It reads the end date from your last posted comment and covers everything from that date to today. No gaps, no overlaps — run it weekly, mid-week, or after a vacation and it adjusts automatically.

Updates are written in a clean, consistent format:

```
Week Mar 2 – Mar 6:
Update: PR #1234 was approved and merged to master. The team aligned on
catalog-style publishing for signal definitions. Onboarding form submitted
with follow-ups on portal issues.
```

All updates are **name-free** and **date-free** within the paragraph — the header provides the time context.

## 📋 Prerequisites

This skill requires the following MCP tools to be configured in your Claude Code environment:

- 🔧 **Azure DevOps MCP** — for reading and writing work items and comments
- 🔧 **WorkIQ MCP** — for querying M365 data (emails, chats, meetings, files)

## 📦 Installation

```bash
claude install github:<your-username>/Weekly-Azure-Devops-Update
```

## 💡 Usage

Simply say:

- 💬 `"update my KRs"`
- 💬 `"post weekly status"`
- 💬 `"update my work items"`
- 🔗 Or paste an ADO query URL and ask for updates

### 🛠️ First-time setup

On first use, the skill will ask you two things:

1. 🔗 **Your ADO query URL** — the query that contains your work items
2. 📝 **Which work item types to update** — e.g., Key Result, User Story, Task

These preferences are saved automatically. You won't be asked again unless you want to change them.

### 🚀 Subsequent runs

Just say `"update my KRs"` and the skill will:

1. 📥 Fetch your work items
2. 🔍 Gather incremental updates from ADO + M365
3. 👀 Show you a summary for review
4. ✅ Post to ADO after your confirmation

## 🔧 Configuration

The skill saves your preferences in Claude's memory:

| Setting | Description | Example |
|---|---|---|
| `default_project` | Your ADO project name | `O365 Core` |
| `default_query_id` | Your ADO query GUID | `040018a6-...` |
| `work_item_types` | Work item types to update | `Key Result, User Story` |

To change any setting, just tell the skill (e.g., "also include Tasks in my updates").

## 📄 License

[MIT](LICENSE)
