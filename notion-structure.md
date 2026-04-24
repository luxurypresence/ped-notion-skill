# PED Notion Workspace — Reference

Reference for how documentation, projects, and work are structured across PED's Notion workspace. Use this when you need to read, navigate, or create content in Notion.

**Notion is the source of truth.** This doc explains the *generic structure* (shells, shared DBs, relations). For specific team or project URLs, query Notion — do not hardcode them here.

Analyzed: 2026-04-15 · Last refreshed: 2026-04-23

> **Reading vs. writing.** This doc is the *read-side* reference — schema, relations, navigation. For the *write-side* (creating or managing rows in `Projects.db`, `Docs.db`, `Work Items.db`, `Meetings.db`), follow `ped-manage-notion-pages-SKILL.md` in this same folder. It encodes the create-and-manage happy paths plus the property-naming gotchas.

> **Collection IDs are a snapshot.** The IDs listed in the shared-DB table below were correct as of the refresh date. They rarely change, but if a `notion-fetch` response returns a different `parent-data-source url="collection://..."`, **trust the live response** and update this doc. Never hardcode IDs in skills or queries — discover them live.

---

## The mental model (read this first)

PED runs on **a small set of shared databases** that hold all content across all teams and projects:

- `Docs.db`, `Work Items.db`, `Projects.db`, `Meetings.db`, `Initiatives.db`, `Milestones.db`, `Teams/Orgs.db`

**Team pages and project pages are thin.** They are **rows** in those shared DBs. Their page body is typically blank — the metadata (properties and relations) is the content.

**The actual content** — memos, specs, PRDs, strategy docs, meeting notes — lives in **`Docs.db` pages**. Docs are linked to projects, teams, meetings, and work items through **relation properties**.

So:

- To find the docs for a project → fetch the project page, read its `Docs` relation, fetch each linked doc.
- To find a team's content → filter any DB by the `Team(s)` relation.
- To find all work under an initiative → filter `Work Items.db` by `Initiatives`.

A project page with a blank body + a populated `Docs` relation is the healthy pattern, **not** a missing-content problem.

---

## How to navigate a project's documentation

To read everything about a project:

1. **Fetch the project page** (`notion-fetch` with URL or ID). The body will often be blank.
2. **Read its relation properties:**
   - `Docs` → linked Docs.db pages (the real written content)
   - `Work Items` → tasks / roadmap items under the project
   - `Meetings` → meetings tied to the project
   - `Initiatives` → parent strategic bets
   - `Milestones` → deliverables / dates
3. **For each linked Doc, fetch the page body.** That's where memos, specs, and decisions actually live.
4. **Check `Source URL` on Docs.** Often points to a predecessor or external canonical version — treat it as a version-history breadcrumb, not a duplicate link.
5. **Check `AI Summary` on Docs** for a quick read when the body is long.

If a linked Doc is itself blank, it's usually a placeholder for upcoming content — note it, don't assume it's broken.

---

## Hierarchy (shape, not specific URLs)

```
PED Home
└── [Group Hub]                e.g. "CRM Group Hub"
    └── [Team Hub]             e.g. "CRM"
        ├── [Sidebar: full-page filtered views of the shared DBs]
        │   ├── <Team> Docs       (filtered view of Docs.db)
        │   ├── <Team> Roadmap    (filtered view of Work Items.db)
        │   ├── <Team> Projects   (filtered view of Projects.db)
        │   └── <Team> Meetings   (filtered view of Meetings.db)
        └── [Inline on main page]
            └── <Team> Projects (inline gallery, linked view of Projects.db filtered to this team)
```

Every Group Hub and Team Hub follows this same shape. They are **shells**: their page bodies carry layout and filtered views, but the real content is in the shared databases. To get specific URLs (team IDs, hub pages, sidebar pages), fetch them from Notion — don't hardcode them here.

---

## Team hub page layout (template)

The team hub page has **3 columns**:

### Column 1 — Navigation Callout
- Link back to PED Home
- Links to the 4 dedicated full-page databases (Docs, Roadmap, Projects, Meetings)

