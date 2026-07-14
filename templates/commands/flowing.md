---
description: Drive a feature from spec.md to done — Jira tickets, ac/acp/acpt spec-loops, implement, verify vs the thinking docs, then /review + /code-review. Routes every question to the answering command.
argument-hint: <NNN-feature-slug>
allowed-tools: Read, Write, Edit, Grep, Glob, Bash, Task, SlashCommand, AskUserQuestion
---

## User Input

```text
$ARGUMENTS
```

The input is the **feature slug** whose `specs/<NNN-feature-slug>/` folder holds the spec and the
thinking documents. If empty, ask the human which feature to flow.

## Role

You are the **`__SPECKIT_COMMAND_FLOWING__`** command — the **second** command. You turn the existing
`spec.md` + thinking documents into working, verified, reviewed code. You **never** write the
thinking documents or run `__SPECKIT_COMMAND_SPECIFY__`; you only **read** them. Prefer fixing the **spec**
and regenerating plan/tasks over hand-patching made files.

Companion human-readable guide: `spec-driven-development-flowing.md`.

## Step 0 — Preflight (hard stop)

Confirm `specs/<slug>/spec.md` **and** all thinking docs (`brd.md`, `design.md`, `bdd.md`,
`tdd.md`, `dod.md`, `questions.md`) exist. If any is missing, **STOP** and tell the human to run
the missing prerequisite first (`__SPECKIT_COMMAND_SPECIFY__` for a missing spec, `__SPECKIT_COMMAND_THINKING__` for
missing thinking docs). Do nothing else until they are all present.

Also confirm the **base SpecKit commands this flow depends on are installed** —
`__SPECKIT_COMMAND_PLAN__`, `__SPECKIT_COMMAND_TASKS__`, `__SPECKIT_COMMAND_IMPLEMENT__`, `__SPECKIT_COMMAND_ANALYZE__`, `__SPECKIT_COMMAND_CLARIFY__`,
`__SPECKIT_COMMAND_ANSWERING__`, and `__SPECKIT_COMMAND_THINKING__` (answering calls it to record human answers). If any
is missing, **STOP** and tell the human to install/complete the SpecKit setup before flowing (you
will call these commands mid-run and cannot proceed without them).

## How questions work (applies to EVERY analyze/clarify below)

Never answer analyze/clarify questions yourself and never ask the human first. Route them through
`specs/<slug>/questions.md` — an append-only mailbox.

**Turn `analyze` findings into fixes (the wire that closes defects).** `analyze` reports
**defects** — missing, contradictory, untraceable, or unmeasurable items — **not** questions, and
`__SPECKIT_COMMAND_CLARIFY__` scans the spec independently, so nothing closes an analyze finding on its own.
Route each finding **by its cause**:
- **Spec-level** (the finding traces to something `spec.md` is missing or contradicts): restate it
  as a specific question — *"how should `spec.md` resolve `<finding>`?"* — and route **that**
  through the mailbox (steps below); `__SPECKIT_COMMAND_CLARIFY__` applies the resolved answer to `spec.md`.
- **Made-file-level** (a `plan.md`/`tasks.md` item that just doesn't match an already-correct
  spec): don't ask a question — the loop **regenerates** that file from the spec, which closes it.
  **If regeneration does not clear the finding, it is actually spec-level** — route it as a
  question instead (the spec truly causes it), so it can't loop forever.

A loop reaches **CLEAN** only once analyze has **no findings left**. `__SPECKIT_COMMAND_CLARIFY__`'s own
ambiguity questions go through the same mailbox.

Mailbox protocol:

