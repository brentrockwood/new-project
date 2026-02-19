# new-project

Personal project bootstrapping tool. One command to go from nothing to a
working devcontainer with your dotfiles, a GitHub repo, and an initial PR.

## Quick start

```bash
new-project
```

See [RUNBOOK.md](RUNBOOK.md) for complete documentation.

## Prerequisites

```bash
brew install gh git devpod
gh auth login
```

## Install

```bash
git clone git@github.com:brentrockwood/new-project.git ~/new-project
ln -s ~/new-project/new-project ~/bin/new-project
```

## What it does

Asks for a name and stack, then:

1. Clones `prjTemplate` → `~/src/<name>`
2. Creates a private GitHub repo
3. Runs a security scan and opens a PR
4. Stands up a DevPod workspace (local Docker or dh1 with `--remote`)

After ~5 minutes you get an SSH address, connect, and start working.

## Contents

```
new-project/
├── new-project                  ← the script (symlink into ~/bin)
├── devcontainers/
│   ├── python/devcontainer.json
│   └── typescript/devcontainer.json
├── postCreate.sh                ← runs inside container at creation
├── scripts/
│   └── security_scan.sh         ← template security scan (with Trufflehog)
└── RUNBOOK.md                   ← full documentation
```
