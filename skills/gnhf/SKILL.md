---
name: gnhf
description: Use when the user asks to run GNHF, says they are going to sleep or leaving and wants an agent-managed coding run, asks to supervise, steer, or review an active GNHF run, or gives feedback on GNHF results.
---

# GNHF

gnhf is an agent orchestrator: it invokes a coding agent in a loop — one fresh,
**stateless** invocation per iteration, ending in exactly one commit on success or a
hard rollback on failure — until a natural-language stop condition is met. Three facts
shape everything below:

- **gnhf performs the commits, the worker judges the stop condition, and `notes.md` is
  the only cross-iteration memory — and it is lossy.** Every iteration re-reads the
  prompt; nothing else survives reliably. This is why the prompt is the product.
- **Statelessness is paid for in input tokens**: empirically ~500K tokens/iteration at
  a small-repo size, 98% input, growing with the codebase. `--max-tokens` is best read
  as a work-timer (~12M ≈ one hour), not a task count.
- **gnhf has no quota awareness, no rate-limit handling, and no agent fallback**
  (verified against source). All limit strategy is the host's.

The stop layers, in intended order: `--stop-when` is the real ending → 3 consecutive
failures catches stuck runs → `--max-iterations` catches trivial non-progress → the
token cap is the spend backstop (the only one that kills mid-iteration). Caps prevent
runaway *spend*, not productive-looking *drift* — drift is contained by prompt
tightness and by landing on a review-before-merge branch, never by a flag.

Full incident evidence, when it exists on this machine (proceed on this file alone if
not): `~/projects/docs/gnhf-guide.md` (operational failures, minutes 2026-07-29) and
`~/projects/standup/docs/planning/gnhf-runs/run-01-retrospective.md` (the
defect-gradient study).

## Mode

Pick one per run.

- **Hands-Off** — bounded task, clear verification, user leaving. Prepare the prompt,
  launch, let the durable monitoring layer watch it, report after exit. Intervene only
  for hard failure, runaway scope, or destructive behavior.
- **Companion** — uncertain, exploratory, or multi-round work; the user is (or will be)
  around. The host supervises checkpoints, steers with bounded relaunches, and treats
  each round of findings as the next acceptance criteria. Prefer a new bounded gnhf
  prompt over manually taking over the worker's scope.

Either way: **"stop condition met" is the worker's opinion, not acceptance.** The host
always re-verifies before presenting the result as done.

## Launch — the run must not live in the harness's process tree

A run inside an agent's process tree dies with the agent's turn (observed: background
Bash tasks killed mid-work when a provider limit aborted the turn).

**The user launches; the host hands over the command.** Standing rule (Ryan,
2026-08-10, after a captain agent-launched a run uninvited — supersedes the earlier
"two equally valid patterns" wording). Perform the pre-launch checks, then hand the
user ONE complete copy-paste command for their own terminal — prompt piped from its
committed file, all flags inline:

```bash
cat docs/planning/gnhf-runs/<run>.md | gnhf --agent codex \
  --max-iterations <n> --max-tokens <n> --prevent-sleep on \
  --meteor-frequency 0 --stop-when "<checklist-backed condition>"
```

The host then monitors; survivability is free. The SAME rule applies to resumes and
reroutes: edit the command, hand it back to the user.

**Agent-launched (detached tmux) ONLY when the user explicitly asks the host to
launch** — never as a default, and never inferred from an approval to "launch" in a
plan (that approves the run, not the host executing it): one launcher script per run
so a reroute is a one-line edit. Keep the prompt in its own file — heredocs inside
`$( )` break under `/bin/sh`.

   ```bash
   tmux new-session -d -s gnhf-<name> -c <dir> "sh <launcher.sh> >> <log> 2>&1"
   ```

Flags that matter: `--agent`, `--max-iterations` (the runaway guard), `--stop-when`,
`--max-tokens`, `--prevent-sleep on`, `--current-branch` only when asked (default is a
`gnhf/` branch; `--worktree` for parallel runs), `--push` stays off,
`--meteor-frequency 0` for quieter logs. Don't invent flags; check `gnhf --help` if
unsure.

Before launch: clean tree (gnhf requires it), right branch/worktree, and in any
fork-flow repo `git branch --unset-upstream` so a bare push cannot reach upstream.

Agent choice: `--agent codex` is the standing preference; `~/.gnhf/config.yml` carries
the model override that activates with it — but note the config's *default* agent is
`claude`, which is a resume trap (below). Split parallel tracks **by verifiability**:
fully-testable work to unattended tracks; work with visual/manual acceptance criteria
can only ever end code-complete, and its prompt must produce a written morning test
plan instead of claiming success.

## The prompt is the product — the defect gradient

Run-01's measured result: **defects cluster inversely with prompt precision.** The
itemized layer came out clean across 54 amnesiac iterations; the "gestured-at" layers
collected every bug; the Hard Constraints section had 100% compliance. Spend prompt
effort accordingly:

- **Itemize what matters.** A numbered list with the rule quoted beats a paragraph
  that gestures. If a layer gets four bullets, expect its bugs.
- **Hard Constraints section** for anything load-bearing — it's the highest-compliance
  real estate in the file. Bans (deferred concepts, forbidden deps, forbidden
  commands) go here with the reason stated.
- **INVARIANTS section** for cross-cutting rules, because notes.md is not reliable
  memory and invariants decay at integration seams. Phrase as per-iteration
  obligations: "every new X must be checked against Y and added to Y's regression test
  in the same iteration."
