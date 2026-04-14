---
name: steady-insights
description: >-
  Analyze Steady/StatusHero check-in data. Connects to the Steady MCP server,
  authenticates, lists teams and members, fetches check-ins for a date range,
  and saves raw output to /tmp/. Use when the user mentions steady analysis,
  statushero report, check-in summary, steady report, or analyze steady.
---

# Analyze Steady / StatusHero

Run through every phase in order. Do NOT skip phases. Each phase has a gate
condition -- only advance when that condition is met.

## Phase 1: MCP Server Availability

Attempt to call the `steady_ping` tool on MCP server `user-steady-mcp`.

### If the tool is not found or the server is unreachable

Tell the user:

> The Steady MCP server is not available in your Cursor environment.
> Follow these steps to set it up:

Provide these numbered steps:

1. Clone the repo:
   ```bash
   git clone https://github.com/arunkalys/steady-mcp.git
   ```
2. Make sure `node` (v18+) is installed:
   ```bash
   node -v
   ```
3. Install dependencies from the repo root:
   ```bash
   cd steady-mcp && npm install
   ```
4. Verify the process is running:
   ```bash
   ps -ef | grep steady-mcp
   ```
5. If it is not running, start it manually:
   ```bash
   node <FULL_PATH>/src/index.js
   ```
   Replace `<FULL_PATH>` with the absolute path to the cloned repo.

Ask the user to confirm the MCP server is running locally before continuing.

Then provide the snippet to add to Cursor's `mcp.json`:

```json
"steady-mcp": {
  "command": "node",
  "args": [
    "<FULL_PATH>/steady-mcp/src/index.js"
  ],
  "env": {
    "STEADY_BASE_URL": "https://app.steady.space"
  }
}
```

Tell the user to replace `<FULL_PATH>` with the actual absolute path and
restart Cursor after editing `mcp.json`.

**Gate:** Do not proceed until `steady_ping` is callable unless user explicitly asks to start with team selection.

---

## Phase 2: Authentication

Call `steady_ping` via MCP:

```
CallMcpTool  server: "user-steady-mcp"  toolName: "steady_ping"  arguments: {}
```

### If ping returns OK

Proceed to Phase 3.

### If ping returns 401 or any non-OK response

Tell the user:

> Steady is not authenticated. The server at https://app.steady.space/
> returned an unauthorized response.

Provide these steps:

1. Open https://app.steady.space/ in your browser and log in.
2. Open DevTools (F12) -> Application -> Cookies -> `https://app.steady.space`.
3. Copy the value of the `_sthr_session` cookie.

Ask the user to paste the cookie value.

Store the pasted value as `USER_STEADY_COOKIE`.

Call MCP to set cookies:

```
CallMcpTool  server: "user-steady-mcp"  toolName: "steady_set_cookies"
arguments: { "cookies": "_sthr_session=<USER_STEADY_COOKIE>" }
```

Call `steady_ping` again to verify. If it still fails, repeat Phase 2
instructions from the beginning.

**Gate:** Do not proceed until `steady_ping` returns OK.

---

## Phase 3: Team Selection

Call MCP to list teams:

```
CallMcpTool  server: "user-steady-mcp"  toolName: "steady_list_teams"  arguments: {}
```

Display the returned teams (name and id) and ask the user to select
**exactly one** team. Use `AskQuestion` if available.
If there is a team called "Virtual Economy Optimization" move it to the top.

Store the selected `team_id`.

**Gate:** Exactly one team must be selected.

---

## Phase 4: Member Selection

Call MCP to list members of the selected team:

```
CallMcpTool  server: "user-steady-mcp"  toolName: "steady_list_members"
arguments: { "team_id": "<selected team_id>" }
```

Display the returned members (name and id). Tell the user they can select
one or more members to generate reports for.

**IMPORTANT – multi-select:** You MUST use `AskQuestion` with
`allow_multiple` set to `true` so the user can pick **multiple** members
in a single step. Example:

