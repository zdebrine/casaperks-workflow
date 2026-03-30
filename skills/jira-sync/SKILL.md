---
name: jira-sync
description: Governs syncing spec content to Jira after a spec is saved. Handles four states — no epic/ticket, epic only, both exist, or dev opts out. Invoked by the spec-writing skill after the spec file is saved.
---

# Skill: Jira Sync

## Purpose

This skill runs after a spec is saved. It syncs the spec content to Jira
by creating or updating epics and tickets using the Atlassian MCP server.
It handles all four possible Jira states and requires explicit dev approval
before any Jira action is taken.

---

## Prerequisites

This skill assumes the developer has configured the Atlassian MCP server
locally in their Claude Code `settings.json`. If Jira MCP tools are not
available, Claude must say:

> "The Atlassian MCP server is not configured. To enable Jira sync, add
> the MCP server to your Claude Code settings.json under `mcpServers`.
> Skipping Jira sync and continuing to `/build`."

Then proceed to `/build` without any Jira action.

---

## Step 1 — Ask whether to sync

Ask the developer:

> "Spec saved. Do you want to sync this spec to Jira? (yes / skip)"

If the developer says **skip**: say "Skipping Jira sync. Proceeding to
`/build`." and proceed directly to `/build`. No further steps.

If the developer says **yes**: continue to Step 2.

---

## Step 2 — Gather Jira context

Ask in a single message:

> "Two quick questions:
> 1. What is the Jira project key? (e.g. CASA, ENG, PLAT)
> 2. Do you have an existing epic key, a ticket key, both, or neither?
>    - Neither → I'll create both
>    - Epic only → provide the epic key, I'll create the ticket
>    - Both → provide the epic key and ticket key, I'll update the ticket"

Wait for the developer's response. Parse it to determine which of the
four states applies.

---

## Step 3 — Generate summary

Before taking any Jira action, generate a concise summary from the saved
spec. The summary must include:

- Feature name (one line)
- What it does and who uses it (2–3 sentences)
- Acceptance criteria as a numbered list (one line each)
- Non-goals as a bulleted list (one line each)

This summary will be used as the Jira description. The full spec markdown
will be added as a comment.

---

## Step 4 — Show preview and get approval

Present the preview before any MCP call:

> "Here is what I will push to Jira:
>
> **Action:** [create epic + ticket / create ticket under [EPIC-KEY] /
> update [TICKET-KEY]]
> **Project:** [PROJECT-KEY]
>
> **Description (goes on epic and/or ticket):**
> [generated summary]
>
> **Comment (goes on ticket):**
> The full contents of `specs/<feature>.md` will be added as a comment.
>
> Approve this push? (yes / skip)"

If the developer says **skip**: say "Skipping Jira sync. Proceeding to
`/build`." and proceed to `/build` with no Jira action.

If the developer says **yes**: continue to Step 5.

---

## Step 5 — Execute Jira action

Use the Atlassian MCP server tools based on the state determined in Step 2.

### State A — Neither epic nor ticket exists

1. Create the epic:
   - Project: developer-provided key
   - Summary: feature name from the spec
   - Issue type: Epic
   - Description: generated summary
2. Create the ticket:
   - Project: same project key
   - Summary: feature name from the spec
   - Issue type: Story (or Task if Story is unavailable)
   - Description: generated summary
   - Parent/Epic link: epic key returned from step 1
3. Add a comment to the ticket with the full spec markdown.

### State B — Epic exists, no ticket

1. Create the ticket:
   - Project: developer-provided key
   - Summary: feature name from the spec
   - Issue type: Story (or Task if Story is unavailable)
   - Description: generated summary
   - Parent/Epic link: developer-provided epic key
2. Add a comment to the ticket with the full spec markdown.

### State C — Both exist

1. Update the ticket description with the generated summary.
2. Add a comment to the ticket with the full spec markdown.

---

## Step 6 — Verify

After each MCP create or update call, read back the affected issue by key
and confirm:

- The description matches the generated summary
- The comment exists and contains the spec markdown

Treat verification as two independent checks. If one passes and the other
fails, do not treat the whole sync as a success:

- **Description correct, comment missing:** Say "The ticket description
  was synced successfully, but the comment could not be confirmed. Paste
  the contents of `specs/<feature>.md` into the ticket comment manually
  to stay consistent with other tickets." Then proceed to Step 7 for the
  description success.
- **Description missing or wrong:** Proceed to the full failure path.
- **Both correct:** Proceed to Step 7.

If verification fails entirely: proceed to the failure path below.

---

## Failure path

If any MCP call fails or verification does not pass, Claude must:

1. State clearly what failed:
   > "The Jira sync failed — [epic / ticket / comment] was not created
   > or updated successfully."

2. Output the generated summary for manual entry:
   > "Here is the summary to paste into the Jira description manually:"
   > [generated summary]

3. Remind the developer to use the spec file for the comment:
   > "For the ticket comment, paste the contents of `specs/<feature>.md`
   > to stay consistent with other tickets."

4. Proceed to `/build` without waiting for any further input.

---

## Step 7 — Confirm and hand off

On success, confirm with the created or updated keys using this exact
format — include only the keys that were created or updated:

> "Jira sync complete. [Epic: EPIC-KEY / ] Ticket: TICKET-KEY"

Then immediately hand off to the `/build` workflow without waiting for
any further developer input.

---

## Edge case handling

**Empty or missing spec file:** Before Step 3, confirm the spec file at
`specs/<feature>.md` exists and is non-empty. If it is missing or empty,
say: "The spec file is empty or could not be read. Skipping Jira sync and
proceeding to `/build`." Do not attempt any Jira action.

**Special characters in feature name:** Pass the feature name to Jira
as-is. Do not sanitize or escape it — the MCP server handles encoding.
If the MCP rejects the name, surface the error and fall through to the
failure path.

**Invalid project key format:** If the MCP returns an error indicating
the project key is invalid (e.g. wrong format, project not found), say:
"The project key [KEY] was not recognized. Please provide a valid Jira
project key (e.g. CASA, ENG, PLAT)." Allow the developer to re-enter
once. If the second attempt also fails, fall through to the failure path.

---

## What this skill never does

- Never populates Jira fields beyond description and comment (no assignee,
  priority, due date, labels, or sprint)
- Never reads from or writes to Jira without explicit dev approval in Step 4
- Never retries a failed MCP call — surface the failure and move on
- Never blocks the developer from reaching `/build` — Jira sync is always
  optional and failure is always recoverable
