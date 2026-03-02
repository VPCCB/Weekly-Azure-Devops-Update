---
name: weekly-kr-update
description: Automates weekly status updates for Azure DevOps work items (Key Results, User Stories, Tasks, Bugs, Features, etc.) by pulling incremental progress from ADO and the M365 ecosystem (emails, Teams chats, meetings, files). Use this skill whenever the user mentions updating KRs, weekly KR updates, posting status on work items, OKR updates, user story updates, or wants to write status updates for their ADO items. Also trigger when the user pastes an ADO query URL and asks for updates, or says things like "update my KRs", "post weekly status", "what did I do this week", "update my work items", "add comments to my user stories", or "update my ADO items".
---

# Weekly KR Update

This skill automates the end-to-end workflow of writing and posting weekly incremental status updates on Key Result (KR) work items in Azure DevOps, using data from both ADO and the broader M365 ecosystem.

## Prerequisites

The following MCP tools must be available:
- **Azure DevOps (ADO)**: `wit_get_query_results_by_id`, `wit_get_work_items_batch_by_ids`, `wit_list_work_item_comments`, `wit_add_work_item_comment`
- **WorkIQ**: `ask_work_iq` (for querying M365 data — emails, chats, meetings, files)

## Workflow

### Step 1: Resolve the ADO Query

**If the user provides a query URL or ID**, extract the query ID and project name from it.
- Example URL: `https://<org>.visualstudio.com/<Project>/_queries/query/<query-guid>/`
- Extract: project = `<Project>`, query ID = `<query-guid>`

**If the user just says "update my KRs"** (no URL provided), check the user's Claude memory files for a saved default query. Look for a `weekly-kr-update-config` section in memory. If no default is found, ask the user for their ADO query URL, then save it to memory for future use.

Also check if the user has a saved preference for which work item types to update. If not, ask: "Which work item types do you want to update? (e.g., Key Result, User Story, Task, Bug, Feature)" and save their selection.

```
## weekly-kr-update-config
- default_project: <project name>
- default_query_id: <query ID>
- work_item_types: Key Result, User Story
```

### Step 2: Fetch and Identify KRs

1. Call `wit_get_query_results_by_id` with the query ID and project, using `responseType: "full"` and `top: 100`
2. Parse the hierarchical `workItemRelations` to identify:
   - Top-level items (where `rel: null`) — these are typically Objectives
   - Child items (where `rel: "System.LinkTypes.Hierarchy-Forward"`) — these are typically KRs
3. Collect ALL unique work item IDs from the results
4. Call `wit_get_work_items_batch_by_ids` with all IDs to get full details
5. Filter to only work items where:
   - `System.WorkItemType` matches one of the user's configured types (from memory, e.g., `Key Result`, `User Story`, `Task`). If no preference is saved, ask the user which types they want to update and save the selection to memory.
   - `System.AssignedTo` matches the current user

### Step 3: Establish Previous Week's Baseline and Determine Date Range

For each KR identified in Step 2:

1. Call `wit_list_work_item_comments` for the work item
2. Identify the most recent comment — this represents the previous update
3. Note the content of that comment as the baseline to diff against
4. **Extract the end date** from the most recent comment's header. Comments follow the format `Week [StartDate] – [EndDate]:`. Parse the `EndDate` — the **start date for this run** is the day after that `EndDate`.
   - Example: if the latest comment says `Week Feb 23 – Feb 27:`, then this run's start date is `Feb 28`
5. If no previous comment exists (first run for this KR), default to Monday of the current week
6. **End date** is always **today's date**

This approach ensures there are no gaps or overlaps between updates, and works regardless of how frequently the skill is run.

This baseline is what makes updates **incremental** — you only write what's new since the last comment.

### Step 4: Gather Updates from M365 for the Determined Date Range

For each KR, call `ask_work_iq` with a targeted question. The question should:
- Reference the KR's title and key topic areas
- Explicitly scope to the date range determined in Step 3 (from day after last update's end date through today)
- Ask for emails, chats, meetings, meeting transcripts, and files
- Be specific enough to get relevant results, not generic

Example question format:
> "What happened specifically between [start date] and [today's date] related to [KR topic keywords]? Only include activity from [date range]. Include emails, chats, meetings, and files."

To maximize efficiency, make parallel WorkIQ calls for KRs that cover different topic areas. If some calls fail, retry them.

### Step 5: Write Incremental Updates

For each KR, compare the M365 findings (Step 4) against the previous week's ADO comment (Step 3):

1. **Identify what's genuinely new** — only include activity, decisions, milestones, or progress that was NOT mentioned in the previous comment
2. **If no incremental updates exist**, write: `NA — No new activity found this week.` followed by a brief note on current status
3. **Strip all personal names** — never include names of people in the update. Use role-neutral language (e.g., "the team agreed" instead of "[Name] agreed", "reviewer feedback was addressed" instead of "[Name] approved")
4. **Strip specific dates** — do not include day-level dates within the update paragraph
5. **Include concrete details** — mention PR numbers, metric values, meeting topics, decisions made, blockers identified. Avoid vague language
6. **Keep it to one paragraph** per KR

### Step 6: Format the Updates

Each update must follow this exact format:

```
Week [StartDate] – [EndDate]:
Update: [one paragraph of incremental progress]
```

Where:
- **StartDate** = day after the previous comment's end date (from Step 3)
- **EndDate** = today's date

Examples:
```
Week Feb 28 – Mar 4:
Update: PR #4914867 for NegativeToneEmailAtMention was approved and merged to master, advancing from "in review" last week. The signal is estimated to ship in build 1.0.3673.0. Reviewer feedback was addressed, scenario naming was aligned for long-term extensibility, and all required reviewer groups approved the change.
```

If run again later in the same week:
```
Week Mar 5 – Mar 7:
Update: Signal onboarding progressed with the dataplane PR passing validation. The onboarding form was submitted and portal issues were resolved.
```

### Step 7: Confirm and Post to ADO

1. Present ALL updates to the user in a summary table before posting:
   - Show KR ID, title, and the proposed update
   - Ask: "Ready to post these updates to ADO?"
2. Once confirmed, call `wit_add_work_item_comment` for each KR with:
   - `format: "markdown"`
   - The formatted update as the comment text
3. Make all comment posts in parallel for efficiency
4. Confirm success with a summary table showing KR ID, title, and post status

## Important Guidelines

- **Incremental only**: The entire value of this skill is in writing ONLY what's new. Always diff against the previous comment. If a metric was already reported last week and hasn't changed, don't repeat it.
- **No names, ever**: This is a firm rule. Updates should be written in a way that focuses on what happened, not who did it.
- **No dates within the paragraph**: The week header provides the date context. The update paragraph itself should not contain specific dates.
- **Concrete over vague**: "PR merged to master" is better than "progress was made". "LU accuracy at 90.4%" is better than "metrics were reviewed".
- **NA is acceptable**: Not every KR will have updates every week. That's fine — marking NA with a brief status note is better than inflating non-progress.
- **Always confirm before posting**: Never post comments to ADO without showing the user first and getting confirmation.
