# `/speckit.flowing` — Run the SpecKit flow

> This is the **second** command. It runs the SpecKit work once the thinking documents and the
> spec already exist.
> It uses the documents written by `/speckit.thinking` (see
> `spec-driven-development-thinking.md`).
> When `analyze`/`clarify` raises a question, this command hands it to **`/speckit.answering`**,
> not to you (the answering job is described in `spec-driven-development-answering.md`).

---

## What this command does

The `/speckit.flowing` command has three jobs:

- **(a) Drive the build to "done"** — make the Jira tickets, run the **spec-loops** (below),
  then `implement`, verify the code against the thinking documents, and review — checking each
  made file yourself before moving on.
- **(b) Feed in the thinking documents** — the BRD and Design Document already went into
  `specify`; here the **BDD, TDD (with Red/Green), and DoD** check the code at the end.
- **(c) Route every clarification question to `/speckit.answering`, not the human.** When
  `analyze`/`clarify` raises a question, this command writes it into the shared `questions.md`
  mailbox and hands it to **`/speckit.answering`**, which answers it from the documents. Only an
  item the documents cannot answer (`NEEDS-HUMAN`) reaches you.

---

## Before you start (prerequisites)

Flowing begins from an existing `spec.md` — it does **not** write the thinking documents or run
`specify`. **Do not start flowing until all of the required steps below are done.** If you are
asked to flow and `spec.md` (or the thinking documents) is missing, stop and run the
prerequisites first — flowing has nothing to harden otherwise.

| Order | Command | Required? | What it produces |
|-------|---------|-----------|------------------|
| 1 | **`/speckit.thinking <feature>`** | **Required** | The thinking documents in `specs/<NNN-feature-slug>/`: `brd.md`, `design.md`, `bdd.md`, `tdd.md`, `dod.md`, and the `questions.md` mailbox. |
| 2 | **`/speckit.constitution`** | *Optional* | The project rules in `.specify/memory/constitution.md`. Every plan/tasks check uses it **if present**. |
| 3 | **`/speckit.specify`** | **Required** | `spec.md`, built from `brd.md` + `design.md`. **This is where flowing starts.** |
| 4 | **`/speckit.checklist <area>`** | *Optional* | "Unit tests for the spec" — checks the *needs* are complete, clear, and measurable. |

- The **thinking documents come only from `/speckit.thinking`** — `brd.md`, `design.md`,
  `bdd.md`, `tdd.md`, `dod.md`, and the `questions.md` mailbox. Flowing reads them; it never
  writes them. If they change, they change through the thinking command, not here.
- **`/speckit.constitution` and `/speckit.checklist` are optional** — skip them if you have a
  reason to, but if a constitution exists it is enforced by every plan and tasks check.
- SpecKit itself — skills, templates, config — must already be installed in the project.

**Required integration (configure once per project):**

- **Jira** — flowing creates the tickets (step 1) through a real Jira integration. Configure the
  connection and project mapping before flowing. If Jira is not configured, flowing does **not**
  stop immediately — it **first asks whether to skip Jira**. If you choose to skip, flowing
  continues without creating tickets; otherwise it stops at step 1 with a clear message so you
  can configure Jira.

`NEEDS-HUMAN` questions need **no** integration — flowing asks the human **directly in Claude
Code** (no external service). See *How questions work* below.

---

## The idea in one picture

