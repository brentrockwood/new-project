# Project System Runbook

This document describes the complete project creation and development workflow.
It covers the tools, repositories, conventions, and the decisions behind them.

---

## Overview

Every new project is bootstrapped by a single script — `new-project` — which:

1. Asks for a project name and stack
2. Clones `prjTemplate` as a starting point
3. Creates a private GitHub repo
4. Stands up a DevPod workspace with your dotfiles and stack tools pre-installed
5. Runs a security scan and opens a PR

From that point forward, every session inside the workspace follows the
Development Operating Agreement (`project/doa.md`), which is part of every
project by default.

---

## Repositories

| Repo | Purpose |
|------|---------|
| `brentrockwood/new-project` | This repo. The `new-project` script and devcontainer templates. |
| `brentrockwood/prjTemplate` | The project scaffold. Cloned fresh for every new project. |
| `brentrockwood/dotfiles` | Shell environment. Installed into every devcontainer. |
| `brentrockwood/<project>` | Each project created by `new-project`. Private by default. |

---

## Prerequisites

The following must be installed on your Mac before using `new-project`:

```bash
# Package manager
brew install gh git

# DevPod CLI
brew install devpod

# Authenticate GitHub CLI
gh auth login
```

Verify everything is ready:

```bash
gh auth status
devpod version
git --version
```

---

## Installation

```bash
# Clone this repo
git clone git@github.com:brentrockwood/new-project.git ~/new-project

# Symlink the script into your PATH
ln -s ~/new-project/new-project ~/bin/new-project
chmod +x ~/bin/new-project

# Make sure ~/bin is in PATH — add to ~/.zshrc.local if not already:
# export PATH="$HOME/bin:$PATH"
```

---

## Creating a New Project

```bash
new-project
```

You will be prompted for:

- **Project name** — lowercase, hyphens only (e.g. `my-cool-tool`)
- **Stack** — `1` for Python, `2` for TypeScript
- **Confirmation** — a summary is shown before anything happens

That's it. The script handles everything else.

### What happens

```
1.  Preflight:     gh, git, devpod present and authenticated
2.  Clone:         prjTemplate → ~/src/<name>
3.  Re-init git:   fresh history, no template commits
4.  Substitute:    [Project Name] tokens replaced throughout
5.  .devcontainer: stack-appropriate devcontainer.json + postCreate.sh
6.  Context:       first entry written to project/context.md
7.  GitHub:        private repo created, main branch pushed
8.  Branch:        initial-scaffold branch created and pushed
9.  send 'er:      security scan → PR: initial-scaffold → main
10. DevPod:        workspace created (local Docker by default)
11. Print:         SSH command and next steps
```

### Remote option (dh1)

```bash
new-project --remote
```

Uses `dh1` (SSH provider) instead of local Docker. Requires the DevPod SSH
provider to be configured:

```bash
devpod provider add ssh
# or follow DevPod docs for SSH provider setup
```

---

## Connecting to the Workspace

After `new-project` completes:

```bash
# SSH directly into the devcontainer
ssh <project-name>.devpod

# DevPod adds this alias to ~/.ssh/config automatically.
# Inside the container:
cd /workspaces/<project-name>
project/scripts/read-context        # see what was done at creation
```

---

## Inside the Container

Every devcontainer has:

| Tool | Notes |
|------|-------|
| zsh | Default shell, vi mode, Starship prompt |
| neovim | `vi` alias, config from dotfiles |
| tmux | Config from dotfiles |
| zoxide | `z` for directory jumping |
| fzf | History search, completion |
| starship | Prompt |
| gh | GitHub CLI |
| trufflehog | Deep secrets scanning |
| git | Pre-installed |

Plus stack-specific tools:

**Python:**
- `uv` — fast package/project manager (replaces pip/venv/pyenv)
- Python 3.12 via `uv python install 3.12`

**TypeScript:**
- Node 22 LTS via nvm
- pnpm
- TypeScript (`tsc`), ts-node

---

## Project Structure

Every project created by `new-project` has this layout:

```
<project>/
├── .devcontainer/
│   ├── devcontainer.json       ← stack-specific, committed
│   └── postCreate.sh           ← runs at container creation
├── src/                        ← your code
├── tests/
├── docs/
│   └── DEPLOYMENT.md
├── scripts/
│   ├── run_checks.sh           ← security + lint + test
│   └── security_scan.sh        ← grep + Trufflehog
├── project/                    ← AI agent operational files
│   ├── doa.md                  ← Development Operating Agreement
│   ├── project.md              ← implementation plan (write-locked)
│   ├── context.md              ← session work log
│   └── scripts/
│       ├── add-context         ← append to context.md
│       ├── read-context        ← read recent entries
│       └── rotate-context      ← archive old entries
├── README.md
├── DEPLOYMENT.md
├── .gitignore
├── .env.example
└── secrets.env.example
```

---

## The Development Operating Agreement (DOA)

Every project includes `project/doa.md`. This is the contract between you and
any AI agent working on the project. Key points:

- Every session starts by reading `project/project.md` and running
  `project/scripts/read-context -n 2`
- Every session ends with `project/scripts/add-context` and a commit
- `project/project.md` is **write-locked** — only humans can change it
- `project/context.md` is **append-only** — entries are never edited
- `send 'er` is the phrase that triggers the full quality gate + push + PR

Agents are expected to follow the DOA without being reminded. If you're
starting a session with an agent, simply say "resume project" and point it at
the project directory.

