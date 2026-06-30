# `/speckit.flowing` — Run the SpecKit flow

> This is the **second** command. It runs the SpecKit work.
> It uses the documents from `/speckit.thinking`.
> When SpecKit asks a question, this command asks a **thinking agent**, not you.

---

## What this command does

- **(a)** Runs the SpecKit commands in order (constitution → specify → plan → tasks → implement → review).
- **(b)** Uses the documents made by `/speckit.thinking` (the BRD and Design Doc go into `specify`; the BDD, TDD (with Red/Green), and DoD check the code at the end).
- **(c)** When `analyze`/`clarify` asks a question, it writes the question into the shared `questions.md` mailbox and runs a **sub-agent with `/speckit.thinking`** to answer it from the documents. Only if that agent says `NEEDS-HUMAN` does it ask you.

**Before you start:** the feature folder `specs/<NNN-feature-slug>/` must exist with `brd.md`, `design.md`, `bdd.md`, `tdd.md`, and `dod.md`. If not, run **`/speckit.thinking <feature>`** first.

---

## The idea in one picture

```
   /speckit.thinking  →  specs/<NNN-slug>/  (brd, design, bdd, tdd, dod, questions)
                              │
                              ▼   (this command, /speckit.flowing)
   constitution → specify → [analyze ⇄ clarify] → spec.md ✓
        → plan  → [analyze ⇄ clarify (⇄ plan)] ✓
        → tasks → [analyze ⇄ clarify (⇄ plan) (⇄ tasks)] ✓
        → implement
                              │  (questions → questions.md → thinking agent, not the human)
                              ▼
   check BDD ▸ check TDD ▸ run tests ▸ check DoD ▸ /review ▸ /code-review
                              │
                              ▼
   found bugs (gaps)?  →  give them to analyze  →  do the cycle again
```

**Main rule:** check each made file **yourself** before the next step.
The made files (plan, tasks) are easy to throw away. **The spec is the source.**

---

## How questions work (the key part)

Questions go through `specs/<slug>/questions.md` — a shared, append-only **mailbox**. The file
is the **truth**; the sub-agent's reply is just a copy. When **`/speckit.analyze`** finds
problems or **`/speckit.clarify`** asks questions:

1. **Do not** answer them yourself, and **do not** ask the human first.
2. **Write a `PENDING` round** into `specs/<slug>/questions.md` (a header + numbered `Q1, Q2…`).
3. Run a **sub-agent** that follows **`/speckit.thinking` (answer mode)** pointed at that round:
   it reads `specs/<slug>/*` and **appends an `ANSWERED` round** (`A1, A2…` with sources).
4. **Re-read `questions.md`** (not just the reply) and put the `ANSWERED` items into
   **`/speckit.clarify`** so they are written into `spec.md`.
5. Items marked **`NEEDS-HUMAN`** (or `CONFLICT / NEEDS-HUMAN`) go to **Slack — one message per
   question**. The human's **first reply** is the answer. The **answering agent** **first** runs
   the **thinking agent** to write it into the right document, and **only then** returns that
   question+answer to flowing as a normal `ANSWERED` item — no re-asking (see
   `spec-driven-development-answering.md`). If Slack isn't set up, ask the human directly.

Every round stays in `questions.md`, so you always have a record of what was asked and why.

---

## Part 1 — Set up SpecKit

### 1. Set up SpecKit
Add SpecKit to the project for Claude (skills, templates, config) if it is not there yet.

### 2. Run constitution
`/speckit.constitution` writes the project rules in `.specify/memory/constitution.md`.
**Do not just copy `CLAUDE.md`.** Take only 5–7 strong rules — each a name, one **MUST**
sentence, and the reason. Drop the tools, commands, and deploy notes. Set version `1.0.0`
and the date. Re-run later only to change a rule.

---

## Part 2 — Write and fix the spec

### 3. Run specify — give it the Design Document and the BRD
`/speckit.specify` turns the documents into a clear `spec.md`. **Give it
`specs/<slug>/brd.md` and `specs/<slug>/design.md`.** Keep the same numbers.

### 4. Make Jira tickets from the Design Document and the BRD
Make tickets from the BRD (business parts) and the Design Document (technical parts), so
the plan and tasks match real tickets.

### 5. Check the spec yourself
Read what you have before the next step.

### 6. Run analyze
`/speckit.analyze` reads the documents and **tells you** the problems. It changes nothing.

### 7. Run clarify
`/speckit.clarify` asks up to 5 questions and writes the answers into `spec.md`.
Use the **question flow above** — the thinking agent answers them.

