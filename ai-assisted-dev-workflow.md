# Working Smart with AI Coding Agents (JetBrains: Junie / Codex / Claude)

A practical guide to getting good results from AI coding agents without burning tokens.

---

## 1. The Core Problem

Token cost scales with **accumulated context**, not task complexity. One long chat that keeps re-sending its full history on every turn gets expensive fast — even if each individual task is simple. The fix isn't "use a cheaper model," it's **controlling how much context each task actually needs.**

---

## 2. Strategy: Isolate Tasks, Don't Bundle Them

Instead of one giant prompt ("build me the whole project"), break work into isolated, well-defined tasks, each in its own fresh chat:

1. Scaffold the project structure
2. Add REST endpoints
3. Add SQS listeners
4. Set up CI/CD
5. Add tests

**Why this works:** each chat starts with minimal context instead of dragging along everything from before. You pay for *this task's* context, not the whole project history.

---

## 3. Which Agent for Which Task

The key question is: **is the "what" already defined, or does it need to be figured out?**

| Task type | Agent | Why |
|---|---|---|
| Architecture decisions, tradeoffs, tricky reasoning (e.g. retry/failure semantics, concurrency design) | **Claude** (Sonnet/Opus) | Best reasoning quality — worth the cost for decisions, not code volume |
| Initial scaffolding (folder structure, boilerplate, `go.mod`, handler stubs) | **Junie or Codex** | Pattern-heavy, well-defined generation — doesn't need top-tier reasoning |
| Multi-file autonomous edits on an existing codebase (new listener matching existing pattern, refactors) | **Junie** | Repo-aware, edits files directly, iterates (edit → test → fix) without you re-supplying context each turn |
| CI/CD YAML, Dockerfiles, config | **Codex or Junie** | Highly templated |
| Quick debugging in an existing session | Whatever's already loaded | Switching models mid-debug just re-burns tokens re-explaining the problem |

**Rule of thumb:** If you could hand the task to a junior dev as a checklist without further input, it's Junie/Codex. If writing that checklist *is* the hard part, it's a Claude task.

**Important:** switching models *within* one long chat does not save tokens — the accumulated context is the same regardless of which model reads it. The savings come from starting fresh, not from which model you pick.

---

## 4. How to Provide Context (Without Guessing)

Fear of "not providing enough context" usually leads to dumping the whole repo into every prompt — that's expensive and often unnecessary, since Junie can already explore the project (file structure, search) on its own.

Scope context to **blast radius**, not to completeness:

- **Task touches specific files only** → reference just those files/folders. Cheaper, and prevents the agent from touching unrelated code.
- **Task needs to fit into an existing pattern** → point it at the specific existing file to mirror ("add a listener like `internal/consumer/orders.go`"), rather than the whole repo.
- **Unsure what's relevant** → ask a cheap exploratory question first ("what files handle SQS consumer registration?"), then scope the real task based on the answer.

Use `.aiignore` to hard-block sensitive files/directories the agent should never read or touch.

---

## 5. Two Files That Do Most of the Work

### `AGENTS.md` — standing rules (read automatically, every task)

- Open, vendor-neutral standard (maintained by the Linux Foundation), supported by Junie, Codex, Claude Code, Cursor, and others.
- Plain Markdown, **no required schema** — just headers and lists, kept short.
- Lives at the project root, version-controlled.
- Gets injected into *every* task automatically — so keep it lean; bloat here has a permanent token cost.

**Example:**

```markdown
# Project Overview
Go backend service exposing REST endpoints and consuming from SQS queues.

# Setup
- Go 1.22+
- Run `docker-compose up` to start local dependencies (LocalStack for SQS)
- Copy `.env.example` to `.env`

# Build & Test
- Build: `go build ./...`
- Run tests: `go test ./...`
- Lint: `golangci-lint run`

# Conventions
- Package layout follows the `cmd/` + `internal/` pattern
- All SQS consumers live under `internal/consumer/`, one file per queue
- Error handling: wrap errors with `fmt.Errorf("%w", err)`, never panic in request handlers
- HTTP handlers return domain errors; translation to status codes happens in middleware only

# Security / Boundaries
- Never commit `.env` or AWS credentials
- Do not modify Terraform files under `infra/` without explicit instruction

# Gotchas
- The `orders` queue has a 30s visibility timeout — long-running handlers must extend it manually
```

