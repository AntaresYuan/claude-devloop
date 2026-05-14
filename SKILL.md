---
name: devloop
description: >-
  Run an autonomous, self-paced "dev loop" — work through a queue of issues/tasks on a git
  repo one at a time: branch → implement (small steps) → build & test hard → PR → independent
  sub-agent code review → merge → verify the deploy → update the issue → log it → next. Self-paced
  via ScheduleWakeup; keeps going unattended (e.g. overnight) until the queue is exhausted, then
  stops. Invoke when the user says things like "work through these issues", "run the loop", "knock
  out the backlog", "do all the doable issues" — especially if they want it to run while they're
  away. NOT for one-off single tasks (just do those directly).
iteration:
  enabled: true
  skill_repo: AntaresYuan/claude-devloop
  threshold: 3
  artifacts:
    friction_log: ./LOOP-FRICTION.jsonl
    run_log: ./LOOP-LOG.md
  state_file: ./.iteration-state.json
---

# devloop — autonomous dev loop

You run a queue of dev tasks autonomously and self-paced, with the user away. Each task lands as a
reviewed, merged, deploy-verified PR. You pace yourself with ScheduleWakeup and keep going until
the queue is done, then stop. You don't wait on the user — but you keep them informed (issue
comments, a `LOOP-LOG.md`, occasional status messages).

## 0. Kickoff — do this BEFORE touching anything

Confirm the project-specific bits with the user, concisely (one message / AskUserQuestion):

1. **The queue.** Which issues/tasks, in what order? Which are explicitly OUT — theirs to do,
   blocked on something only they can do (a 2FA OTP, a credential, an external account), or
   needing a decision you shouldn't make?
2. **Build & verify.** Build command (e.g. `node scripts/build.js`, `npm run build`)? Test suite
   (`npm test`)? If no test suite, "test" = build passes + idempotent + manual output checks +
   `curl` the deployed site. How is "deployed" verified (a live URL? a preview?) and roughly how
   long does a deploy take?
3. **Guardrails.** Anything you must NOT touch without showing a plain-text draft first (content
   files, copy, secrets, infra/deploy config, CMS config)? Any "always do X" conventions?
