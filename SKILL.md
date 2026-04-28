---
name: sprint
description: >
  Scrum-like development workflow using GitHub Issues. Commands: plan (create requirements),
  pick (claim & implement), decide (record decisions), status (dashboard), refine (groom issues),
  setup (bootstrap labels). Uses gh CLI for issue management and .dev/decisions/ for ADRs.
allowed-tools: Bash(gh *) Bash(git *) Bash(ls *) Bash(mkdir *) Bash(cat *) Read Write Edit Glob Grep Agent
argument-hint: "<plan|pick|decide|status|refine|setup> [arguments]"
effort: high
---

# Sprint — Development Workflow Skill

You manage a scrum-like development workflow using GitHub Issues as the single source of truth for requirements, assignment, and tracking. Architectural decisions are stored in `.dev/decisions/` in the repo.

**Key principles:**
- GitHub Issues are requirements. Assignment on GitHub is the concurrency lock.
- Acceptance criteria use WHEN/THEN/SHALL format for testability.
- Implementation is phased — implement, test, and verify each phase before moving on.
- Decisions capture the *why* alongside the *what*.
- **Never commit, push, or create PRs without explicit user permission.** Always present changes for review first and ask the developer to confirm before any git commit, git push, or PR creation.

The skill templates are in the `templates/` directory and label definitions in `setup/labels.json`, both relative to this skill's directory: `${CLAUDE_SKILL_DIR}`.

## User instructions

$ARGUMENTS

## Command routing

Parse the first word of the user's arguments to determine the command:

