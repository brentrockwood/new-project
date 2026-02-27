# `dob` v0.2 — Implementation Spec for Claude Code

## Context

`dob` (Date of Birth) replaces the existing `new-project` script in the
`brentrockwood/new-project` repo. It is part of the DOA Framework — a system
for structured AI agent collaboration on software projects.

The existing `new-project` script handles greenfield project bootstrapping.
`dob` extends this to handle any project state: new, existing-unadopted, or
already-adopted. One command, one path, behavior determined by what's found at
the target path.

The existing `new-project` script is at:
https://github.com/brentrockwood/new-project

Read it before starting. Port its logic faithfully — do not reinvent it.

---

## Repo Changes

- Repo `brentrockwood/new-project` will be renamed to `brentrockwood/dob`
  (out of scope for this task — just be aware)
- Script name: `dob` (replaces `new-project`)
- Single script file output — the entire implementation lives in one Bash file
- Everything else in the repo stays as-is:
  `devcontainers/`, `postCreate.sh`, `scripts/security_scan.sh`
- Update `RUNBOOK.md` to reflect new command name and behavior

---

## Invocation

```
dob <path> [--dry-run] [--force] [--remote]
```

- `<path>` — required. Target directory. Created if it doesn't exist.
- `--dry-run` — run evaluate, print migration plan, exit 0. No files written,
  no git state modified.
- `--force` — bypass hard fails after explicit per-fail confirmation prompt.
  Full trust after confirmation. No further interference.
- `--remote` — use dh1 SSH DevPod provider instead of local Docker. Pass
  through to DevPod unchanged from existing `new-project` behavior.

No subcommands. The path determines everything.

---

## Prerequisites

Inherited from `new-project`. Must be present and will be checked at startup:

- `bash` 4+ (Homebrew bash assumed — macOS ships Bash 3, which lacks
  associative arrays)
- `gh` (GitHub CLI)
- `git`
- `devpod`

---

## High-Level Flow

```
main()
  parse_args()
  evaluate()
  build_migration_plan()
  print_report()
  → if --dry-run: exit 0
  → if hard fails present and not --force: exit 1
  → if hard fails present and --force: confirm_force() per fail, exit 1 if any denied
  migrate()
```

---

## Script Structure

Use functions throughout. No logic at the top level beyond calling `main()`.

Use Bash 4+ associative arrays for check results:

```bash
declare -A CHECK_STATUS       # pass | warn | fail | auto
declare -A CHECK_MESSAGE      # human-readable description of finding
declare -A CHECK_REMEDIATION  # what user must do (fail only, else empty string)
declare -A CHECK_ACTION       # function name to call in migrate (auto only, else empty string)
declare -A CHECK_ORDER        # integer, determines migrate step execution order
```

---

## Evaluate Phase

Read-only. Idempotent. No files written. No git state modified.

Runs all checks unconditionally (do not short-circuit on first fail — collect
all results so the report is complete).

### Checks

#### `check_uncommitted`

Detect uncommitted files in the working tree.