4. **Permissions** (so the loop runs unattended without prompting them every command): permission
   mode = **"Accept Edits"** (Shift+Tab cycles to it, or `/config`), PLUS a bash allowlist in the
   user-level settings file (`~/.claude/settings.json` — that's the project root when Claude Code
   was launched from the home dir; verify) under `permissions.allow`: `Bash(git:*)`, `Bash(gh:*)`,
   the build runner (`Bash(node:*)` / `Bash(npm:*)` / `Bash(npx:*)`), `Bash(curl:*)`, `Agent`,
   `WebFetch`, `WebSearch`, and read-only utils (`ls`/`cat`/`grep`/`rg`/`find`/`head`/`tail`/`wc`/
   `sed`/`awk`/`sort`/`uniq`/`diff`/`mkdir`/`touch`/`cp`/`mv`/`echo`/`printf`/`true`/`xxd`/`jq`/
   `pwd`); under `permissions.deny`: `Bash(rm:*)`, `Bash(sudo:*)`, `Bash(git push --force:*)`,
   `Bash(git push -f:*)`, `Bash(git push --force-with-lease:*)`, `Bash(git reset --hard:*)`,
   `Bash(git clean:*)`, `Bash(git rebase:*)`, `Bash(git filter-branch:*)`, `Bash(git update-ref:*)`,
   `Bash(gh repo delete:*)`, `Bash(gh secret:*)`, `Bash(gh auth:*)`. NOTE: that allowlist is global
   if it's in `~/.claude/settings.json` — say so. The nuclear alternative (`--dangerously-skip-permissions`)
   needs a CLI restart and loses the session — don't recommend it. Offer to write the allowlist
   yourself (the user approves the one settings-file edit) OR have them set it up; either way,
   confirm it actually works (run one harmless `git -C <repo> status` and check it didn't prompt).

If the queue / build / guardrails aren't clear, ASK — don't guess.

## 1. Command-pattern constraints — THIS is the thing that bites you

The harness only auto-approves bash commands that are SIMPLE and START WITH an allowlisted command:
- ✅ `git -C /abs/path/to/repo status -sb` · `gh pr create -R owner/repo --head feat/x --base main --title "..." --body-file /tmp/pr.md` · `node /abs/path/scripts/build.js` · `node -c /abs/path/scripts/x.js` · `curl -s URL -o /tmp/site.html` · `grep -c PATTERN /tmp/site.html`
- ❌ `cd /repo && git ...` — the `cd` prefix triggers an "untrusted hooks" warning AND the command no longer starts with `git`. Use `git -C <repo>` and absolute paths instead; never `cd`.
- ❌ `REPO=... && git ...` (starts with a var assignment) · `git ... && git ... && git ...` (compound `&&`) · `... | grep ...` (pipe) · `git commit -m "$(cat <<'EOF' ... EOF)"` (`$(...)` substitution).
- Multi-line commit messages: repeated `-m "..."` flags (one per paragraph). Add a `Co-Authored-By:` trailer via a final `-m` if the environment expects one.
- PR bodies / issue comments containing `## headings` (i.e. a newline followed by `#`): write the
  text to a temp file with the **Write tool**, then `gh pr create ... --body-file /tmp/pr.md` /
  `gh issue comment N -R owner/repo --body-file /tmp/c.md`. Inline `--body "...##..."` trips a gh
  prompt. Single-line `--body "..."` (no `#`) is fine.
- Verify deploy: `curl -s URL -o /tmp/site.html` then `grep PATTERN /tmp/site.html` — SEPARATE
  commands, no pipe.
- Append to a log file: `cat >> /abs/path/LOOP-LOG.md <<'EOF' ... EOF` — works (single command
  starting with `cat`).

## 2. The per-task loop

For each task in the queue, in order:

1. **`git -C <repo> log --oneline -3`** first — confirm the ACTUAL repo state. (If you're resuming
   from a ScheduleWakeup whose prompt has a stale "STATE:" claim — which happens — trust git, not
   the prompt.)
2. **Branch:** `git -C <repo> checkout main` → `git -C <repo> pull --ff-only origin main` →
   `git -C <repo> checkout -b feat/<short-name>` (use the repo's branch-naming convention if it
   has one — `feat/` / `fix/` / `perf/` / etc.).
3. **Implement** in small steps. Read the surrounding code first; match its conventions, comment
   density, naming, idioms.
4. **Build & test hard.** Run the build — it must pass AND be idempotent (run it twice; no diff on
   the second run). `node -c` (or the equivalent lint) any new/changed source files. Grep the built
   output for the markup/strings you expect. Run the test suite if there is one. Goal: "force no
   problems" — don't rush.
5. **Self-review the diff** (`git -C <repo> diff main...feat/<name>`) before committing.
6. **Commit** (`git -C <repo> add <files>` then `git -C <repo> commit -m "..."` — conventional-commit
   style; `-m` again for body paragraphs / the Co-Authored-By trailer).
7. **Push** the branch: `git -C <repo> push -u origin feat/<name>`.
8. **Open the PR:** write the body to `/tmp/pr.md` (Write tool), then
   `gh pr create -R owner/repo --head feat/<name> --base main --title "..." --body-file /tmp/pr.md`.
   Body: what changed, why, what was tested + results, anything out of scope / known follow-ups.
9. **Independent review:** spawn a fresh general-purpose sub-agent (`Agent` tool, `subagent_type:
   general-purpose`) to code-review the PR — instruct it to use ONLY Read/Grep/Glob (NO Bash, NO
   Edit — so it doesn't trigger permission prompts), give it context + the files to read, ask for:
   bugs/logic errors, build breakage, regressions to existing behaviour, a11y, edge cases, codebase
   consistency, console errors / leaked globals — ending with a one-line verdict
   ("OK to merge" / "OK to merge after fixing: …" / "Do not merge: …"). Post a brief summary of the
   review on the PR (single-line `--body` or `--body-file`).
10. **Address** anything serious it flags — NEW commits to the branch (never amend a pushed commit;
    force-push is denied). Cheap nits: fix them; the rest: note in the PR / LOOP-LOG.
11. **Merge:** `gh pr merge <N> -R owner/repo --squash --delete-branch`.
12. **Sync:** `git -C <repo> checkout main` → `git -C <repo> pull --ff-only origin main`.
13. **Verify the deploy.** ScheduleWakeup ~150s (or the deploy duration). Next turn:
    `curl -s <live-url> -o /tmp/site.html` ; `head -c 50 /tmp/site.html` (it's HTML) ;
    `grep -c '<marker for the feature you just shipped>' /tmp/site.html` (=1) plus a couple of
    unrelated markers (=1, regression check). If it's not live yet, ScheduleWakeup ~90s and retry
    once — don't assume failure on the first try.
14. **Update the issue:** `gh issue comment <N> -R owner/repo --body-file /tmp/c.md` (or single-line
    `--body "..."` if no `#` headings). Close the issue if it's fully done:
    `gh issue close <N> -R owner/repo`.
15. **Log it.** Append an entry to `LOOP-LOG.md` at the repo root (create it on the first task) —
    branch, what was done, what was tested + results, PR + commit, the review verdict, any follow-ups.
    Commit + push it to main (docs-only commit; won't trigger build/CI if your paths-filter excludes it).
16. **Next task.** ScheduleWakeup with a prompt that re-enters this loop. Short delay (~60s) when
    you're just continuing; ~150s when waiting on a deploy/CI.

## 3. Pacing

ScheduleWakeup is how you span turns. ~60s for "just keep working"; ~150s for "waiting on a deploy";
longer (1200s+) only if there's genuinely nothing to check. The harness re-invokes you when it fires.
If multiple wakeups queue up and a prompt's "STATE:" looks stale, ALWAYS `git -C <repo> log` at the
top of the turn and reason from the real state.

## 4. STOP conditions — pause, leave a note (in LOOP-LOG.md + a user message), don't guess

- A build won't pass and you can't fix it cleanly.
- A merge conflicts (don't force anything — leave it).
- The sub-agent reviewer flags something serious you can't resolve.
- A task needs a product/design decision that's the user's to make.
- A task is blocked on something only the user can do (2FA OTP, an external account, a credential, an asset they have to produce).
- You'd hit a guardrail (something you said you wouldn't touch without a draft).
- Permissions break — you start prompting the user repeatedly. Stop, explain why, let them fix it; don't keep firing prompts.

## 5. When the queue is exhausted

- Append a "Loop complete" section to `LOOP-LOG.md`: a short summary + all PR/commit links + what's
  still on the user (with the *why* for each). Commit + push.
- Hand off to `/iterate` for retrospective analysis (see §7 below). This runs BEFORE the final
  user message so the message can reference the RETRO file + any auto-opened PRs.
- Post a final user-facing message: what shipped (with PR numbers), what still needs them (and why),
  housekeeping (permission mode they can revert, the allowlist — note if it's global, any temp
  files / the LOOP-LOG that can be deleted), and pointers to the RETRO report + any iteration
  PRs `/iterate` opened.
- **Stop. Do not ScheduleWakeup again.**

## 6. Friction logging during the run

Throughout the loop, append one JSON line per friction event to `LOOP-FRICTION.jsonl` at the
working repo root. This feeds `/iterate` (§7). Init at start of loop:
`touch <repo>/LOOP-FRICTION.jsonl`.

**When to log (and the `type`):**

| When | `type` | Notes |
|---|---|---|
| A bash command hits a permission prompt | `permission_blocked` | log even if user approves inline |
| Same command blocked ≥2× in one run | `permission_thrash` | indicates real allowlist gap |
| A command had to retry (build failure, network, etc.) | `retry` | include `attempts` + `reason` |
| ScheduleWakeup undershot a deploy / CI wait | `pacing_mismatch` | include `scheduled_wait_s` vs `actual_ready_s` |
| Sub-agent reviewer rejected ≥2 cycles on one PR | `subagent_flag_persistent` | include cycle count |
| `git log` showed unexpected state vs a STATE: claim | `unexpected_state` | log assumption vs actual |
| A STOP condition fired (see §4) | `stop_condition_hit` | include which one |
| A `deny:` rule blocked a destructive command you didn't mean to fire | `safety_intentional` | **inverse of friction** — flags the guardrail working as designed; `/iterate` won't propose relaxing it |

**Format** (one line per event):

```jsonl
{"ts": "2026-05-13T16:42:11Z", "run_id": "<this-loop-id>", "task": "issue-42", "type": "pacing_mismatch", "details": {"step": "deploy_verify", "scheduled_wait_s": 150, "actual_ready_s": 245}, "lesson_hint": "Default 150s undershoots this repo's deploy"}
```

**How to append** (one of the allowed bash patterns — single command starting with `cat`,
no compound):

```
cat >> /abs/path/to/repo/LOOP-FRICTION.jsonl <<'EOF'
{"ts": "...", "run_id": "...", "task": "...", "type": "...", "details": {...}, "lesson_hint": "..."}
EOF
```

Generate `run_id` once at loop start (e.g. timestamp-based) and reuse for every event in this
run. Keep `lesson_hint` short and concrete — that's what /iterate's analyzer reads as a
human-authored hypothesis.

## 7. Iteration handoff (`/iterate`)

After §5 (Loop complete written, before the final user message), invoke `/iterate devloop`.

The `/iterate` meta-skill reads this skill's `iteration:` frontmatter to know where the
artifacts are, runs a retrospective sub-agent over `LOOP-FRICTION.jsonl` + `LOOP-LOG.md` +
recent git history, produces a `RETRO-<YYYY-MM-DD>.md` in the working repo, accumulates
friction counts in `.iteration-state.json` across runs, and — when a class crosses
`threshold` (default `3`) — opens a PR to `AntaresYuan/claude-devloop` with a proposed
`SKILL.md` diff backed by friction evidence.

**`/iterate` never auto-merges** and **never proposes weakening safety entries** (deny list,
hooks, force-push protections). PRs are for human review.

If `~/.claude/skills/iterate/` is not installed, skip the handoff silently — `LOOP-FRICTION.jsonl`
just keeps accumulating for the next time `/iterate` is available. Note the missing iteration
in the final user message so they can install it: `git clone
https://github.com/AntaresYuan/claude-skill-iterate ~/.claude/skills/iterate`.
