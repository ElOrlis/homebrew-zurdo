# Zurdo

> A Rust CLI that drives LLM agents through a PRD's tasks in a loop — and **independently verifies** every acceptance criterion instead of trusting the agent's self-report.

This is the official Homebrew tap for Zurdo, and — since Zurdo is proprietary, closed-source software — also its public documentation home. The formula in this tap installs a licensed pre-built binary from the public [zurdo-dist](https://github.com/ElOrlis/zurdo-dist) releases; no toolchain required.

Zurdo is a hardened Rust rewrite of [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop). It keeps ralph-loop's pragmatic "shell out to a coding agent CLI" model and adds the missing accountability layer: a strict PRD grammar, machine-checkable acceptance criteria, deterministic per-task state, and crash-safe resume.

- **Provider-agnostic.** Works with the [`claude`](https://docs.claude.com/en/docs/claude-code/overview), [`codex`](https://github.com/openai/codex), and [`copilot`](https://github.com/github/gh-copilot) CLIs, via `std::process::Command` — no SDKs, no API calls.
- **No git automation.** Zurdo runs the agent against the working tree and verifies. Whether to branch, commit, or open a PR is your call.
- **State lives next to your repo, not next to the PRD.** Everything goes under `.zurdo/<slug>/` at the repo root, atomically written, resumable after Ctrl-C or a crash.

---

## Why

A coding agent that grades its own homework will eventually mark its own homework `passing`. The whole point of Zurdo is that the agent never gets to decide whether a task passed:

1. The PRD's acceptance criteria carry explicit, machine-checkable hints — `[shell:]`, `[http:]`, `[file-exists:]`, `[grep:]`, `[file-absent:]`, `[no-grep:]`, or `[manual]`.
2. After every agent iteration, **Zurdo** runs every hint against the working tree and decides pass/fail. Multiple hints on one criterion are AND'd.
3. A task is `passed` only when all automated hints pass. `[manual]` carries zero machine signal; it surfaces a human-review obligation in reports, nothing more.

If the agent claims success but `cargo test` exits non-zero, the task fails and the loop tries again — up to the per-task `Max-Attempts` budget.

---

## Install

### Homebrew (macOS and Linux)

```sh
brew install ElOrlis/zurdo/zurdo
```

This pulls a pre-built binary from the public [zurdo-dist](https://github.com/ElOrlis/zurdo-dist) releases. Shipped targets: macOS Apple Silicon, and Linux x86_64 / aarch64 via [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux). (Intel macOS is not pre-built yet.)

> **Upgrading from a pre-1.0 install?** Older versions were tapped with
> `brew tap ElOrlis/zurdo https://github.com/ElOrlis/zurdo` — pointing at the
> source repo, which is now private and no longer carries the formula. Re-tap
> once against this public tap:
>
> ```sh
> brew untap elorlis/zurdo
> brew install ElOrlis/zurdo/zurdo
> ```

### From a release tarball

```sh
# Replace VERSION and TARGET to taste. Available targets:
#   aarch64-apple-darwin
#   x86_64-unknown-linux-gnu · aarch64-unknown-linux-gnu
VERSION=1.1.3
TARGET=aarch64-apple-darwin
curl -fsSL "https://github.com/ElOrlis/zurdo-dist/releases/download/v${VERSION}/zurdo-v${VERSION}-${TARGET}.tar.gz" \
  | tar -xz -C /usr/local/bin zurdo
```

The release page also ships a `checksums.txt` and per-file `.sha256` siblings — verify before installing in CI.

### Prerequisites

Zurdo shells out to an agent CLI for the executor role. Install at least one:

| Provider  | CLI                                                              | Auth                                   |
| --------- | ---------------------------------------------------------------- | -------------------------------------- |
| Anthropic | [`claude`](https://docs.claude.com/en/docs/claude-code/overview) | `claude login` or `ANTHROPIC_API_KEY`  |
| OpenAI    | [`codex`](https://github.com/openai/codex)                       | `codex login` or `OPENAI_API_KEY`      |
| GitHub    | [`copilot`](https://github.com/github/gh-copilot)                | `copilot auth login` or `GITHUB_TOKEN` |

`zurdo init` writes a `.zurdo/config.toml` with all three providers wired; switching is a one-line edit.

---

## Quick start

```sh
# 1. From the root of the repo you want Zurdo to drive:
zurdo init                            # writes .zurdo/config.toml, installs bundled skills

# 2. Write a PRD:
cat > prds/hello.md <<'EOF'
# PRD: Hello

## Task: task-build — Write the greeter
**Effort**: low
**Depends-on**: []

### Description
Create `src/greeter.txt` containing the single line `hello world`.

### Acceptance Criteria
- [ ] greeter file exists [file-exists: src/greeter.txt]
- [ ] greeter says hello world [grep: hello world in src/greeter.txt]
EOF

# 3. Validate the grammar before paying for tokens:
zurdo validate prds/hello.md

# 4. (optional) Static + LLM analysis of the PRD itself:
zurdo --analyze prds/hello.md

# 5. Drive the loop:
zurdo run prds/hello.md

# 6. Inspect what happened:
zurdo report prds/hello.md           # JSON by default; pass --format md for markdown
zurdo state list                     # every .zurdo/<slug>/ at this repo root
```

A bare `zurdo <prd>` is sugar for `zurdo run <prd>`.

---

## What a run looks like

`zurdo run` emits a live narration on stdout: a startup banner, per-task headers, per-criterion pass/fail, then a final summary table.

```
═══════════════════════════════════════════════════════════
  Zurdo v1.1.3
  PRD:      prds/auth.md
  Slug:     auth-a1b2
  Executor: anthropic (effort_map: low=claude-haiku-4-5,
                                   medium=claude-sonnet-4-6,
                                   high=claude-opus-4-7)
  Tasks:    7 (4 pending, 2 passed, 1 blocked-by-dependency)
═══════════════════════════════════════════════════════════

─── task-auth: Set up authentication ─── effort=medium, deps=[]
  → iteration 1 of 5 (max-attempts=5, agent-timeout=30m)
  ⠋ task-auth iter 1 — claude-sonnet-4-6 — 12s — 4.1 KB stdout
  ✓ agent completed: exit=0, 47s, 18.4 KB stdout
                     tokens in=3,421 out=8,209 — $0.12 est.
  → running 5 criteria
    ✓ shell: cargo build (1.2s)
    ✓ shell: cargo test (8.4s)
    ✗ shell: cargo clippy -- -D warnings (3.1s)
      stderr tail: error: this loop never actually loops
  iteration 1: 4/5 criteria passed; will retry
  ...
  ✓ task-auth: passed in 2 iterations (8m 14s)

═══ Run Summary ═══
  task-id          status                  attempts   wall-clock   criteria
  ──────────────────────────────────────────────────────────────────────────
  task-auth        passed                  2/5        8m 14s       5/5
  task-login       passed-pending-review   1/5        3m 02s       4/4
  task-rate-limit  failed                  5/5        12m 41s      3/4
  ──────────────────────────────────────────────────────────────────────────
  totals           1 passed, 1 pending-review, 1 failed                24m 57s
  iterations       8 used (of unlimited)
  tokens & cost    in=142,308 out=384,902 — $0.34 est.
```

Glyph legend: `→` action, `✓` pass, `✗` fail, `⊘` skipped/manual, `⚠` warning. The progress stream is on stdout; a parallel JSONL event log lands at `.zurdo/<slug>/progress.log` for tooling. Tunables: `--no-progress` silences the stream, `--quiet-agent` drops the live tee of agent output (spinner stays), `--no-color` strips ANSI.

---

## Commands

| Subcommand                        | Purpose                                                                                                                                                                                                        |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `zurdo init`                      | Write a default `.zurdo/config.toml` and install bundled skills to the provider discovery path                                                                                                                 |
| `zurdo init --sync`               | Refresh bundled skill installs to the provider discovery path without rewriting config                                                                                                                         |
| `zurdo init --check-models`       | Probe every `effort_map` entry against its provider CLI and print a status table                                                                                                                               |
| `zurdo check-models`              | Probe the **existing** `.zurdo/config.toml`'s models without writing anything; adds a row per configured analyzer model. Exits `0` when all ok, `4` on any unknown/unsupported (matching the `run` pre-flight) |
| `zurdo validate <prd>`            | Deterministic grammar + dep-graph checks; no LLM, no execution                                                                                                                                                 |
| `zurdo run <prd>`                 | Drive the PRD through the agent loop                                                                                                                                                                           |
| `zurdo run <prd> --resume`        | Resume an interrupted run; no prompt                                                                                                                                                                           |
| `zurdo run <prd> --reset`         | Archive existing state under `.zurdo/<slug>/.archive/<ts>/` and start fresh                                                                                                                                    |
| `zurdo run <prd> --analyze`       | Full pre-flight (deterministic + LLM); never proceeds to execution                                                                                                                                             |
| `zurdo run <prd> --analyze --fix` | Iterative refinement loop; writes `<prd>.proposed.md` and asks before overwriting                                                                                                                              |
| `zurdo verify <prd>`              | Re-run every terminal task's criteria against the current working tree                                                                                                                                         |
| `zurdo report <prd>`              | Build a curated run report (`--format json` default, `--format md` supported)                                                                                                                                  |
| `zurdo state list`                | List every `.zurdo/<slug>/` at the repo root                                                                                                                                                                   |
| `zurdo state where <prd>`         | Print the absolute `.zurdo/<slug>/` path a PRD would resolve to                                                                                                                                                |
| `zurdo skills list`               | List bundled opinionated skills compiled into the binary                                                                                                                                                       |
| `zurdo skills install <name>`     | Install a bundled skill directly into the provider discovery path (`--all`, `--provider`, `--all-providers`)                                                                                                   |

Run `zurdo <subcommand> --help` for the full flag surface.

### Global flags

These flags apply across multiple subcommands. Subcommand-specific flags (`--analyze`, `--fix`, `--resume`, `--reset`, `--format`, `--check-models`) are in the commands table above.

| Flag                        | Effect                                                                                                    |
| --------------------------- | --------------------------------------------------------------------------------------------------------- |
| `--no-prompt`               | Suppress every interactive prompt (CI-safe). The resume prompt defaults to *Resume*.                      |
| `--max-iterations N`        | Cap total agent invocations across the run. Overrides `[defaults] max_total_iterations`. `0` = unlimited. |
| `--skip-model-check`        | Disable the pre-run model probe. For CI against fake CLIs or unverified models.                           |
| `--no-progress`             | Suppress the stdout progress stream (banner, per-task headers, summary).                                  |
| `--quiet-agent`             | On TTY, drop the live tee of agent stdout/stderr. Spinner and heartbeats remain.                          |
| `--no-color`                | Disable ANSI escapes in all output (stdout progress stream and stderr diagnostics).                       |
| `--log-file <path>`         | Tee the stderr diagnostic log to a file. Independent of `progress.log`.                                   |
| `--log-level <level>`       | One of `error`, `warn`, `info`, `debug`, `trace`. Default `info`.                                         |
| `-v` / `-q`                 | Aliases for `--log-level=debug` / `--log-level=warn`. Mutually exclusive with `--log-level`.              |
| `--log-format <text\|json>` | Diagnostic-log format. Default `text`. Does not affect the stdout progress stream.                        |

---

## PRD format (60-second tour)

```markdown
# PRD: <free-form title>

## Task: task-1 — <task title>
**Effort**: medium
**Depends-on**: []
**Max-Attempts**: 5         # optional, falls back to config default
**Skills**: rust-style       # optional, comma-separated
**Agent-timeout**: 30m       # optional, units required (s/m/h)
**Category**: Backend        # optional, used to group reports

### Requirements
- req-tests-green: The workspace test suite passes.
- req-health-endpoint: The service exposes a working /health endpoint.

### Description
Free-form prose. Passed verbatim to the executor agent.

### Acceptance Criteria
- [ ] cargo tests pass [shell: cargo test --workspace] [proves:req-tests-green]
- [ ] binary exists [file-exists: target/release/zurdo]
- [ ] no panics in main [grep: panic! in src/main.rs]
- [ ] /health returns 200 [http: GET http://localhost:8080/health -> 200] [proves:req-health-endpoint]
- [ ] design review signed off [manual]
```

The load-bearing surprises:

- The task heading separator between id and title is a real **em-dash** (U+2014 `—`), not `-` or `–`. macOS: `Option+Shift+-`. Linux Compose key: `Compose - - -`.
- The metadata block starts **immediately** after the `## Task:` heading — no blank line between them.
- `**Effort**` and `**Depends-on**` are required on every task; `Effort` values must be keys in the active executor's `effort_map` (see [Configuration](#configuration)).
- Every acceptance criterion is a `- [ ]` checkbox line and needs **at least one** hint; use `[manual]` if it's a human-review-only criterion.
- Section headings under a task are a closed enum: `### Requirements` (optional), `### Description`, `### Acceptance Criteria` — in that order, order enforced.
- `zurdo validate <prd>` checks all of this deterministically — run it before paying for tokens.

**Requirement traceability (optional).** The `### Requirements` section names what the task must achieve, one bullet per requirement: `- <req-id>: <free-form text>`, where `req-id` matches `^req-[a-z0-9-]+$` and is unique within the task. A criterion can then link back with a `[proves:<req-id>]` modifier after its hints. `[proves:]` is not a hint — it runs no check and never gates the criterion; it declares that when the criterion passes, the named requirement has evidence. Validation enforces the links both ways: a `[proves:]` referencing an undeclared `req-id` is an error, and a declared requirement no criterion proves is surfaced as an uncovered-requirement warning by `--analyze`. Multiple criteria may prove the same requirement.

The bundled `zurdo-prd-author` skill (see [Bundled skills](#bundled-skills)) teaches your agent CLI the full grammar and walks a PRD from intent to a `zurdo validate`-clean file.

---

## Verification hints

| Hint                                 | Behavior                                                                                           |
| ------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `[shell: <cmd>]`                     | Run command; pass iff exit 0. Working dir = repo root.                                             |
| `[http: <method> <url> -> <status>]` | Make request; pass iff response status matches.                                                    |
| `[file-exists: <path>]`              | Pass iff file exists at path (relative to repo root).                                              |
| `[grep: <pattern> in <file>]`        | Pass iff regex pattern is found in file.                                                           |
| `[file-absent: <path>]`              | Pass iff no file exists at path (relative to repo root).                                           |
| `[no-grep: <pattern> in <file>]`     | Pass iff the file is readable and the regex is **not** found; a missing/unreadable file **fails**. |
| `[manual]`                           | Out-of-band human review; never machine-checked.                                                   |

Multiple hints on one criterion are AND'd. `[manual]` mixed with automated hints means the automated portion still gates; the manual portion surfaces a review obligation in reports. A task with **only** `[manual]` criteria short-circuits at pre-flight to `passed-pending-review` and never invokes the agent.

Two authoring gotchas worth knowing up front:

- `[grep:]` / `[no-grep:]` must target a **file**, not a directory — a directory path always fails (instantly, which is the tell).
- A hint that can't fail proves nothing (`[shell: true]`, `[file-exists: README.md]`). `zurdo --analyze` flags many such no-ops as warnings.

---

## State directory layout

Per-PRD state lives at `.zurdo/<slug>/` under the **repo root** (never beside the PRD). The slug is deterministic: `<basename>-<sha1(repo-relative-path)[0..4]>`.

```
.zurdo/
├── config.toml                      # provider config, effort map, defaults
└── <slug>/
    ├── prd.json                     # terminal source of truth, atomic writes
    ├── progress.log                 # append-only JSONL event stream
    ├── lock                         # pid + ISO-8601 start time
    ├── iterations/
    │   ├── <task-id>-<attempt>.out
    │   ├── <task-id>-<attempt>.err
    │   └── <task-id>-<attempt>.prompt
    └── reports/
        └── <timestamp>.{json,md}
```

Add `.zurdo/` to your `.gitignore`. Zurdo prints a one-time hint if you forget — but it never modifies your `.gitignore` itself.

---

## Resume, locks, and recovery

`zurdo run` is crash-safe by design. Four mechanisms cooperate:

**The lock file.** While Zurdo is running it holds `.zurdo/<slug>/lock` (pid + ISO-8601 start time). A second `zurdo run` against the same PRD refuses with exit `3` and the offending pid. Stale locks (dead pid) are taken over automatically with a `warn`-level log — no manual cleanup needed.

**The interactive resume prompt.** When you run against an existing `.zurdo/<slug>/`, Zurdo asks:

```
What would you like to do?
  [R] Resume from current state           (default)
  [X] Reset — archive state and start over
  [A] Abort
```

`R` (Enter) continues; `X` archives state under `.zurdo/<slug>/.archive/<ts>/` and starts fresh; `A` exits 0. The prompt is skipped — and defaults to `R` — when stdin is not a TTY, or when any of `--resume`, `--reset`, or `--no-prompt` was passed.

**PRD-hash drift.** `prd.json` records the SHA-1 of the PRD at the last successful parse. If the live PRD now hashes differently, `zurdo run` refuses with exit `4` and asks for `--reset`. Editing a PRD mid-run is never silently absorbed — your old `prd.json` is archived, not overwritten.

**Ctrl-C.** First press triggers a graceful exit: the current iteration is allowed to finish, the summary table is rendered with partial state, and the next run resumes cleanly. Second press is a hard kill — the in-flight iteration is dropped at resume time via `progress.log` reconciliation, and `attempts` is not incremented for the dropped iteration.

---

## Configuration

`zurdo init` writes a default `.zurdo/config.toml`:

```toml
[roles.executor]
provider = "anthropic"            # or "codex" or "copilot"
# model is determined by [effort_map.<provider>] below

[roles.analyzer]                  # required for --analyze
provider = "anthropic"
model    = "claude-haiku-4-5"

[effort_map.anthropic]
low    = "claude-haiku-4-5"
medium = "claude-sonnet-4-6"
high   = "claude-opus-4-7"

[effort_map.codex]
low    = "gpt-5.5"
medium = "gpt-5.5"
high   = "gpt-5.5"

[effort_map.copilot]                # `auto` lets the Copilot CLI pick the model
low    = "auto"                      # concrete dotted ids (e.g. claude-sonnet-4.6)
medium = "auto"                      # are plan-gated; probe with `zurdo check-models`
high   = "auto"

[defaults]
max_attempts         = 5          # per-task fallback
max_total_iterations = 0          # 0 = unlimited

[timeouts]
criterion_seconds = 300           # shell + http only
agent_seconds     = 1800

[providers.anthropic]
cli        = "claude"
extra_args = []

[providers.codex]
cli        = "codex"
extra_args = []

[providers.copilot]
cli        = "copilot"
extra_args = []
```

`Effort` values in a PRD must be keys in the **active executor's** `effort_map`. The grammar does not hardcode `low|medium|high` — define whatever tiers you like, as long as PRD and config agree. `zurdo check-models` probes every configured model against its provider CLI before you commit to a run.

---

## Bundled skills

Zurdo ships a small set of bundled opinionated skills compiled into the binary. They teach an LLM the Zurdo-specific bits — PRD grammar, hint authoring, run-state interpretation — so the agent doesn't have to rediscover them per call. `zurdo skills list` prints what's available; `zurdo skills install <name>` installs one directly into the provider's discovery path (`.claude/skills/<name>/` for Anthropic, `.agents/skills/<name>/` for Codex/Copilot).

| Skill                 | Reach for it when …                                                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `zurdo-prd-author`    | Authoring a PRD end-to-end. Runs a grill-style interview from intent → tasks → grammar, then pressure-tests every acceptance criterion until its hint actually verifies. |
| `zurdo-hint-debugger` | A criterion failed and you need to know whether the hint is wrong or the code is. Correlates the hint with iteration logs and the tree.                                  |
| `zurdo-state-summary` | You want a human summary of `prd.json` + `progress.log` with a recommended next action (`--resume`, `--reset`, or fix-then-resume).                                      |

Install all bundled skills at once with `zurdo skills install --all`, or install into more than one provider in a single shot with `--provider <name>` (repeatable) or `--all-providers`. Both flags compose: `zurdo skills install --all --all-providers` installs every bundled skill into every configured provider. Bundled-skill installs go straight to the provider discovery path (idempotent via a `.zurdo-managed` sentinel file in each installed skill); to customize, edit the installed copy.

**PRD-referenced skills** (declared in task `**Skills**` metadata) are user-managed — install them to your provider's discovery path (`.claude/skills/` or `.agents/skills/` at project-level or globally), or point `[skills] search_paths` at custom directories. Zurdo verifies they exist at pre-flight (warn-only); you own the installation workflow.

---

## Exit codes

| Code | Meaning                                                                                   |
| ---- | ------------------------------------------------------------------------------------------ |
| `0`  | Success                                                                                    |
| `1`  | General failure (unhandled error)                                                          |
| `2`  | PRD parse / validation error                                                               |
| `3`  | Pre-flight failure (missing config, missing CLI on PATH, lock held)                        |
| `4`  | State mismatch requires `--reset`, or pre-flight model probe rejected the model            |
| `5`  | One or more tasks finished `failed` or `blocked-by-dependency`                             |
| `6`  | Iteration budget exhausted (`--max-iterations` or `--analyze --fix --max-iterations`)      |
| `7`  | `--analyze --fix` thrash detected (warning count non-decreasing across last 3 iterations)  |
| `8`  | `--analyze --fix` halted — LLM produced an unparseable PRD; last-good iteration preserved  |

CI wrappers can branch cleanly on `5` (real failure) vs `6` (budget) vs `7/8` (analyze-fix didn't converge).

---

## CI integration

The exit-code surface is designed for clean branching from a shell wrapper:

```sh
#!/usr/bin/env bash
set -euo pipefail

ec=0
zurdo run prds/feature.md \
    --no-prompt \
    --no-color \
    --max-iterations 50 \
    --log-format json --log-file zurdo.log \
  || ec=$?

case "$ec" in
  0)    echo "all tasks passed" ;;
  2)    echo "PRD grammar broken";   exit 1 ;;
  3|4)  echo "infra / pre-flight";   exit 1 ;;
  5)    echo "task failure";         exit 1 ;;  # real regression
  6)    echo "budget exhausted";     exit 1 ;;  # bump --max-iterations
  7|8)  echo "analyze-fix non-convergent"; exit 1 ;;
  *)    echo "unexpected ec=$ec";    exit 1 ;;
esac
```

Notes:

- `--no-prompt` makes the resume prompt non-interactive (defaults to *Resume*) so the run never blocks on stdin.
- `--no-color` keeps logs grep-friendly in CI's plain-text artifact viewer.
- `--log-format json` plus `--log-file` gives you a structured diagnostic log alongside the JSONL `progress.log`.
- Capture `.zurdo/<slug>/reports/*.json` and `.zurdo/<slug>/iterations/*` as build artifacts for post-mortems.
- Cap spend with `--max-iterations` (global) and per-task `Max-Attempts` (PRD metadata). Exit `6` distinguishes "budget hit" from `5` ("a task actually failed"), so dashboards can split flakes from regressions.

---

## Troubleshooting

| Symptom                                                            | Cause                                                                                | Fix                                                                                                          |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `lock held by pid <n>` (exit `3`)                                  | Another `zurdo run` is in flight against the same PRD.                              | Wait for it. If no zurdo is running, the lock is stale and the next run will take it over.                   |
| `state mismatch — pass --reset` (exit `4`)                         | The PRD has changed since the last successful run; `prd_hash` no longer matches.    | If the edits were intentional, run with `--reset` (old state archives under `.zurdo/<slug>/.archive/<ts>/`). |
| `unknown model anthropic:<x>` (exit `4`)                           | The model under `[effort_map.<provider>]` is unknown or unsupported under your auth. | Fix the entry (probe with `zurdo check-models`), or pass `--skip-model-check`.                               |
| `task heading uses hyphen-minus where em-dash required` (exit `2`) | The H2 task heading uses `-` or `–` instead of U+2014 `—`.                          | Insert a real em-dash. macOS: `Option+Shift+-`. Linux Compose key: `Compose - - -`.                          |
| `acceptance criterion has no hint` (exit `2`)                      | A `- [ ]` line lacks any hint.                                                      | Add at least one hint, or `[manual]` if it's a human-review-only criterion.                                  |
| Agent runs forever, no progress                                    | Agent is hung or slow.                                                              | Lower `Agent-timeout` in the task metadata or `[timeouts] agent_seconds` in config.                          |
| Criterion `passes` but proves nothing                              | Hint is too loose (`[shell: true]`, `[file-exists: README.md]`).                    | Tighten the hint. `zurdo --analyze` flags many such no-ops as warnings.                                      |
| `[grep:]` hint fails instantly (0ms)                               | The hint targets a directory; grep hints require a file path.                       | Point the hint at a specific file.                                                                           |
| `[Y/n]` prompts appear in CI logs                                  | Default mode is interactive when stdin happens to be a TTY.                         | Always pass `--no-prompt` in CI (plus `--resume` or `--reset` to declare intent explicitly).                 |
| `zurdo: command not found` after `brew install`                    | Homebrew bin directory not in PATH (Linux specifically).                            | `eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"` in your shell rc.                                   |

If your symptom isn't here, run with `-v` (`--log-level=debug`) and check `.zurdo/<slug>/progress.log` — every state transition is structured there.

---

## About this tap

The formula in [`Formula/zurdo.rb`](Formula/zurdo.rb) is **auto-generated** by the release workflow in the private source repo on every `v*.*.*` tag — it points at the matching [zurdo-dist](https://github.com/ElOrlis/zurdo-dist) release and carries per-target SHA256s. Don't edit it by hand; changes will be overwritten on the next release.

Release history for Zurdo itself is published as release notes on [zurdo-dist](https://github.com/ElOrlis/zurdo-dist/releases).

---

## License

Zurdo is proprietary, closed-source software — Copyright © 2026 El Orlis. All rights reserved. The binary is distributed under the Zurdo EULA shipped with each release. Zurdo is an independent reimplementation inspired by [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop) (MIT).

## Acknowledgements

Zurdo owes its loop shape and overall philosophy to Paul Lovvik's [ralph-loop](https://github.com/paullovvik/ralph-loop). The verification-first guarantee, strict PRD grammar, and crash-safe state machine are the new contributions.
