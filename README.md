# devloop — an autonomous dev-loop skill for Claude Code

`/devloop` turns Claude Code into an unattended worker that grinds through a queue of issues on a
git repo — one task at a time, properly: branch → implement in small steps → build & test hard →
open a PR → get an independent code review (a fresh sub-agent) → merge → verify the live deploy →
update the issue → log it → next. It paces itself with `ScheduleWakeup` and keeps going until the
queue is empty, then stops — so you can fire it off and walk away (e.g. overnight).

It's a single [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) — one `SKILL.md`.

## Install

Drop `SKILL.md` into a `devloop/` folder under your Claude Code skills directory:

```sh
mkdir -p ~/.claude/skills/devloop
curl -sL https://raw.githubusercontent.com/AntaresYuan/claude-devloop/main/SKILL.md \
  -o ~/.claude/skills/devloop/SKILL.md
```

(Per-project instead of global: put it in `<repo>/.claude/skills/devloop/SKILL.md`.)

Then in Claude Code: `/devloop`.

## Use it

```
/devloop
```

First it runs a short **kickoff** — it'll ask you the project-specific stuff: which issues (and in
what order, and which are *out* — yours, blocked, or a decision it shouldn't make), the build/verify
commands, your guardrails (what it must not touch without showing a draft first), and how to set
permissions so it can run unattended without prompting you on every command. Once that's settled it
just goes.

You'll see progress as it works: issue comments, a running `LOOP-LOG.md` it commits to the repo
root, the merged PRs (each with the sub-agent's review summary), and a final wrap-up message when
the queue's done.

## What's encoded in it

- **Kickoff checklist** — queue, build/verify, guardrails, permission setup. It won't start until
  those are clear; it asks rather than guesses.
- **The per-task loop** — branch → implement → `build` must pass *and* be idempotent → self-review
  the diff → commit → push → open PR (body written to a temp file, `--body-file`) → spawn a fresh
  general-purpose sub-agent to code-review it (Read/Grep/Glob only, so it doesn't trip your
  permission prompts) → address findings (new commits, never amend a pushed one) → squash-merge →
  pull → `ScheduleWakeup` for the deploy → `curl` the live URL + grep for the feature you shipped
  plus a couple of unrelated markers (regression check) → comment/close the issue → append to
  `LOOP-LOG.md` → next.
- **Self-pacing** — `ScheduleWakeup` (~60s to keep working, ~150s to wait on a deploy/CI). If a
  resumed turn's context looks stale, it re-checks `git log` and reasons from the real state.
- **STOP conditions** — it pauses and leaves a note (in `LOOP-LOG.md` + a message) rather than
  guessing, when: a build won't fix, a merge conflicts, the reviewer flags something serious, a
  task needs a product/design decision that's yours, a task is blocked on something only you can do
  (a 2FA OTP, a credential, an asset), it'd hit a guardrail, or permissions break.
- **The bash command-pattern constraints** that the harness imposes on auto-approved commands
  (simple, single, starts with an allow-listed command — no `cd && …`, no `VAR=… && …`, no
  `&&`/`;`/`|` chains, no `$(…)`; `git -C <repo>` + absolute paths; `--body-file` for PR/comment
  markdown). Baked in so the loop doesn't stall on permission prompts.

## Safety / caveats — read this

This skill exists to run **unattended with broad auto-approval**. That means:

- You're trusting Claude to commit, push, open PRs, and **merge to your default branch** (= deploy,
  if you have CI/CD) without you watching each step. Set up the permission allowlist and the
  guardrails deliberately; the kickoff walks you through it.
- It works on **branches → PRs → squash-merge**, leaving a reviewable trail; it does not
  force-push or rewrite history (those are in the recommended `deny` list).
- The independent sub-agent review is a real second pass, but it's still Claude reviewing Claude —
  treat the merged PRs as "reviewed enough to ship a personal project," not "reviewed by a senior
  human."
- It's only as safe as your guardrails. Tell it, in the kickoff, exactly what it must not touch
  without showing you a draft first (content/copy, secrets, infra/deploy config, …).
- Skim what it did afterward — the `LOOP-LOG.md` is the play-by-play, and the PRs have the diffs +
  review summaries.

## Self-iteration (opt-in)

`/devloop` is integrated with [`/iterate`](https://github.com/AntaresYuan/claude-skill-iterate),
a meta-skill that improves skills based on their own runs. While the loop runs, devloop appends
friction events (permission prompts, retries, pacing mismatches, persistent sub-agent
rejections, etc.) to a `LOOP-FRICTION.jsonl` file in the working repo. When the queue is
exhausted, devloop auto-hands-off to `/iterate`, which produces a `RETRO-<date>.md` report and
— when the same friction class has recurred ≥3 times across runs — opens a PR to this skill's
own repo with a proposed `SKILL.md` diff backed by the evidence.

`/iterate` **never auto-merges** and **never proposes weakening safety designs** (deny entries,
hook bypass, force-push). PRs land for human review.

To enable, install `/iterate`:
```bash
git clone https://github.com/AntaresYuan/claude-skill-iterate ~/.claude/skills/iterate
```
If `/iterate` is not installed, devloop runs unchanged; the friction log just accumulates for
the next time it is available.

## Where this came from

Built (and battle-tested) running an unattended dev loop on a real project — a personal site —
overnight: it shipped a batch of features, each as a reviewed-merged-deploy-verified PR, and the
friction it hit (mostly the bash command-pattern stuff above) is exactly what's now baked into the
skill so the next run is smoother.

## License

MIT — see [LICENSE](LICENSE).