- **Externally-pinned gates.** The worker must not author the gate it is graded by
  (a real worker wrote a tsconfig that silently excluded a package, then honestly
  reported green every iteration after). State gates as exact commands with expected
  scope, and require new packages/modules to join the gate in the iteration that
  creates them.
- **Falsifiable tests.** "A rule test must be able to fail; if a rule is enforced
  structurally, test at the layer where the two concepts meet." Otherwise you get
  coverage theater.
- **Checklist-backed stop condition**, never prose. Bad: "supports the full flow."
  Good: a maintained checklist file mapping every numbered requirement to the control
  and test that proves it; stop = every row checked. The worker judges `--stop-when`,
  so an ambiguous one *will* be read generously.
- **An adversarial final lap**: when the stop condition appears met, spend one
  iteration attacking boundary cases before declaring it.
- **Re-read triggers** for spec-heavy areas ("before touching X, re-read spec §N") —
  requirements referenced indirectly degrade over the run's lifetime; requirements in
  the prompt don't.
- **An ambiguity outlet**: conservative reading + record the question in a
  QUESTIONS.md + move on. Cheap, high-value morning review targets.
- **Self-contained.** The Codex worker sees AGENTS.md, never CLAUDE.md; anything that
  must bind it lives in the prompt or AGENTS.md. State authorization overrides in so
  many words (e.g. commits allowed despite a repo rule), or the worker obeys the repo
  and produces nothing.
- **Blocked ≠ failed**: "if a task requires a forbidden action, document it with
  evidence and move on; never fake success."
- **Continuation-safe phrasing** ("continue from the current repo state") so the run
  can be resumed or rerouted to another agent by editing `--agent` in its launcher.

## Reading a live run — the log is the only truth

**A finished run is indistinguishable from a hung one by process inspection.** After
finishing, gnhf keeps its TUI alive: process up, CPU burning, log mtime fresh, no
worker child. Every cheap signal lies. Judge only by:

```bash
strings <log> | tail -c 200000 | grep -nE "stop condition|You've hit your|limit"
```

- `stop condition` → **finished** (grep the short phrase — the TUI wraps mid-word, and
  one real run's log ended at `stop condition m`).
- a limit message → **blocked**, not hung; won't recover until the window resets.
- neither, with no worker child and no commit/notes movement → **actually hung**.

Progress = **commit count + notes-file mtime moving between checks**. Nothing else
counts. Logs are append-only across relaunches (scan the tail, or an old limit message
reports "blocked" forever) and full of ANSI escapes (`strings`, not `tail`).

Durable monitoring must live outside agent turns — a supervising agent that gets
blocked stops recording, silently. If this machine has the watchdog
(`~/projects/scripts/gnhf-watchdog.sh` + its launchd plist), register the run and load
it; in-session polling is then the interactive supplement, and the first thing to read
after any gap is the watchdog log (`grep SUMMARY ~/.gnhf-watchdog/watchdog.log`), not
the process table. Without the watchdog, fall back to log inspection and accept the
blind spots. Quota gating via `codexbar guard` helps but has two verified holes: Codex
has no session window (guard it on `weekly`), and codexbar cannot see the Claude
monthly spend limit — only the run's own log catches that one.

## Resume and reroute

- **The iteration counter is cumulative across resumes** (`startIteration` in the
  log); `--max-iterations` counts against the cumulative total — **raise the cap on
  every resume**. The token counter resets per process, so a re-passed `--max-tokens`
  is a fresh budget.
- **Re-pass `--agent` and all caps every time.** Only `--stop-when` persists per run;
  the config default agent is `claude`, so a bare `gnhf` resume silently switches a
  codex run to Claude.
- **Re-piping the identical prompt is the cleanest resume** — gnhf detects the match
  and continues the same branch and numbering without interrogation. An interrupted
  iteration rolls back cleanly.
- **Reroute** = kill the tmux session, edit `--agent` in the launcher, relaunch; the
  new agent picks up from committed state. This is what continuation-safe prompts buy.

## Review — never trust the run's own notes

Workers self-report optimistically, and every independent check below has caught a
real problem at least once:

1. Read the exit state from the **log** first (finished vs blocked vs iteration-cap —
   a finished-looking TUI is not stop-condition evidence), then notes and any
   QUESTIONS.md, then `git log --oneline <base>..HEAD` for the narrative.
2. **Re-run the suites yourself.** Never quote the worker's counts.
3. **Reproduce one falsification**: break the thing a key test guards (or run the test
   at the pre-fix commit) and watch it go red. An unfalsified assertion is worse than
   none.
4. **Check scope structurally**, not by claim: `git show --name-only` against expected
   files, build artifacts, attribution trailers, `git branch -r --contains HEAD`
   (should be empty).
5. **Run an independent adversarial review** over `git diff <base>...` — it is the
   layer the loop cannot provide for itself and has caught 100% of the serious
   findings in real runs — then **verify the reviewer's findings too**; a finding you
   can't reproduce is a finding to drop.
6. Decide: **Mergeable** / **Needs a follow-up bounded run** / **Do not merge**. Never
   merge without explicit authorization. Reword gnhf's own commit subjects
   (`gnhf 1: …`) before anything becomes a PR.

When the user returns with findings, convert each into an observable correction
(preserve their wording and scope), relaunch on the same branch with the finding as
the sole bounded objective and an evidence-based stop condition, and review again.
Anything unverifiable overnight (UI, hardware) is labelled **unverified** with a
written test plan, not claimed.

## Safety

- Preserve user changes; no destructive git against a run branch.
- Nothing gets pushed unless the user authorized that exact push.
- While the user is away: produce branches, evidence, and a status report — not
  irreversible actions.