- **pass**: `git status --porcelain` returns empty, or directory has no `.git`
  yet (not a repo — can't have uncommitted files)
- **fail**: uncommitted files exist
- Remediation hint: `git add -A && git commit -m "..."` or `git stash`

#### `check_partial_adoption`

Detect partial DOA adoption.

Ownership marker: `project/doa.md`

Sentinel files:
- `project/context.md`
- `project/scripts/add-context`
- `project/scripts/read-context`
- `project/scripts/rotate-context`

Logic:
- `project/doa.md` present → fully adopted → **pass**
- `project/doa.md` absent AND all sentinels absent → not adopted → **pass**
  (will be handled by migration plan)
- `project/doa.md` absent AND any sentinel present → partial adoption → **fail**
- Remediation hint: list exactly which sentinel files were found, suggest
  either completing adoption manually or removing them before running `dob`

#### `check_security`

Run `scripts/security_scan.sh` (already exists in this repo) against the
target path.

- **pass**: exit code 0
- **fail**: exit code 1 (issues found)
- **error**: exit code 2 (script error) — treat as warn, note the scan
  could not complete
- Remediation hint: `run scripts/security_scan.sh and resolve all findings`

If `scripts/security_scan.sh` is not found relative to the `dob` script
location, skip with a warn noting the scan was unavailable.

#### `check_gh_auth`

Run `gh auth status`.

- **pass**: authenticated
- **warn**: not authenticated — migration will fail at GitHub steps, but
  evaluate should not hard-fail on this

#### `check_stack`

Detect stack from files present in target path:

| File | Stack |
|---|---|
| `package.json` | TypeScript / Node |
| `pyproject.toml` or `requirements.txt` | Python |
| `go.mod` | Go |

- **pass**: stack detected, store detected value for use in migration
- **warn**: stack unknown — migration will use a generic template

---

## Build Migration Plan

After evaluate, assemble an ordered list of migration steps. A step is added
to the plan only when its condition is not already met.

| Order | Step Function | Condition to add |
|---|---|---|
| 10 | `step_create_project_dir` | `project/` does not exist |
| 20 | `step_copy_scripts` | `project/scripts/` missing or incomplete |
| 30 | `step_write_doa` | `project/doa.md` does not exist |
| 40 | `step_write_project_md` | `project/project.md` does not exist |
| 50 | `step_init_git` | `.git` does not exist |
| 60 | `step_write_gitignore` | `.gitignore` does not exist |
| 70 | `step_write_secrets_example` | `secrets.env.example` does not exist |
| 80 | `step_add_context_entry` | Always (documents the migration) |
| 90 | `step_initial_commit` | Any files were written |
| 100 | `step_create_github_repo` | No GitHub remote detected |
| 110 | `step_create_devpod` | Always (after repo exists) |
| 120 | `step_open_pr` | Always (after push) |

**Already-adopted project**: all steps are pass → plan is empty → report +
exit 0.

**Net-new empty directory**: most steps added → full bootstrap.

**Existing unadopted project**: steps 10–40, 80–120 added; 50–70 likely
skipped.

---

## Print Report

Always printed to stdout, even on success. Format:

```
dob v0.2.0 — evaluating /path/to/project

  ✓  working tree clean
  ✓  no partial adoption detected
  ✗  secrets detected in repo                    [FAIL]
     → run scripts/security_scan.sh and resolve findings before proceeding
  ⚠  gh not authenticated                        [WARN]
  ⚠  stack unknown                               [WARN]

migration plan (5 steps):
  1. create project/ directory
  2. copy project/scripts/ from DOA repo
  3. write project/doa.md
  4. write project/project.md (onboard template)
  5. write first context entry

1 failure. resolve before migrating, or run with --force.
```

If plan is empty (already adopted, nothing to do):

```
dob v0.2.0 — evaluating /path/to/project

  ✓  working tree clean
  ✓  fully adopted
  ✓  security scan clean
  ✓  gh authenticated
  ✓  stack: typescript

nothing to do. project is fully adopted.
```

---

## `--force` Confirmation

One prompt per hard fail, in the order fails were encountered. If the user
answers `n` to any prompt, exit 1 immediately. If all answered `y`, proceed
to migrate.

**Uncommitted files:**
```
WARNING: Uncommitted files detected. If migration fails or corrupts your
working tree, there is no recovery path. You will lose unsaved work.
Proceed anyway? (y/n)
```

**Partial adoption:**
```
WARNING: Partial DOA adoption detected. Proceeding may overwrite or conflict
with existing framework files. Review what's present before continuing.
Proceed anyway? (y/n)
```

**Security scan:**
```
WARNING: Secrets or sensitive data detected in this repository. Proceeding
will commit DOA framework files alongside potentially exposed credentials.
Proceed anyway? (y/n)
```

---

## Migrate Phase

Execute steps from the migration plan in order. Each step function:

- Announces what it's doing (`echo "  → step description"`)
- Does its work
- Exits non-zero on failure (migrate aborts, prints error, exits 1)

### Step Implementations

#### `step_create_project_dir`
```bash
mkdir -p "$TARGET/project"
```

#### `step_copy_scripts`
Clone or fetch `brentrockwood/doa` into a temp directory, copy
`project/scripts/` into the target. Make scripts executable.

DOA repo URL: `https://github.com/brentrockwood/doa`

Use `gh repo clone brentrockwood/doa <tmpdir>` if gh is authenticated,
otherwise `git clone`. Clean up tmpdir on exit (trap).

#### `step_write_doa`
Copy `project/doa.md` from the cloned DOA repo (reuse tmpdir from
`step_copy_scripts` — clone once, reuse).

#### `step_write_project_md`
Write `project/project.md` from the appropriate template (see Templates
section below). Select template based on whether existing code was detected
in the target directory.

"Existing code detected" = files present beyond `.git`, `.gitignore`,
`secrets.env.example`, and the `project/` directory itself.

#### `step_init_git`
```bash
git -C "$TARGET" init
git -C "$TARGET" checkout -b main
```

#### `step_write_gitignore`
Write a standard `.gitignore` appropriate to the detected stack. If stack
unknown, write a minimal generic one. Include `secrets.env` at minimum.

#### `step_write_secrets_example`
Write a minimal `secrets.env.example` with a header comment explaining its
purpose.

#### `step_add_context_entry`
Run `project/scripts/add-context` with:
- `--agent "dob"`
- `--model "v0.2.0"`
- `--output project/context.md`
- Body: `"DOB migration complete. Project bootstrapped by dob v0.2.0. Branch: main"`

#### `step_initial_commit`
```bash
git -C "$TARGET" add -A
git -C "$TARGET" commit -m "DOB: Initial DOA framework scaffold"
```

#### `step_create_github_repo`
Port directly from existing `new-project` logic. Create private repo,
push main branch, create and push `initial-scaffold` branch.

#### `step_create_devpod`
Port directly from existing `new-project` logic. Respect `--remote` flag.

#### `step_open_pr`
Port directly from existing `new-project` logic.

---

## Templates

### `project/project.md` — New Project

Use the existing template from `new-project` / `prjTemplate`. No changes.

### `project/project.md` — Onboard (existing code detected)

```markdown
# [Project Name]

## Discovered State

<!-- Populated by dob. Do not edit manually. -->
- **Stack**: [detected stack or "unknown"]
- **Detected files**: [top-level directory listing]

## Ratified Decisions

<!-- To be filled in during first planning session with agent. -->

## Open Questions

<!-- Ambiguities surfaced during onboarding. -->

## Known Debt / Deferred Items

<!-- Findings from dob evaluate that need attention but are not blocking. -->
```

Substitute `[Project Name]` with the directory basename.
Substitute `[detected stack]` with check_stack result.
Substitute `[top-level directory listing]` with `ls` of target root,
excluding `.git`.

---

## Version String

Hardcoded at top of script:

```bash
DOB_VERSION="0.2.0"
```

Print in report header as `dob v0.2.0`.

---

## Environment Variable Overrides

Inherit from `new-project`. Add one new one:

```bash
# GitHub username (default: brentrockwood)
NEW_PROJECT_GITHUB_USER

# Template repo (default: github.com/brentrockwood/prjTemplate)
NEW_PROJECT_TEMPLATE

# Where new projects land (default: ~/src)
NEW_PROJECT_SRC_DIR

# DOA repo for scripts/doa.md source (default: github.com/brentrockwood/doa)
DOB_DOA_REPO
```

---

## Error Handling

- Every external command call checks exit code
- Migrate aborts immediately on any step failure
- Print clear error message indicating which step failed
- Exit 1 on any failure
- Use `trap` to clean up tmpdir on exit (normal and error)

---

## RUNBOOK.md Updates

Update the existing RUNBOOK.md to reflect:

- Command renamed from `new-project` to `dob`
- Installation symlink: `ln -s ~/dob/dob ~/bin/dob`
- New invocation syntax and flags
- Brief explanation of the DOB/DOA naming (intentional, not a typo)
- Updated "What it does" section covering the unified path behavior
- `--dry-run` and `--force` documented
- Remove references to `new-project` subcommand — there are no subcommands

---

## What NOT to Change

- `devcontainers/` — untouched
- `postCreate.sh` — untouched
- `scripts/security_scan.sh` — untouched, invoked by `dob`
- Existing GitHub Actions if any — untouched

---

## Acceptance Criteria

- [ ] `dob ~/src/brand-new-project` on a nonexistent path runs full bootstrap,
      produces a working DevPod workspace with DOA files in place
- [ ] `dob ~/src/existing-code` on an existing repo with no DOA files runs
      evaluate, detects unadopted state, migrates, opens PR
- [ ] `dob ~/src/adopted-project` on a fully adopted project prints clean
      report and exits 0 without modifying anything
- [ ] `dob ~/src/dirty-project` with uncommitted files fails evaluate and
      prints remediation hint
- [ ] `dob ~/src/dirty-project --force` prompts with warning, proceeds if
      confirmed
- [ ] `dob ~/src/any-project --dry-run` prints evaluate report and migration
      plan, makes no changes
- [ ] All existing `new-project` behavior is preserved under the new script name
