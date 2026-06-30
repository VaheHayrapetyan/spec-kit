# Spec-Driven Development with SpecKit + Claude

> This is how we build a feature step by step.
> First Claude writes documents. Then SpecKit turns them into work.
> You check the work at every step.

---

## The idea in one picture

```
   Claude writes the "thinking" documents
   ┌──────────────────────────────────────────────────────────────┐
   │  BRD → Design Doc → BDD → TDD → TDD Red/Green → tests  + DoD │
   └──────────────────────────────────────────────────────────────┘
                              │  (give BRD + Design Doc to SpecKit)
                              ▼
   SpecKit makes the work
   ┌──────────────────────────────────────────────────────────────┐
   │  constitution → specify → [analyze ⇄ clarify] → spec.md ✓    │
   │       → plan  → [analyze ⇄ clarify (⇄ plan)] ✓               │
   │       → tasks → [analyze ⇄ clarify (⇄ plan) (⇄ tasks)] ✓     │
   │       → implement                                            │
   └──────────────────────────────────────────────────────────────┘
                              │  (check the implementation code with the documents)
                              ▼
   check BDD ▸ check TDD ▸ run tests ▸ check DoD ▸ /review ▸ /code-review
                              │
                              ▼
   found bugs (gaps)?  →  give them to analyze  →  do the cycle again
```

**Main rule:** check each document **yourself** before you make the next one.
A small mistake early becomes a big mistake later.

---

# Part 0 — Write the "thinking" documents (with Claude)

You write these **before** SpecKit. Later you give the BRD and Design Document to
`specify`. You use the BDD, TDD, and DoD to check the code at the end.

### 1. Make the Business Requirement Document (BRD)
**What it is.** This document says *why* we build the feature and *what* it must do.
It uses business words, not code. It has the goals, what is in and out, the people,
the rules, and how we know it works. The product owner can read it and say "yes".
> Prompt: *"Write a BRD for `<feature>`. Use simple business words, not code. Add the
> goals, the scope (in and out), the people, the rules, the success metrics, the risks,
> and the acceptance criteria. Give every item a number."*

### 2. Make the Design Document from the BRD
**What it is.** This document says *how* we build it. It turns each business need into
a technical plan: which parts of the code, which contracts (API, events), the data, and
how to turn it off if there is a problem. It is the bridge from business to code.
> Prompt: *"From this BRD, write a Design Document. For each need, give the technical
> way: the parts, the contracts, the data, and a safe way to turn it off. Show anything
> that is not clear in the BRD."*

### 3. Make the Behaviour-Driven Development document (BDD) from the BRD
**What it is.** This document writes the needs as small examples you can test.
It uses **Given / When / Then**. Each example shows one behaviour and the edge cases.
Each example links back to a need, so you can follow it.
> Prompt: *"From this BRD, write a BDD document with Given/When/Then. One feature for
> each need, with normal and edge cases. Link each example to the need it tests."*

### 4. Make the Test-Driven Development document (TDD) from the BDD
**What it is.** This document turns each BDD example into a *test plan*. For each test it
says: the name, the type (unit or integration), the code it tests, the file, and how to
run it. It is the plan for the tests.
> Prompt: *"From this BDD, write a TDD plan. For each test, give the name, the type, the
> code it tests, the file, and the run command."*

### 5. Make TDD Red-Green
**What it is.** This is the way to use the TDD plan:
- **Red** — write the test first. It fails, because the code is not there yet.
- **Green** — write a little code to make the test pass.
- **Refactor** — clean the code. The test keeps you safe.

Red-Green shows the test really works. A test that never fails is not useful.
> Prompt: *"Mark each test Red or Green. For Red tests, write the failing test now. Make
> it check the right behaviour and fail."*

### 6. Make the tests with TDD
**What it is.** Write the Red tests as real test files. Run them. Check that they fail for
the **right reason** — the feature is not built yet, not a typo. These tests are the
contract. The code must make them pass later.
> Prompt: *"Make the Red test files in the right place, like our other tests. Run them and
> show that they fail because the feature is not built."*

### 7. Make the Definition of Done (DoD) from the BRD
**What it is.** This is the list that says "we can ship it now". Each item is a check
linked to a need. Some items block the merge. Some are for production. Some need another
team to sign. The work is "done" only when the list is done — not by feeling.
> Prompt: *"From this BRD, write a Definition of Done as a list of gates. Each gate is a
> check linked to a need. Mark it: blocks merge / for production / needs sign-off. Add the
> status."*

---

# Part 1 — Set up SpecKit

### 8. Run the SpecKit command and make the skills and commands
**What it is.** Add SpecKit to the project. Then the steps become commands like
`/speckit-specify` and `/speckit-plan`. This makes the SpecKit skills, templates, and
config in the project.
> Command to Claude: *"Set up SpecKit in this project for Claude and make the SpecKit
> skills and commands."*

### 9. Run constitution — the best way (NOT working good ⚠️)
**What it is.** `/speckit-constitution` writes the project rules in
`.specify/memory/constitution.md`. Every later step is checked against these rules.