---

## Context Management

The `project/scripts/` tools manage `context.md`:

```bash
# Add an entry (agent does this after every interaction)
project/scripts/add-context \
  --agent "Claude.ai" \
  --model "claude-sonnet-4-5" \
  "Summary of what was done and what's next."

# Read last 2 entries (start of session)
project/scripts/read-context -n 2

# Read just headers
project/scripts/read-context --headers-only -n 5

# Rotate when file gets large (default: 1MB)
project/scripts/rotate-context
```

---

## Security Scanning

Every project has `scripts/security_scan.sh` which runs:

1. **Sensitive file check** — secrets.env, credentials.json, token.json, etc.
   Verifies each is gitignored.
2. **Pattern scan** — grep for API keys, tokens, passwords, private keys, IPs,
   database URLs across all source files.
3. **Git history check** — looks for sensitive filenames ever committed.
4. **Trufflehog** — deep entropy-based scan. Always available inside the
   devcontainer. Gracefully skipped on the host if not installed.

Run it:

```bash
./scripts/security_scan.sh

# Or via the full check suite:
./scripts/run_checks.sh
```

Exit codes: `0` = clean, `1` = issues found, `2` = script error.

---

## send 'er

`send 'er` is a phrase spoken to an AI agent that triggers the full
release gate:

1. Security scan on entire project
2. All tests must pass
3. Linter must be clean
4. Build (if applicable)
5. Summary shown, push confirmed
6. Push to origin
7. PR opened

At project creation time, `new-project` runs a trimmed version: security scan
only (no code yet), then commits and opens the initial PR.

---

## Updating the Template

When you improve the template (e.g. add a tool, fix a script, update the DOA):

```bash
cd ~/src/prjTemplate   # or wherever you keep it
# make changes
git add -A
git commit -m "Template: describe what changed"
git push
```

New projects created after the push will automatically get the improvements.
Existing projects do **not** auto-update — that's intentional. If you want
an existing project to get a template fix, cherry-pick or manually apply it.

---

## Updating the Devcontainer

If `postCreate.sh` or `devcontainer.json` changes in this repo:

```bash
cd ~/new-project
git add -A
git commit -m "DevContainer: describe what changed"
git push
```

To apply the change to an existing workspace:

```bash
# On the host:
devpod up <project-name> --recreate
```

This rebuilds the container. Your code (in `/workspaces/<project>`) is
preserved. Shell history and any manual changes to the container layer are
lost — which is fine, that's the point.

---

## Adding a New Stack

To add Go (or any other stack):

1. Create `devcontainers/go/devcontainer.json` in this repo
   - Set `"STACK": "go"` in `containerEnv`
2. Add a `go)` case to `postCreate.sh` that installs Go tools
3. Add `"go"` as option `3` in the `new-project` prompt
4. Commit and push this repo

---

## dh1 (Remote Docker / Stretch Goal)

`dh1` is an LXC on the Proxmox rack. To use it as a DevPod provider:

### Option A: Docker TCP (simpler)

On dh1, expose the Docker daemon over TCP (with TLS):

```bash
# On dh1
# Edit /etc/docker/daemon.json to add:
# { "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"] }
# Set up TLS certs (see Docker docs)
```

Then on your Mac:

```bash
devpod provider add docker
devpod context use default
# Set DOCKER_HOST in your environment or devpod provider options
```

### Option B: SSH Provider (recommended)

```bash
devpod provider add ssh
```

Then use `new-project --remote` which passes:

```
--provider ssh --provider-option HOST=dh1
```

dh1 must have Docker installed and your SSH key authorized.

### Why this works with no script changes

The `--remote` flag in `new-project` is the only switch needed. Everything
else — the devcontainer, postCreate, dotfiles — runs identically because
DevPod abstracts the provider away.

---

## Troubleshooting

### `gh: not authenticated`

```bash
gh auth login
# Choose: GitHub.com → SSH → browser auth
```

### `devpod: command not found`

```bash
brew install devpod
# or download from https://devpod.sh
```

### `devpod up` fails: Docker not running

```bash
open -a Docker   # start Docker Desktop on Mac
# wait for the whale to settle, then retry
```

### Container builds but `.zshrc` errors on startup

Your `.zshrc` expects `zoxide`, `fzf`, and `starship` to be installed.
`postCreate.sh` installs them, but if it failed partway through:

```bash
# Inside the container:
bash /workspaces/<project>/.devcontainer/postCreate.sh
```

### `add-context` not executable

```bash
chmod +x project/scripts/add-context \
         project/scripts/read-context \
         project/scripts/rotate-context
```

### Template substitution missed a file

`new-project` only substitutes in a known list of files. If you add a file to
`prjTemplate` that contains `[Project Name]`, add it to `SUBSTITUTION_FILES`
in the `new-project` script.

---

## Environment Variables

All configuration in `new-project` is overridable via environment variables.
Add these to `~/.zshrc.local` to change defaults:

```bash
# GitHub username (default: brentrockwood)
export NEW_PROJECT_GITHUB_USER="brentrockwood"

# Template repo to clone (default: github.com/brentrockwood/prjTemplate)
export NEW_PROJECT_TEMPLATE="github.com/brentrockwood/prjTemplate"

# Where new projects land on your Mac (default: ~/src)
export NEW_PROJECT_SRC_DIR="$HOME/src"
```

As always: when environments vary, use an environment variable.
