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
codex exec --sandbox workspace-write --full-auto --skip-git-repo-check --json \
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
OUTPUT: End with a report: files changed (one line each: path + what/why),
  the proof output, the critique + evidence when CRITIQUE applies, and any
  deviations from the spec with reasons.
```

## Level 10 review mechanics (after every build call)

1. **Delegate the diff read to a Sonnet reviewer subagent** (XELAH amendment, 2026-07-25): dispatch it via the Agent tool with `model: "sonnet"`, pointing it at the rock's contract and `git diff`. Its brief: read every hand-written source change in full (skip lockfiles and generated/vendor output), check the diff against the contract's GOAL/CONSTRAINTS/NON-GOALS and the smell vocabulary, and report structured findings with file:line citations plus a severity per finding. The Visionary reads the findings, not the diff. Codex's report is a claim, the reviewer's report is evidence to adjudicate, neither is proof.
2. Run `PROOF_CMD` yourself. Only your run counts.
3. Adjudicate each reviewer finding accept or reject with a reason (Rule 5). Accepted findings within the rock's scope: send ONE consolidated fix-round prompt (resume, same session): "Fix these N items, nothing else, re-run the proof." Max 2 fix rounds per rock.
4. Still broken after round 2: takeover is pre-authorised (Alex, 2026-07-25) and bounded by the finishing rule. Last-mile residue: the Visionary finishes it, announces it, notes it in the scorecard. Unsalvageable diff: revert the rock, rewrite the contract, one fresh Codex run; still failing, stop and flag `needs-alex`. Never rebuild a rock inline.
5. Deviations Codex reported: judge each against the Core Focus. Execution-detail deviations stand (the Integrator is the Tie Breaker). Vision or scope deviations get reverted in a fix round, with the reason logged.
6. Commits: user-gated, authored by the driving agent, conventional format. Codex never commits.

## Codex Cloud (laptop-off lane)

Environments are web-UI-only: no CLI exists to create or list them, so record env IDs the moment they are discovered (the live env map lives in memory `project_night_build_lanes`). Submit work with `codex cloud exec`; use `--branch` to build on a PR branch when a rock's prerequisites live in an unmerged PR. Auth rides the ChatGPT login (subscription), not an API key.

## Failure handling

**A call succeeded only if ALL of:** exit code 0, the round's `-o` file exists and is non-empty, and (for meeting rounds) its last line greps a `VERDICT:`. Anything less is a failure, even if the stream showed `thread.started`. Never read a verdict from a file the current round did not freshly write.

- Fresh call fails or times out (no thread id captured): retry ONCE with a fresh session and a new round filename. Do not "resume" a thread that never started.
- Resume call fails or times out: retry ONCE with the same explicit thread id and a new round filename. Second failure: fall back to a fresh session carrying a one-paragraph summary of the meeting so far, and SAY SO to the user in one line (session continuity broke; the round count continues, the new session cannot verify its own prior findings).
- Still failing: stop and surface the error (rerun the identical command WITHOUT `2>/dev/null` to capture stderr). Never silently continue without the review.
- Interrupted build (session restart or kill mid-run, no `-o` report written): the Codex thread usually survives. Grep the run's stream jsonl for the `thread.started` id, then `codex exec resume <id>` with a finish-the-contract prompt ("complete the contract in <file>; report files changed + proof output"). Only start a fresh build if no stream file or thread id exists (2026-07-25, recovered two orphaned night builds this way).
- Auth errors: tell the user to run `codex login`. Broken install (`spawn ... ENOENT`): `npm i -g @openai/codex@latest`.
- Prerequisite floor: Codex CLI >= 0.130; contract verified on 0.143.0.
