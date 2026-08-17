# Codex Integrator: the invocation contract

Every Codex call in this skill goes through these exact patterns. They encode real traps; do not improvise around them. Verified live on codex-cli 0.143.0.

## The non-negotiables (every call)

1. **Prompts go in via a file on stdin** (`- <"$FILE"`), never as an argv string. This kills two traps at once: shell-quoting bugs and the stdin hang (`codex exec` reads stdin in addition to any argv prompt; under a non-TTY harness an unredirected call blocks forever at 0 CPU). If you ever must pass an argv prompt, append `< /dev/null`.
2. `2>/dev/null` always. Codex streams thinking tokens to stderr; they would flood the driving context.
3. **Capture the JSONL stream to a file and parse it after exit.** Never pipe the stream through `grep` directly: `grep -m1` exits at first match, closes the pipe, and can kill Codex mid-run before the output file is written.
4. `--json -o <outfile>`: the answer is read from the `-o` file, the stream file is parsed only for the thread id.
5. **One fresh temp dir per run, fresh filenames per round** (`RUN=$(mktemp -d /tmp/rf.XXXXXX)`). Never reuse an output path across rounds: after a timeout or failure, a reused path serves the PREVIOUS round's verdict and looks exactly like success.
6. Resume by explicit thread id, never `--last`. A wrong session looks exactly like success.
7. Never pin `-m`. Model pins 400 on ChatGPT-account auth. Before round 1, echo the active model if `~/.codex/config.toml` has a `model` line, else say "Codex CLI default".
8. Bash tool `timeout: 600000` (10 min) on every call. Codex writes output only at completion; a killed call is silently empty.
9. `--skip-git-repo-check` on every call (Codex refuses untrusted directories without it).
10. **Snapshot before every call** (the three snapshot lines below, together, in every snippet): status, tracked diff, and untracked-file hashes. If a read-only call changes files anyway: paths that were CLEAN before get reverted; tracked-dirty or untracked paths whose content no longer matches the snapshot get a STOP, not an auto-revert (surface it, let the user judge). Never blanket-revert: the user or the Visionary may have legitimate uncommitted work.

```bash
snap() { git status --porcelain > "$RUN/pre-$1.txt"; git diff > "$RUN/pre-$1.patch";
  git ls-files --others --exclude-standard -z | xargs -0 shasum > "$RUN/pre-$1.sha" 2>/dev/null || true; }
```
11. A stream event complaining about "skills context budget" is benign Codex housekeeping, not a failure.
12. **One tool call per step, never one compound command.** Write the prompt/contract file with the Write tool (not a heredoc), run the snapshot as its own small Bash call, then run the bare `codex exec` line. A single compound command chaining these trips the harness auto-mode classifier and stalls the run on a permission prompt. Same rule for post-run housekeeping: `gh issue close`, branch deletes, etc. go as separate small commands.
13. **Never chain `codex exec` behind a conditional command.** A `grep -c` that exits 1 on zero matches silently kills the launch and the background task reports exit=1 with no report file. The exec line is always a bare invocation, and a missing `-o` report file means "did not run", not "failed" (2026-07-26).
14. **Worktrees for Codex builds live in `~/worktrees/`, NEVER /tmp.** The macOS /tmp cleaner deletes worktree `.git` link files and tracked files mid-run (84 files lost once, recovered via `git ls-files -d -z | xargs -0 git restore --` then `git worktree repair` twice, then `git worktree move`). Run temp dirs for OUTPUTS (`$RUN`) may stay in /tmp; the working tree may not. Before any long unattended run, commit the current state as an explicit `wip: ... UNREVIEWED` checkpoint and push; review fixes land as their own commit on top (2026-07-31).
15. **Know the sandbox's hard limits and contract around them.** workspace-write blocks socket binds (every Chrome/puppeteer launch, `npx tsx --test` IPC pipes), port listens, `rm -f` (use plain rm or overwrite), and any write outside the project dir (name /tmp paths for OUTPUTS only via shell redirection, pre-create directories). In-sandbox proofs are therefore STATIC (node --check, bash -n, direct-loader tests); anything launching a browser or binding a port is the Visionary's live proof outside the sandbox, and the contract says so explicitly so Codex never sits wedged trying (2026-07-30, two wedges in one night).
16. **The never-started signature:** process alive at ~0 CPU, empty `-o` file, and no new rollout file in `~/.codex/sessions/` for the launch timestamp means Codex never started (usually a stdin block). Kill and relaunch with stdin closed instead of waiting (2026-07-30, a 4-hour wedge).

## Review calls (Same Page Meeting, read-only)

Fresh session:

