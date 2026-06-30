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
`specify`. Make sure these are done first:

- **`/speckit.thinking <feature>`** — creates the thinking documents in
  `specs/<NNN-feature-slug>/`: `brd.md`, `design.md`, `bdd.md`, `tdd.md`, `dod.md`, and the
  `questions.md` mailbox. **Required.**
- **`/speckit.constitution`** — writes the project rules in `.specify/memory/constitution.md`.
  *Optional* (but every plan/tasks check uses it if present).
- **`/speckit.specify`** — turns `brd.md` + `design.md` into `spec.md`. **Required** — this is
  where flowing starts.
- **`/speckit.checklist <area>`** — "unit tests for the spec": checks the *needs* are complete,
  clear, and measurable. *Optional*.

(SpecKit itself — skills, templates, config — must already be installed in the project.)

---

## The idea in one picture

```
   prerequisites:  /speckit.thinking → [/speckit.constitution] → /speckit.specify → [/speckit.checklist]
                              │
                              ▼   (this command, /speckit.flowing)
   Jira tickets (design.md + brd.md)
        → ac-loop   (analyze ⇄ clarify ⇄ analyze)                          ✓
        → plan   → analyze(plan.md)  → clarify → acp-loop                   ✓
        → tasks  → analyze(tasks.md) → clarify → acpt-loop                  ✓
        → implement → analyze ⇄ clarify
                    → generate tests (tdd.md) ▸ run tests
                    → verify code vs dod/design/brd/bdd/tdd/questions
        → /review  ▸  /code-review   (loop ×2+)
                              │  (any bug/gap → copy into analyze → acpt-loop → back to implement)
                              ▼
   all loops + reviews clean → done
```

**Main rule:** check each made file **yourself** before the next step. A small mistake early
becomes a big mistake later. The made files (plan, tasks) are easy to throw away — **the spec
is the source.** Fix from the spec down, never as a quick code patch.

---

## How questions work (the key part)

Questions move through `specs/<slug>/questions.md` — a shared, append-only **mailbox**. The
file is the **truth**; the answer you get back is only a copy of the latest round. This is the
flowing side of the protocol that `spec-driven-development-answering.md` describes; both
documents must say the same thing in the same terms. When **`/speckit.analyze`** finds problems
or **`/speckit.clarify`** asks questions:

1. **Do not** answer them yourself, and **do not** ask the human first.
2. **Write a `PENDING` round** into `specs/<slug>/questions.md` — a fixed header plus numbered
   `Q1, Q2…`:
   ```
   ## Round <n> — <date> — status: PENDING — asked-by: flowing
   Q1: …
   Q2: …
   ```
3. **Hand the round to `/speckit.answering`.** It reads `specs/<slug>/*`, answers **only from
   the documents** with a source tag for each answer, and **appends a matching `ANSWERED`
   round** (`A1, A2…` aligned to the `Q#`).
4. **Re-read `questions.md`** — not just the returned message — and feed the `ANSWERED` items
   into **`/speckit.clarify`** so they are written into `spec.md`.
5. Items marked **`NEEDS-HUMAN`** (or **`CONFLICT / NEEDS-HUMAN`**) are the only ones that reach
   a person. Each goes to **Slack as one message per question** (set up once via
   `/speckit.slack`). The human's **first reply** is the answer. `/speckit.answering` **first**
   writes that answer into the right document, and **only then** returns the question+answer to
   flowing as a normal `ANSWERED` item — there is no re-asking, and the answer never disagrees
   with the documents (see `spec-driven-development-answering.md`). If Slack is not configured,
   surface the item to the human directly.

Every round stays in `questions.md`, so you always have a record of what was asked and how it
was resolved.

---

## Spec-loop strategies

The flow below is built from three named loops. Each runs **analyze first** (so clarify asks
about real problems, not guesses), routes every question through `/speckit.answering`, and keeps
going until a full pass of **analyze and clarify returns no bugs and no gaps**. Run each loop a
**minimum of 3 times**, even if an early pass looks clean — later passes catch what early ones
miss.

### `ac-loop` — spec only
```
analyze → clarify → analyze   (repeat until clean; min 3×)
```
The lightest loop: tighten `spec.md` alone.

### `acp-loop` — spec + plan
```
(analyze → clarify) ×2 → plan → analyze   (repeat until clean; min 3×)
```
- **Skip `plan`** in a cycle when clarify **did not change `spec.md`** — there is nothing new
  for the plan to absorb. Fix the cause (the spec), not the result (the plan).

### `acpt-loop` — spec + plan + tasks
```
(analyze → clarify) ×2 → plan → tasks → analyze   (repeat until clean; min 3×)
```
- **Skip `plan`** when clarify **did not change `spec.md`**.
- **Skip `tasks`** when `plan` was skipped **or** `plan` **did not change `plan.md`** — with no
  plan change there are no new tasks to regenerate.

In every loop the made files are regenerated, never hand-patched: if the plan or tasks are
wrong, fix the spec and make them again.

---

## Part 1 — Tickets and the spec

### 1. Make the Jira tickets — the first step
**What it is.** Turn the agreed scope into real work. Make tickets from **`brd.md`** (the
business parts) and **`design.md`** (the technical parts), so the plan and tasks below map onto
real tickets.

### 2. Run the `ac-loop` on the spec
**What it is.** With the tickets in place, run the **`ac-loop`** (`analyze → clarify → analyze`)
on `spec.md` until analyze and clarify find no bugs and no gaps — at least 3 times. This is
where the spec is hardened before any plan exists.