### `DECISIONS.md` — historical rationale (NOT read automatically by default)

- A log of *why* the project was built a certain way — not conventions, a record of reasoning and tradeoffs.
- Only relevant when someone (human or agent) is about to touch that area and needs the "why," not the "how."

**Example entries:**

```markdown
## 2026-09-05 — Failed queue consumption always deletes the message
We ack-and-drop on consumer failure rather than relying on SQS redrive.
Reason: duplicate side effects (double-charging downstream API) are worse
than losing a message. Failures are logged to CloudWatch for manual replay
instead of automatic retry.

## 2026-08-20 — SNS fan-out instead of direct multi-queue publish
Chosen over publishing to each SQS queue individually from the producer,
so adding a new consumer doesn't require producer code changes.
Tradeoff: slightly higher latency, accepted since none of our consumers
are latency-sensitive.
```

---

## 6. Making Decision-Logging Automatic

`DECISIONS.md` isn't read automatically on its own — but you can make agents maintain it automatically by pointing to it from `AGENTS.md`, which *is* always read. Add a dedicated, precisely-scoped section:

```markdown
# Decision Logging
When you make or are told to make an architectural or behavioral decision
that isn't obvious from the code itself (e.g. how errors are handled,
which AWS pattern was chosen and why, a tradeoff between two approaches),
append an entry to DECISIONS.md in this format:

## YYYY-MM-DD — <short title>
<what was decided, and the reasoning/tradeoff>

Do not log routine implementation work (adding a field, fixing a typo,
following an existing pattern) — only decisions a future developer would
otherwise have to guess or ask about.
```

Two things matter here:
- **Name the target file explicitly** ("append to `DECISIONS.md`") — otherwise an agent might misread "document decisions in this file" as meaning `AGENTS.md` itself, which should stay short and stable.
- **Scope what counts as a decision** — without this, agents may log trivial changes and pollute the file (and waste tokens every time it's read).

**Tradeoff to be aware of:** referencing `DECISIONS.md` from `AGENTS.md` means it's read on *every* task (more consistent, slightly higher per-task cost). The alternative — only reading it on demand when explicitly asked — is cheaper but relies on you remembering to point to it when relevant.

---

## 7. Getting-Started Sequence for a New Project

1. Think through the architecture and key decisions yourself first (don't let an agent "discover" the shape of the project by accident).
2. Write those decisions into `DECISIONS.md` as a starting point.
3. Write `AGENTS.md` with project layout, conventions, setup/build/test commands, and the decision-logging instruction above.
4. Scaffold the project with Junie/Codex, scoped to the initial structure.
5. Build out features task-by-task, each in its own chat, scoping context to what each task actually touches.
6. Bring in Claude specifically when a task requires resolving a real tradeoff — then capture the outcome in `DECISIONS.md` before handing the implementation to Junie/Codex.

---

## 8. Token-Saving Checklist

- ✅ One well-defined task per chat — don't bundle unrelated work
- ✅ Reference specific files/folders instead of the whole repo when scope allows
- ✅ Keep `AGENTS.md` short — it's injected into every single task
- ✅ Use `DECISIONS.md` for rationale, referenced from `AGENTS.md` only if you want it auto-loaded everywhere
- ✅ Use Claude for reasoning/tradeoffs, Junie/Codex for defined, mechanical work
- ❌ Don't switch models mid-chat expecting savings — the accumulated context cost is the same
- ❌ Don't dump the entire codebase into a prompt "just in case" — scope to blast radius