**Do not just copy `CLAUDE.md`.** `CLAUDE.md` mixes tools, commands, and deploy
notes with the real rules. So the result is messy. The constitution needs only clear,
strong rules — not "how to" notes.

**Best way (two passes):**
1. First ask Claude to read `CLAUDE.md` and take out only the *rules* — about 5 to 7 of
   them. Each rule: a name, one **MUST** sentence, and the reason.
2. Check the list yourself. Drop the tools, commands, and deploy notes. Keep only the
   clear, strong rules.
3. Now run `/speckit-constitution` with this clean list. Set the version to `1.0.0` and
   the date.
4. Read the result. Every rule must be clear and easy to test. No soft "should" without a
   reason.
5. Keep the tools and commands in `CLAUDE.md`. The constitution has rules only.
6. Later, run constitution again only to change a rule. It raises the version and updates
   the templates.
> Prompt: *"Read CLAUDE.md and write 5–7 constitution rules. Each one: a name, one MUST
> sentence, and the reason. No tools, no commands, no deploy notes. Then run
> /speckit-constitution with these rules, version 1.0.0."*

---

# Part 2 — Write and fix the spec

### 10. Run specify — give it the Design Document and the BRD
**What it is.** `/speckit-specify` turns plain text into a clear `spec.md`: user stories,
needs, success criteria, and edge cases. Do not write the feature from memory. **Give it
the BRD and the Design Document.** Then the spec keeps the agreed scope.
> Command: *"/speckit-specify — build the spec from the BRD and the Design Document. Keep
> the same numbers so we can follow them."*

### 11. Make Jira tickets from the Design Document and the BRD
**What it is.** Turn the scope into real work. Make tickets from the BRD (business parts)
and the Design Document (technical parts). Then the plan and tasks match real tickets.

### 12. Check every step yourself before the next one
**What it is.** This is the most important habit. Before you make the next document, read
the one you have. Check the BRD before the Design Document. Check the Design Document
before specify. Check `spec.md` before plan. Find problems early, when they are cheap.

### 13. Run analyze
**What it is.** `/speckit-analyze` reads the documents and looks for problems. It checks
the spec (and later the plan and tasks) for things that do not match, things that are
missing, and things that are not clear. It only **tells you** the problems. It does not
change anything.

### 14. Run clarify
**What it is.** `/speckit-clarify` finds the unclear parts of the spec. It asks up to
**5 questions**. Then it **writes your answers into `spec.md`**. This is where you fix the
problems that analyze found.

### 15. Do analyze → clarify → analyze, 3 times
**What it is.** Do **analyze → clarify → analyze** about three times. Always do **analyze
first**, so clarify asks about real problems, not guesses. Each loop makes the spec
better. Stop when analyze and clarify finds nothing.

### 16. Check the spec.md file(s)
**What it is.** Open `spec.md` and read it yourself. This file is the most important one.
Check that the user stories, needs, and success criteria match the feature you want. Do
not leave this to the tools.

### (Optional) Run checklist on the spec
**What it is.** `/speckit-checklist` is like "unit tests for the spec". It does **not**
test the code. It checks the *needs*: are they complete, clear, the same, and easy to
measure? You can make one file for each area, like `security.md` or `api.md`.

**When.** Use it on the spec, after clarify and before plan. It works well with the other
tools: analyze checks that documents match, clarify asks you questions, and checklist
checks the quality of the needs.

**Checklist is not the DoD. They are different:**
- **Checklist** = "is the spec written well?" (about the needs)
- **DoD** = "is the code done and safe to ship?" (about the build)

So do not use checklist to test the code, and do not use it in place of the DoD. But you
can use it for one good link: every DoD gate must come from a clear, measurable need in
the spec. If a gate has no clear need, fix the spec.
> Command: *"/speckit-checklist security — check the quality of the security needs."*

---

# Part 3 — Plan

### 17. Run plan
**What it is.** `/speckit-plan` makes the plan and design files (`plan.md`, plus research,
data model, contracts). It also checks the plan against the rules from step 9.

### 18. Run analyze with plan.md
**What it is.** Run `/speckit-analyze` again, now with `plan.md`. Check that the plan
matches the spec and the rules. No extra parts, no missing parts.

### 19. Do analyze → clarify → plan → analyze, 3 times
**What it is.** Loop until the plan is clean. Do analyze first to find problems.
**Shortcut:** if clarify finds a problem in **`spec.md`**, fix the spec. Then you can
**skip plan** this time. A bad plan is only the result. The spec is the real cause. Fix
the cause.

### 20. Check spec.md — not plan.md or tasks.md
**What it is.** When you check again, read **`spec.md`**, not `plan.md` or `tasks.md`.
Checking the made files is a waste of time. **If the plan or tasks are wrong, do not fix
them by hand. Fix the spec and make them again.** The made files are easy to throw away.
The spec is the real source.

---

# Part 4 — Tasks

### 21. Run tasks — with the Jira tasks
**What it is.** `/speckit-tasks` makes `tasks.md`. The tasks are in the right order, with
groups by user story. Match them with the Jira tickets from step 11. Then the tickets and
the tasks stay the same.