### Column 2 — Quick Actions (Buttons)
4 action buttons (Notion automations):
- **Create Doc** — creates a doc pre-tagged to the team
- **Create Project** — creates a project pre-tagged to the team
- **Create Work Item** — creates a work item pre-tagged to the team
- **Start New Meeting** — creates a meeting pre-tagged to the team

### Column 3 — Welcome Callout (gray bg)
- Team description placeholder text

### Below Columns — Inline Database
- Inline gallery of projects (linked view of `Projects.db`, filtered to this team)

---

## Shared databases (global, used across all teams)

All databases are shared globally and filtered per-team via the `Team(s)` relation property.

| Database | Collection URL | Purpose |
|----------|---------------|---------|
| Docs.db | `collection://96069c52-23e8-8297-95e8-87ebe38332ac` | All documents |
| Work Items.db | `collection://8c169c52-23e8-8389-a8b5-07b69a35b3e9` | Tasks / roadmap items |
| Projects.db | `collection://72969c52-23e8-82df-8ad3-070f4f756409` | Projects |
| Meetings.db | `collection://51d69c52-23e8-8302-8ba1-879b98da9eeb` | Meetings |
| Orgs/Teams.db | `collection://2e469c52-23e8-82b4-bab9-07b09d10ed9d` | Teams + Orgs (used for filtering) |
| Initiatives.db | `collection://a3d69c52-23e8-8232-8e4e-072ddcdc857a` | Initiatives |
| Milestones.db | `collection://dcd17c47-129e-46b3-8035-3a8f92c0653d` | Milestones |

**To resolve a team's URL:** query `Teams.db` (collection `2e469c52-23e8-82b4-bab9-07b09d10ed9d`) by team name, or ask the user. The page URL of the team's row in Teams.db is what gets used in every `Team(s)` relation filter across the other DBs.

---

## Database schemas

### 1. Docs.db (`collection://96069c52-23e8-8297-95e8-87ebe38332ac`)

**When you'd query this:** when you need the actual written content behind a project, decision, or memo. This is the primary content DB.

| Property | Type | Notes |
|----------|------|-------|
| Name | title | |
| Type | select | Document/Freeform, Decision Memo, PRD, Pre/Post Mortems, PRFAQ, Product Review, Project Proposal, Roadmap Overview, Strategy Memo, Weekly Update, Brief, SOP, Playbook |
| Status | status | Not started, In progress, Done |
| Verification | verification | Verified / Expired |
| Owner(s) | person | |
| Collaborators | person | |
| Last reviewed by | person | max 1 |
| Review date | date | |
| Date Completed | date | |
| Date Archived | date | |
| Team(s) | relation → Teams.db | **key filter field** |
| Org | relation → Teams.db | |
| Project(s) | relation → Projects.db | |
| Initiative(s) | relation → Initiatives.db | |
| Work Items | relation → Work Items.db | |
| Meetings | relation → Meetings.db | |
| AI Summary | text | AI-generated |
| Source URL | url | Often points to a predecessor / external canonical doc |
| File | file | |
| Access | formula | |
| Visibility | button | |

**Typical views on a team's Docs page:**
- My Docs (table) — filtered to current user as Owner or Collaborator + team
- All Docs (table) — all team docs
- PRDs — filtered by Type=PRD
- PRFAQs — filtered by Type=PRFAQ
- Roadmap Overviews — filtered by Type=Roadmap Overview
- All Verified — verification filter
- Expired Verification — verification filter

---

### 2. Work Items.db (`collection://8c169c52-23e8-8389-a8b5-07b69a35b3e9`)

**When you'd query this:** when you need tasks, roadmap items, or execution status for a project, initiative, or team.