### 8. Do analyze → clarify → analyze, 3 times
Always analyze first, so clarify asks about real problems. Stop when nothing is found.

### 9. Read the spec.md file
Open `spec.md` and read it yourself. Check the user stories, needs, and success criteria.

### (Optional) Run checklist on the spec
`/speckit.checklist <area>` is "unit tests for the spec" — it checks the *needs*, not the
code, and is **not** the DoD. Use it after clarify, before plan.

---

## Part 3 — Plan

### 10. Run plan
`/speckit.plan` makes `plan.md` plus research, data model, and contracts, and checks
against the rules.

### 11. Run analyze with plan.md
Check the plan matches the spec and the rules. No extra parts, no missing parts.

### 12. Do analyze → clarify → plan → analyze, 3 times
Analyze first; route questions through the thinking agent. **If a gap is in `spec.md`, fix
the spec and skip plan this time** — fix the cause, not the result.

### 13. Check spec.md — not plan.md or tasks.md
If the plan or tasks are wrong, **do not fix them by hand. Fix the spec and make them
again.** The made files are easy to throw away.

---

## Part 4 — Tasks

### 14. Run tasks — with the Jira tasks
`/speckit.tasks` makes `tasks.md`, in order, grouped by user story. Match the Jira tickets.

### 15. Run analyze with plan.md and tasks.md
Full check on **spec + plan + tasks**: every need has a task, nothing missing or wrong.

### 16. Do analyze → clarify → analyze, 3 times
The same loop on the tasks. Fix problems in the spec, not the tasks.

### 17. If bugs come back: run plan and tasks again, then loop again
Make the files again — do not fix by hand. Keep going until clarify finds **no problems**
in spec, plan, and tasks. Then start the code.

---

## Part 5 — Build and check

### 18. Run implement
`/speckit.implement` does the tasks in `tasks.md` in order, with checks.

### 19. Check the BDD with the code
Read `specs/<slug>/bdd.md`. Each Given/When/Then must match the running code.

### 20. Check the TDD with the code
Read `specs/<slug>/tdd.md`. Every test is there and tests the right code. None skipped.

### 21. Run the TDD tests with the code
The tests marked **Red** in `tdd.md` must be **Green** now. This proves the code
does what the documents say.

### 22. Check the DoD with the code
Read `specs/<slug>/dod.md`. Tick only what is really done. Show the rest honestly.

### 23. If there are bugs or problems: give them back and start again
Copy the problem text into `/speckit.analyze` and do the full loop again (questions still
go to the thinking agent). Fix *from the spec down*, not as a quick code patch.

---

## Part 6 — Review and finish

### 24. Run /review and /code-review
When the SpecKit loops are clean:
- **`/review`** — checks the code against the spec for correct and safe contracts.
- **`/code-review`** — checks the diff for bugs and clean-up.

### 25. Run analyze with the bugs you found, then loop again
Give the review results to `/speckit.analyze` and do the loop one more time. Keep going
until the reviews **and** analyze are both clean.

---

## Cheat sheet

| Part | Step | Command |
|------|------|---------|
| pre | documents exist? | `/speckit.thinking <feature>` (the other command) |
| 1 | Constitution | `/speckit.constitution` (clean it, don't copy CLAUDE.md) |
| 2 | Specify | `/speckit.specify` (give `brd.md` + `design.md`) |
| 2 | Jira tickets | from BRD + Design Doc |
| 2 | Analyze / Clarify ×3 | `/speckit.analyze` → PENDING in `questions.md` → thinking agent answers → re-read → `/speckit.clarify` |
| 2 | Read spec | read `spec.md` |
| 2 | Checklist *(opt)* | `/speckit.checklist <area>` (spec quality, **not** DoD) |
| 3 | Plan | `/speckit.plan` → analyze → loop ×3 (skip plan if gap is in spec) |
| 4 | Tasks | `/speckit.tasks` → analyze on spec+plan+tasks → loop ×3 → make again |
| 5 | Implement + check | `/speckit.implement` → BDD / TDD / run tests / DoD |
| 5 | On bugs | copy problem → `/speckit.analyze` → full loop |
| 6 | Review | `/review` + `/code-review` → analyze → loop until clean |

## Things to remember

1. **Documents first, then code.** Use the thinking documents; don't work from memory.
2. **Check every step yourself before the next** — small mistakes grow.
3. **The made files are easy to throw away. The spec is the source.** Fix the spec, then make plan and tasks again.
4. **Analyze first, then clarify** — ask about real problems, not guesses.
5. **Questions go to the thinking agent** — the human only answers `NEEDS-HUMAN`.
6. **Clean after many loops, not after one try.** The DoD says "done", not your feeling.
