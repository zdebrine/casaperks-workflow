# Spec: Jira Sync

## Feature Summary

After a spec is written and saved via `/spec`, Claude will automatically
sync the spec content to Jira using the Atlassian MCP server. The feature
handles three Jira states (no epic/ticket, epic only, both exist) and
includes a dev-approved preview gate before any Jira action is taken.
Developers and stakeholders can skip the manual step of copying spec
content into Jira, keeping the board accurate without extra effort.

---

## Acceptance Criteria

1. **No epic or ticket exists:**
   Given the spec is saved and the dev confirms no epic or ticket exists,
   when the dev approves the Jira preview, then Claude creates an epic and
   a linked child ticket, sets the description of both to the
   Claude-generated summary, and attaches the spec markdown as a comment
   on the ticket.
   _Verification: read back both the epic and ticket by key and confirm
   description and comment content match what was sent._

2. **Epic exists, no ticket:**
   Given the spec is saved and the dev confirms an epic exists but no
   ticket, when the dev approves the Jira preview, then Claude creates a
   ticket linked to the existing epic, sets the ticket description to the
   Claude-generated summary, and attaches the spec markdown as a comment.
   _Verification: read back the ticket by key and confirm content matches._

3. **Both epic and ticket exist:**
   Given the spec is saved and the dev provides an existing epic and ticket
   key, when the dev approves the Jira preview, then Claude updates the
   ticket description with the Claude-generated summary and adds the spec
   markdown as a new comment.
   _Verification: read back the ticket by key and confirm description and
   new comment match what was sent._

4. **Dev opts out:**
   Given the spec is saved and the dev chooses to skip Jira, then no Jira
   actions are taken and the workflow continues to `/build`.
   _No failure case — nothing is attempted._

---

## Non-Goals

1. No Jira fields populated beyond ticket/epic description and spec
   markdown comment. Priority, due date, assignee, labels, and all other
   fields are left to the dev manager in Jira.
2. No bidirectional sync. Changes made to the Jira ticket after the push
   are not reflected back into the spec file.

---

## Test Spec

### Criterion 1 — No epic or ticket
- **Realistic inputs:** Project key `CASA`, no epic or ticket key provided.
- **Happy path:** Epic `CASA-1` and ticket `CASA-2` are created; read back
  by returned keys; `CASA-1` description matches Claude-generated summary;
  `CASA-2` description matches summary; `CASA-2` has exactly one comment
  containing the full spec markdown. Claude outputs:
  "Jira sync complete. Epic: CASA-1 / Ticket: CASA-2" then proceeds to
  `/build`.
- **Partial read-back failure:** Epic created successfully but comment is
  missing from ticket on read-back. Claude reports the comment was not
  confirmed, outputs the spec markdown for manual paste, then proceeds to
  `/build`.
- **Full failure path:** Claude reports that the epic or ticket was not
  created, displays the generated summary for manual entry into the Jira
  description, reminds dev to paste `specs/<feature>.md` into the ticket
  comment, then proceeds to `/build`.
- **False positive check:** A 200 response from the MCP does not confirm
  correct content. Read-back verification catches malformed content, wrong
  project, wrong parent linkage, or missing comment.

### Criterion 2 — Epic exists, no ticket
- **Realistic inputs:** Project key `CASA`, epic key `CASA-1` provided,
  no ticket key.
- **Happy path:** Ticket `CASA-3` created linked to `CASA-1`; read back
  confirms description matches summary and comment contains full spec
  markdown. Claude outputs: "Jira sync complete. Ticket: CASA-3" then
  proceeds to `/build`.
- **Partial read-back failure:** Same as criterion 1 partial failure.
- **Full failure path:** Same as criterion 1 full failure.
- **False positive check:** Same read-back verification pattern.

### Criterion 3 — Both exist
- **Realistic inputs:** Project key `CASA`, epic key `CASA-1`, ticket key
  `CASA-2` provided.
- **Happy path:** `CASA-2` description updated with summary; new comment
  added containing full spec markdown; read back confirms both. Claude
  outputs: "Jira sync complete. Ticket: CASA-2" then proceeds to `/build`.
- **Partial read-back failure:** Description updated but new comment not
  found on read-back. Claude reports the comment was not confirmed, outputs
  spec markdown for manual paste, then proceeds to `/build`.
- **Full failure path:** Same as criterion 1 full failure.
- **False positive check:** Same read-back verification pattern.

### Criterion 4 — Opt out
- **Happy path:** No Jira action taken. Claude outputs: "Skipping Jira
  sync. Proceeding to `/build`." No MCP calls made.
- **Failure path:** N/A.

### Edge cases
- **Empty spec file:** If `specs/<feature>.md` is empty or missing when
  jira-sync runs, Claude must abort the sync and say: "The spec file is
  empty or could not be read. Skipping Jira sync and proceeding to
  `/build`."
- **Special characters in feature name:** Feature names containing
  characters like `/`, `&`, `<`, or `>` must not break the Jira summary
  field. Claude must pass the name as-is and let the MCP handle encoding.
- **Invalid project key format:** If the developer provides a project key
  that does not match Jira's key format (e.g. all-lowercase `casa` or a
  key with spaces), Claude must surface the error from the MCP and ask
  the developer to re-enter a valid key before retrying once. If the
  second attempt fails, fall through to the failure path.

---

## Architecture Sketch

**Files changed:**
- `skills/spec-writing/SKILL.md` — add a handoff step at the end: after
  the spec is saved, invoke the Jira sync skill.

**Files created:**
- `skills/jira-sync/SKILL.md` — new skill that owns the full Jira flow.

**Data flow:**
```
/spec command
  → spec-writing skill (existing flow unchanged)
  → spec saved as specs/<feature>.md
  → jira-sync skill invoked
      → ask dev: Jira project key?
      → ask dev: epic key? ticket key? (determines which of 4 states)
      → generate Claude summary from full spec markdown
      → show preview (summary + confirmation of what will be created/updated)
      → dev approves or skips
      → if approved: call Atlassian MCP tools
          → create_issue / update_issue / add_comment as appropriate
          → read back to verify
          → on failure: surface content for manual entry
      → proceed to /build
```

**Credentials:**
Stored in the developer's local Claude Code `settings.json` under
`mcpServers`. Never stored in the plugin repo. Each developer configures
their own Atlassian MCP server instance.

---

## Open Questions & Assumptions

1. **Assumed:** The Atlassian MCP server exposes tools sufficient to create
   epics, create linked child tickets, update issue descriptions, add
   comments, and read back issues by key.
2. **Decided:** Claude will ask the dev which Jira project to push to at
   the start of the Jira sync step — no assumption of a single project.
3. **Assumed:** Spec markdown fits within Jira field and comment size limits.
4. **Assumed:** The dev has read/write permissions for the Jira project
   they specify.