| Property | Type | Notes |
|----------|------|-------|
| Name | title | |
| Status | status | Backlog, To Do, In progress, Blocked, In review, Done, Archive |
| Priority | select | 🔴 Urgent, 🟠 High, 🟡 Medium, 🟢 Low |
| Delivery Cycle | select | DC2.2026 through DC8.2026 |
| Date | date | due date (format: MMM d) |
| Date Completed | date | |
| Date Archived | date | |
| Description | text | short description |
| Est. Hrs. | number | estimated hours |
| Actual (hrs) | number | actual hours |
| Owner(s) | person | max 1 |
| Collaborators | person | |
| Team(s) | relation → Teams.db | **key filter field** |
| Org | relation → Teams.db | |
| Project(s) | relation → Projects.db | |
| Initiatives | relation → Initiatives.db | |
| Milestone | relation → Milestones.db | max 1 |
| Docs | relation → Docs.db | |
| Meeting(s) | relation → Meetings.db | |
| Parent item | relation → self (Work Items.db) | max 1, for subtasks |
| Sub-item | relation → self (Work Items.db) | children |
| Access | formula | |
| Visibility | button | |
| Place | place | |

**Views on a team's Roadmap page:**
- Roadmap by Project (board, grouped by Project, no subitems)
- Roadmap View (Timeline) (timeline by Date, grouped by Project)
- Roadmap by DC (board, grouped by Delivery Cycle, no subitems)
- Completed Roadmap Items (board, filtered Status=Complete)

**Views on the team hub page (inline):**
- Roadmap View (board, grouped by Project)
- Roadmap View (Timeline) (timeline)

---

### 3. Projects.db (`collection://72969c52-23e8-82df-8ad3-070f4f756409`)

**When you'd query this:** when you need the metadata / index for a project. The project's actual content lives in its related Docs, not in this page's body.

| Property | Type | Notes |
|----------|------|-------|
| Name | title | project name |
| Status | status | Not started, In progress, At risk, Blocked, On hold, Complete, Archived |
| Priority | select | 🔴 High, 🟡 Medium, 🟢 Low |
| Sequence | select | For Review, Now, Next, Later, Parked, Ongoing, Done |
| Date | date | timeline/due dates |
| Date Completed | date | |
| Date Archived | date | |
| Summary | text | project summary |
| Estimate | select | (empty options — custom per workspace) |
| Progress % | number | shown as bar |
| Progress (WIP) | rollup | % of Work Items with Complete status |
| Owner(s) | person | max 1 |
| Collaborators | person | |
| Team(s) | relation → Teams.db | **key filter field** |
| Org | relation → Teams.db | |
| Initiatives | relation → Initiatives.db | |
| Work Items | relation → Work Items.db | |
| Docs | relation → Docs.db | **the real content lives here** |
| Meetings | relation → Meetings.db | |
| Milestones | relation → Milestones.db | |
| Access | formula | |
| Visibility | button | |

**Views on a team's Projects page:**
- Active (gallery, card, filtered Status=In progress OR Not started)
- Completed (gallery, filtered Status=Complete)

**Views on the team hub inline:**
- All Projects (gallery, all team projects)

---

### 4. Meetings.db (`collection://51d69c52-23e8-8302-8ba1-879b98da9eeb`)

**When you'd query this:** when you need meeting notes, attendees, or decisions tied to a project or team.

| Property | Type | Notes |
|----------|------|-------|
| Name | title | |
| Type | select | Product Review, Team Meeting, Project Meeting |
| Date | date | meeting date/time (format: MMM d) |
| Meeting Attendees | person | |
| Collaborators | person | |
| Team(s) | relation → Teams.db | **key filter field** |
| Org | relation → Teams.db | |
| Projects | relation → Projects.db | |
| Initiatives | relation → Initiatives.db | |
| Work Items | relation → Work Items.db | |
| Docs | relation → Docs.db | |
| Access | formula | |
| Visibility | button | |

**Views on a team's Meetings page:**
- Team Meetings (table, grouped by relative date, filtered to the team)
- My Meetings (table, filtered to current user as Attendee or Collaborator)

---

### 5. Initiatives.db & Milestones.db — schema gap

These two databases appear in the shared-DB table and as relation targets across Docs / Work Items / Projects / Meetings, but their **internal schemas are not yet documented here**. If you need to read or query them, fetch a known row first to capture the property list, then add a section above. If you need to *write* to them, follow the same pattern as the documented DBs but **verify property names live** — none of the gotchas in `ped-manage-notion-pages-SKILL.md` have been validated against these two yet.