```bash
RUN=$(mktemp -d /tmp/rf.XXXXXX)
# write the review prompt to "$RUN/prompt-r1.md" first
snap r1
codex exec -s read-only --skip-git-repo-check --json -o "$RUN/out-r1.txt" \
  - <"$RUN/prompt-r1.md" > "$RUN/stream-r1.jsonl" 2>/dev/null
```

After it exits, extract the thread id from the stream file and echo it visibly:

```bash
THREAD_ID=$(grep -m1 '"type":"thread.started"' "$RUN/stream-r1.jsonl" \
  | sed 's/.*"thread_id":"\([^"]*\)".*/\1/')
```

Resume the SAME session for later rounds. THE SAFETY LINE: `resume` rejects `-s`; without `-c sandbox_mode="read-only"` Codex inherits the config default and can WRITE files mid-review.

```bash
snap rN
codex exec resume "$THREAD_ID" -c sandbox_mode="read-only" --skip-git-repo-check --json \
  -o "$RUN/out-rN.txt" - <"$RUN/prompt-rN.md" > "$RUN/stream-rN.jsonl" 2>/dev/null
```

Then Read the round's `-o` file and grep its last line for `VERDICT:`.

## Build calls (rock execution, sandboxed write)

Gate first, in this order:

1. No git repo yet (fresh kickoff): `git init`, then commit ONLY this run's artifacts. Never sweep unrelated pre-existing files into the baseline: they may be secrets or junk. If any exist, list them and ask the user once (commit, stash, or ignore; "ignore" means writing the paths into `.gitignore` and committing it, so the gate actually reads clean).
2. Commit the planning artifacts (plan, VTO, log, issues) before the first build. These baseline commits are part of the machinery: announce them in one line, do not ask. Only CODE commits (rock results) are user-gated. The gate is not "nothing changed", it is "everything is baselined": `git status --porcelain` must be empty when Codex launches, so the build diff is exactly Codex's work and nothing else.
3. Pre-existing dirty files that are not this run's artifacts: stop and ask the user to commit or stash them.

Default build mode is Codex's OWN sandbox with write access to the workspace. Do NOT reach for `--yolo` / `--dangerously-bypass-approvals-and-sandbox` by default: sandboxed mode builds real features, and the bypass flags run an unsandboxed agent, which needs the user's explicit per-run approval.

```bash
snap build   # pre-build.txt must be empty (everything baselined)
~/.claude/scripts/codex-run.sh exec --sandbox workspace-write -c approval_policy="never" --skip-git-repo-check --json \
  -o "$RUN/build-out.txt" - <"$RUN/contract.md" > "$RUN/stream-build.jsonl" 2>/dev/null
BUILD_THREAD=$(grep -m1 '"type":"thread.started"' "$RUN/stream-build.jsonl" \
  | sed 's/.*"thread_id":"\([^"]*\)".*/\1/')   # fix rounds resume THIS id; echo it visibly
```

If the rock needs network (installing deps), add `-c sandbox_workspace_write.network_access=true` and say so in one line before running. Fix rounds resume the same session with the sandbox forced via `-c`:

```bash
snap fixN
codex exec resume "$BUILD_THREAD" -c sandbox_mode="workspace-write" --skip-git-repo-check --json \
  -o "$RUN/fix-outN.txt" - <"$RUN/fixN.md" > "$RUN/stream-fixN.jsonl" 2>/dev/null
```

## The build contract template

```
GOAL: <one paragraph: what done looks like for THIS rock>
SPEC: Read <actual plan file path>, rock <N>. It is frozen and already
  reviewed. Implement it exactly. If a detail is impossible as written but the
  intent is unambiguous, implement the closest faithful version and report the
  deviation. If the impossibility is MATERIAL (would change behavior, scope, or
  an interface), do not improvise: stop, output `BLOCKED: <reason>` as your
  report, and wait. Do not redesign.
KEY PATHS: <files/dirs to touch, files to read first>
CONSTRAINTS: <do-not-touch list, style rules, deps that must not change>
NON-GOALS: <explicitly out of scope, including every other rock>
PROOF: Run `<PROOF_CMD>` and include its full output in your report.
CRITIQUE: <only for rocks with a user-facing surface> Render or run the result
  the way a user would experience it (screenshot the page desktop + mobile, open
  the export, run the binary). Then critique it hostilely in writing: rhythm,
  alignment, contrast, dead zones, anything that smells like an AI default. Fix
  what you find, then add ONE deliberate refinement beyond the fix list. Include
  the critique and the render evidence in your report. A report without them is
  incomplete.
DATA FACTS: <only for rocks touching a served payload> The exact key values for
  every lookup this rock performs, verified against the LIVE API, not the source.
OUTPUT: End with a report: files changed (one line each: path + what/why),
  the proof output, the critique + evidence when CRITIQUE applies, and a
  DRIFT INVENTORY: an enumerated list of EVERY delta versus the baseline
  artifact or prior behaviour (visual, structural, contract, timing), each
  marked intended-per-contract or unrequested, with "none" written per
  category if empty. A report whose inventory a reviewer later falsifies
  counts as a failed round, not a note.
```