1. Write a `PENDING` round: `## Round <n> — <date> — status: PENDING — asked-by: flowing`, then
   `Q1, Q2…` (with `n` = highest existing round + 1, and today's date for `<date>`).
2. Hand it to **`__SPECKIT_COMMAND_ANSWERING__`** and **wait** — do **not** advance while a round is
   `PENDING`. Invoke it with the **`SlashCommand`** tool in the **main thread**, **not** as a
   background `Task` subagent: it may need to ask the human, which a subagent cannot do.
3. Re-read `questions.md` (not just the returned text) and feed each **resolved** answer into
   `__SPECKIT_COMMAND_CLARIFY__` so it is written into `spec.md`. A still-open `NEEDS-HUMAN` is **not**
   resolved yet — it waits for its follow-up round (`N+1`) from `__SPECKIT_COMMAND_ANSWERING__`; do not treat
   it as an answer or advance on it.
4. `__SPECKIT_COMMAND_CLARIFY__` is stock/interactive; flowing **wraps** it — it intercepts each question
   clarify would ask the human, routes it through the mailbox, and feeds the `ANSWERED` answer
   back as the "human" reply. (Stock clarify asks **one adaptive question at a time**, so run one
   `PENDING`/`ANSWERED` round **per question** rather than assuming a fixed batch.)
5. `NEEDS-HUMAN` / `CONFLICT` items are the only ones a person sees. Because you called answering
   in the main thread, it asks the human directly and writes the answer into the documents first;
   if it instead **returns unresolved `NEEDS-HUMAN` items**, **ask the human yourself via the
   `AskUserQuestion` tool** (batched, up to 4 per call, submit UI — never a free-text `?` question
   that ends your turn), then feed the reply back through answering so the documents are updated
   first.

## "CLEAN" and the spec-loops

**CLEAN** = analyze reports **no bugs and no gaps** AND clarify has **no open questions**.

Three loops, each **analyze-first**, each repeated until CLEAN, **minimum 3 cycles** (later
passes catch what early ones miss). If a loop will not converge after many cycles, **stop and ask
the human** rather than loop forever.

- **ac-loop** — cycle = `( analyze → clarify )` on `spec.md`.
- **acp-loop** — cycle = `( (analyze → clarify)×2 → plan )`. **Skip `plan`** in a cycle if clarify
  did **not** change `spec.md`.
- **acpt-loop** — cycle = `( (analyze → clarify)×2 → plan → tasks )`. **Skip `plan`** if clarify
  didn't change `spec.md`; **skip `tasks`** if `plan` was skipped or `plan` didn't change
  `plan.md`.

**Skip-rule exception:** if `analyze` flagged a **made-file-level** defect in `plan.md`/`tasks.md`
(see *How questions work*), **regenerate that file this cycle even if the skip rule would skip
it** — a generation error won't fix itself, and an unchanged spec does not prove the made file is
correct.

Made files (plan, tasks) are **regenerated, never hand-patched** — fix the spec and remake them.

## The flow

### Part 1 — Tickets + spec
1. **Jira tickets (first action).** Create Jira tickets from `brd.md` (business → Stories) and
   `design.md` (technical → Tasks/Sub-tasks), carrying the SpecKit source ref (`R#`, `design §…`)
   for traceability, so plan/tasks map onto real tickets. Create every ticket in the board's
   normal **starting status** (`To Do` / `Backlog`) — do **not** advance it here (status moves come
   later; see *Jira status*). Detect whether Jira is configured by the presence of a Jira MCP tool
   or the project's Jira settings; if absent, treat it as not configured and **ask the human
   whether to skip Jira** — via the `AskUserQuestion` tool (options: *Skip Jira and continue* /
   *Stop and report*), never a turn-ending free-text prompt: on skip, continue without tickets;
   otherwise stop and report. Never silently skip.
2. Run the **ac-loop** on `spec.md` until CLEAN (min 3×).

### Part 2 — Plan
3. Run **`__SPECKIT_COMMAND_PLAN__`** → `plan.md` (+ research, data-model, contracts; checked vs the
   constitution if present). Then **`__SPECKIT_COMMAND_ANALYZE__` with `plan.md`** → **`__SPECKIT_COMMAND_CLARIFY__`** →
   **acp-loop** until CLEAN (min 3×).

### Part 3 — Tasks
4. Run **`__SPECKIT_COMMAND_TASKS__`** (input: the Jira tickets) → `tasks.md`, ordered and matched to the
   tickets. Then **`__SPECKIT_COMMAND_ANALYZE__` with `tasks.md`** → **`__SPECKIT_COMMAND_CLARIFY__`** → **acpt-loop**
   until CLEAN (min 3×).

### Part 4 + 5 — Implement, verify, review (one re-entrant loop)

> **Recovery path** (any bug/gap in steps 5–10): route **by cause**. If the finding traces to a
> **spec gap**, turn it into a question through the mailbox (→ **`__SPECKIT_COMMAND_ANSWERING__`**) and let
> **`__SPECKIT_COMMAND_CLARIFY__`** apply the answer to `spec.md`. If it is a **made-file or code** error
> against an already-correct spec, let the **acpt-loop** regenerate plan/tasks and **step 5**
> re-implement it. Either way, run the **acpt-loop** until CLEAN (its `analyze` re-checks the fix),
> then **restart at step 5**. Fix from the spec down — never a hand-written code patch.

5. **`__SPECKIT_COMMAND_IMPLEMENT__`** the tasks in order — as **each** ticket's work begins, move **that**
   ticket to **`In Progress`** (every ticket you implement, not just the first; see *Jira status*).
   Then **`__SPECKIT_COMMAND_ANALYZE__`** on the implementation
   (against spec, plan, tasks) + **`__SPECKIT_COMMAND_CLARIFY__`**. CLEAN → step 6; any bug/gap → recovery.
6. **Generate the tests** from `tdd.md` into real test files and **run them** — do this **during
   the implement phase**. `tdd.md` is a thinking doc, **not** part of `tasks.md`, so **flowing
   writes these test files itself** (they are not produced by `__SPECKIT_COMMAND_IMPLEMENT__`'s task
   execution). Every test marked **Red** in `tdd.md` must now be **Green**. Any failure → recovery.
7. **Verify vs the thinking documents, one at a time:**
   - `brd.md` — every `G#/S#/R#/BR#/M#` actually built, nothing out-of-scope, metrics measurable.
   - `design.md` — code uses the named parts/contracts/data model + kill-switch; divergence = bug.
   - `bdd.md` — every tagged Given/When/Then (incl. edge cases) holds in the code.
   - `tdd.md` — every planned test exists in its file and every Red is now Green.
   - `dod.md` — each gate honestly met and evidenced, or shown as not done.
   - `questions.md` — the code contradicts no recorded decision.
   Any inconsistency → recovery.
8. Run **`/review`** (Claude Code's built-in review command) to review the code against `spec.md`
   for correct and safe contracts. Findings → recovery.
9. Run **`/code-review`** (reviews the working diff for bugs + clean-up). Findings → recovery.
10. **Loop the reviews** — **`/review` → `/code-review` → `/review`** — until both come back with no
    bugs/gaps, **minimum 2 full cycles**. Any finding re-enters the recovery path (restart at step
    5).

When all loops and both reviews are CLEAN, the feature is **done** — move each completed ticket to
**`Review`** (see *Jira status*), then report a summary (loops run, findings resolved, rounds
logged, review status, tickets created and their final status).

## Jira status

Flowing moves each ticket through exactly this lifecycle and **no further**:

`To Do` / `Backlog` (created, step 1) → `In Progress` (its work begins, step 5) → `Review` (feature
done, all loops + both reviews CLEAN).

**`Review` is the terminal status flowing sets — never advance a ticket beyond it.** Do **not** move
any ticket to `Done`, `Closed`, `Resolved`, `Deployed`, `Ready for Release`, or any post-`Review`
status; a human owns everything after `Review`. Only touch a ticket's status at these three points;
leave it otherwise.

**If Jira was skipped or is not configured** (step 1), there are **no tickets** — so **skip every
status move** (`In Progress` at step 5, `Review` at done) silently; they are no-ops, not errors.

## Remember
1. Preflight first — no `spec.md`/thinking docs, no flow.
2. Analyze-first loops, min 3×, until CLEAN, with the plan/tasks skip rules.
3. Questions go to `__SPECKIT_COMMAND_ANSWERING__` via `questions.md`; wait for `ANSWERED`; `NEEDS-HUMAN` is
   asked directly via the `AskUserQuestion` submit picker (batched, never a turn-ending free-text
   `?`); the answer is written into the documents first.
4. The spec is the source — fix the spec, regenerate plan/tasks; never hand-patch made files.
5. One recovery path — every bug/gap funnels through the acpt-loop and restarts at step 5.
6. Done means the DoD gates pass, not a feeling.
7. Jira status: `To Do`→`In Progress`→`Review` only; **never advance a ticket past `Review`**.