- **`plan`** (or no arguments) → Go to [Plan Command](#plan-command)
- **`pick`** → Go to [Pick Command](#pick-command)
- **`decide`** → Go to [Decide Command](#decide-command)
- **`status`** → Go to [Status Command](#status-command)
- **`refine`** → Go to [Refine Command](#refine-command)
- **`setup`** → Go to [Setup Command](#setup-command)

If the first word doesn't match any command, treat the entire input as a plan description and go to [Plan Command](#plan-command).

---

## Ensure Setup

**Every command must run these checks first before doing anything else.** Stop with a clear error message if any check fails.

### Step 1: Verify prerequisites

Run these checks in parallel:

1. `gh auth status` — if this fails, tell the user: "GitHub CLI is not authenticated. Run `gh auth login` first."
2. `git rev-parse --is-inside-work-tree` — if this fails, tell the user: "Not inside a git repository."
3. `gh repo view --json nameWithOwner -q .nameWithOwner` — if this fails, tell the user: "No GitHub remote found. Add one with `git remote add origin <url>`." Save the result as `REPO` (e.g., `owner/repo`) for use in later API calls.

### Step 2: Ensure labels exist

Run: `gh label list --search "type:" --limit 1 --json name -q '.[0].name'`

If no result is returned, the sprint labels haven't been created yet. Read the label definitions from `${CLAUDE_SKILL_DIR}/setup/labels.json` and create each one:

```
gh label create "<name>" --color "<color>" --description "<description>" --force
```

Use `--force` so this is idempotent (updates existing labels rather than failing).

### Step 3: Ensure decisions directory

Run: `mkdir -p .dev/decisions`

---

## Plan Command

**Purpose**: Interactive session to refine and create one or more structured GitHub Issues.

### Step 1: Discuss and decompose requirements

Ask the developer what they want to build, fix, or improve. They may give anything from a one-liner ("add social login") to a detailed spec.

**If the requirement is large or spans multiple concerns**, break it down into smaller, independently implementable issues. For example, "add social login" might decompose into:
- OAuth provider integration (Google, Apple)
- Account linking/merging flow
- Session management and token refresh
- UI for sign-in screen

Present the proposed breakdown to the developer and discuss each piece:

1. **Propose the decomposition**: "This looks like it breaks into N pieces: [list]. Does this split make sense, or would you group things differently?"
2. **Discuss each piece one by one**: For each proposed issue, probe for:
   - What problem does this solve?
   - Who is affected?
   - What are the edge cases and constraints?
   - Is this a feature, bug fix, or refactor?
   - What priority — high, medium, or low?
   - Are there dependencies between this and the other pieces?
3. **Get confirmation**: Don't create any issues until the developer confirms the full set. They may want to merge, split, reprioritize, or defer some pieces.

If the requirement is small and self-contained, skip the decomposition step and discuss it directly.

Note dependencies between issues in the Notes section of each issue (e.g., "Depends on #42 for OAuth token flow").

### Step 2: Structure each requirement

For each requirement discussed, read the template from `${CLAUDE_SKILL_DIR}/templates/issue-body.md` and fill it in:

- **User Story**: Write in "As a [role], I want [capability] so that [benefit]" format.
- **Acceptance Criteria**: Write each criterion using the WHEN/THEN/SHALL format. Cover:
  - Normal/happy path flows
  - Edge cases
  - Error handling
  - Persistence expectations (if applicable)
  - UI/UX behavior (if applicable)
- **Implementation Phases**: Break the work into 2-4 ordered phases. Each phase should be independently testable. Don't make phases too granular — each should represent a meaningful chunk of work.
- **Risk Assessment**: Note technical risks, dependencies on other issues or services, and unknowns.
- **Notes**: Add any context from the discussion, links to related decisions, or references.

### Step 3: Select labels

For each issue, determine labels from the discussion:

- **Type** (exactly one): `type:feature`, `type:bug`, or `type:refactor`
- **Priority** (exactly one): `priority:high`, `priority:med`, or `priority:low`
- **Package** (optional, for monorepos): If the project is a monorepo and the requirement targets a specific package, create a `package:<name>` label if it doesn't exist: `gh label create "package:<name>" --color "c5def5" --description "Package: <name>" --force`
- **Readiness**: Add `ready` if the requirement is fully refined. Add `needs-refinement` if it still has unknowns that need resolving.

### Step 4: Milestone (optional)

If the developer mentions a sprint name or release target, check for an existing milestone:

```
gh api repos/{REPO}/milestones --jq '.[] | select(.title=="<name>") | .number'
```

If none exists, create one:

```
gh api repos/{REPO}/milestones -f title="<name>"
```

### Step 5: Create issues

For each requirement, create a GitHub Issue:

```
gh issue create --title "<concise title>" --body "<structured body>" --label "<label1>,<label2>,..." [--milestone "<name>"]
```

### Step 6: Summary

After all issues are created, present a summary table:

```
Created N issues:
  #42  Feed freezes on background return     [type:bug, priority:high, ready]
  #43  Search filter support                 [type:feature, priority:med, ready]
  #44  Artist social links                   [type:feature, priority:low, needs-refinement]
```

---

## Pick Command

**Purpose**: Claim a GitHub Issue and implement it phase by phase.

Arguments after `pick` are optional:
- `/sprint pick` — show available issues and let the developer choose
- `/sprint pick 42` — directly pick issue #42

### Step 1: Show available work

Run these in parallel:

1. `gh issue list --label ready --no-assignee --json number,title,labels --limit 20` — available work
2. `gh issue list --assignee @me --state open --json number,title,labels,assignees --limit 10` — already assigned to current user

Present the results:

```
YOUR WORK
  #42  Feed freeze fix          [type:bug, priority:high, in-progress]

AVAILABLE TO PICK
  #43  Search filters           [type:feature, priority:med]
  #44  Artist social links      [type:feature, priority:low]
```

If an issue number was provided as argument, skip this display and go directly to Step 2.

### Step 2: Claim the issue

Once the developer picks an issue (or one was specified via argument):

1. Check if it's already assigned: `gh issue view N --json assignees -q '.assignees[].login'`
   - If assigned to someone else, warn: "This issue is assigned to @username. Do you want to reassign it to yourself?"
   - If assigned to the current user, proceed (already claimed).
2. Assign to self: `gh issue edit N --add-assignee @me`
3. Update labels: `gh issue edit N --add-label in-progress --remove-label ready`

### Step 3: Create branch

1. Get the issue title: `gh issue view N --json title,labels -q '.title'`
2. Determine branch prefix from labels:
   - `type:bug` → `bug/`
   - `type:feature` → `feat/`
   - `type:refactor` → `refactor/`
   - Default → `feat/`
3. Generate branch name: `<prefix><N>-<slugified-title>` (lowercase, hyphens, max 50 chars)
   - Example: `bug/42-fix-feed-freeze-on-background`
4. Create and switch: `git checkout -b <branch-name>`
   - If the branch already exists, ask the developer: switch to it or create an alternate name?

### Step 4: Read the issue spec

Fetch the full issue body: `gh issue view N --json body,title -q '.body'`

Parse the issue body to extract:
- **Acceptance Criteria** — these drive what to test
- **Implementation Phases** — these drive the work order
- **Risk Assessment** — be aware of these during implementation

### Step 5: Implement phase by phase

For each phase listed in the Implementation Phases section:

1. **Implement** the phase — write the code, following existing project patterns and conventions.

   **Dependency safety gate**: Before editing any manifest file (`package.json`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, etc.) or running any install/add command (`npm install`, `pnpm add`, `yarn add`, `pip install`, `cargo add`, `go get`, `mvn install -Dversion=`, etc.) that introduces a new dependency or changes a resolved version, run the [Dependency Safety Check](#dependency-safety-check). The check must complete with a clean pass, an accepted fallback substitution, or an explicit override before the manifest edit/install proceeds. Do not edit the manifest first and check afterward — the check gates the edit.
2. **Test** — write or update tests that verify the acceptance criteria covered by this phase. Run the test suite to confirm.
3. **Update the issue** — mark the phase checkbox as complete on GitHub:
   - Fetch current body: `gh issue view N --json body -q '.body'`
   - Find the specific `- [ ] Phase X:` line and replace `- [ ]` with `- [x]`
   - Update: `gh api repos/{REPO}/issues/N -f body="<updated body>"`

Repeat for each phase. If a phase reveals problems or new requirements, discuss with the developer before proceeding.

**Do NOT commit anything yet.** Code changes are local only at this point.

### Step 6: Final review

After all phases are complete:

1. Run the full test suite
2. Check for lint or type errors if the project has those tools
3. Review all changes: `git diff` to show the developer what was changed
4. Present a summary of all changes, organized by phase

### Step 7: User review and commit

**Ask the developer to review the changes before committing.** Present:
- A summary of files changed and what each change does
- The proposed commit message(s)
- Ask: "Would you like to review the diff, adjust anything, or proceed with committing?"

**Only after the developer explicitly approves**, commit the changes:
- `git add <specific files>` (never `git add -A`)
- Create commit(s) referencing the issue: `Refs #N: <phase description>`

**Do NOT push to remote yet.**

### Step 8: Push and create PR

**Ask the developer for permission before pushing and creating the PR.** Present:
- The commit(s) that will be pushed
- The proposed PR title and body
- Ask: "Ready to push and create the PR?"

**Only after the developer explicitly approves**, push and create the PR:

1. Push: `git push -u origin <branch-name>`
2. Create PR:

```
gh pr create --title "<concise title>" --body "Closes #N

## Summary
<bullet points of what was done>

## Acceptance Criteria Verified
<list each criterion and how it was verified>

## Test Plan
<what tests were added/modified>

## Dependency Safety
<Render the verification log captured during the per-phase Dependency Safety Check. One row per package added or upgraded:
  - `package@version` (final, after any fallback)
  - Originally requested version (if different from final)
  - Publish date
  - Advisories checked (IDs or 'none')
  - Fallback reason (if a substitution was applied)
  - Override justification (if a hard-block was overridden)
If no dependencies were added or upgraded in this PR, write 'No dependency changes.'>
"
```

### Step 9: Report

Show the PR URL and a summary of what was implemented, which phases were completed, and any remaining notes.

---

## Decide Command

**Purpose**: Record an architectural or design decision with full rationale.

Arguments after `decide` are optional:
- `/sprint decide` — interactive, Claude asks what the decision is about
- `/sprint decide "Use PostgreSQL for event store"` — title provided upfront

### Step 1: Find next decision number

```
ls .dev/decisions/D-*.md 2>/dev/null | sort -V | tail -1
```

Extract the number from the last file and increment. If no files exist, start at D-001. Pad to 3 digits.

### Step 2: Discuss the decision

If no title was provided as argument, ask the developer:
- What decision needs to be recorded?
- What was the context — what problem or situation prompted this?
- What alternatives were considered?
- Why was this option chosen over the alternatives?
- What are the consequences (positive and negative)?

If a title was provided, still probe for context, rationale, and alternatives.

### Step 3: Link to issues

Ask: "Are there GitHub Issues related to this decision?"

If yes, note the issue numbers for cross-linking.

### Step 4: Write the decision record

Read the template from `${CLAUDE_SKILL_DIR}/templates/decision-record.md` and fill it in with the discussed details.

Write to `.dev/decisions/D-<NNN>-<slugified-title>.md`

Example: `.dev/decisions/D-003-use-postgresql-for-event-store.md`

### Step 5: Cross-link to issues

For each related GitHub Issue, add a comment:

```
gh issue comment N --body "Decision recorded: [D-<NNN>: <title>](.dev/decisions/D-<NNN>-<slug>.md)

**Decision**: <one-line summary>
**Rationale**: <one-line summary>"
```

### Step 6: Report

Show the file path, a summary of the decision, and which issues were linked.

---

## Status Command

**Purpose**: Bird's eye view of the current sprint state.

Run all of these in parallel:

1. `gh issue list --label in-progress --state open --json number,title,labels,assignees --limit 20`
2. `gh issue list --label ready --no-assignee --json number,title,labels --limit 20`
3. `gh issue list --label needs-refinement --json number,title,labels --limit 20`
4. `gh issue list --state closed --json number,title,closedAt --limit 10 --sort updated`
5. `ls .dev/decisions/D-*.md 2>/dev/null`
6. `gh pr list --state open --json number,title,author,headRefName --limit 10`

Format as a dashboard:

```
=== Sprint Status ===

IN PROGRESS
  #42  Feed freeze fix              @nirmal    [type:bug, priority:high]
  #43  Search filter support         @alice     [type:feature, priority:med]

READY TO PICK
  #44  Artist social links          [type:feature, priority:low]
  #45  Refactor API layer           [type:refactor, priority:med]

NEEDS REFINEMENT
  #46  Performance improvements     [needs-refinement]

RECENTLY CLOSED
  #40  Setup CI pipeline            closed 2d ago
  #41  Fix memory leak              closed 1d ago

OPEN PRs
  #12  bug/42-feed-freeze-fix       @nirmal

DECISIONS
  D-001  Use Supabase for auth
  D-002  Monorepo structure
  D-003  JSON file persistence
```

If any section is empty, show "(none)" rather than omitting it.

For decisions, read just the first line (title) from each file to display the name.

---

## Refine Command

**Purpose**: Take a rough issue and add structured acceptance criteria, phases, and risks to make it implementable.

Arguments after `refine` are optional:
- `/sprint refine 46` — refine issue #46 directly
- `/sprint refine` — list issues labeled `needs-refinement` and let the developer choose

### Step 1: Select issue

If no issue number provided, list issues needing refinement:

```
gh issue list --label needs-refinement --json number,title --limit 20
```

Present the list and ask which one to refine.

### Step 2: Read the issue

Fetch the full issue with comments:

```
gh issue view N --json body,title,labels,comments
```

Display the current state to the developer.

### Step 3: Interactive refinement

Discuss with the developer to fill in what's missing:

- Are the acceptance criteria complete? Add missing WHEN/THEN/SHALL statements.
- Are implementation phases defined? Break the work into testable phases.
- Are risks identified? Note technical risks, dependencies, unknowns.
- Is the user story clear? Refine it if needed.

### Step 4: Update the issue

Construct the updated issue body incorporating the refined content. Preserve any existing content that's still valid.

Update via the GitHub API:

```
gh api repos/{REPO}/issues/N -f body="<updated body>"
```

### Step 5: Update labels

```
gh issue edit N --remove-label needs-refinement --add-label ready
```

### Step 6: Report

Show what was added/changed and confirm the issue is now marked as `ready`.

---

## Setup Command

**Purpose**: Explicitly bootstrap the sprint workflow for a repository. This runs the same checks as [Ensure Setup](#ensure-setup) but reports what was created.

### Step 1: Run Ensure Setup

Follow the [Ensure Setup](#ensure-setup) steps.

### Step 2: Report

Tell the developer what was set up:

- Which labels were created (or "all labels already exist")
- Whether `.dev/decisions/` was created (or "already exists")
- The repo name and current authentication status
- Remind them of available commands: `/sprint plan`, `/sprint pick`, `/sprint status`, `/sprint decide`, `/sprint refine`

---

## Dependency Safety Check

**Purpose**: Before any sprint task adds or upgrades a third-party dependency, verify that the target version is not freshly published (supply-chain compromise window) and not subject to a known high/critical advisory. If unsafe, automatically redirect to the closest clean version.

This check is invoked from the [Pick Command](#pick-command) per-phase loop. It runs **before** any manifest edit (`package.json`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, etc.) and **before** any install/add command (`npm install`, `pnpm add`, `yarn add`, `pip install`, `cargo add`, `go get`, `mvn ... -Dversion=`, etc.).

### Triggers

Run the check when a phase implementation will:

- Add a new dependency entry.
- Update an existing dependency to a different version (including lockfile-only bumps that change the resolved version of a top-level dep).
- Replace one package with another.

A pure lockfile regeneration that resolves to the **same** versions does not trigger the check.

### Supported ecosystems

Fully supported in v1: **npm**, **PyPI**, **crates.io**, **Go modules**, **Maven Central**.

If a manifest belongs to any other ecosystem (CocoaPods, Hex, NuGet, RubyGems, Composer, etc.), **hard-block** with the message: `Dependency safety check: ecosystem not supported. Override required to proceed.`

### Caching

Maintain an in-memory cache for the duration of the pick, keyed on `ecosystem:package:version`. Each entry stores the full check result: publish date, advisory list, install-script flags, and final verdict (pass / blocked-with-reason / accepted-fallback-target).

- Steps 1, 2, and 4 (and the per-candidate re-checks inside Step 5's fallback walk) MUST consult the cache before making a network call. A hit returns the stored result without re-querying the registry or OSV.
- The verification log (Step 7) reads from the same cache, so the PR body reflects exactly what was checked.
- The cache lives only for the current pick — a fresh pick (or a new Claude session) starts empty. Do not persist it to disk.
- Typosquat (Step 3) does not need the cache; it operates on the package name alone.

### Step 1: Resolve publish date (7-day age guard)

For the target `package@version`, look up the publish timestamp. If the publish date is **less than 7 days ago** from today, the version is blocked. Per-ecosystem recipes:

- **npm**:
  ```
  npm view <pkg>@<version> time --json
  ```
  Use the entry keyed by the exact version string. (Falls back to `https://registry.npmjs.org/<pkg>` JSON, field `time.<version>`.)

- **PyPI**:
  ```
  curl -s https://pypi.org/pypi/<pkg>/<version>/json
  ```
  Use `urls[0].upload_time_iso_8601` (or earliest of the `urls` array).

- **crates.io**:
  ```
  curl -s https://crates.io/api/v1/crates/<pkg>/<version>
  ```
  Use `version.created_at`.

- **Go modules**:
  ```
  curl -s https://proxy.golang.org/<module>/@v/<version>.info
  ```
  Use the `Time` field. List versions via `https://proxy.golang.org/<module>/@v/list`.

- **Maven Central**:
  ```
  curl -s "https://search.maven.org/solrsearch/select?q=g:<group>+AND+a:<artifact>+AND+v:<version>&core=gav&rows=1&wt=json"
  ```
  Use `response.docs[0].timestamp` (epoch ms).

If the lookup fails (network error, 404, malformed response), **fail closed** — hard-block with a remediation message naming the registry and command that failed.

### Step 2: Advisory scan

Query OSV.dev as the unified backend for advisories:

```
curl -s -X POST https://api.osv.dev/v1/query -H 'Content-Type: application/json' \
  -d '{"package":{"name":"<pkg>","ecosystem":"<ECOSYSTEM>"},"version":"<version>"}'
```

Ecosystem strings: `npm`, `PyPI`, `crates.io`, `Go`, `Maven`.

If the response includes any vulnerability with a `database_specific.severity` or CVSS score in the **HIGH** or **CRITICAL** range (or, when severity is missing, any vulnerability at all that affects this exact version), the version is blocked. Capture each vulnerability's `id`, `summary`, and reference URL for the report.

If the OSV call fails, **fail closed**.

### Step 3: Typosquat heuristic

For new dependencies (not version bumps of an existing package), compute the Levenshtein distance between the requested package name and a list of popular packages on the same ecosystem. If distance ≤ 2 from a popular name **and** the requested name is not itself a popular name, hard-block with: `Possible typosquat: '<requested>' is distance N from popular package '<popular>'. Override required.`

The popular-package seed list is a small static set per ecosystem (e.g., for npm: `react`, `lodash`, `express`, `axios`, `chalk`, `commander`, `moment`, `request`, `webpack`, `typescript`, `eslint`, `jest`, `vue`, `next`, `dotenv`). Treat the seed as illustrative — extend as needed when false positives or misses appear.

### Step 4: Install-script check

If the package declares install-time scripts, hard-block:

- **npm**: `npm view <pkg>@<version> scripts --json` — block if any of `preinstall`, `install`, `postinstall` is set.
- **PyPI**: block if the distribution includes a `setup.py` (sdist with executable setup, as opposed to a wheel-only release). Detect via the PyPI JSON `urls` array — if no entry has `packagetype == "bdist_wheel"`, treat as install-script risk.
- **crates.io**: block if the crate declares a `build.rs` (check via the published crate metadata; if unverifiable, note as unknown and require override).
- **Go modules**: no install scripts in the module system; skip.
- **Maven Central**: skip; build plugins are scoped to the consuming project, not install-time.

Override requires explicit justification recorded in the PR + issue comment.

### Step 5: Clean-version fallback

If Step 1, 2, 3, or 4 blocks the requested version, automatically search for a clean adjacent version:

1. **Prefer fixed-in versions** (advisory-block only): If the OSV response includes `affected[].ranges` with `fixed` versions, walk them in ascending order. For each candidate, run Step 1 and Step 2 again. Return the **lowest** fixed-in version that passes both (i.e., is ≥ 7 days old and free of high/critical advisories).
2. **Fall back to highest prior unaffected version**: If no fixed-in version passes (or none was listed), walk the registry's version list in **descending** order from the version immediately below the requested one. For each, run Step 1 and Step 2. Return the first version that passes both.
3. **No clean version found**: If neither direction yields a passing candidate, hard-block and report the search trace (versions tried + reason each was rejected) so the developer can pick a different package.

When a fallback is found, propose it to the developer with the reason, e.g.:

```
v2.3.4 is affected by GHSA-xxxx-yyyy (HIGH).
Suggest v2.3.5 — fixed-in, published 14 days ago, no advisories.
```

The developer must confirm the substitution before the manifest is edited.

### Step 6: Override flow

A hard-block can be overridden, but only with explicit justification:

1. The assistant explains exactly why the version was blocked.
2. The developer types an override justification (free text, must be non-empty).
3. The justification is recorded in **both** the PR body's "Dependency Safety" section and a comment on the issue:
   ```
   gh issue comment N --body "Dependency safety override: <pkg>@<version> — <justification>"
   ```
4. The phase proceeds.

### Step 7: Record verification

On success (clean pass or accepted fallback), append a row to an in-memory verification log for this branch. At PR time, the log is rendered into the PR body's "Dependency Safety" section (see [Step 8 of Pick Command](#step-8-push-and-create-pr)). Each row captures:

- `package@version` (final, after any fallback)
- Originally requested version (if different)
- Publish date
- Advisories checked (IDs or "none")
- Fallback reason (if applicable)
- Override justification (if applicable)

---

## Error Handling

Apply these rules across all commands:

- **Network errors from `gh`**: Report the error clearly and suggest the developer check their internet connection and `gh auth status`.
- **Permission errors**: Report that the user may not have write access to the repository.
- **Empty results**: Never show raw empty output. Always say "No issues found matching criteria" or similar.
- **Malformed issue body**: When reading an issue body that doesn't follow the template structure (e.g., created manually on GitHub without the template), work with what's available. Don't fail — adapt. If refining, add the structure that's missing.
- **Rate limiting**: If `gh api` returns a rate limit error, tell the developer and suggest waiting.
