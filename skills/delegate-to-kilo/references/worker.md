# Worker reference

Details for `scripts/claude-kilo-worker` that the main flow in SKILL.md
does not need. Read only the section you need.

## Exit codes

- `0`: Kilo finished, no permission was denied, and the check (if any)
  passed.
- `2`: usage error, not inside a git repository, or a requested model that
  Kilo does not offer; nothing ran.
- `3`: Kilo finished, but a permission was denied or auto-rejected; the
  report lists which. The task may be incomplete.
- `4`: a read-only run changed the working tree; review `Worker changes` and
  tell the user.
- `5`: a file changed inside `KILO_WORKER_READ_DIRS`.
- `6`: the `KILO_WORKER_CHECK` command failed; its output tail is in the
  report.
- `124`: Kilo timed out; changes may be partial.

Kilo itself exits 0 even after denied permissions; the worker turns those
into `3`.

## Report

The worker prints Kilo's final answer (capped at 3000 characters, 12000
in read mode; `KILO_WORKER_ANSWER_CHARS` changes it), a tool count, failed
and denied tool calls, Kilo's token usage, the path of the full log, a
`Worker changes` section, and the `Session:` line.

`Worker changes` shows only what Kilo changed, not changes that were
already in the working tree: a diffstat, a `git diff <baseline>` command, new
untracked files, and already-untracked files Kilo changed (which
`git diff` does not show). When the whole diff, new files included, is at most
`KILO_WORKER_DIFF_LINES` lines (default 60; `0` disables it), it is
printed inline.

The check in `KILO_WORKER_CHECK` runs last, in the project directory, with
`KILO_WORKER_CHECK_TIMEOUT` seconds (default 300). Files it creates are not
reported as Kilo's changes. It does not run in read mode or after a
timeout.

Read the full log only for a specific question, with `grep` or `jq`, never
whole. `KILO_WORKER_VERBOSE=1` also prints the transcript, for debugging
the worker itself.

## Fixing after review

Send only the correction to the same Kilo session; it keeps the context of
its earlier work:

```bash
KILO_WORKER_SESSION=ses_... KILO_WORKER_CHECK='<same check>' bash <skill-base-dir>/scripts/claude-kilo-worker <<'TASK'
<only what must change, and why>
TASK
```

The resumed run keeps the session's model, and its report shows only that
round's changes. Small fixes are cheaper to make yourself.

Stop after two correction rounds that still fail review or the check: finish
the task yourself from where Kilo left it. A third round rarely converges
and costs more than the remaining work.

## Read-only analysis

For questions about the code, run with `KILO_WORKER_MODE=read`. Kilo
uses its `plan` agent with file edits denied and bash limited to commands that
only read (`git log`, `git show`, `git diff`, `git blame`, `git grep`,
`git status`, `git ls-files`, `ls`, `cat`, `head`, `tail`, `grep`, `wc`,
`sort`, `uniq`). It answers with findings instead of a diff.

The answer is a lead, not a fact: open the files and lines it cites for every
point that matters before acting on it. Exit `3` in read mode often means
Kilo tried a command outside the list and then answered another way; read
the answer before treating it as incomplete.

### Second opinion

Read mode also reviews your own work cheaply, because the answer is short: ask
Kilo to critique a plan file or `git diff <base>` and list concrete bugs,
missed cases and contradictions with the project's rules, each with file and
line. Use it after a large change of yours, before telling the user it is
done. Treat each point as a lead and verify it; drop the ones that do not
hold.

## Long tasks

Keep each task small enough to finish in a few minutes, and split larger work
into sequential delegations. The worker stops Kilo after 570 s, below the
600 s maximum of the Bash tool. For a task that cannot be split, run it with
`run_in_background: true` and `KILO_WORKER_TIMEOUT=1500`, and wait for the
completion notification.

## Files outside the repository

To let Kilo read docs in a sibling repository or a runbook, pass their
directories in `KILO_WORKER_READ_DIRS` (colon-separated absolute paths).
Kilo may read them, never edit them; the worker exits `5` if anything
there changes. Without it, Kilo cannot open them.

For a long task, write it once to a file in your scratchpad and pass it with
`< task.md`, so a run that fails before doing anything can be repeated without
writing the task again.

## What the worker blocks

Kilo's own permission system denies, for delegated runs only: git commit,
push, reset, clean, checkout, switch, restore, stash, rebase and merge;
`rm -rf`; any file access outside the project directory, except Kilo's own
scratch and truncated-output directories and `KILO_WORKER_READ_DIRS`; and
every pattern in `~/.config/claude-kilo-worker/deny` (one shell glob per
line, e.g. `*supabase db reset*`). The rules reach Kilo through
`KILO_CONFIG_CONTENT`.
