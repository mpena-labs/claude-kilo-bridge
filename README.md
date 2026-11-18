# Claude Code → Kilo Delegation Bridge

[![skills.sh](https://skills.sh/b/mpena-labs/claude-kilo-bridge)](https://skills.sh/mpena-labs/claude-kilo-bridge)

A small, version-controlled setup for delegating straightforward coding tasks from Claude Code to Kilo, with the main Opus session performing the final review.

## What this project does

When Claude Code is running with Opus or Fable, the `delegate-to-kilo` skill can be selected automatically for simple, mechanical, repetitive, or well-defined coding tasks. The skill invokes `claude-kilo-worker`, a script bundled inside the skill folder, which runs Kilo in non-interactive mode. After the worker finishes, the main Opus session reviews the diff and validation results before accepting the work as complete.

When Claude Code is running with Sonnet, the skill explicitly tells the model not to use this workflow. This behavior was validated manually in Claude Code.

The setup does **not** disable or replace Claude Code's normal subagents. Explore, Plan, general-purpose, and user-defined subagents remain available.

## Architecture

```text
Claude Code / Opus
        |
        | automatic skill selection
        v
 delegate-to-kilo
        |
        v
 claude-kilo-worker
        |
        v
    kilo run
        |
        v
 Kilo code agent
        |
        v
 configured Kilo model
        |
        v
  implementation result
        |
        v
Main Opus session reviews diff + tests
        |
        v
      accepted
```

The worker runs Kilo CLI (`kilo run`), validated with Kilo `7.8.27`.

## Files

- `skills/delegate-to-kilo/SKILL.md`: Claude Code skill definition, including the Opus-only delegation rule and mandatory main-session review.
- `skills/delegate-to-kilo/scripts/claude-kilo-worker`: wrapper that launches Kilo with a timeout, deny rules and a report of what changed.
- `skills/delegate-to-kilo/references/review-checklist.md`: what the main session checks in every delegated diff.
- `skills/delegate-to-kilo/references/worker.md`: exit codes, report, read mode, session resume and other worker details, loaded only when needed.
- `commands/delegate-kilo.md`: the `/delegate-kilo` command, the same flow as the skill without the turn to load it.
- `bench/`: tasks and scripts that measure the Opus cost of delegating.
- `install.sh`: installs the skill folder into `~/.claude/skills/` and the command into `~/.claude/commands/`.
- `uninstall.sh`: removes the installed files while creating a timestamped backup first.
- `tests/test-install.sh`: verifies the installed files match the repository and checks basic metadata.
- `tests/test-worker.sh`: exercises the worker against a stub `kilo`, without calling any model.
- `.github/workflows/ci.yml`: runs ShellCheck, the worker tests and an install/uninstall cycle in a temporary `HOME`.
- `VERSION`: project/configuration version.

## Installation

With the [skills CLI](https://skills.sh):

```bash
npx skills add mpena-labs/claude-kilo-bridge -g -a claude-code
```

Or clone this repository and run:

```bash
./install.sh
```

Both place:

```text
~/.claude/skills/delegate-to-kilo/SKILL.md
~/.claude/skills/delegate-to-kilo/scripts/claude-kilo-worker
~/.claude/skills/delegate-to-kilo/references/review-checklist.md
~/.claude/skills/delegate-to-kilo/references/worker.md
~/.claude/commands/delegate-kilo.md
```

`install.sh` creates a timestamped backup if a target file already exists, and removes the `~/.local/bin/claude-kilo-worker` left by versions before 1.3.0 (after backing it up).

## Requirements

- Claude Code with support for Skills.
- Kilo CLI installed and available as `kilo` in `PATH`.
- Working Kilo `code` and `plan` agents (`plan` is used by read-only runs).
- The project being changed must be a git repository. The worker refuses to run elsewhere, because without git there is no diff to review.

### Choosing the model

The worker does not pick a model: Kilo uses its own default, so you change the delegated model where you already choose models, in Kilo (or pin one with `"model"` in `kilo.jsonc`). The worker prints the model Kilo used last so you can see it.

When you ask Claude for a specific model, the skill passes it as `KILO_WORKER_MODEL`:

```bash
export KILO_WORKER_MODEL='provider/model'
```

Before dispatching, the worker checks that model against `kilo models` and refuses it (exit 2) if Kilo no longer offers it, which happens often with free models. The skill tells Claude never to choose a model on its own, since most entries in `kilo models` are billed per token. A resumed session keeps its model unless `KILO_WORKER_MODEL` is set.

### Timeout

The worker stops Kilo after 570 seconds and exits with code 124, staying below the 600-second maximum of Claude Code's Bash tool. Override it with:

```bash
export KILO_WORKER_TIMEOUT=300
```

For a long task that cannot be split, the skill tells Claude to run the worker in the background with a higher `KILO_WORKER_TIMEOUT`, where the Bash tool limit does not apply.

### Extra deny rules

Commands that must never run in a delegated session of your projects (database resets, production deploys) go in `~/.config/claude-kilo-worker/deny`, one shell glob per line. Blank lines and lines starting with `#` are ignored. Another path can be set with `KILO_WORKER_DENY_FILE`.

```text
# Database and production deploys stay with the main session
*supabase db reset*
*supabase db push*
*vercel*--prod*
```

### Fixing after review

Each run ends with a `Session:` line. When the review finds something to fix, the skill tells Claude to send only the correction to that same Kilo session, which keeps the context of its earlier work:

```bash
KILO_WORKER_SESSION=ses_... bash ~/.claude/skills/delegate-to-kilo/scripts/claude-kilo-worker <<'TASK'
Also handle the empty-list case in parse_items.
TASK
```

The worker finds the session id by giving each new run a unique title, and pins it with `--session` rather than `--continue`, which could resume an unrelated interactive session in the same directory.

### Read-only analysis

For questions about the code (every call site of a function, how a module computes something, which routes skip a check), the skill runs the worker with `KILO_WORKER_MODE=read`:

```bash
KILO_WORKER_MODE=read bash ~/.claude/skills/delegate-to-kilo/scripts/claude-kilo-worker <<'TASK'
Goal: list every caller of create_token and what each one passes.
Done when: each caller is cited with file:line.
TASK
```

Kilo then runs its `plan` agent with file edits denied and bash limited to commands that only read: `git log`, `show`, `diff`, `blame`, `grep`, `status` and `ls-files`, plus `ls`, `cat`, `head`, `tail`, `grep`, `wc`, `sort` and `uniq`. Bash is not denied outright because Kilo's free tier rejects requests without the bash tool. An allowed command could still write through a redirect, so after the run the worker checks the working tree against its snapshot and exits with code 4 if anything changed. Commands in pipes are checked one by one, so `git log | head` passes and `git log | xargs rm` does not. The skill tells Claude to treat the answer as a lead and to open the cited lines before acting on it.

### Review checklist

`references/review-checklist.md` lists what the main session checks before accepting delegated work: edits that weaken existing tests, re-running the checks itself, scope creep and shortfall, and failure patterns a green test run does not show (hardcoded success, swallowed errors, APIs that do not exist in the installed version, dead code). It is adapted from the review guide of [amelnagdy/delegate-skills](https://github.com/amelnagdy/delegate-skills) (MIT).

### Exit codes

| Code | Meaning |
|---|---|
| 0 | Kilo finished and no permission was denied |
| 2 | Usage error, not inside a git repository, or a requested model that Kilo does not offer; nothing ran |
| 3 | Kilo finished, but a permission was denied or auto-rejected; the task may be incomplete |
| 4 | A read-only run changed the working tree |
| 5 | A file changed inside a directory shared through `KILO_WORKER_READ_DIRS` |
| 6 | The check command in `KILO_WORKER_CHECK` failed |
| 124 | Timeout; changes may be partial |
| 127 | `kilo` not found |

### Short report, full log on disk

The point of delegating is to make the Claude session cost less, and Claude pays for every line the worker prints. The worker runs Kilo with `--format json`, keeps the full transcript and stderr in `~/.local/state/claude-kilo-worker/` (deleted after 7 days; `KILO_WORKER_LOG_DIR` changes the place) and prints only:

- Kilo's final answer, capped at 3000 characters (12000 in read mode; `KILO_WORKER_ANSWER_CHARS` changes it);
- a one-line count of the tools it used, and the failed and denied tool calls;
- Kilo's token usage;
- the path of the full log.

A typical edit run now reports in under 1000 characters, where the old streamed transcript ran to tens of thousands. `KILO_WORKER_VERBOSE=1` adds the transcript back, for debugging the worker.

### One turn instead of three

Every extra turn re-reads the whole Claude context, which is what made delegating cost more than writing the code. The worker therefore also does the two steps Claude used to take separate turns for:

- **Inline diff.** When the whole diff, new files included, is at most `KILO_WORKER_DIFF_LINES` lines (default 60; `0` disables it), it is printed after `Worker changes`.
- **Check command.** `KILO_WORKER_CHECK='python3 -m unittest'` runs after Kilo finishes, in the project directory, with `KILO_WORKER_CHECK_TIMEOUT` seconds (default 300). The worker prints whether it passed and the tail of its output, and exits 6 when it failed. It runs after the change report, so files it creates are not counted as Kilo's changes. It is skipped in read mode and after a timeout.

The skill keeps only the main flow in `SKILL.md` (about 4.8 KB) and moves exit codes, read mode, session resume and the other details to `references/worker.md`, read only when needed, because the skill text is re-read on every turn after it loads.

### Measured cost

Fresh `claude -p` Opus 5.5 sessions on the same task, measured when the worker still drove OpenCode 2.0.31 with `muse-spark-1.3-contributor-free` (not repeated with Kilo):

| Task | Opus alone | Delegated | Quality |
|---|---|---|---|
| Write a unittest suite for a 10-function module | $0.42 to $0.48 (6 runs) | Skill: $0.31 (2 runs), about 30% less. `/delegate-kilo`: $0.30 to $0.32 (2 runs), about 35% less | Opus: 68 to 91 tests, 10/10 planted bugs caught. `/delegate-kilo`, with the edge cases named in `Done when`: 90 to 98 tests, 9 to 10/10. Skill before that change: 49 to 56 tests, 8 to 9/10 |
| Replace `print` with `logging` in 12 modules | $0.35 to $0.38 (4 runs) | $0.35 (2 runs) | Both correct; Opus solved it with a short script, so there was nothing to save |

About $0.25 to $0.30 of each session is the fixed start-up context (system prompt, skill list, `CLAUDE.md`), which delegating cannot reduce. In these runs Opus did not pick the skill on its own; it delegated when the prompt asked for the skill or came through `/delegate-kilo`. `bench/` holds the tasks and scripts to repeat these measurements.

### `/delegate-kilo`

`/delegate-kilo <task>` (installed to `~/.claude/commands/delegate-kilo.md`) carries the same flow as the skill in a few lines. A command is expanded into your message, so it also saves the turn Claude spends loading the skill, and you decide when to delegate instead of waiting for Claude to pick the skill.

### Files outside the project

Kilo cannot open files outside the project, so a task about a doc in a sibling repository used to paste the doc in, or Kilo guessed. `KILO_WORKER_READ_DIRS` takes colon-separated absolute directories that Kilo may read; the prompt tells it never to edit them, and the worker exits 5 if any file in them changed during the run.

### Report of what changed

Before running, the worker snapshots the working tree with `git stash create`, which does not touch the tree or the stash list. When Kilo finishes it prints a `Worker changes` section with a diffstat against that snapshot and a ready `git diff <snapshot>` command, so changes you already had are not mixed with the worker's. It also lists new untracked files and files that were already untracked and that Kilo changed, which `git diff` does not show.

## Tests

```bash
./tests/test-worker.sh    # worker behavior, no Kilo or model needed
./tests/test-install.sh   # installed files match this repository
```

CI runs both, plus ShellCheck and an install/uninstall cycle in a temporary `HOME`.

## Manual verification

### Verify the worker directly

From a git project directory:

```bash
bash ~/.claude/skills/delegate-to-kilo/scripts/claude-kilo-worker <<'TASK'
Create a file named kilo-delegation-test.txt containing exactly two lines: DELEGATION_TEST and Executed by an external worker.
TASK
```

The task can also be passed as arguments, but stdin through a quoted heredoc keeps `$`, backticks and quotes literal. The skill tells Claude to use the heredoc form.

### Verify automatic delegation with Opus

Use a simple task and do **not** explicitly invoke the skill. For example:

```text
Create a file named kilo-delegation-test.txt in the current directory containing exactly these two lines:

DELEGATION_TEST
Executed by an external worker

The task is deliberately simple. After completing it, verify the file and tell me how the task was executed.
```

Evidence of actual delegation should include something like:

```text
Skill(delegate-to-kilo)
Bash(bash ~/.claude/skills/delegate-to-kilo/scripts/claude-kilo-worker ...)
=== Kilo Worker ===
Kilo: ...
```

### Verify the Opus review step

For a delegated task, the main Opus session should continue after Kilo
finishes and inspect the resulting diff and relevant tests. Evidence should
show that the Kilo command completed first, followed by Claude Code
reading/reviewing the changed files and running validation as appropriate.

### Verify non-delegation with Sonnet

Repeat the same task in a Sonnet session. The expected behavior is that Claude Code uses its own file-writing tool instead of invoking the Kilo wrapper.

## Safety boundaries

The skill tells Claude not to delegate:

- architecture decisions
- ambiguous requirements
- security-sensitive decisions
- difficult debugging that needs substantial reasoning
- work where the correct implementation approach is unclear

The worker blocks destructive operations in two layers, both scoped to delegated runs only:

1. **Permission rules enforced by Kilo.** The worker sets `KILO_CONFIG_CONTENT` with `deny` rules for `git commit`, `push`, `reset`, `clean`, `checkout`, `switch`, `restore`, `stash`, `rebase` and `merge`, for `rm -rf`, for file access outside the project (`external_directory`), and for every pattern in your deny file. Patterns start with `*` because command-rewriting plugins such as rtk turn `git commit` into `rtk git commit` before permissions are evaluated. Your regular Kilo sessions are not affected. If `KILO_CONFIG_CONTENT` is already set, the worker keeps yours and warns that its rules are not applied.
2. **Prompt instructions.** The task is prefixed with an instruction to leave changes uncommitted, stay inside the project directory and avoid the denied commands, so the model does not waste turns on them.

This is a guardrail, not a sandbox. A pattern-based rule does not catch forms like `git -C . commit` or a script that calls git, and the `code` agent can still edit any file in the project. The main session's diff review remains the real safety check.

### Kilo non-interactive behavior

Verified with Kilo 2.0.31 (and 1.18.35 where noted):

- The worker passes `--dir` with the project path and sets `KILO_CONFIG_CONTENT`, which Kilo merges into its config (checked with `kilo debug config`). Kilo has no `--standalone` flag and needs none.
- Any permission that resolves to `ask` is auto-rejected in `kilo run` (`permission requested: ...; auto-rejecting`). The run does not hang waiting for approval.
- A command matching a `deny` rule fails with `Permission denied: <tool>` or `The user has specified a rule which prevents you from using this specific tool call`.
- Kilo still exits with code 0 in both cases. The worker looks for these messages and exits with code 3 instead.
- Kilo reads the project's `CLAUDE.md`, but not Claude Code's memory directory, so the skill asks Claude to copy relevant project rules into the task.
- `--auto` would approve every permission that is not explicitly denied. The worker does not use it.

## Important behavior

The Opus/Sonnet distinction is a model instruction inside the skill, not a hard programmatic router. The validated behavior is:

```text
Opus   → may automatically load the skill for suitable tasks
Sonnet → told not to use the skill
```

This repository does not implement ACP. Kilo is used through `kilo run`, because the current setup does not require an ACP bridge.

## Restoring the setup later

```bash
./install.sh
```

The installer will back up an existing target before replacing it.

## Uninstall

```bash
./uninstall.sh
```

The uninstall script backs up the current installed files before removing them.

## Security

This repository must not contain:
- API keys
- tokens
- credentials
- personal Claude Code config files
- `.env` files
- conversation history

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