```
        PREREQUISITES  —  done before flowing (see the table above)
   ══════════════════════════════════════════════════════════════════════
   /speckit.thinking → [/speckit.constitution] → /speckit.specify → [/speckit.checklist]
   ⇒  spec.md  +  thinking docs:  brd · design · bdd · tdd · dod · questions
                                    │
                                    ▼   run  /speckit.flowing
   ═══════════════  PART 1 · TICKETS + SPEC  ════════════════════════════
   [1] preflight — are spec.md AND the thinking docs present?
          ├─ NO  →  STOP: run the missing prerequisite first
          └─ YES →  create Jira tickets  ← brd.md (Stories) + design.md (Tasks)
                    Jira not configured?  ask "skip Jira?"  →  skip | STOP
   [2] ac-loop on spec.md:   ( analyze → clarify )  ↺  until CLEAN  (min 3×)
                                    │
                                    ▼
   ═══════════════  PART 2 · PLAN  ══════════════════════════════════════
   [3] /speckit.plan  →  plan.md  (+ research, data-model, contracts)
       acp-loop:   ( analyze(plan.md) → clarify )×2 → plan   ↺  until CLEAN (min 3×)
                                    │
                                    ▼
   ═══════════════  PART 3 · TASKS  ═════════════════════════════════════
   [4] /speckit.tasks  (input: Jira tickets)  →  tasks.md
       acpt-loop:  ( analyze(tasks.md) → clarify )×2 → plan → tasks  ↺  CLEAN (min 3×)
                                    │
                                    ▼
   ═════════  PART 4 + 5 · IMPLEMENT · VERIFY · REVIEW  ══════════════════
   one re-entrant loop — any bug/gap sends you back to [5]

    ┌─▶ [5]  implement → analyze(with implementation) → clarify
    │   [6]  generate tests (tdd.md) DURING implement → run → Red ⇒ Green
    │   [7]  verify code vs  dod · design · brd · bdd · tdd · questions
    │   [8]  /review  →  [9] /code-review  →  [10] repeat reviews (≥2×)
    │              │
    │        any bug / gap?  ─── NO ──▶  all gates CLEAN  ──▶  ✅ DONE
    │              │ YES
    │              ▼
    └── RECOVERY:  copy finding → analyze(finding) → clarify → acpt-loop  (back to [5])

   CLEAN = analyze: no bugs / no gaps   AND   clarify: no open questions
```

---

## How questions work (the key part)

Whenever **`/speckit.analyze`** finds problems or **`/speckit.clarify`** asks questions, those
questions are **not yours to answer.** They move through `specs/<slug>/questions.md` — a shared,
append-only **mailbox**. The file is the **truth**; the message you get back is only a copy of
the latest round. This is the flowing side of the protocol that
`spec-driven-development-answering.md` describes; both documents must say the same thing in the
same terms. (`/speckit.answering` is the **answering job of `/speckit.thinking`** — that is why
its rounds are tagged `by: thinking`.)

**Turn `analyze` findings into fixes.** `/speckit.analyze` reports **defects** — missing,
contradictory, untraceable, or unmeasurable items — not questions, and `/speckit.clarify` scans
the spec on its own, so nothing closes an analyze finding by itself. Route each finding **by its
cause**: a **spec-level** finding (something `spec.md` is missing or contradicts) is restated as a
question — *"how should `spec.md` resolve `<finding>`?"* — sent through the mailbox below, and
`/speckit.clarify` writes the answer into `spec.md`; a **made-file-level** finding (a
`plan.md`/`tasks.md` item that just doesn't match an already-correct spec) needs **no** question —
the loop **regenerates** that file, which closes it (and if regeneration doesn't clear it, it was
spec-level after all, so route it as a question). A loop is only **clean** once analyze has **no
findings left**.

**Flowing wraps the stock clarify — it does not fork it.** `/speckit.clarify` is normally
interactive (it asks the human up to five questions and writes the answers into `spec.md`).
Flowing keeps that command unchanged and puts a wrapper around it: it **intercepts** each
question clarify would put to the human, writes it as a `PENDING` round, calls
`/speckit.answering`, and feeds the `ANSWERED` answers back into clarify **as if they were the
human's replies** — so clarify encodes them into `spec.md` exactly as usual. (Because stock
clarify asks **one adaptive question at a time**, run one `PENDING`/`ANSWERED` round **per
question**, not a fixed batch.) Only `NEEDS-HUMAN` items ever reach a person, asked **directly in
Claude Code**.

**Both `/speckit.clarify` and `/speckit.flowing` must wait for `/speckit.answering` to answer.**
Nothing advances while a round is still `PENDING`: clarify holds each question open until its
answer comes back, and flowing does not move to the next step until `/speckit.answering` has
appended the matching `ANSWERED` round to `questions.md`. No proceeding on an unanswered
question, and no guessing while you wait.

