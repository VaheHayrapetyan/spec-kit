---
description: Write the thinking documents (BRD, Design, BDD, TDD+Red/Green, DoD) for a feature — each built with a review loop (min 3x) until clean.
argument-hint: <feature description or NNN-feature-slug>
allowed-tools: Read, Write, Edit, Grep, Glob, Bash, Task
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input above. It is the feature to think about (a description, or
an existing `NNN-feature-slug`). If it is empty, ask the human what feature to work on before
doing anything else.

## Two invocation modes

- **Full mode (default).** The input names a feature (description or slug) → write or refresh
  **all five** documents in order, each with the review loop.
- **Targeted mode.** Triggered when the input **starts with the marker `TARGETED <slug>/<doc>:`**
  (this is how `__SPECKIT_COMMAND_ANSWERING__` hands you one question + its human answer to record, naming the
  feature `<slug>` and the document `<doc>`). In this mode you **must not regenerate all five
  documents**. Instead:
  1. In `specs/<slug>/`, write the decision into the **named document** (`<doc>`) in place,
     keeping every stable ID.
  2. **Cascade downstream:** also update every document **below it in the chain**
     (`brd → design → bdd → tdd → dod`) that the decision affects — e.g. a value written into
     `design.md` usually needs matching updates in `bdd.md`, `tdd.md`, and `dod.md`; a resolved
     **conflict** almost always cascades.
  3. Run the review loop on **each document you touched** (not the whole set), then stop.
  Leave documents the decision does not affect untouched, and never renumber IDs.

## Role

You are the **`__SPECKIT_COMMAND_THINKING__`** command — the **first** command in the flow. You write the
documents that say *why* and *how*, so `__SPECKIT_COMMAND_FLOWING__` can later turn them into work. You
**write documents only** — you never write code and never run `__SPECKIT_COMMAND_FLOWING__`.

Companion human-readable guide: `spec-driven-development-thinking.md`. Answering
`__SPECKIT_COMMAND_FLOWING__`'s questions later is a separate command, `__SPECKIT_COMMAND_ANSWERING__`.

## Where the documents live

Everything for a feature lives in **one folder**, `specs/<NNN-feature-slug>/`:

- `brd.md` — why & what
- `design.md` — how
- `bdd.md` — Given/When/Then examples
- `tdd.md` — test plan + Red/Green tests (Markdown, **not** code)
- `dod.md` — the "done" gates
- `questions.md` — the clarification Q&A log (create it empty if missing)

- If the folder does **not** exist, create it with auto-numbering (`001-`, `002-`, …); the slug
  is kebab-case, 2–4 words. `__SPECKIT_COMMAND_SPECIFY__` later reuses this same folder.
- If it **already exists**, reuse it and **update** the documents in place — never overwrite from
  scratch.
- The Red/Green tests are written **inside `tdd.md` as Markdown** (described in words). No test
  files are created and nothing is run here; real tests come later in
  `__SPECKIT_COMMAND_FLOWING__` → `__SPECKIT_COMMAND_IMPLEMENT__`.

## The review loop — use it for EVERY document

Never write a document in one pass. For each document:

1. **Create or update** it from its contract (below). If it exists, keep the correct parts, keep
   stable IDs (`R1`, `G1`, test IDs, gate IDs…) and any human edits; change only what the input
   or an upstream document requires. Adding a requirement must not renumber existing ones.
2. **Review** it, in this order:
   - **First**, read previous features' documents (`specs/<NNN-*>/`, thinking **and** SpecKit
     docs) as a **reference only** — reuse existing names/interfaces/contracts, and **spot
     conflicts and duplication**. Never adopt a prior decision unless *this* feature's BRD/design
     justifies it; if a prior doc conflicts, **flag it — do not silently conform**.
   - **Then**, review the current document for **bugs and gaps**: anything missing, unclear,
     contradictory, not measurable, not testable, not numbered, not traceable to its source, or a
     forgotten edge case. List each as a concrete finding.
3. **Fix** using the findings.
4. **Loop 2 → 3** until a review returns **no bugs, no gaps, and no conflicts** — **at least 3
   times**, even if early passes look clean.

Only move to the next document when the current one passes a clean review.

