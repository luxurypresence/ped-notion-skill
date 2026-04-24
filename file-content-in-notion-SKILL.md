---
name: file-content-in-notion
description: >
  File content into shared PED Notion databases. Four operations, used alone or together:
  (A) create a new `Projects.db` row for a project, (B) create a new `Docs.db` row that points
  to an existing page via `Source URL` with the right Project and Team relations,
  (C) create or manage a `Work Items.db` row attached to a project, team, and orgs, and
  (D) create or manage a `Meetings.db` row (meeting notes, attendees, decisions) linked to a
  project, team, work items, and docs. Use this skill whenever the user says "create a project
  in Notion," "add a project to [team] in Notion," "add this project to the [team] team," "file
  this under [project]," "add this doc to [project] in Notion," "link this page to [project],"
  "create a Docs entry for this," "create a work item," "add a work item to [project]," "add this
  to the roadmap," "put this on the [team] roadmap," "assign this work item to [person],"
  "link this work item to [milestone]," "mark this work item done," "archive this work item,"
  "create a meeting," "log a meeting for [project]," "add a meeting to
  [team]," "link this meeting to [project / work item / doc]," "add attendees to [meeting],"
  "reschedule [meeting]," or drops a Notion URL and asks for it to be associated with a project,
  team, work item, or meeting. Always use this skill — the happy path has several gotchas
  (property naming, team URL ambiguity, hub-page-vs-Teams.db-row confusion, `userDefined:` prefix
  rules, multi-value relation array format, `update_properties` replacing instead of appending)
  that the skill handles.
compatibility: "Requires Notion MCP"
---

# File Content into Shared Notion Databases

Four related operations, often done together:

- **A. Create a new `Projects.db` row** for a project that doesn't exist yet.
- **B. Create a new `Docs.db` row** that points to an existing page via `Source URL`, with relations to a Project and Team.
- **C. Create or manage a `Work Items.db` row** attached to a project, team, and orgs (covers create plus common follow-up updates: status change, owner reassignment, milestone linking, mark Done, archive).
- **D. Create or manage a `Meetings.db` row** for meeting notes / attendees / decisions, linked to a team, optionally to projects, work items, docs, and initiatives (covers create plus common follow-up updates: add/remove attendees, reschedule, link to additional projects or work items).

**Do NOT move or duplicate the original page content** for operation B. The Docs.db row is a thin metadata record; the real content stays where it already lives. This is the "project-as-thin-row" pattern applied to docs: the DB row is structure; the page it points to is content.

This skill assumes a Notion workspace with shared databases for Projects, Docs, Work Items, Meetings, and Teams that use relation properties to connect them. Collection IDs, team URLs, org URLs, and user IDs are **not baked into this skill** — discover them from the live workspace (see "Resolving IDs" below).

## When to do which

- User drops a Notion URL and names a project that already exists → skip to Part B (file the doc).
- User asks to "create a project" or names a project that doesn't exist → Part A first (create the project), then Part B if they also dropped a doc.
- User asks to create a work item under an existing project → skip to Part C.
- User asks to "add something to the roadmap," "put this on the [team] roadmap," or otherwise references roadmap items → Part C. In PED, roadmap views are projections of `Work Items.db`; creating a roadmap entry means creating a work item.
- User asks to update status / owner / milestone, mark Done, or archive an existing work item → Part C, management subsection (C.7).
- If the project a new work item should attach to doesn't exist → Part A first, then Part C.
- User asks to log / create a meeting (with notes, attendees, links to projects or work items) → skip to Part D.
- User asks to add attendees, reschedule, or link an existing meeting to more projects / work items / docs → Part D, management subsection (D.7).
- For "find meetings about X" or "list meetings on team Y," prefer the dedicated `notion-query-meeting-notes` MCP tool before falling back to direct fetches.
- Before creating a project, **always confirm it doesn't already exist** by searching `Projects.db`.

## Resolving IDs (do this every time)

Do not hardcode collection IDs, team URLs, org URLs, or user IDs. Discover them live:

1. **Find the Projects.db / Docs.db / Work Items.db / Meetings.db / Teams.db collection IDs.** Fetch any known project, doc, work item, meeting, or team page with `notion-fetch`. The response's `parent-data-source url="collection://..."` tag gives the collection ID. Alternatively, your workspace may have a reference doc listing these — check the tool's own documentation if available.
2. **Find the Team URL.** Search `Teams.db` by team name with `notion-search` and the team DB's `data_source_url`. The team's row URL in Teams.db is what goes into the `Team(s)` relation — **not** a hub/shell page with the same name.
3. **Find the Org URL(s).** The easiest way: fetch a sibling project on the same team and copy its `Org` values. Most workspaces use multiple orgs per project (often 2). Never guess Org URLs; derive them from a sibling.
4. **Find the Owner user ID.** Default to the user running the skill. If a different owner is needed, ask the user for a mention or search users.

If any of these lookups fail, ask the user for the missing ID before proceeding.

## Common prerequisites

For any of the four operations, you need:

- **Team URL** (the team's row in Teams.db)
- **Org URL(s)** (usually multi-value; copied from a sibling project — note: Meetings.db does not require Org)
- **Owner(s)** / **Attendees** (user mentions in `user://UUID` format — Parts A/B/C use `Owner(s)` single-person; Part D uses `Meeting Attendees` multi-person)

Always ask the user to confirm the team if it's ambiguous (e.g., a team called "X" vs. a group page also called "X"). Team names shared with hub/group pages are a common source of errors.

---

## Part A — Create a new Project in Projects.db

### A.1 Confirm the project doesn't exist

Search `Projects.db` for the proposed name using `notion-search` with the Projects.db `data_source_url`. If a match exists, ask the user whether to reuse it before creating a duplicate.

### A.2 Gather the fields

| Field | Value source |
|---|---|
| `Name` | From user (confirm exact wording) |
| `Status` | Usually the "in progress" value for new projects. Valid values depend on the workspace — check a sibling project if unsure |
| `Owner(s)` | User mention. Default to the current user unless told otherwise |
| `Team(s)` | From prerequisites |
| `Org` | Multi-value. Copy from a sibling project on the same team |
| `Summary` | Short one-liner describing the project's purpose. Ask the user if not obvious |
| `icon` | Optional emoji |

Leave blank by default: priority, sequence, date, initiatives, milestones, collaborators, estimate, progress.

### A.3 Propose the draft to the user

Present a table of proposed values **before** writing. Wait for confirmation.

### A.4 Create the row (workaround for multi-value Org)

Multi-value relations are inconsistent on `notion-create-pages` across databases. For `Projects.db` specifically, creating with the JSON-array string has failed in practice. Reliable pattern:

1. **Create** the project with a **single** Org URL (the first one).
2. **Update** the Org property afterward using the full JSON-array string of all Orgs.

```
# Step A.4a — create with one Org
notion-create-pages
  parent:
    type: "data_source_id"
    data_source_id: "<Projects.db collection ID>"
  pages:
    - properties:
        Name: "<project name>"
        Status: "<in-progress status value>"
        Owner(s): "user://<UUID>"
        Team(s): "<team page URL>"
        Org: "<first org URL only>"
        Summary: "<one-liner>"
      icon: "<emoji>"

# Step A.4b — set full Org array via update
notion-update-page
  page_id: "<new project ID>"
  command: "update_properties"
  properties:
    Org: "[\"<org1 URL>\",\"<org2 URL>\",...]"
```

**Critical:** `update_properties` **replaces** the relation, so step A.4b must pass the **full** list including the first Org. Do not pass only the new Orgs or you'll lose the first.

### A.5 Verify

Fetch the new project page and confirm all expected properties (especially all Orgs) are set. Return the URL to the user.

---

## Part B — File a Doc into Docs.db

### B.1 Fetch the source page

Use `notion-fetch` with the page URL or ID. Confirm the page exists. Capture:

- **Title** (becomes `Name`)
- **Author / Owner hints** (look for "Author: X" lines, metadata tables, or explicit ownership)
- **Audience / Purpose** lines (help infer `Type`)
- **Creation / publication date** (helps infer `Status`)
- **Icon** (if the source page has one, reuse it on the Docs.db row)

If the page is already a row in `Docs.db`, stop and tell the user — no need to duplicate.

### B.2 Resolve the Project URL

If the user gave a URL, use it. If they gave a name, search `Projects.db`. If multiple matches, ask to disambiguate. **Never guess.** If the project doesn't exist, run Part A first.

### B.3 Resolve the Team URL

See prerequisites.

### B.4 Draft the Docs.db row and confirm

Show the user a table of proposed values **before** writing. Required fields:

| Field | Value source |
|---|---|
| `Name` | From source page title |
| `Type` | Valid values depend on the workspace — check a sibling doc if unsure (common options include Strategy Memo, PRD, Decision Memo, PRFAQ, Project Proposal, Roadmap Overview, Brief, SOP, Playbook) |
| `Status` | Typically `Not started` / `In progress` / `Done` — confirm with a sibling doc |
| `Owner(s)` | User mention |
| `Project(s)` | From B.2 (full Notion page URL) |
| `Team(s)` | From B.3 (full Notion page URL) |
| `Source URL` | The original page URL — **this is the whole point** |

Optional fields (set if there's a clear signal, otherwise skip):

- `Org` — match the Project's `Org` values (multi-value; see gotchas)
- `Collaborators`, `Review date`, `Date Completed`, `Date Archived`

**Leave blank by default:** `AI Summary`, `File`, `Initiative(s)`, `Work Items`, `Meetings`, `Verification`.

### B.5 Create the row

For single-value relations (`Project(s)`, `Team(s)`), pass a plain URL string.
For multi-value `Org`, pass a JSON-array string in the same `create-pages` call — this typically works on create for Docs.db (unlike Projects.db, which requires the two-step workaround above).

```
notion-create-pages
  parent:
    type: "data_source_id"
    data_source_id: "<Docs.db collection ID>"
  pages:
    - properties:
        Name: "<from B.4>"
        Type: "<from B.4>"
        Status: "<from B.4>"
        Owner(s): "user://<UUID>"
        Project(s): "<project page URL>"
        Team(s): "<team page URL>"
        Org: "[\"<org1 URL>\",\"<org2 URL>\",...]"
        Source URL: "<original page URL>"
      icon: "<emoji if the source page has one>"
```

Leave `content` unset (blank page body). The Source URL carries readers to the real content.

Return the new Docs.db row URL to the user.

---

## Part C — Create or manage a Work Item in Work Items.db

Work items are the third major node in the project-as-thin-row model — they live in `Work Items.db` and attach to a Project, a Team, and one or more Orgs. In PED, work items are what surface in **Roadmap views** (the team's Roadmap page, board / timeline / DC views). Creating a properly-attached work item is what makes it appear on the right team's roadmap. Schema is fully documented in the workspace's notion-structure reference (the `Work Items.db` section). Refer to that for the full property list; this part covers the happy paths.

### C.1 Resolve the Project URL

If the user gave a URL, use it. If they gave a name, search `Projects.db`. If multiple matches, ask to disambiguate. **Never guess.** If the project doesn't exist, run Part A first, then come back here.

### C.2 Resolve the Team URL

See prerequisites. The Team URL must be the project's team — confirm by fetching the resolved Project page and reading its `Team(s)` value.

### C.3 Gather fields

Required:

| Field | Value source |
|---|---|
| `Name` | From user (the work item title) |
| `Status` | Default `To Do` for new items — confirm valid values via a sibling work item on the same project (status labels can drift; see gotchas) |
| `Owner(s)` | User mention. Default to the current user unless told otherwise |
| `Project(s)` | From C.1 (full Notion page URL) |
| `Team(s)` | From C.2 (full Notion page URL) |
| `Org` | Multi-value. **Copy from the resolved Project's `Org` values** — do not guess |

Optional (set if there's a clear signal, otherwise skip):

- `Priority` (select)
- `Delivery Cycle` (select)
- `Date` (due date)
- `Description` (short text)
- `Initiatives` (relation, multi-value)
- `Milestone` (relation, max 1)
- `Collaborators` (person, multi-value)

**Leave blank by default:** `Sub-item`, `Parent item`, `Docs`, `Meeting(s)`, `Est. Hrs.`, `Actual (hrs)`, `Date Completed`, `Date Archived`, `Place`.

### C.4 Draft and confirm

Show the user a table of proposed values **before** writing. Wait for confirmation.

### C.5 Create the row (single-shot)

`Work Items.db` accepts the multi-value `Org` JSON-array string on `notion-create-pages` — same as `Docs.db`, unlike `Projects.db`. **Do not** use the Projects.db two-step create-then-update workaround here unnecessarily.

```
notion-create-pages
  parent:
    type: "data_source_id"
    data_source_id: "<Work Items.db collection ID — discover live, do not hardcode>"
  pages:
    - properties:
        Name: "<from C.3>"
        Status: "<from C.3, e.g. 'To Do'>"
        Owner(s): "user://<UUID>"
        Project(s): "<project page URL>"
        Team(s): "<team page URL>"
        Org: "[\"<org1 URL>\",\"<org2 URL>\",...]"
```

Leave `content` unset (blank page body) unless the user provided body text.

### C.6 Verify and return URL

Fetch the new work item page and confirm `Status`, `Project(s)`, `Team(s)`, and the full `Org` array are present. Return the URL to the user.

### C.7 Common follow-up management operations

All updates use `notion-update-page` with `command: "update_properties"`.

- **Status change** — `properties: { Status: "<new status>" }`
- **Reassign owner** — `properties: { Owner(s): "user://<new UUID>" }`
- **Link to milestone** — `properties: { Milestone: "<milestone page URL>" }` (single-value relation; plain URL string)
- **Mark Done** — `properties: { Status: "Done", Date Completed: "<today's date>" }`
- **Archive** — `properties: { Status: "Archive", Date Archived: "<today's date>" }`
- **Multi-value field updates** (`Org`, `Initiatives`, `Docs`, `Meeting(s)`, `Collaborators`) — `update_properties` REPLACES, never appends. Always pass the **full** array including any existing values you want to keep.

For any update, fetch the work item first to confirm current values (especially before touching multi-value fields).

### C.8 Out of scope for v1 (flagged for later)

- Subtask flow (`Parent item` / `Sub-item` self-relation)
- Bulk reassignment across many work items in one call
- Cross-project moves (changing `Project(s)` on an existing item)

If a request needs any of these, treat it as a Part C extension — don't invent a parallel skill.

---

## Part D — Create or manage a Meeting in Meetings.db

Meetings live in `Meetings.db` and link laterally across the workspace — to a Team (required), and optionally to Projects, Work Items, Docs, and Initiatives. Schema is fully documented in the workspace's notion-structure reference (the `Meetings.db` section). Refer to that for the full property list; this part covers the happy paths.

**Important shape differences from Parts A/B/C:**
- No `Status`, no `Owner(s)`, no `Priority`, no `Delivery Cycle`. Meetings are events, not workflow items.
- Attendees use `Meeting Attendees` (multi-value person), not `Owner(s)`.
- The project relation is **`Projects`** (plural, no parentheses) — different from Work Items' `Project(s)`. Property names are case- and punctuation-sensitive; don't substitute.
- `Org` is optional on Meetings.db (unlike Parts A and C, where it's required).
- For "find / list meetings about X," prefer `notion-query-meeting-notes` over hand-rolled queries.

### D.1 Resolve the Team URL

See prerequisites. Every meeting belongs to at least one team. Confirm with the user if ambiguous.

### D.2 Resolve any related Project / Work Item / Doc / Initiative URLs (optional)

If the user named a project, search `Projects.db` (as in B.2 / C.1). If they named a work item, search `Work Items.db` similarly. Same disambiguate-or-ask rule. None of these are required — a meeting can be team-only.

### D.3 Gather fields

Required:

| Field | Value source |
|---|---|
| `Name` | From user (the meeting title) |
| `Date` | Meeting date/time. Format: ISO date (`2026-04-23`) or full datetime if known |
| `Team(s)` | From D.1 (full Notion page URL) |

Recommended:

| Field | Value source |
|---|---|
| `Type` | Select. Valid values: `Product Review`, `Team Meeting`, `Project Meeting`. Default `Team Meeting` if generic; `Project Meeting` if linked to a specific project |
| `Meeting Attendees` | Multi-value person. Default to the current user; ask for additional attendees |

Optional:

- `Collaborators` (multi-value person — distinct from Attendees; use for people involved but not present)
- `Projects` (plural relation, multi-value)
- `Initiatives` (relation, multi-value)
- `Work Items` (relation, multi-value)
- `Docs` (relation, multi-value)
- `Org` (relation, multi-value — copy from a related Project if any)

**Leave blank by default:** `Access` (formula, auto-computed), `Visibility` (button), `Place` (only if it's a physical/recurring location).

### D.4 Draft and confirm

Show the user a table of proposed values **before** writing. Wait for confirmation. Always confirm `Date`, `Type`, and the `Meeting Attendees` list explicitly — these are the most common things to get wrong.

### D.5 Create the row (single-shot)

`Meetings.db` accepts multi-value relations on `notion-create-pages` in JSON-array string form (same shape as Docs.db / Work Items.db).

```
notion-create-pages
  parent:
    type: "data_source_id"
    data_source_id: "<Meetings.db collection ID — discover live, do not hardcode>"
  pages:
    - properties:
        Name: "<from D.3>"
        Type: "<from D.3, e.g. 'Project Meeting'>"
        Date: "<ISO date or datetime>"
        Team(s): "<team page URL>"
        Meeting Attendees: "[\"user://<UUID1>\",\"user://<UUID2>\"]"
        Projects: "[\"<project page URL>\"]"          # optional, multi-value
        Work Items: "[\"<work item URL>\"]"            # optional, multi-value
        Docs: "[\"<doc URL>\"]"                         # optional, multi-value
        Org: "[\"<org URL>\"]"                          # optional, multi-value
```

If notes content is provided, pass it as the `content` field (Notion-flavored Markdown). Otherwise leave `content` unset — the meeting can be filled in later from Granola or other notes capture.

### D.6 Verify and return URL

Fetch the new meeting page and confirm `Date`, `Team(s)`, `Meeting Attendees`, and any relations landed correctly. Return the URL to the user.

### D.7 Common follow-up management operations

All updates use `notion-update-page` with `command: "update_properties"`.

- **Add attendees** — fetch first, then `properties: { Meeting Attendees: "[\"<existing user URLs>\",\"<new user URL>\"]" }` (full array; `update_properties` REPLACES).
- **Remove an attendee** — fetch, drop the URL, pass the remaining full array.
- **Reschedule** — `properties: { Date: "<new ISO date>" }`.
- **Link to additional projects / work items / docs / initiatives** — fetch the current relation array, append, pass the full array. Same replace-not-append rule applies.
- **Change `Type`** — `properties: { Type: "<new select value>" }`.
- **Append meeting notes to the body** — use `notion-update-page` with the appropriate body-append command (not `update_properties`).

For any update touching a multi-value field, fetch the meeting first to confirm current values.

### D.8 Out of scope for v1 (flagged for later)

- Recurring meeting series (creating N meetings from one template)
- Auto-generating meeting notes from external sources (Granola transcripts, calendar invites) — that belongs in a separate ingestion skill, not this one
- Bulk attendee changes across many meetings

If a request needs any of these, treat it as a Part D extension — don't invent a parallel skill.

---

## Property-naming gotchas

Known from hitting them manually:

- **Property names are exact and case-sensitive.** Names like `Source URL`, `Owner(s)`, `Project(s)`, `Team(s)`, `Org` — preserve parentheses, spaces, and capitalization exactly as they appear in the DB schema.
- **The `userDefined:` prefix is ONLY for properties literally named `url` or `id`.** A property called `Source URL` is a full, unique name — do NOT prefix it. Passing `userDefined:Source URL` returns `400 Property "userDefined:Source URL" not found`.
- **Relations take full Notion page URLs**, not bare IDs.
- **Multi-value relations must be passed as a JSON-array string** in properties: e.g., `"Org": "[\"url1\",\"url2\"]"`. Comma- or newline-separated values fail with `Invalid page URL`.
- **Multi-value relations on `notion-create-pages` are inconsistent across databases.** Sometimes the JSON-array works on create (observed: Docs.db); sometimes it fails (observed: Projects.db). If create fails, use the two-step create-then-update workaround.
- **`update_properties` REPLACES, never appends.** Passing a single URL to a multi-value field overwrites existing values. Always pass the full array on updates.
- **`Owner(s)` values are user mentions** in the format `user://UUID` (no `mention-user` wrapper when passing to `notion-create-pages`).
- **Status is a `status` property, not a `select`.** Valid values and capitalization can differ between databases in the same workspace (e.g., `In progress` in one DB vs. `In Progress` in another). Check a sibling row if unsure.
- **Work Items.db accepts multi-value `Org` on `notion-create-pages`** (single-shot, JSON-array string) — same as Docs.db, unlike Projects.db. Don't apply the Projects.db create-then-update workaround unnecessarily. If a future workspace ever rejects it, fall back to the two-step pattern from A.4.
- **Meetings.db relation property names differ subtly from other DBs.** The project relation on Meetings.db is **`Projects`** (plural, no parens), not `Project(s)`. Don't blindly copy property names from Work Items / Docs. Always confirm against a sibling row or the schema reference.
- **Meetings have no `Status` or `Owner(s)`.** Don't try to set them — passing those properties on Meetings.db will fail (or silently no-op) because the schema doesn't define them. Use `Meeting Attendees` (multi-value person) for ownership semantics instead.
- **`Meeting Attendees` is multi-value person**, passed as a JSON-array of `user://UUID` strings. Single-attendee values still need the array wrapper.

---

## Worked examples (regression checks)

These examples describe the **shape** of what happened, not the real IDs.

### Example 1 — File an existing page into Docs.db (Part B only)

User says: "Add [this existing Notion page] to the [Existing Project] project. Don't recreate the content. Team is [Team]. Status = Done."

- **B.1:** Fetch the page. Capture title, author, icon. Infer Type from the doc's nature (e.g., Strategy Memo for a lay-of-the-land doc).
- **B.2:** User provided the project URL directly — no search.
- **B.3:** User specified the team.
- **B.4:** Draft the table and confirm.
- **B.5:** Create. If the first attempt returns a `userDefined:Source URL` error, remove the prefix and retry.

### Example 2 — Create a project, then file a doc (Part A + Part B)

User says: "Add a project to Notion for [Project Name], I don't think it exists. Then add [this doc] to it."

**Part A — Create the project:**
- **A.1:** Search Projects.db. Confirm no existing match.
- **A.2:** Copy Orgs from a sibling project on the same team.
- **A.3:** Draft and confirm with user.
- **A.4a:** Create with a **single** Org URL. Attempting to pass multiple Orgs comma- or newline-separated will fail with `Invalid page URL`.
- **A.4b:** Set the full Org array via `update_properties` with a JSON-array string. Passing a single URL on update will **replace** the first Org — always pass the full array.
- **A.5:** Fetch the new project and verify all Orgs are present.

**Part B — File the doc:** proceed as in Example 1, using the new project URL from Part A.

**Key learnings baked into the skill:**
- Always search Projects.db before creating a project (A.1).
- Copy Org values from a sibling project (prerequisites) — never guess or ask for them if a sibling exists.
- For Projects.db, use the create-then-update workaround for multi-value Org (A.4).
- For Docs.db, the JSON-array typically works on create (B.5) — if it doesn't, fall back to the two-step workaround.
- `update_properties` replaces, so always pass the full array.

### Example 3 — Create a work item under an existing project (Part C, create)

User says: "Add a work item called [X] to the [Y] project on the [Z] team."

- **C.1:** Resolve the project URL (search Projects.db if only a name was given).
- **C.2:** Confirm the team URL matches the project's `Team(s)`.
- **C.3:** Copy the project's `Org` array (often 2–3 URLs); default `Status="To Do"`, `Owner(s)`=current user.
- **C.4:** Show the proposed table and wait for confirmation.
- **C.5:** Single-shot `notion-create-pages` with the multi-value `Org` JSON-array string. Do NOT use the Projects.db two-step workaround — Work Items.db accepts multi-value Org on create.
- **C.6:** Fetch the new work item, verify all Orgs landed, return URL.

### Example 4 — Manage an existing work item (Part C, management)

User says: "Mark work item [W] as done." or "Reassign work item [W] to [person]." or "Archive work item [W]."

- Fetch [W] first to confirm current `Status` and any multi-value fields you might touch.
- `notion-update-page` with `command: "update_properties"`:
  - Mark Done → `{ Status: "Done", Date Completed: "<today>" }`
  - Reassign → `{ Owner(s): "user://<new UUID>" }`
  - Archive → `{ Status: "Archive", Date Archived: "<today>" }`
- For any multi-value field update, always pass the full array — `update_properties` replaces, never appends.

**Key learnings baked into Part C:**
- Always copy `Org` from the resolved Project — never guess.
- Work Items.db is single-shot on create for multi-value `Org` (unlike Projects.db).
- Status labels can drift between databases — check a sibling work item if unsure of valid values.
- Subtasks (Parent item / Sub-item) and bulk operations are out of scope for v1.

### Example 5 — Create a meeting linked to a project and work item (Part D, create)

User says: "Log a meeting called [X] on [date] for the [Y] project. Attendees: me and [person]. Link it to work item [W]."

- **D.1:** Resolve the team URL from the project's `Team(s)`.
- **D.2:** Resolve the project URL (search Projects.db if only a name was given) and the work item URL.
- **D.3:** Build attendees array (current user + named person → `user://UUID` mentions). Default `Type="Project Meeting"` because it's project-scoped.
- **D.4:** Show the proposed table — confirm Date, Type, Attendees explicitly.
- **D.5:** Single-shot `notion-create-pages` with `Projects` (plural!), `Work Items`, and `Meeting Attendees` as JSON-array strings.
- **D.6:** Fetch the new meeting, verify Date / Team(s) / Attendees / relations, return URL.

### Example 6 — Manage an existing meeting (Part D, management)

User says: "Add [person] to meeting [M]." or "Reschedule meeting [M] to [new date]." or "Link meeting [M] to project [P]."

- Fetch [M] first to capture current `Meeting Attendees` and any relation arrays you'll touch.
- `notion-update-page` with `command: "update_properties"`:
  - Add attendee → `{ Meeting Attendees: "[\"<existing>\",\"<new>\"]" }` (full array)
  - Reschedule → `{ Date: "<new ISO date>" }`
  - Link to additional project → `{ Projects: "[\"<existing project URLs>\",\"<new project URL>\"]" }`
- For any multi-value update, always pass the full array including existing values — `update_properties` replaces.

**Key learnings baked into Part D:**
- Meetings have no `Status` or `Owner(s)` — use `Meeting Attendees` instead.
- The project relation on Meetings.db is `Projects` (plural, no parens) — different from Work Items' `Project(s)`.
- `Org` is optional on Meetings.db (unlike Projects/Work Items where it's required).
- For "find/list meetings" requests, prefer `notion-query-meeting-notes` over hand-rolled queries.
- Recurring series and Granola-driven note ingestion are out of scope for v1.