```json
{
  "questions": [
    {
      "id": "members",
      "prompt": "Select the team members you want reports for:",
      "allow_multiple": true,
      "options": [
        { "id": "<user_id_1>", "label": "<User Name 1>" },
        { "id": "<user_id_2>", "label": "<User Name 2>" }
      ]
    }
  ]
}
```

Do NOT use `allow_multiple: false` (or omit it). The user must be able to
choose more than one member.

Store each selected `user_id` and `user_name`.

**Gate:** At least one member must be selected.

---

## Phase 5: Date Range

Present the user with predefined date-range options using `AskQuestion`:

```json
{
  "questions": [
    {
      "id": "date_range",
      "prompt": "Select the date range for the report:",
      "options": [
        { "id": "1m", "label": "Last 1 month" },
        { "id": "3m", "label": "Last 3 months" },
        { "id": "6m", "label": "Last 6 months" },
        { "id": "other", "label": "Other (custom range)" }
      ]
    }
  ]
}
```

### If the user selects 1m, 3m, or 6m

Set `end_date` to today's date (`YYYY-MM-DD`). Compute `start_date` by
subtracting the corresponding number of months from today.

### If the user selects "Other"

Ask the user for:

- **Start date** (required)
- **End date** (optional -- defaults to today's date)

Convert the provided dates to `YYYY-MM-DD` format. If a date is invalid or
unparseable, tell the user and ask again.

Store as `start_date` and `end_date`.

**Gate:** `start_date` must be a valid `YYYY-MM-DD` date. `end_date` must be
>= `start_date`.

---

## Phase 6: Fetch and Save

For **each** selected member (`user_id` / `user_name`):

1. Call MCP to get check-ins:
   ```
   CallMcpTool  server: "user-steady-mcp"  toolName: "steady_get_checkins"
   arguments: { "user_id": "<user_id>", "start_date": "<start_date>", "end_date": "<end_date>" }
   ```
2. Build the output filename:
   - Pattern: `/tmp/raw_<user_name>_<start_date>_<end_date>.txt`
   - Replace all spaces in `user_name` with underscores.
3. Write the full MCP response content to that file.

After all members are processed, display the list of files written and note
that these files only contain the raw data:

```
Files written (raw data only):
- /tmp/raw_Jane_Doe_2025-01-01_2025-06-30.txt
- /tmp/raw_John_Smith_2025-01-01_2025-06-30.txt

Note: These files only contain the raw check-in data from Steady.
```

Proceed to Phase 7.

---

## Phase 7: Process and Summarize

For **each** raw data file produced in Phase 6:

1. Read the file (e.g. `/tmp/raw_<user_name>_<start_date>_<end_date>.txt`).
2. Process the contents with the following prompt — apply it yourself as the
   LLM, do **not** send it to an external service:

   > Process the raw check-in data below and produce a structured summary
   > for this team member. Extract and organize the following sections:
   >
   > **Main Areas of Work** — A detailed description of the primary areas
   > and projects the person focused on during this period.
   >
   > **Achievements** — A bullet list of concrete accomplishments,
   > milestones reached, and deliverables completed.
   >
   > **Work Categories** — Classify and highlight the types of work
   > performed. Call out any of the following that apply (omit categories
   > with no evidence):
   >   - Reliability / Engineering Efficiency work
   >   - Design Doc / EDD (Engineering Design Document) related work
   >   - Mentoring / Coaching work
   >   - Collaboration / Coordination across teams or orgs
   >
   > **Challenges** — Blockers, difficulties, or areas where progress was
   > slowed. Include context on how they were addressed if available.

3. Write the processed summary to a new file:
   - Pattern: `/tmp/summary_<user_name>_<start_date>_<end_date>.md`
   - Use the same `user_name` (spaces replaced with underscores),
     `start_date`, and `end_date` as the corresponding raw file.

After all members are processed, display the list of summary files:

```
Summary files written:
- /tmp/summary_Jane_Doe_2025-01-01_2025-06-30.md
- /tmp/summary_John_Smith_2025-01-01_2025-06-30.md
```

**Gate:** Every raw file from Phase 6 must have a corresponding summary file.

Done.