**Filling gaps:** anything the input doesn't state but a complete document needs is filled from
**researched best practice** and prefixed **[Research best practice]** with a short reason, so a
human can tell given facts from assumptions.

## Write the documents, in order (each with the review loop)

### 1. `brd.md` — Business Requirement Document
*Why* we build it and *what* it must do, in business words (no code). Must contain, every item
with a stable numbered ID:
- Purpose / problem; **Goals** (`G1…`) and Non-goals.
- **Scope** — in-scope (`S1…`) and out-of-scope, each with a one-line reason.
- Stakeholders & personas.
- **Functional requirements** (`R1…`) — each a single testable "the system MUST…".
- **Business rules & constraints** (`BR1…`).
- **Success metrics / KPIs** (`M1…`) — measurable, with a target and how it is measured.
- **Risks & assumptions** (`A1…`) — impact + mitigation.
- Dependencies.
- **Acceptance criteria** — for each `R`, the condition that proves it is met.

### 2. `design.md` — Design Document
*How* we build it: turn each BRD requirement into a technical plan. Solution overview + the BRD
IDs it covers, then **for each `R#`**: components/modules touched; data model (entities, fields,
types, ownership, migrations); **contracts** (API/events: method/path or topic, request/response
schemas, status/error codes, idempotency, versioning); main flow (happy + key error paths);
failure handling (timeouts, retries, fallbacks); **kill-switch / rollout** (flag, safe default);
observability; security & privacy; performance & scale; open questions. Plus a **traceability
list**: every BRD ID → the design parts, flagging any BRD ID with no design and any design with
no BRD ID.

### 3. `bdd.md` — Behaviour-Driven Development
Needs as small testable **Given/When/Then** examples (Gherkin), sourced from `brd.md` (with
`design.md` for technical edges): one **Feature** per requirement tagged with its ID (e.g. `@R3`);
scenarios for happy path, boundary values, each error/validation case, and state/permission
variations; **Scenario Outline** + Examples for one behaviour over many inputs; **negative
scenarios**; concrete observable steps only (no vague "works correctly"). Plus a **coverage
list**: each BRD ID → its scenarios, flagging any with none.

### 4. `tdd.md` — TDD plan + Red/Green (Markdown, no code)
Turn each BDD scenario into a test plan **and** a Red/Green test described **in words**. For each
scenario: test ID + descriptive name; type (unit/integration/contract/e2e — the smallest that
proves it); component/function under test; target test **file path** (this project's layout and
naming); preconditions/fixtures/mocks, inputs, expected assertions, run command; links back to
the BDD tag and BRD ID; **Red/Green state** — **RED** (must fail now, feature not built) or
**GREEN** — with the assertion in plain words (Arrange → Act → Assert), the **exact reason it is
Red now**, and the **condition that flips it to Green**. Plus a **coverage list** (each scenario →
its test(s)) and a **Red vs Green count**.

### 5. `dod.md` — Definition of Done
The gates that mean "we can ship it now". A **table of gates** sourced from `brd.md`; for each:
gate ID + checkable description; the BRD ID(s) it traces to (flag any that trace to none);
category (*blocks-merge* / *for-production* / *needs-sign-off*); how to verify; owner; status.
Cover at least: acceptance criteria met; tests written & passing; code review done;
security/privacy checks; performance targets; observability in place; documentation updated; and
a working rollback / kill-switch.

## Finish

When all five pass a clean review, **print a next-step message** (do **not** run it yourself):

> Done. All five documents are ready in `specs/<NNN-feature-slug>/`. Next, run
> **`__SPECKIT_COMMAND_FLOWING__ <feature-slug>`**.

## Remember
1. Documents first, then code — each comes from the one before it.
2. Every document gets the review loop (min 3x) until no bugs, gaps, or conflicts.
3. Missing input → research best practice, fill the gap, and mark it **[Research best practice]**.
4. Red/Green tests live in `tdd.md` as Markdown, not code in any language.
5. Create-or-update — never overwrite; keep stable IDs and human edits.
6. Answering `__SPECKIT_COMMAND_FLOWING__`'s questions is the separate `__SPECKIT_COMMAND_ANSWERING__` command.
