# PED Notion Skill

A reference + skill for working with Luxury Presence's PED Notion workspace from an AI assistant (Claude Code, Claude Desktop, Cursor, etc.). Two files:

- **[notion-structure.md](notion-structure.md)** — the *read-side* reference. How PED's shared databases are wired together (Projects, Docs, Work Items, Meetings, Initiatives, Milestones, Teams/Orgs), the project-as-thin-row mental model, schemas, relations, and how to navigate any project's documentation.
- **[ped-manage-notion-pages-SKILL.md](ped-manage-notion-pages-SKILL.md)** — the *write-side* skill. Step-by-step instructions for creating and managing rows in `Projects.db`, `Docs.db`, `Work Items.db`, and `Meetings.db`, including the property-naming gotchas that bite anyone going freehand.

## Who this is for

PMs (and engineers / designers / EMs) at Luxury Presence whose AI assistant already has the Notion MCP integration enabled, and who want that AI to:

- Read the right docs when they share a Notion URL
- Create projects, file docs, add work items to the roadmap, log meetings — without having to spell out the property names, relations, and gotchas each time
- Use the team's actual conventions (project-as-thin-row, `Source URL` as a version breadcrumb, multi-value Org from sibling projects, etc.) instead of inventing its own

If your AI doesn't have the Notion MCP yet, set that up first — neither file does anything without it.

## Two patterns for Docs.db rows

PED uses `Docs.db` for two subtly different things. The skill handles both, but the distinction matters when you're the one deciding what to do:

1. **Docs.db row = the source of truth.** The document's content lives on the Docs.db row itself. You open the row and you're reading the doc. Status, owner, team, project relations all hang off that same page. No `Source URL`, or `Source URL` only points to a superseded predecessor (version-history breadcrumb). This is the default for most team docs created fresh in Notion.

2. **Docs.db row = a thin pointer to a source-of-truth page elsewhere.** The real content lives on a separate Notion page (or, in some cases, an external URL). The Docs.db row exists so the doc shows up in the team's DB views, search, and relation graphs — but its job is to redirect readers to the actual source. This is the pattern when you're filing an *already-existing* page into the team's DB, or when you want the content page to be a clean deliverable (shareable, owned separately) while still tracking it through PED's shared DBs.

The skill's Part B ("File a Doc into Docs.db") produces pattern 2 by default: it sets `Source URL`, populates `AI Summary` with a scannable description, and puts a single prominent link in the body pointing at the source. If you're producing pattern 1 instead (the row *is* the canonical doc), skip the body-as-pointer step and let the content itself live on the row.

When asked to "add this document to Notion" without an existing source page, the skill's happy path is: create the content page first, then create a Docs.db row pointing at it (pattern 2). This keeps the authoritative content on a clean, shareable page while the Docs.db row handles discovery and relations.

## Adding this to your local workspace

The intended pattern is to clone this repo into your workspace as a tool, then point your AI at it.

### 1. Clone into your workspace

Pick whatever folder structure your workspace uses for AI tools / skills. Example layouts:

```bash
# Example 1: workspace with a tools/ folder
cd ~/your-workspace
git clone git@github.com:luxurypresence/ped-notion-skill.git tools/ped-notion-work

# Example 2: dropping it next to other Claude skills
cd ~/.claude/skills
git clone git@github.com:luxurypresence/ped-notion-skill.git ped-notion-work
```

The folder name on disk doesn't need to match the repo name — pick what fits your existing structure.

### 2. Tell your AI when to read each file

Add a note to your workspace's top-level instructions (e.g. `CLAUDE.md`, `.cursorrules`, system prompt, or whatever your tool uses) so the AI loads the right file at the right time. Adapt this to your paths:

```markdown
- When the user shares a Notion URL or asks you to read, navigate, or
  understand PED Notion content, read tools/ped-notion-work/notion-structure.md
  before responding.

- When the user asks you to create a project, file a page into Docs.db,
  link a page to a project/team, create or manage a work item, or create
  or manage a meeting, also read tools/ped-notion-work/ped-manage-notion-pages-SKILL.md
  and follow it step-by-step.
```

The skill file's frontmatter `description` includes a long list of natural-language triggers ("create a project in Notion," "add this to the roadmap," "log a meeting for X," etc.) so your AI can recognize when to load it on its own once it knows the file exists.

### 3. Keep it current

This is a regular git repo. Pull occasionally:

```bash
cd path/to/ped-notion-work
git pull
```

PED's Notion structure and the workspace conventions evolve — the docs here lag reality, but they get refreshed periodically. Check the "Last refreshed" date at the top of `notion-structure.md`.

## What's intentionally NOT in this repo

- **Hardcoded team URLs, project URLs, owner UUIDs.** The skill discovers all IDs live via `notion-fetch`. Collection IDs in the structure doc are a snapshot — if a fetch returns a different `parent-data-source`, trust the live response.
- **Per-team workflow** (Engage's roadmap conventions vs. CRM's, etc.). Both files are intentionally generic. Team-specific rules belong in your own workspace, not here.
- **Anything that requires write access beyond Notion MCP.** No Slack posting, no calendar invites, no GitHub issue creation. If you want those orchestrations, build them on top of this skill, not inside it.

## Contributing

This repo is maintained by the PED PM team at Luxury Presence. If you hit a property-naming gotcha, schema drift, or missing happy path, open a PR — the gotchas section of the skill is built from real failures, and the more we capture there the less time everyone wastes re-discovering them.

If you're filling the schema gap for `Initiatives.db` or `Milestones.db` (currently undocumented — see §5 of the structure doc), that would be especially welcome.
