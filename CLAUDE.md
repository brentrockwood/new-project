# Claude Code Operating Instructions

This file is read automatically by Claude Code at the start of every session.
Follow all instructions here without being reminded.

## Session Startup (every session, no exceptions)

1. Read `project/project.md` — this is the write-locked implementation plan.
2. Run `project/scripts/read-context -n 2` — resume context from last session.
3. Verify you are on the correct feature branch. If not, create one before doing anything else.

## Development Operating Agreement

All work on this project is governed by `project/doa.md`. Read it now if you
have not already. It is authoritative. When in doubt, follow it.

## AI Session Logging (required submission artifact)

Every prompt received from the human must be logged to `AI_SESSION.md` using
the session script. Do this **before** beginning work on the prompt.

### Logging a prompt

```bash
project/scripts/add-session-entry \
  --agent "Claude Code" \
  --model "<your model version>" \
  --type prompt \
  --output AI_SESSION.md \
  "Exact text of the human's prompt, copied verbatim"
```

### Logging a summary after non-trivial work

After completing any meaningful unit of work, add a summary entry:

```bash
cat << 'EOF' | project/scripts/add-session-entry \
  --agent "Claude Code" \
  --model "<your model version>" \
  --type summary \
  --output AI_SESSION.md
What was done, what decisions were made, what files changed, what comes next.
Branch: <current-branch>
EOF
```

### Rules

- `AI_SESSION.md` is **append-only**. Never edit existing entries.
- Log the prompt **verbatim** — do not paraphrase or summarize the human's words.
- Include `AI_SESSION.md` in every commit alongside the work it documents.
- If a session ends abruptly, add a retroactive summary entry before the next commit.

## Go-Specific Rules

These apply to all Go projects and supplement the DOA.

- Always run `go test -race ./...` — never `go test ./...` alone.
- Run `gofmt -w .` or `goimports -w .` before every commit.
- All exported symbols require godoc comments.
- Use table-driven tests as the default pattern.
- Define interfaces at the consumer side, not the producer.
- Use `context.Context` as the first argument for any function that may block
  or call an external service.
- Wrap errors with `fmt.Errorf("component: %w", err)` for inspectable chains.
- No global mutable state. Inject dependencies.

## The `send 'er` Gate (Go)

When the human says "send 'er", execute in order:

1. `gosec ./...` — security scan. Report findings.
2. `go vet ./...` — must be clean.
3. `golangci-lint run` — must be clean.
4. `go test -race ./...` — all tests must pass.
5. `go build ./...` — must succeed.
6. Add final context entry via `project/scripts/add-context`.
7. Add final session summary via `project/scripts/add-session-entry --type summary`.
8. Show: branch name, files changed, commits to push.
9. Prompt: "Ready to push to origin? (y/n)" — wait for confirmation.
10. Push to origin. Open a pull request.
