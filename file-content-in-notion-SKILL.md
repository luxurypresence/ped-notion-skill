---
name: file-content-in-notion
description: >
  File content into shared PED Notion databases. Two operations, used alone or together:
  (A) create a new `Projects.db` row for a project, and (B) create a new `Docs.db` row that points
  to an existing page via `Source URL` with the right Project and Team relations. Use this skill
  whenever the user says "create a project in Notion," "add a project to [team] in Notion," "add
  this project to the [team] team," "file this under [project]," "add this doc to [project] in
  Notion," "link this page to [project]," "create a Docs entry for this," or drops a Notion URL
  and asks for it to be associated with a project or team. Always use this skill — the happy path
  has several gotchas (property naming, team URL ambiguity, hub-page-vs-Teams.db-row confusion,
  `userDefined:` prefix rules, multi-value relation array format, `update_properties` replacing
  instead of appending) that the skill handles.
compatibility: "Requires Notion MCP"
---

# File Content into Shared Notion Databases

Two related operations, often done together:

- **A. Create a new `Projects.db` row** for a project that doesn't exist yet.
- **B. Create a new `Docs.db` row** that points to an existing page via `Source URL`, with relations to a Project and Team.

**Do NOT move or duplicate the original page content** for operation B. The Docs.db row is a thin metadata record; the real content stays where it already lives. This is the "project-as-thin-row" pattern applied to docs: the DB row is structure; the page it points to is content.

This skill assumes a Notion workspace with shared databases for Projects, Docs, and Teams that use relation properties to connect them. Collection IDs, team URLs, org URLs, and user IDs are **not baked into this skill** — discover them from the live workspace (see "Resolving IDs" below).

## When to do which

- User drops a Notion URL and names a project that already exists → skip to Part B (file the doc).
- User asks to "create a project" or names a project that doesn't exist → Part A first (create the project), then Part B if they also dropped a doc.
- Before creating a project, **always confirm it doesn't already exist** by searching `Projects.db`.

## Resolving IDs (do this every time)

Do not hardcode collection IDs, team URLs, org URLs, or user IDs. Discover them live:

1. **Find the Projects.db / Docs.db / Teams.db collection IDs.** Fetch any known project, doc, or team page with `notion-fetch`. The response's `parent-data-source url="collection://..."` tag gives the collection ID. Alternatively, your workspace may have a reference doc listing these — check the tool's own documentation if available.
2. **Find the Team URL.** Search `Teams.db` by team name with `notion-search` and the team DB's `data_source_url`. The team's row URL in Teams.db is what goes into the `Team(s)` relation — **not** a hub/shell page with the same name.
3. **Find the Org URL(s).** The easiest way: fetch a sibling project on the same team and copy its `Org` values. Most workspaces use multiple orgs per project (often 2). Never guess Org URLs; derive them from a sibling.
4. **Find the Owner user ID.** Default to the user running the skill. If a different owner is needed, ask the user for a mention or search users.

If any of these lookups fail, ask the user for the missing ID before proceeding.

## Common prerequisites

For either operation, you need:

- **Team URL** (the team's row in Teams.db)
- **Org URL(s)** (usually multi-value; copied from a sibling project)
- **Owner(s)** (user mention in `user://UUID` format)

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