### 22. Run analyze with plan.md and tasks.md
**What it is.** Full check: `/speckit-analyze` on **spec + plan + tasks**. Check that every
need has a task, with nothing missing and nothing wrong.

### 23. Do analyze → clarify → analyze, 3 times
**What it is.** The same loop on the tasks. Analyze to find problems. Clarify to fix them
in the spec. Analyze to check.

### 24. If bugs come back: run plan and tasks again, then loop again
**What it is.** If a loop finds real problems, **make the files again** — do not fix by
hand. Run `/speckit-plan` and `/speckit-tasks` again and do the loop again. Keep going
until `/speckit-clarify` finds **no problems in spec.md, plan.md, and tasks.md**. Then you
can start the code.

---

# Part 5 — Build and check

### 25. Run implement
**What it is.** `/speckit-implement` does the tasks in `tasks.md` in order, step by step,
with checks. The documents are clean now, so the code is easy.

### 26. Remember: steps 3 → 7 were to test the code
The BDD, TDD, TDD Red-Green, the tests, and the DoD were made for this moment. Now we use
them.

### 27. Check the BDD with the code
**What it is.** Read each Given / When / Then example. Check that the running code does the
same thing. If not, it is a bug or a change you did not write down.

### 28. Check the TDD with the code
**What it is.** Check that every test in the TDD plan is there and tests the right code. No
test is skipped.

### 29. Run the TDD tests with the code
**What it is.** Run the tests from steps 5–6. The Red tests must be **Green** now. This is
the proof that the code does what the BDD and TDD say.

### 30. Check the DoD with the code
**What it is.** Read each DoD gate. Tick only what is really done. Show what is not done,
honestly. The DoD says "done", not your feeling.

### 31. If there are bugs or problems: give them back and start again
**What it is.** If the code has bugs or design problems, **copy the problem text into
`/speckit-analyze`** and do the full loop again. The fix comes *from the spec down*, not as
a quick code patch. This keeps the documents true.

---

# Part 6 — Review and finish

### 32. When analyze and clarify find no problems: run /review and /code-review
**What it is.** When the SpecKit loops are clean, use the review skills:
- **`/review`** — checks the code against the spec for correct and safe contracts.
- **`/code-review`** — checks the diff for bugs and clean-up.

Collect the bugs and problems they find.

### 33. Run analyze with the bugs you found, then do the whole loop again
**What it is.** Give the `/review` and `/code-review` results to `/speckit-analyze`. Then do
the loop one more time. Keep going until the reviews **and** analyze are both clean. The
goal is a clean result after many loops, not after one try.

---

## Cheat sheet

| # | Step | Command / Prompt |
|---|------|------------------|
| 1 | BRD | "Write a BRD for `<feature>`…" |
| 2 | Design Doc | "From the BRD, write a Design Document…" |
| 3 | BDD | "From the BRD, write Given/When/Then examples…" |
| 4 | TDD | "From the BDD, write a test plan…" |
| 5 | TDD Red-Green | "Mark tests Red/Green; write the failing tests…" |
| 6 | Make tests | "Make the Red tests and show they fail…" |
| 7 | DoD | "From the BRD, write a Definition of Done…" |
| 8 | Set up SpecKit | "Set up SpecKit (Claude) + skills/commands" |
| 9 | Constitution | `/speckit-constitution` *(clean it, do not just copy CLAUDE.md)* |
| 10 | Specify | `/speckit-specify` (give BRD + Design Doc) |
| 11 | Jira tickets | from BRD + Design Doc |
| 12 | Self-check | read every document before the next |
| 13 | Analyze | `/speckit-analyze` |
| 14 | Clarify | `/speckit-clarify` |
| 15 | Loop ×3 | analyze → clarify → analyze |
| 16 | Read spec | read `spec.md` |
| 16+ | Checklist *(optional)* | `/speckit-checklist security` (spec quality, **not** the DoD) |
| 17 | Plan | `/speckit-plan` |
| 18 | Analyze + plan | `/speckit-analyze` |
| 19 | Loop ×3 | analyze → clarify → plan → analyze (skip plan if the gap is in spec) |
| 20 | Read spec only | make plan/tasks again, do not fix by hand |
| 21 | Tasks | `/speckit-tasks` (match Jira) |
| 22 | Analyze + tasks | `/speckit-analyze` on spec + plan + tasks |
| 23 | Loop ×3 | analyze → clarify → analyze |
| 24 | Make again | re-plan + re-tasks until clarify is clean |
| 25 | Implement | `/speckit-implement` |
| 26–30 | Check | check BDD / TDD / run tests / check DoD |
| 31 | On bugs | copy problem → `/speckit-analyze` → full loop |
| 32 | Review | `/review` + `/code-review` |
| 33 | Finish | analyze with the results → loop until clean |

## Things to remember

1. **Documents first, then code.** Each document comes from the one before it. Check it too.
2. **Check every step yourself before the next** — small mistakes grow.
3. **The made files are easy to throw away. The spec is the source.** Fix the spec, then make plan and tasks again.
4. **Analyze first, then clarify** — ask about real problems, not guesses.
5. **Clean after many loops, not after one try.**
6. **The DoD says "done"** — not your feeling.