### Contract hardening (field-proven, 2026-07-29 to 07-31)

- **Served keys, never display labels.** Any contract touching a served payload carries the DATA FACTS section above; builders binding lookups to display strings ("marketing" vs served key "attract", `.includes("show rate")` also matching "No-show rate") cost a fix round each time. Reviewers check every find/filter binds to keys.
- **Name the middleware chain.** A rock adding an HTTP route to a repo with app-wide gating middleware names that middleware in the contract, and REACHING the route (not just building it) is in the proof bar. Two routes once deployed unreachable because the contract never mentioned `src/middleware.ts`.
- **State the applied-migration ceiling.** On any repo with checked-in migrations, the contract states the currently APPLIED ceiling as fact; migration drift is the first hypothesis on a "nothing works" report (prod once sat eight migrations behind main while features merged compile-verified).
- **Fixtures mirror the messiest real rows.** A form or import rock prefilling from registry/free-form data gets fixtures with prose money ("RM2,000/mo"), unmapped enums, and nulls, never only canonical values; verify fixtures mirror production constraints exactly (real roles for SECURITY DEFINER re-checks, unique constraints, error-checked inserts, try/finally cleanup with a zero-leftovers assertion). The go-live walk on the FIRST REAL record is part of the run, not after it.
- **User-facing server actions return expected errors as values (`{ok, error}`), never throws.** Next.js redacts thrown server-action errors in production, so every validation reason becomes a useless digest error that dev mode never shows you.

## Proof discipline (Level 10, field-proven)

- **A proof that cannot fail is not a proof.** Every verification names the condition under which it would FAIL and the run recreates it. Prove write paths with a marker value that has never been rendered before, so a cache or stale render cannot impersonate a pass.
- **HTTP 200 is not proof for a client-rendered page.** curl proves the shell, never the content, on anything assembled in the browser; use a browser check or assert on a server-rendered marker.
- **Proof output is captured whole, filtered after.** Output goes to a log file first and gets grepped there; a proof piped through `tail`/`head` mid-command once swallowed 38 FAIL lines and read as a clean pass.
- **New harnesses pass seeded calibration first.** Plant one known violation (must flag) and one known-good case (must stay silent) before any harness output gates a rock; uncalibrated harnesses have billed 600+ phantom failures.
- **Enumerate a browser harness's blind spots up front.** elementsFromPoint skips pointer-events:none, getBoundingClientRect reports unclipped geometry, computed gradients need their own parser; each unlisted blind spot has cost a cycle.
- **A finding that survives a correct fix unchanged indicts the harness, not the artifact.** Check the measurement semantics (clipping, visibility, stacking) before spending another fix round.
- **Curl-proof strings come from the exact rendered variant, apostrophe-free.** React escapes apostrophes to `&#x27;`; a quoted string from source copy false-fails a correct build. When a leak grep fires, READ the matching text; when a presence check fails, screenshot first. Canonical patterns: currency leak `/RM[0-9,.]/` case-sensitive digit-anchored; label presence checks case-insensitive (CSS text-transform changes innerText casing).
- **Name what a live-account action writes before running it.** Diagnostic-looking actions can mutate (a Stripe Checkout link mints an order row); during anyone's live test, state the write or do not act.
- **Next.js worktree hygiene:** never run a dev server on a worktree carrying a proof-build `.next` (or vice versa), and after ANY edit while a symlinked-node_modules worktree dev server runs: stop, `mv .next` aside, restart, warm with curl. `rm -rf .next` is permission-blocked; `mv` aside is the recovery.

## Level 10 review mechanics (after every build call)

1. **Delegate the diff read to a Sonnet reviewer subagent** (XELAH amendment, 2026-07-25): dispatch it via the Agent tool with `model: "sonnet"`, pointing it at the rock's contract and `git diff`. Its brief: read every hand-written source change in full (skip lockfiles and generated/vendor output), check the diff against the contract's GOAL/CONSTRAINTS/NON-GOALS and the smell vocabulary, and report structured findings with file:line citations plus a severity per finding. The Visionary reads the findings, not the diff. Codex's report is a claim, the reviewer's report is evidence to adjudicate, neither is proof. Scope every reviewer brief to an explicit file list and a bounded check list; open-ended full-repo briefs stall on the watchdog (3 occurrences). On a stall, resume the SAME agent with "consolidate what you have into the report now" rather than respawning.
2. Run `PROOF_CMD` yourself. Only your run counts.
3. Adjudicate each reviewer finding accept or reject with a reason (Rule 5). Accepted findings within the rock's scope: send ONE consolidated fix-round prompt (resume, same session): "Fix these N items, nothing else, re-run the proof." Max 2 fix rounds per rock.
4. Still broken after round 2: takeover is pre-authorised (Alex, 2026-07-25) and bounded by the finishing rule. Last-mile residue: the Visionary finishes it, announces it, notes it in the scorecard. Unsalvageable diff: revert the rock, rewrite the contract, one fresh Codex run; still failing, stop and flag `needs-alex`. Never rebuild a rock inline.
5. Deviations Codex reported: judge each against the Core Focus. Execution-detail deviations stand (the Integrator is the Tie Breaker). Vision or scope deviations get reverted in a fix round, with the reason logged.
6. Commits: user-gated, authored by the driving agent, conventional format. Codex never commits.

