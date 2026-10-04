---
name: rocket-fuel
description: Runs the orchestrating session (Opus 5.5 by default, Fable by escalation) and an Integrator (Codex CLI for large Lane 3 builds, or an Opus integrator subagent) as co-founders on the Rocket Fuel operating system. The Visionary holds vision, plan, standards, final review; the Integrator filters the plan, executes the build, reports. Lane 3 code always, Lane 2 code when contracted, Lane 1 never. Context-routed, one entry point, five functions: KICKOFF a new project, CLARITY BREAK to refactor an existing one, SAME PAGE MEETING to pressure-test a plan, BUILD A ROCK to hand the Integrator a scoped task, RUN A WAVE to produce many parallel units under one constitution without a human in the loop. Use when the user says /rocket-fuel, wants to start or refactor a project with Codex, wants a plan reviewed by a second model, wants to delegate a build to Codex or an integrator, or wants a batch of same-kind units built in parallel.
---

# Rocket Fuel: the Visionary/Integrator OS

It takes two. One sees the future, one makes it happen. Vision without execution is just hallucination.

Before any build, load the checklist in Learned corrections below (it is the rule register's compact form); open full rows in SYSTEM/Wiki/live/rule-register.md when a trigger applies.

## Quick start

The user runs `/rocket-fuel <anything in plain English>`. You detect the function and the lane (Phase 0), confirm both in one line, and drive the whole thing. The user makes decisions at exactly three gates: the Grill answers, the deadlock tie-break (rare), and the final commit. Everything else runs itself.

**Guide the user the whole way.** Assume they have never seen this system. After confirming the function, tell them in 2-3 lines which stages are coming and where their three gates are. Announce every stage transition in one line ("Same Page Meeting, round 2: Codex is verifying its round-1 findings were addressed"). When the run ends, list the artifacts that now exist and the single next step. Never go silent for a long Integrator call: say what is running and roughly how long it takes before you launch it.

## Which lane uses rocket-fuel (HR20)

- **Lane 3 (risky), always.** Any of schema or migrations, RLS, permissions, money, outbound side effects, auth, prod scripts or sweeps, import/merge/delete, patient-file storage, LLM data egress. The full run: Grill, PRD, Same Page Meeting to a verdict, frozen contract, Integrator build, Sonnet full-diff review, Level 10. The orchestrator finishes last-mile residue only.
- **Lane 2 (multi-file, no Lane 3 trigger), when contracted.** A lean contract (GOAL, KEY PATHS, CONSTRAINTS, NON-GOALS, PROOF, CRITIQUE where surfaced); Grill and PRD only when scope is unclear; at most ONE Same Page round, after which you adjudicate each finding and freeze; one review. The orchestrator may finish or fix any residue itself.
- **Lane 1 (small), never.** The session writes the code directly; this skill does not run.
- The lane is announced in one line before any contract and can only go UP: a plan or diff that crosses into Lane 3 stops and re-enters under the Lane 3 run. Runs whose PRD was frozen before 2026-10-04 close on the protocol they started under.

## The Accountability Chart (one person per seat)

| Seat | Held by | Five accountabilities |
|---|---|---|
| **Visionary** | the orchestrating session (Opus 5.5 by default, Fable by escalation) | new ideas and R&D · creative problem solving · architecture and standards · the hard design calls · Level 10 final review |
| **Integrator** | Codex CLI for large Lane 3 builds, or an Opus integrator subagent | filter the Visionary's plan against the Core Focus · execute the plan · integrate the parts · resolve cross-cutting implementation issues · report honestly |

When more than one is accountable, nobody is. In Lane 3 the orchestrator does not author implementation code: it finishes last-mile residue only, under the takeover rule in the Level 10 review, and anything more goes back out as a fresh contract. In Lane 2 it may finish or fix any residue itself (HR20).

## The 5 Rules

1. **Same page before code.** No build starts until the Same Page Meeting ends in `VERDICT: SAME PAGE`, or the user explicitly overrides. Lane 2: one round, then you adjudicate and freeze.
2. **No End Runs.** Mid-build change requests, whether from the user or from you, go onto the Issues List for the next review. Never into the working diff.
3. **The Integrator is the Tie Breaker** on execution details: file layout, implementation order, library choice within stated constraints. The Visionary trumps only on vision or scope deviations, and logs why.
4. **Bounded meetings.** Hard caps: 5 review rounds (Lane 2: 1), 2 fix rounds per rock (the user may set different caps up front; whatever the number, it is a hard cap). A flagged deadlock beats a fake approval. At the cap, the user decides: it is more important THAT you decide than WHAT you decide.
5. **Mutual respect.** Every Integrator or reviewer finding lands in the log verbatim, and you answer each one with accept or reject plus a reason. Never silently drop a finding.

## Phase 0: Preflight and Read the Room

Preflight once per session whenever Codex will hold a seat (a Same Page Meeting or a Codex build): run `codex --version` (Bash). If it fails (including a `spawn ... ENOENT` from a broken npm install), fix with `npm i -g @openai/codex@latest`; if auth errors appear later, the user runs `codex login`. Never fake the Integrator: if Codex stays down, say so, then run the seat on an Opus integrator per "Opus integrator mechanics" below, announced in one line. Never an untracked helper, never the orchestrator itself.

Then detect the function:

| Signals in the request + repo state | Function |
|---|---|
| New idea, empty or near-empty directory, "start / kick off / build me a" | **KICKOFF** |
| Existing codebase and "refactor / clean up / audit / what should we fix" | **CLARITY BREAK** |
| An existing plan, spec, or PRD file that Codex has not reviewed | **SAME PAGE MEETING** (standalone) |
| A frozen spec or one clearly scoped task ("have codex build X") | **BUILD A ROCK** (standalone) |
| Many units of the same kind ("build N sites/pages/variants", "run a wave") | **RUN A WAVE** |

Confirm function and lane in one line, then go: "Running a KICKOFF for <thing>, Lane 3 (migration). Say stop if you wanted something else." If genuinely ambiguous, ask ONE question. Never a questionnaire.

## Function 1: KICKOFF

1. **The Grill.** Interview the user one question at a time (AskUserQuestion), with your recommended answer attached to every question. Facts you can find by exploring the codebase or the web are yours to find; only decisions belong to the user. 5 to 9 questions for a real project, 3 to 5 for something small. Do not proceed until the user confirms shared understanding. Write `VTO.md`: Core Focus (one sentence), what done looks like, non-goals, constraints and stack, and 3 to 7 goals.
   - Completion: user has confirmed; `VTO.md` exists with a one-sentence Core Focus.
2. **Draft the plan.** Write `PLAN.md`: 3 to 7 Rocks in dependency order. Every rock has a "done looks like" and a proof command (a test, a build, a curl); wherever a test makes sense, the proof is a failing test written before the build (TDD framing). If you have more than 7 rocks you are stuffing 100 pounds into a 50-pound bag: cut or merge. Do less better.
   - Completion: every rock has a runnable proof command.
   - **Contract gates (XELAH amendment, 2026-07-28, from night-build field lessons).** Before freezing any rock: (a) its prerequisites exist ON MAIN, verified by grepping for the symbol, not assumed; work sitting in an unmerged PR means adjudicate and merge that PR first, or contract the rock onto that branch (`codex cloud exec --branch`). (b) No constraint that contradicts the repo's own rules (a TODO-copy guard over legitimately-placeholder fields can never be wired; Codex ships dead code rather than flagging it unless the BLOCKED instruction catches it). (c) A rock adding a NOT NULL column must include a repo-wide sweep of existing INSERTs into that table (tests, seeds, fixtures) and name the FULL suite as the proof bar, not just touched files: typecheck and builder-local checks miss fixture breakage.
   - **Approved-artifact fidelity gate (XELAH amendment, 2026-07-28).** When the thing being built is a port, rebuild, or re-platforming of an artifact the user has ALREADY approved (a static site, a deck, a design, a document), the contract names that artifact as the verbatim spec and instructs the Integrator to emulate it exactly as the target stack allows: same layout, spacing, type scale, imagery treatment, motion. Never contract it as "reproduce the design language" or any other re-derivation framing. When the approved artifact contains working code (CSS, JS, effects), the contract mandates TRANSPLANTING that code verbatim, renamed only where the target architecture requires, never re-implementing behaviour from a description of it: every re-implementation pass drops details (hover effects, animations, spacing) that the original already got right. The proof for such a rock includes a side-by-side comparison against the approved original, judged by the Visionary. Origin: the fable-templates renderer port landed visibly below the approved bar because the contract asked for the language, not the artifact.
   - **context7 grounding gate (XELAH amendment, 2026-07-25).** Before freezing any contract that touches a library, framework, SDK, or cloud API: resolve the library on context7 (`resolve-library-id`, then `query-docs`) and pin the verified signatures, versions, and config shapes into the contract as stated fact. Codex has no context7 and builds from its own training cut; a remembered signature in a frozen contract is a wasted fix round waiting to happen. Never write an API signature into a contract from memory (Guardrail 2 extended to code).
3. **Same Page Meeting.** First `git init` if there is no repo yet and commit the planning artifacts (the snapshot discipline in the invocation contract depends on git existing before the FIRST Codex call). Then run the meeting per [SAME-PAGE-MEETING.md](SAME-PAGE-MEETING.md). Codex reviews read-only, you argue back, bounded rounds, verdict line, full log.
   - Completion: `VERDICT: SAME PAGE` in `SAME-PAGE-LOG.md`, or user override recorded.
4. **The Integrator builds.** For each rock in order, hand the Integrator a frozen build contract: Codex per [CODEX-INTEGRATOR.md](CODEX-INTEGRATOR.md), or an Opus integrator per "Opus integrator mechanics" below. Baseline first: `git init` if there is no repo yet, and commit the planning artifacts before the first rock, so `git status` is empty when the Integrator launches and the build diff is exactly its work. While it runs, you do not touch the code (Rule 2).
   - **Self-critique clause (XELAH amendment, 2026-07-28, the Autonomous Loop).** Any rock with a user-facing surface (a page, a UI, an export, a rendered anything) gets the CRITIQUE line in its contract: the Integrator renders its own output the way a user would experience it, critiques it hostilely in writing, fixes what it finds, adds one deliberate refinement, and includes the critique + evidence in its report. A report without the critique on a surfaced rock is incomplete and goes back before review.
   - Completion: the Integrator reports files changed + proof output for the rock (+ critique and evidence where the clause applies).
5. **Level 10 Review** (per rock). Delegate the full-diff read to a Sonnet reviewer subagent (XELAH amendment, 2026-07-25): it reads `git diff` in its own context and reports structured findings against the contract and the smell vocabulary (duplicated code, mysterious names, feature envy, message chains, speculative generality), citing file:line for each. You adjudicate each finding accept or reject with a reason (Rule 5, verbatim log), and you run the proof command yourself: neither the Integrator's pasted output nor the reviewer's summary counts as proof. This review is the director seat of the Autonomous Loop (XELAH amendment, 2026-07-28): where the rock has a rendered surface, the reviewer also reads the Integrator's self-critique evidence, and your proof includes experiencing the artifact itself (load the page, open the export), because a green build is not a green render. Classify every accepted finding introduced-by-this-rock vs pre-existing-made-reachable (XELAH amendment, 2026-07-28): the rock ships when it strictly improves the prior state; a pre-existing defect gets re-contracted as its own issue, never absorbed into the rock and never allowed to block it. Re-run the clean-tree check when the rock finishes, not only at launch, and attribute any non-Integrator drift before committing. Anything off-track goes back as a fix round (resume the same Integrator session, max 2). At the cap, takeover is pre-authorised (Alex, 2026-07-25) and bounded: if the residue is last-mile, finish it yourself; if the diff is unsalvageable, revert the rock, rewrite the contract, re-run the Integrator once, and still failing, stop and flag `needs-alex`. Never rebuild a rock inline. (Lane 2: one review; you may fix any residue yourself.)
   - Completion: proof passes when YOU run it.
6. **Close the meeting.** Report the scorecard: rocks done / total, proof results, deviations accepted, issues deferred. Offer the commit. Code commits are user-gated and orchestrator-authored (the planning-artifact baseline commits are machinery: announced, not asked); the Integrator never commits.
7. **Cold review + rules conversion (XELAH amendment, 2026-07-28, all four functions).** Before ending the run, scan its findings and fix rounds cold. Any defect class that appeared twice or more converts to a binding rule now, not next time: run/project-scoped rules go into a `## STANDARDS` section of `VTO.md` (or the wave's `BRIEFS.md`) so every later rock inherits them; rules durable beyond the project append to `~/xelah-brain/_learning/skill-addenda/rocket-fuel.md` below its marker line `<!-- NEW LESSONS BELOW: folded weekly into SYSTEM/Wiki/live/rule-register.md -->` (the source; the vault copy is a sync-brain mirror, never the write target) followed by `~/.claude/scripts/sync-brain.sh` and `~/.claude/scripts/sync-skills.sh` (auto-binding on the next invocation; the weekly `/lessons-cluster-scan` fold turns them into rule-register rows and regenerates the checklist). Autonomous, reported in the scorecard, never gated. OS-level rules (CLAUDE.md, guardrails, hooks) are proposed to the user, never auto-written.

## Function 2: CLARITY BREAK (refactor an existing repo)

1. **State of the Company.** Audit solo: map the architecture, name the smells with the vocabulary above, find the debt and the risks. Write `ISSUES.md`, the wish-list dump: every issue on one line with impact (high/med/low). This is the Visionary's 32-item list; do not filter yet.
2. **Pick the Rocks.** Show the user the list ranked by impact. They pick (or you propose and they confirm) 3 to 7 for this cycle. When everything is important, nothing is important.
3. **Refactor plan.** Write `PLAN.md`: per rock, the files touched, the approach, and the proof (existing tests must stay green; name the command).
4. **Same Page Meeting** on the refactor plan (Codex read-only: what breaks, what is simpler, what is not worth it).
5. **Build + Level 10 Review** exactly as KICKOFF steps 4 to 6.

## Function 3: SAME PAGE MEETING (standalone)

The file the user pointed at becomes the canonical plan file for the whole run: every revision lands in THAT file, not in a new `PLAN.md`. If it lives outside the repo, copy it in as `RF-PLAN.md` first (Codex reviews inside the repo; the original stays untouched). Run [SAME-PAGE-MEETING.md](SAME-PAGE-MEETING.md) against it, deliver the verdict, the revised plan, and the log. Then offer, once: continue into BUILD A ROCK, or stop here.

## Function 4: BUILD A ROCK (standalone)

If the user handed you a frozen spec file, use it. If they handed you a sentence, write the one-rock contract yourself (GOAL, SPEC, KEY PATHS, CONSTRAINTS, NON-GOALS, PROOF, CRITIQUE where surfaced) and show it in one message before launching. Then: clean-tree gate, build, Level 10 review, user-gated commit, per [CODEX-INTEGRATOR.md](CODEX-INTEGRATOR.md) or "Opus integrator mechanics" below.

## Function 5: RUN A WAVE (XELAH amendment, 2026-07-28, the Autonomous Loop)

Many units of the same kind, one constitution, no human between the two gates. The user approves the wave brief (intent) and the ship; everything between runs itself. Full protocol: the vault article [[autonomous-loop]].

1. **The constitution.** Write `BRIEFS.md`: the non-negotiable standards every unit must meet (real copy, accessibility, responsiveness, zero console errors, the project's own bars), plus a `## STANDARDS` section that grows as rules convert mid-wave. One Grill pass with the user to set the wave's scope, count, and axes; this is the intent gate.
2. **The registry.** In `BRIEFS.md`, one line per unit recording its axes (domain, technique, palette, type, mood, or whatever axes fit the medium). Every new unit's brief may share AT MOST ONE axis with anything already in the registry. Convergence is the failure mode; the registry is the immune system.
3. **Per-unit briefs.** For each unit, a frozen contract per [CODEX-INTEGRATOR.md](CODEX-INTEGRATOR.md): concept, its axes, its proof, and the CRITIQUE clause always on. Builders get autonomy inside the brief, never over the constitution.
4. **Assets stay with the director.** You generate/source all media and hand paths to builders: one queue, one budget, no cap collisions. Builders never mint assets.
5. **Fan out.** Parallel Integrator sessions (Codex, or one Opus integrator per unit, each in its own worktree), each owning one unit's folder, no shared components between units. Before launching, write the file-ownership map (which unit owns which paths) and the intended merge order into EVERY sibling contract, so no collision is possible and a fresh session can take over mid-flight. Stagger render passes: headless-Chrome screenshots hang under parallel load and on autoplaying loop videos; when a screenshot path is known to hang, the unit verifies via built-DOM fetch instead.
6. **Per unit: three-pass critique, then director review.** The builder renders, critiques hostilely, refines plus one deliberate upgrade, reports with evidence. You review the evidence, run the proof, send specific fix feedback (max 2 fix rounds, then the Level 10 takeover bounds).
7. **Rules convert mid-wave.** A defect class seen twice anywhere in the wave becomes a `## STANDARDS` rule immediately; later units build under rules earlier units paid for. Durable rules follow the cold-review step's addenda path (brain-side write below the marker, then the two syncs).
8. **Close.** Scorecard across units, registry final state, rules minted, then the ship gate: nothing deploys or publishes without the user.

## Opus integrator mechanics

How a Lane 3 or contracted Lane 2 build runs when the Integrator seat is an Opus subagent instead of Codex. Same contract, same gates, same proof rules; only the transport changes. The Codex-down fallback uses this path.

1. **Workspace.** Dispatch with the Agent tool, `model: "opus"` (`"fable"` only by HR20 escalation: a fix that already failed once, or a Lane 3 design call), `isolation: "worktree"`; or create an explicit git worktree under `~/worktrees/<slug>` and name it in KEY PATHS ("work only inside <path>"). hpa-portal builds use the `ship.sh new` worktree per portal-ship. The clean-tree gate applies unchanged: `git status --porcelain` is empty at launch. Plan-gate binds subagents too: the approved plan exists before dispatch.
2. **The brief is the frozen contract pasted in full** (the template in [CODEX-INTEGRATOR.md](CODEX-INTEGRATOR.md): GOAL, SPEC with the BLOCKED rule, KEY PATHS, CONSTRAINTS, NON-GOALS, PROOF, CRITIQUE where surfaced, DATA FACTS where served, OUTPUT with the DRIFT INVENTORY), never a summary or a pointer to it. Fixed lines appended to every brief: the three-pass rule (render the real artifact, hostile written critique, refine plus one deliberate upgrade, HR21); "A permission or confirmation prompt is a gate, never an obstacle: report a denial as a result, never bypass it (guardrail 28)"; "A blocked write stays blocked: never re-route it through Bash, Python, or any other tool; stop and report (guardrail 41)"; "Do not commit, push, or deploy."
3. **The report has the shape Codex returns:** files changed (path + what/why), the full proof output, critique + evidence when CRITIQUE applies, the drift inventory, or `BLOCKED: <reason>`. Save it verbatim with the agent id to the run home (`~/.hpa-verify/<run>/`).
4. **Review and proof are unchanged.** Sonnet reviewer on the full diff in the worktree, you adjudicate each finding, you run the proof yourself and experience any surfaced artifact; the integrator's pasted output is a claim, never proof.
5. **Fix rounds** go to the SAME agent via SendMessage with its id (the analogue of resuming the Codex thread): one consolidated prompt, "fix these N items, nothing else, re-run the proof." Max 2, then the Level 10 takeover bounds. If the agent is gone (session restart), dispatch a fresh Opus integrator with the full contract plus the explicit list of what remains, and say so in one line.
6. **Same Page reviewer.** Codex when it is up (cross-model review is the point). Codex down: a fresh Opus subagent with no Edit/Write tools (`subagent_type: "Plan"`, with the snapshot check around it as for a Codex review) runs the same review prompt and must end with the same verdict line.

## Anti-patterns (the tells)

| Anti-pattern | The tell | The rule it breaks |
|---|---|---|
| End Run | "just quickly change X" mid-build, and you reach for Edit | Rule 2: onto the Issues List |
| Fake approval | accepting Integrator output with no `VERDICT:` line found | Rule 4: no verdict, no approval |
| Consensus trap | round 4+ re-litigating a settled point | Rule 4: cap it, user decides |
| 100-pound bag | PLAN.md with 8+ rocks | Do less better: cut to 7 max |
| Genius with a thousand helpers | the orchestrator implementing Lane 3 rocks itself because "it's faster" | The Chart: wrong seat |
| Trusting the intern's demo | claiming done off the Integrator's pasted proof | Level 10: run it yourself |

## Artifacts

`VTO.md` (kickoff only) · `PLAN.md` (the what) · `SAME-PAGE-LOG.md` (the why, round by round) · `ISSUES.md` (clarity break + every deferred end-run). All at repo root, committed as a baseline before the first build.

**Collision rule:** before writing any artifact, check whether that filename already exists with unrelated content. If it does, prefix every artifact this run with `RF-` (`RF-PLAN.md`, `RF-ISSUES.md`, ...) and tell the user in one line. Never overwrite a file this skill did not write. Everywhere this skill names `PLAN.md`, `ISSUES.md`, or the log, it means the run's ACTUAL paths after this rule.

---
Based on the Visionary/Integrator operating system from *Rocket Fuel* by Gino Wickman and Mark C. Winters.
