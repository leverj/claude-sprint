# claude-sprint

A scrum-like development workflow that runs across multiple AI coding tools. Requirements live as structured GitHub Issues, decisions are recorded as ADRs in the repo, and any developer can pick up where someone left off.

## Prerequisites

- A supported AI coding tool (see below)
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated (`gh auth login`)
- A git repo with a GitHub remote

## Supported tools

The canonical workflow lives in `SKILL.md`. `AGENTS.md` is a git symlink to it (mode `120000`), and `GEMINI.md` imports it via `@./SKILL.md` — so every tool reads the same source with no manual sync.

| Tool | Discovery file | Install doc |
| --- | --- | --- |
| Claude Code | `~/.claude/skills/sprint/SKILL.md` | [docs/install/claude-code.md](docs/install/claude-code.md) |
| Codex CLI | `AGENTS.md` (project root) | [docs/install/codex.md](docs/install/codex.md) |
| Gemini CLI | `GEMINI.md` (project root) | [docs/install/gemini-cli.md](docs/install/gemini-cli.md) |
| Cursor | `AGENTS.md` or `.cursor/rules/*.mdc` | [docs/install/cursor.md](docs/install/cursor.md) |
| OpenCode | `AGENTS.md` (project root) | [docs/install/opencode.md](docs/install/opencode.md) |
| GitHub Copilot CLI | `AGENTS.md` (project root) | [docs/install/copilot-cli.md](docs/install/copilot-cli.md) |

## Quick install

Claude Code (native skill):

```bash
git clone https://github.com/leverj/claude-sprint ~/.claude/skills/sprint
```

Other tools (clone once, link from each project):

```bash
git clone https://github.com/leverj/claude-sprint ~/sprint-workflow
cd /path/to/your/project
ln -s ~/sprint-workflow/AGENTS.md AGENTS.md   # or GEMINI.md, .cursor/rules/sprint.mdc, etc.
```

On Windows, enable git symlinks (`git config --global core.symlinks true`, run as admin) or `cp` the file and re-copy on update.

## Updating

```bash
cd ~/.claude/skills/sprint && git pull   # or ~/sprint-workflow
```

## Quick Start

```
/sprint setup          # Bootstrap labels on your GitHub repo
/sprint plan           # Create structured requirements as GitHub Issues
/sprint pick           # Claim an issue and implement it
/sprint status         # See what's in-progress, ready, and needs work
```

## Commands

| Command | Purpose |
|---------|---------|
| `/sprint plan` | Discuss and create one or more structured GitHub Issues with acceptance criteria, implementation phases, and risk assessment |
| `/sprint pick [N]` | List available issues, claim one, create a branch, implement phase by phase, create a PR |
| `/sprint decide ["title"]` | Record an architectural decision in `.dev/decisions/`, cross-link to GitHub Issues |
| `/sprint status` | Dashboard of in-progress, ready-to-pick, needs-refinement, recently closed, open PRs, and decisions |
| `/sprint refine [N]` | Take a rough issue and add structured acceptance criteria and phases to make it implementable |
| `/sprint setup` | Bootstrap sprint labels and the `.dev/decisions/` directory |

## How It Works

### Requirements as GitHub Issues

`/sprint plan` creates Issues with a structured format:

- **User Story** — "As a [role], I want [capability] so that [benefit]"
- **Acceptance Criteria** — WHEN/THEN/SHALL format, directly testable
- **Implementation Phases** — ordered checkboxes, each independently verifiable
- **Risk Assessment** — technical risks, dependencies, unknowns

### Concurrency via GitHub Assignment

When a developer runs `/sprint pick`, Claude assigns the issue to them on GitHub before starting work. Other developers running `/sprint pick` will see that issue as taken. GitHub assignment is the lock — no file-based coordination needed.

### Decisions as ADRs

`/sprint decide` creates files in `.dev/decisions/` with context, rationale, alternatives considered, and consequences. These are version-controlled alongside the code they govern. GitHub Issues get a comment linking to the decision for cross-reference.

### Phase-by-Phase Implementation

`/sprint pick` doesn't implement everything at once. It works through the Implementation Phases defined on the issue:

1. Implement the phase
2. Write/update tests
3. Update the phase checkbox on GitHub
4. Move to next phase

After all phases are complete, Claude presents the changes for review. **Code is never committed, pushed, or turned into a PR without explicit developer approval.** The developer gets a chance to review the diff, adjust the commit message, and confirm before anything leaves their local machine.

If a session ends mid-work, the next developer can see which phases are checked off.

## Label Convention

| Label | Meaning |
|-------|---------|
| `type:feature` | New functionality |
| `type:bug` | Something broken |
| `type:refactor` | Internal improvement |
| `priority:high` | High priority |
| `priority:med` | Medium priority |
| `priority:low` | Low priority |
| `ready` | Refined, has acceptance criteria, implementable |
| `needs-refinement` | Logged but needs structured criteria before implementation |
| `in-progress` | Currently being worked on (assigned to someone) |

`package:*` labels are created on-the-fly for monorepo projects (e.g., `package:ui`, `package:server`).

## Decision Records

Decisions are stored in `.dev/decisions/` with this naming convention:

```
.dev/decisions/D-001-use-supabase-for-auth.md
.dev/decisions/D-002-monorepo-structure.md
```

Each file captures: Context, Decision, Rationale, Alternatives Considered, and Consequences.

## Customization

Fork this repo to customize:

- **Labels**: Edit `setup/labels.json` to change names, colors, or add new labels
- **Issue template**: Edit `templates/issue-body.md` to change the issue structure
- **Decision template**: Edit `templates/decision-record.md` to change the ADR format
- **Workflow**: Edit `SKILL.md` to modify command behavior

## Uninstalling

```bash
# Global
rm -rf ~/.claude/skills/sprint

# Per-project
rm -rf .claude/skills/sprint
```

## License

MIT
