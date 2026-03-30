# AI Dev Workflow — Casa Plugin

A structured, human-in-the-loop development workflow for teams using Claude Code.
Built to make developers better programmers — not just faster ones.

## Installation

> **Note:** Claude Code does not yet support installing local plugins via the CLI.
> Until it does, registration requires a one-time manual step.

**Step 1: Clone the plugin**

```bash
git clone https://github.com/zdebrine/casaperks-workflow ~/.claude/plugins/casaperks-workflow
```

**Step 2: Register it manually**

Open `~/.claude/plugins/installed_plugins.json` and add the following entry inside the `"plugins"` object (create the file with `{"version": 2, "plugins": {}}` if it doesn't exist):

```json
"casaperks-workflow@local": [
  {
    "scope": "user",
    "installPath": "/YOUR_HOME_DIR/.claude/plugins/casaperks-workflow",
    "version": "1.0.0",
    "installedAt": "2026-01-01T00:00:00.000Z",
    "lastUpdated": "2026-01-01T00:00:00.000Z"
  }
]
```

Replace `/YOUR_HOME_DIR` with your actual home directory (run `echo $HOME` if unsure).

Or use this one-liner to do it automatically:

```bash
node -e "
const fs = require('fs');
const file = \`\${process.env.HOME}/.claude/plugins/installed_plugins.json\`;
const data = JSON.parse(fs.readFileSync(file, 'utf8'));
data.plugins['casaperks-workflow@local'] = [{
  scope: 'user',
  installPath: \`\${process.env.HOME}/.claude/plugins/casaperks-workflow\`,
  version: '1.0.0',
  installedAt: new Date().toISOString(),
  lastUpdated: new Date().toISOString()
}];
fs.writeFileSync(file, JSON.stringify(data, null, 2));
console.log('Plugin registered.');
"
```

**Step 3: Symlink the commands globally**

Claude Code only loads plugin commands for marketplace-installed plugins. Until local plugin installs are supported natively, symlink the plugin's `commands/` directory so the slash commands are available everywhere:

```bash
claude config add plugins ~/.claude/plugins/casaperks-workflow
```

This keeps the plugin as the single source of truth — edits to the plugin's commands are picked up immediately, no copying required.

**Step 4: Restart Claude Code**

The commands are now available in every project.

To verify, run `/spec` in any project.

---

## The problem this solves

AI coding tools are fast. But speed creates a trap: you can ship code you don't
understand, can't safely change, and can't debug when it breaks. This workflow
makes comprehension mandatory at every stage of development.

## Three goals, in priority order

1. **Dev comprehension** — the developer understands the architecture of what
   was just built, well enough to diagnose problems and explain tradeoffs
2. **Dev education** — the developer is becoming a better programmer by using
   this workflow, not just a faster one
3. **Code robustness** — the code is well-tested, and the tests actually catch bugs

## How it works

```
/casa:spec → /casa:build → /casa:explain → /casa:approve → /casa:build → ... → /casa:review-tests → /casa:retro
```

Every feature starts with a spec the developer writes (not Claude). Every build
stage ends with a gate the developer must pass by demonstrating comprehension.
Every feature ends with a retro that captures what was learned.

## Commands

### `/casa:spec [feature-name]`

Guides you through writing a feature spec before any code is written.
Claude asks questions — you write the answers. Produces `specs/<name>.md`.

### `/casa:build [stage-number]`

Implements one stage of the feature. Stops after the stage and will not
continue until you pass the gate.

### `/casa:approve`

Passes the current stage gate. Claude will ask you three comprehension
questions first. You must answer them. This is not optional.

### `/casa:explain [file-or-function]`

Deep explanation of code that was just written. Covers: approach and
tradeoffs, failure modes, change surface, what to test, what to refactor.
Run this whenever something feels unclear — and especially when it doesn't.

### `/casa:review-tests`

Full test quality audit in four phases: coverage check, false positive
detection (break tests), rubric evaluation, and graded summary with
action items.

### `/casa:retro [feature-name]`

Post-feature retrospective. Five questions, one at a time. Produces
`retros/<name>.md`. Over time this becomes your team's learning record.

## Plugin structure

```
casaperks-workflow/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── commands/
│   ├── spec.md                  # /casa:spec command
│   ├── build.md                 # /casa:build command
│   ├── approve.md               # /casa:approve command
│   ├── explain.md               # /casa:explain command
│   ├── review-tests.md          # /casa:review-tests command
│   └── retro.md                 # /casa:retro command
└── skills/
    ├── approve-gate/SKILL.md    # Gate review protocol
    ├── spec-writing/SKILL.md    # Spec guidance
    └── test-quality/SKILL.md    # Test evaluation rubric
```

## Adapting to your stack

The commands and skills are stack-agnostic. The `/casa:build` command's default
four stages (scaffold → backend → frontend → polish) work for any architecture.
The test-quality skill works for any test framework.

## Philosophy

The three comprehension questions at every gate are uncomfortable on purpose.
Being asked to explain code you didn't fully understand is the moment learning
happens. The goal is not to make development frictionless — it's to make
the friction productive.

> "AI can generate a lot of code quickly, but speed is only valuable if
> developers still understand what was created, can safely change it later,
> and can trust it won't regress."