1. **Do not** answer questions yourself, and **do not** ask the human first.
2. **Write a `PENDING` round** into `specs/<slug>/questions.md` — a fixed header plus numbered
   `Q1, Q2…`:
   ```markdown
   ## Round 2 — 2026-06-29 — status: PENDING — asked-by: flowing
   Q1: What is the session retry limit?
   Q2: Which roles can delete an account?
   ```
3. **Hand the round to `/speckit.answering`** (invoke it in the **main thread** via `SlashCommand`,
   not a background subagent — it may need to ask the human). It reads `specs/<slug>/*`, answers
   **only from the documents** with a source tag for each answer, and **appends a matching
   `ANSWERED` round** (`A1, A2…` aligned to the `Q#`):
   ```markdown
   ## Round 2 — 2026-06-29 — status: ANSWERED — by: thinking
   A1: 3 attempts, then a 15-minute lock.   [source: design.md §Failure handling]
   A2: Owner and Admin only.                [source: brd.md R7]
   ```
4. **Re-read `questions.md`** — not just the returned message — and feed the `ANSWERED` items
   into **`/speckit.clarify`** so they are written into `spec.md`.
5. Items marked **`NEEDS-HUMAN`** (or **`CONFLICT / NEEDS-HUMAN`**) are the only ones that reach
   a person, asked **directly in Claude Code**, one question at a time. Because you call
   `/speckit.answering` in the main thread, it asks the human, writes the answer into the right
   document **first**, and returns a follow-up resolution round (`N+1`). If instead it **returns
   unresolved `NEEDS-HUMAN` items** (it ran where it couldn't reach the human), **ask the human
   yourself** and feed the reply back through `/speckit.answering` so the documents are updated
   first. Feed only **resolved** answers into `/speckit.clarify`; a still-open `NEEDS-HUMAN` waits
   for its `N+1` round (see `spec-driven-development-answering.md`).

Every round stays in `questions.md`, so you always have a record of what was asked and how it
was resolved.

> **Where `/speckit.answering` fits.** Any time you read "→ clarify" or "→ analyze" in the steps
> below, the same rule applies: if a question comes up, route it through `/speckit.answering` via
> the `questions.md` mailbox. It is not a separate step you have to remember — it is how every
> analyze/clarify in this whole flow handles its questions.

---

## Spec-loop strategies

The flow below is built from three named loops. Each runs **analyze first** (so clarify asks
about real problems, not guesses), routes every question through `/speckit.answering`, and keeps
**repeating its cycle until a full pass of analyze and clarify returns no bugs and no gaps**.
Run each loop a **minimum of 3 cycles**, even if an early cycle looks clean — later passes catch
what early ones miss. "Clean" means analyze reports **no bugs and no gaps**, and clarify has
**no open questions**.

### `ac-loop` — spec only
```
cycle = ( analyze → clarify )            (repeat the cycle until clean; min 3×)
```
The lightest loop: tighten `spec.md` alone, before any plan or tasks exist. Used right after the
Jira tickets are made. **The cycle is `analyze → clarify`**, repeated until clean.

### `acp-loop` — spec + plan
```
cycle = ( (analyze → clarify) ×2 → plan )        (repeat the cycle until clean; min 3×)
```
- **The cycle is `(analyze → clarify) ×2 → plan`.** Run **analyze → clarify twice** at the top
  of each cycle, then regenerate the **plan**, then **run the cycle again**.
- **Skip `plan`** in a cycle when clarify **did not change `spec.md`** — there is nothing new
  for the plan to absorb, so re-running it would only churn the file. Fix the cause (the spec),
  not the result (the plan).

### `acpt-loop` — spec + plan + tasks
```
cycle = ( (analyze → clarify) ×2 → plan → tasks )        (repeat the cycle until clean; min 3×)
```
- **The cycle is `(analyze → clarify) ×2 → plan → tasks`.** Run **analyze → clarify twice**,
  then regenerate **plan**, then **tasks**, then **run the cycle again**.
- **Skip `plan`** when clarify **did not change `spec.md`** (same rule as `acp-loop`).
- **Skip `tasks`** when `plan` was **skipped** *or* `plan` **did not change `plan.md`** — with no
  plan change there are no new tasks to regenerate.

**Skip-rule summary (per cycle):**

| In this cycle, clarify changed `spec.md`? | Re-run `plan`? | plan changed `plan.md`? | Re-run `tasks`? |
| ----------------------------------------- | -------------- | ----------------------- | --------------- |
| yes                                       | yes            | yes                     | yes             |
| yes                                       | yes            | no                      | no              |
| no                                        | no             | n/a                     | no              |

**Skip-rule exception:** if `/speckit.analyze` flags a **made-file-level** defect in
`plan.md`/`tasks.md`, regenerate that file this cycle **even if the skip rule would skip it** — a
generation error won't fix itself, and an unchanged spec doesn't prove the made file is correct.

In every loop the made files are **regenerated, never hand-patched**: if the plan or tasks are
wrong, fix the spec and make them again.

---

## Part 1 — Tickets and the spec

Part 1 is where **`/speckit.flowing` actually begins** — it picks up exactly where the picture
above hands off, **right after the prerequisites**
(`/speckit.thinking → [/speckit.constitution] → /speckit.specify → [/speckit.checklist]`) have
produced `spec.md` and the thinking docs, and after the **Jira integration** is configured — or
you have chosen to skip Jira. None of that is re-run here; flowing only
**reads** those inputs. Because everything downstream depends on them, Part 1 **opens by
re-confirming they are all present** (the preflight in step 1) before doing any work.

### 1. Make the Jira tickets — the first step
**Preflight (hard stop).** Before doing anything else, re-confirm the entry conditions from
*Before you start* are met: **`spec.md`** and **all the thinking documents** (`brd.md`,
`design.md`, `bdd.md`, `tdd.md`, `dod.md`, `questions.md`) exist. If any is missing, **STOP the
whole process here in step 1** and run the missing prerequisite first (`/speckit.specify` for a
missing `spec.md`, `/speckit.thinking` for missing thinking docs). Also confirm the base SpecKit
commands this flow calls are installed — `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`,
`/speckit.analyze`, `/speckit.clarify`, `/speckit.answering`, and `/speckit.thinking`; if any is
missing, **STOP** and have it installed first. Do not create tickets or run any loop until they
are all present.

**What it is.** Turn the agreed scope into real work. This is the **very first action of
flowing**. Make **Jira** tickets from **`brd.md`** (the business parts) and **`design.md`** (the
technical parts) through the configured Jira integration, so the plan and tasks below map onto
real tickets.
- Cover both: business needs from `brd.md` (→ Stories), technical work from `design.md`
  (→ Tasks/Sub-tasks).
- Carry the SpecKit source ref (e.g. `brd.md R3`, `design.md §Contracts`) onto each ticket so it
  is traceable back to the documents.
- Keep the ticket IDs — `tasks.md` (Part 3) is matched back to these tickets.
- Detect whether Jira is configured by the presence of a Jira MCP tool or the project's Jira
  settings. If it is **not** configured, **ask first whether to skip Jira**: on **skip**, continue
  without tickets; otherwise **stop here** and report it. Do not silently skip ticket creation.

### 2. Run the `ac-loop` on the spec
**What it is.** With the tickets in place, run the **`ac-loop`** (cycle = `analyze → clarify`)
on `spec.md` until analyze and clarify find no bugs and no gaps — **at least 3 cycles**. This is
where the spec is hardened before any plan exists. Route every question through
`/speckit.answering`.

---

## Part 2 — Plan

### 3. Run plan, then the `acp-loop`
**What it is.** Run **`/speckit.plan`** to make `plan.md` plus the research, data-model, and
contract files (checked against the constitution rules if present). Then:
1. Run **`/speckit.analyze` with `plan.md`**.
2. Run **`/speckit.clarify`** (questions → `/speckit.answering`).
3. Enter the **`acp-loop`** until it comes back clean (min 3 cycles).

---

## Part 3 — Tasks

### 4. Run tasks, then the `acpt-loop`
**What it is.** Run **`/speckit.tasks`** with the **Jira tickets from Part 1 as input** to make
`tasks.md` — ordered, grouped by user story, and matched back to those tickets. Then:
1. Run **`/speckit.analyze` with `tasks.md`**.
2. Run **`/speckit.clarify`** (questions → `/speckit.answering`).
3. Enter the **`acpt-loop`** until it comes back clean (min 3 cycles).

---

## Part 4 — Implement and verify

Parts 4 and 5 are **one re-entrant loop.** Every check from here on — implement, tests, verify,
`/review`, `/code-review` — feeds the same **recovery path**:

> **Recovery path** — whenever a check turns up a bug or gap, route it **by cause.** If it traces
> to a **spec gap**, turn it into a question through the mailbox (→ **`/speckit.answering`**) and
> let **`/speckit.clarify`** apply the answer to `spec.md`. If it is a **made-file or code** error
> against an already-correct spec, let the **`acpt-loop`** regenerate plan/tasks and **step 5**
> re-implement it. Either way, run the **`acpt-loop`** until clean, then **start again from step 5
> (`implement`).** Fix from the spec down, never as a quick code patch.

### 5. Run implement, then analyze and clarify
**What it is.** Run **`/speckit.implement`** to do the tasks in `tasks.md` in order. Then run
**`/speckit.analyze`** on the implementation (against the spec, plan, and tasks) and
**`/speckit.clarify`**.
- If they return **no bugs and no gaps** → continue to step 6.
- If they return **any bug or gap** → take the **recovery path** above (which lands you back
  here, at step 5).

### 6. Generate the tests from `tdd.md` and run them
**What it is.** During implementation, turn the plan in **`tdd.md`** into **real test files** in
the right place (TDD), then **run them**. This is where the Markdown Red/Green plan becomes
actual test code. Note `tdd.md` is a thinking doc, **not** part of `tasks.md`, so **flowing writes
these test files itself** — they are not produced by `/speckit.implement`'s task execution.
- The tests marked **Red** in `tdd.md` must be **Green** now.
- If any test exposes a bug or gap, **copy the failing test's bug/gap text** and take the
  **recovery path** (back to step 5).

### 7. Verify the code against the thinking documents
**What it is.** Check the running implementation, document by document, for any inconsistency with
the six thinking documents. Go through them **one at a time** — do not spot-check — and for each
one confirm the code matches what the document records. Check them in this order:

- **`brd.md` (why & what).** Walk every numbered business item — goals (`G#`), scope (`S#`),
  functional requirements (`R#`), business rules (`BR#`), and success metrics (`M#`). Each must
  be **actually built** in the code: every `R#` implemented, every `BR#` enforced, nothing that
  is **out of scope** (`S#` "out") slipped in, and every `M#` measurable from what shipped. A
  requirement with no corresponding code is a **gap**.
- **`design.md` (how).** For each design decision, confirm the code uses the **named parts,
  contracts (API/events), and data model** the document specifies — same endpoints, payloads,
  field names, and error shapes — and that the **kill-switch / safe-off** path exists if the
  design calls for one. Code that diverges from the design (different contract, missing toggle)
  is a **bug**.
- **`bdd.md` (Given/When/Then).** Take each scenario and its requirement tag (e.g. `@R3`) and
  confirm the code produces exactly that behaviour — the **normal path and every edge case**.
  Each Given/When/Then should map to a real, passing test or an observable behaviour. A scenario
  with no matching behaviour is a **gap**.
- **`tdd.md` (test plan + Red/Green).** Confirm every planned test exists as a **real test file**
  (step 6), sits in the file the plan names, and that each test marked **Red** is now **Green**.
  A missing or still-**Red** test is a **bug**.
- **`dod.md` (done gates).** Go gate by gate: each must be **honestly met and evidenced**, or
  explicitly **shown as not done** — never assumed. An unmet or unverifiable gate blocks "done".
- **`questions.md` (recorded decisions).** Read every `ANSWERED` round and confirm the code
  honours each recorded decision (values, roles, limits, conflicts resolved). Nothing in the
  implementation may **contradict** a decision written here.

On **any** inconsistency in **any** document, copy the exact bug/gap text and take the **recovery
path** (back to step 5).

---

## Part 5 — Review and finish

### 8. Run /review
**What it is.** **`/review`** checks the code against the spec for correct and safe contracts.
If it returns bugs or gaps, **copy them** and take the **recovery path** (back to step 5).

### 9. Run /code-review
**What it is.** **`/code-review`** checks the diff for bugs and clean-up. If it returns bugs or
gaps, **copy them** and take the **recovery path** (back to step 5).

### 10. Loop the reviews until clean
**What it is.** Run **`/review` → `/code-review` → `/review`** and keep cycling until both come
back with no bugs and no gaps — the full review cycle a **minimum of 2 times**. The goal is a
clean result after many loops, not after one try. Any bug/gap from either review re-enters the
recovery path and restarts at step 5; the reviews only "pass" when a full `/review` **and**
`/code-review` both come back clean.

---

## Cheat sheet

| Part | Step | Name          | Command / loop                                                                                          |
| ---- | ---- | ------------- | ------------------------------------------------------------------------------------------------------- |
| pre  | —    | thinking docs | `/speckit.thinking <feature>` *(required)*                                                              |
| pre  | —    | constitution  | `/speckit.constitution` *(optional)*                                                                    |
| pre  | —    | spec          | `/speckit.specify` (give `brd.md` + `design.md`) *(required)*                                           |
| pre  | —    | checklist     | `/speckit.checklist <area>` *(optional)*                                                                |
| 1    | 1    | Jira tickets  | from `brd.md` + `design.md` (first step)                                                                |
| 1    | 2    | Harden spec   | **ac-loop**: cycle = `analyze → clarify` (min 3×)                                                       |
| 2    | 3    | Plan          | `/speckit.plan` → analyze(`plan.md`) → clarify → **acp-loop** (skip plan if spec unchanged)             |
| 3    | 4    | Tasks         | `/speckit.tasks` (input: Jira tickets) → analyze(`tasks.md`) → clarify → **acpt-loop** (skip per rules) |
| 4    | 5    | Implement     | `/speckit.implement` → analyze(with implementation) → clarify                                           |
| 4    | 6    | Tests         | generate from `tdd.md` **during implement** → run → Red must be Green                                   |
| 4    | 7    | Verify        | code vs `dod/design/brd/bdd/tdd/questions` (document by document)                                       |
| 5    | 8    | Review        | `/review` → copy any finding → recovery                                                                 |
| 5    | 9    | Code-review   | `/code-review` → copy any finding → recovery                                                            |
| 5    | 10   | Loop reviews  | `/review` → `/code-review` → `/review` (full cycle min 2×)                                              |
| 4–5  | rec  | Recovery path | any bug/gap → copy → `/speckit.analyze` → clarify → **acpt-loop** → back to step 5                      |

---

## Things to remember

1. **Prerequisites first.** Thinking docs, (optional) constitution, `specify`, (optional)
   checklist all happen **before** flowing; flowing starts from `spec.md`. Don't flow until
   `spec.md` and the thinking documents exist.
2. **Three loops, analyze first.** `ac-loop` hardens the spec, `acp-loop` adds the plan,
   `acpt-loop` adds the tasks — each repeats until analyze and clarify are clean, minimum 3×,
   with the skip rules for unchanged `spec.md`/`plan.md`.
3. **Questions go to `/speckit.answering`** — flowing wraps stock clarify and routes its
   questions through the `questions.md` mailbox; never answer from memory. The human only sees
   `NEEDS-HUMAN` items (asked directly in Claude Code), and the answer is written back into the
   documents first.
4. **The made files are easy to throw away. The spec is the source.** Fix the spec, then
   regenerate plan and tasks — never hand-patch them.
5. **One recovery path, always back to implement.** Tests, verification, `/review`, and
   `/code-review` all funnel any bug/gap through the `acpt-loop` and restart at Part 4, step 5.
6. **Clean after many loops, not after one try.** The DoD says "done", not your feeling.