---

## Part 2 — Plan

### 3. Run plan, then the `acp-loop`
**What it is.** Run **`/speckit.plan`** to make `plan.md` plus the research, data-model, and
contract files (checked against the constitution rules if present). Then run
**`/speckit.analyze` with `plan.md`**, run **`/speckit.clarify`**, and enter the **`acp-loop`**
until it comes back clean. Read **`spec.md`** when you re-check — if a gap is in the spec, fix
the spec and let the loop skip `plan` that cycle.

---

## Part 3 — Tasks

### 4. Run tasks, then the `acpt-loop`
**What it is.** Run **`/speckit.tasks`** to make `tasks.md` — ordered, grouped by user story,
matched to the Jira tickets from Part 1. Then run **`/speckit.analyze` with `tasks.md`**, run
**`/speckit.clarify`**, and enter the **`acpt-loop`** until it comes back clean. Keep fixing the
spec, not the tasks; the loop regenerates plan and tasks for you.

---

## Part 4 — Implement and verify

This is a **re-entrant block.** At any check below, if analyze/clarify, the tests, or the
verification turn up bugs or gaps: **copy that text into `/speckit.analyze`**, run
`/speckit.clarify`, run the **`acpt-loop`** until clean, and then **start this part again from
step 5.** Fix from the spec down, never as a quick code patch.

### 5. Run implement, then analyze and clarify
**What it is.** Run **`/speckit.implement`** to do the tasks in `tasks.md` in order. Then run
**`/speckit.analyze`** on the implementation and **`/speckit.clarify`**. If they return no bugs
and no gaps, continue; otherwise re-enter the block (above) and come back here.

### 6. Generate the tests from `tdd.md` and run them
**What it is.** During implementation, turn the plan in **`tdd.md`** into real test files in the
right place, then **run them**. The tests marked **Red** in `tdd.md` must be **Green** now. If
any test exposes a bug or gap, copy it and re-enter the block.

### 7. Verify the code against the thinking documents
**What it is.** Check the running implementation for any inconsistency with the thinking
documents — **`dod.md`, `design.md`, `brd.md`, `bdd.md`, `tdd.md`, and `questions.md`.** Each
BDD Given/When/Then must match the code, each DoD gate must be honestly met or shown as not
done, and nothing may contradict a recorded decision. On any inconsistency, copy it and
re-enter the block.

---

## Part 5 — Review and finish

### 8. Run /review
**What it is.** **`/review`** checks the code against the spec for correct and safe contracts.
If it returns bugs or gaps, copy them into **`/speckit.analyze`**, run `/speckit.clarify`, run
the **`acpt-loop`**, and **start again from Part 4, step 5.**

### 9. Run /code-review
**What it is.** **`/code-review`** checks the diff for bugs and clean-up. If it returns bugs or
gaps, copy them into **`/speckit.analyze`**, run `/speckit.clarify`, run the **`acpt-loop`**,
and **start again from Part 4, step 5.**

### 10. Loop the reviews until clean
**What it is.** Run **`/review` → `/code-review` → `/review`** and keep going until both come
back with no bugs and no gaps — a **minimum of 2 times**. The goal is a clean result after many
loops, not after one try.

---

## Cheat sheet

| Part | Step | Command / loop |
|------|------|----------------|
| pre | thinking docs | `/speckit.thinking <feature>` *(required)* |
| pre | constitution | `/speckit.constitution` *(optional)* |
| pre | spec | `/speckit.specify` (give `brd.md` + `design.md`) *(required)* |
| pre | checklist | `/speckit.checklist <area>` *(optional)* |
| 1 | Jira tickets | from `brd.md` + `design.md` (first step) |
| 1 | Harden spec | **ac-loop**: `analyze → clarify → analyze` (min 3×) |
| 2 | Plan | `/speckit.plan` → analyze(`plan.md`) → clarify → **acp-loop** (skip plan if spec unchanged) |
| 3 | Tasks | `/speckit.tasks` → analyze(`tasks.md`) → clarify → **acpt-loop** (skip plan/tasks per rules) |
| 4 | Implement | `/speckit.implement` → analyze → clarify |
| 4 | Tests | generate from `tdd.md` → run → Red must be Green |
| 4 | Verify | code vs `dod/design/brd/bdd/tdd/questions` |
| 4 | On bug/gap | copy → `/speckit.analyze` → clarify → **acpt-loop** → back to step 5 |
| 5 | Review | `/review` → `/code-review` → `/review` (min 2×); any bug/gap → acpt-loop → back to step 5 |

---

## Things to remember

1. **Prerequisites first.** Thinking docs, (optional) constitution, `specify`, (optional)
   checklist all happen **before** flowing; flowing starts from `spec.md`.
2. **Three loops, analyze first.** `ac-loop` hardens the spec, `acp-loop` adds the plan,
   `acpt-loop` adds the tasks — each runs until analyze and clarify are clean, minimum 3×, with
   the skip rules for unchanged `spec.md`/`plan.md`.
3. **Questions go to `/speckit.answering`** — never answer from memory; the human only sees
   `NEEDS-HUMAN` items, and the answer is written back into the documents first.
4. **The made files are easy to throw away. The spec is the source.** Fix the spec, then
   regenerate plan and tasks — never hand-patch them.
5. **On any bug or gap, re-enter from implement.** Tests, verification, `/review`, and
   `/code-review` all loop back through the `acpt-loop` to Part 4, step 5.
6. **Clean after many loops, not after one try.** The DoD says "done", not your feeling.