## Codex Cloud (laptop-off lane)

Environments are web-UI-only: no CLI exists to create or list them, so record env IDs the moment they are discovered (the live env map lives in memory `project_night_build_lanes`). Submit work with `codex cloud exec`; use `--branch` to build on a PR branch when a rock's prerequisites live in an unmerged PR. Auth rides the ChatGPT login (subscription), not an API key.

## Dual-account failover (XELAH amendment, 2026-08-17)

Two Codex accounts live on this Mac: A at `~/.codex` (primary), B at `~/.codex-b` (overflow), selected per-invocation via `CODEX_HOME`. The wrapper `~/.claude/scripts/codex-run.sh` owns the choice: it reads the live weekly meter (`~/.claude/scripts/usage-meter.sh codex`, source: the wham/usage backend endpoint) and dispatches on B once A is at or above 95% (Alex's binding rule), retrying once on the other account on a usage-limit error. Every switch appends to `~/.claude/scripts/subscription-switches.log`.

Rules that keep this safe:

1. FRESH dispatches (build, review, cloud) go through `codex-run.sh`. RESUME calls NEVER do: threads live inside one home, so `codex exec resume` is always a bare `codex` call with `CODEX_HOME` pinned to the home that ran the original attempt. Which home that was: check the switches log around the dispatch timestamp; no line means it ran on A.
2. A limit hit mid-thread means re-contract on the other account, never resume across homes.
3. Cloud env IDs are account-scoped. `codex-run.sh` translates A-account env IDs to B's via its `# ENV_MAP` section; an unmapped ID makes it refuse (exit 78) rather than run against the wrong account. Keep the map current when environments are created.
4. Flag note (0.147): `--full-auto` no longer exists on `codex exec`; the equivalent is `-c approval_policy="never"` with `--sandbox workspace-write`, as the build block above shows.
5. Resume cwd gotcha (0.147, field-proven 2026-08-17): `codex exec resume` accepts no `-C` and takes its workspace from the INVOKING shell's cwd. A resume launched from the wrong directory gives the thread the wrong writable scope and Codex reports BLOCKED. Always `cd` into the rock's workspace before any resume call.

## Failure handling

**A call succeeded only if ALL of:** exit code 0, the round's `-o` file exists and is non-empty, the stream jsonl has NO `turn.failed` event (a mid-run backend disconnect exits looking clean; the diff survives, resume the same thread with an explicit list of what remains), and (for meeting rounds) its last line greps a `VERDICT:`. Anything less is a failure, even if the stream showed `thread.started`. Never read a verdict from a file the current round did not freshly write.

Watch loops polling an external API (Render, Vercel, CI) parse tolerantly (`items[0].get('deploy', items[0])`-style fallbacks for shape variants) and PRINT parse errors instead of looping past them; a hard-indexed shape has burned 10 silent minutes twice in one night.

- Fresh call fails or times out (no thread id captured): retry ONCE with a fresh session and a new round filename. Do not "resume" a thread that never started.
- Resume call fails or times out: retry ONCE with the same explicit thread id and a new round filename. Second failure: fall back to a fresh session carrying a one-paragraph summary of the meeting so far, and SAY SO to the user in one line (session continuity broke; the round count continues, the new session cannot verify its own prior findings).
- Still failing: stop and surface the error (rerun the identical command WITHOUT `2>/dev/null` to capture stderr). Never silently continue without the review.
- Interrupted build (session restart or kill mid-run, no `-o` report written): the Codex thread usually survives. Grep the run's stream jsonl for the `thread.started` id, then `codex exec resume <id>` with a finish-the-contract prompt ("complete the contract in <file>; report files changed + proof output"). Only start a fresh build if no stream file or thread id exists (2026-07-25, recovered two orphaned night builds this way).
- Auth errors: tell the user to run `codex login`. Broken install (`spawn ... ENOENT`): `npm i -g @openai/codex@latest`.
- Prerequisite floor: Codex CLI >= 0.130; contract verified on 0.143.0.