Collection IDs:
- Initiatives.db: `collection://a3d69c52-23e8-8232-8e4e-072ddcdc857a`
- Milestones.db: `collection://dcd17c47-129e-46b3-8035-3a8f92c0653d`

---

## Key design patterns

### Team filtering
Every database uses a `Team(s)` relation to `Teams.db` (collection `2e469c52-23e8-82b4-bab9-07b09d10ed9d`). All views filter by:
```
Team(s) relation_contains <team-page-url>
```
where `<team-page-url>` is the team's row in Teams.db. Resolve this by fetching from Notion — don't hardcode.

### Cross-database relations
The shared databases are extensively interconnected:
- Docs ↔ Work Items (bilateral)
- Docs ↔ Projects (bilateral)
- Docs ↔ Meetings (bilateral)
- Work Items ↔ Projects (bilateral)
- Work Items ↔ Meetings (bilateral)
- Projects ↔ Meetings (bilateral)
- Work Items ↔ Milestones
- Projects ↔ Milestones
- Docs / Work Items / Projects / Meetings → Initiatives (relation; directionality from Initiatives.db not yet verified — see schema gap below)

**Property-naming subtlety.** The same logical relation can have *different* property names across databases. Most notably: the project relation is `Project(s)` (parens) on Docs.db and Work Items.db, but `Projects` (plural, no parens) on Meetings.db. Similarly `Meeting(s)` on Work Items.db vs `Meetings` on Projects.db / Docs.db. Property names are case- and punctuation-sensitive when writing — always check the relevant schema table above (or fetch a sibling row) before passing properties to `notion-create-pages` / `notion-update-page`.

### Project-as-thin-row
Project and team pages are **metadata rows**, not content pages. Their body is usually blank. The actual writing lives in related Docs. If you open a project page and see nothing, open its `Docs` relation — that's where the content is.

### `Source URL` is a version breadcrumb
When a Doc has a `Source URL` pointing to another Notion page, treat it as a pointer to a predecessor or external canonical version — not a duplicate link. A new Doc with a populated `Source URL` often means "this is the new canonical version that supersedes the URL shown."

### Quick Actions pattern
Each team page has 4 Notion button automations that create records in the shared databases, pre-setting:
- `Team(s)` = the team's page URL
- `Org` = the group/org page URL

### Delivery Cycles
Work items are assigned to named delivery cycles: DC2.2026 through DC8.2026. Smart logic is intended to auto-assign based on start/end dates.

### Verification
Docs have a `Verification` property (Notion's native verification type) — shows "Verified" or "Expired Verification" based on review date. Key views surface expired docs for maintenance.

---

## Common tasks

### Read a project's full documentation
1. Fetch the project page.
2. Read the `Docs` relation → fetch each linked Doc body.
3. Read `Work Items`, `Meetings`, `Initiatives`, `Milestones` relations as needed.
4. If a Doc has a `Source URL`, that's a version/predecessor pointer — fetch it only if you need the older version.

### List all docs for a team
Query `Docs.db` filtered by `Team(s) contains <team-page-url>`.

### Find every meeting tied to a project
Fetch the project page, read its `Meetings` relation.

### Find all work items in a delivery cycle for a team
Query `Work Items.db` filtered by `Team(s) contains <team-page-url>` AND `Delivery Cycle = DC<n>.<year>`.

### Create a new project under a team
Add a row to `Projects.db` with `Team(s)` set to the team's page URL. The team's Quick Action button ("Create Project") does this automatically and pre-fills `Team(s)` and `Org`.

### Create a new team page
1. Create a `Teams.db` entry for the new team → get its page URL.
2. Create the team hub page with the 3-column layout above.
3. Add the 4 Quick Action buttons (automations targeting the shared DBs with `Team(s)` pre-filled).
4. Add sidebar database links (filtered views of Docs, Roadmap, Projects, Meetings).
5. Add an inline Projects gallery on the hub page.
6. All shared databases automatically include the new team's content when filtered by the new team URL.
