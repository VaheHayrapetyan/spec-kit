# `/speckit.thinking` — Write the "thinking" documents

> This is the **first** command. It writes the documents that say *why* and *how*.
> Later, `/speckit.flowing` turns these documents into work.

---

## What this command does

The `/speckit.thinking` command has two jobs:

- **(a) Write the documents** (this document) — BRD, Design Doc, BDD, TDD plan (with Red/Green), DoD.
- **(b) Answer questions** — when `/speckit.flowing` asks a question, it is answered from the
  documents, not from memory. That job runs as its **own command, `/speckit.answering`**, and is
  described in **`spec-driven-development-answering.md`**.

This document covers job (a): **writing the documents.**

**Main rule:** make each document with a **review loop** (below) before you start the next one.
A small mistake early becomes a big mistake later.

---

## Full mode vs targeted mode

`/speckit.thinking` runs in one of two ways:

- **Full mode (default).** You give it a feature (a description or an `NNN-feature-slug`) and it
  writes or refreshes **all five** documents in order, each with the review loop below.
- **Targeted mode.** `/speckit.answering` hands it a **single decision to record** — the input
  starts with the marker **`TARGETED <slug>/<doc>:`** (naming the feature `<slug>` and the
  document `<doc>`). In this mode it does **not** regenerate all five documents. Instead it:
  1. writes the decision into the **named document** (`<doc>`) in `specs/<slug>/`, keeping every
     stable ID;
  2. **cascades downstream** — updates every document **below it in the chain**
     (`brd → design → bdd → tdd → dod`) that the decision affects (a value set in `design.md`
     usually needs matching updates in `bdd.md`, `tdd.md`, and `dod.md`; a resolved conflict almost
     always cascades);
  3. runs the review loop on **each document it touched** (not the whole set), then stops.
  It leaves documents the decision does not affect untouched, and never renumbers IDs. This is how
  a human answer to a `NEEDS-HUMAN` question is written back into the documents (see
  `spec-driven-development-answering.md`).

---

## Where the documents live

SpecKit keeps everything for a feature in **one feature folder**, `specs/<NNN-feature-slug>/`
(that is where `spec.md`, `plan.md`, `tasks.md`, `research.md`, `data-model.md`, and
`contracts/` go). The thinking documents go **in the same folder**, next to them, so
`/speckit.flowing` and the SpecKit commands find them in one place:

```
specs/<NNN-feature-slug>/
  brd.md             # why and what
  design.md          # how
  bdd.md             # Given / When / Then examples
  tdd.md             # the test plan + Red/Green tests (a Markdown document, not code)
  dod.md             # the "done" gates
  questions.md       # clarification Q&A log — see spec-driven-development-answering.md
  …                  # spec.md, plan.md, tasks.md … come later, from SpecKit
```

If the feature folder does **not** exist, this command creates it (the same way the other
SpecKit commands do, with auto-numbering like `001-`, `002-`) so that `/speckit.specify` later
reuses the **same** folder instead of making a new one. If the folder **already exists**, reuse
it and **update** the documents inside it rather than starting over. The slug is kebab-case,
2–4 words.

**Note on the Red/Green tests:** they are written **inside `tdd.md`** (together with the test
plan) **as Markdown** — described in words, not as real test code in Go or any other language.
No test files are created and nothing is run at this stage. The real test code comes later,
during `/speckit.flowing` → `/speckit.implement`.

---

## How to make each document (the review loop)

Do **every** document below with this loop — never write one in a single pass:

- **(a) Create or update** the document from its prompt. If the file **does not exist**, create
  it. If it **already exists**, **update it in place** — do not overwrite from scratch: keep the
  parts that are still correct, keep the stable IDs (`R1`, `G1`, test IDs, gate IDs…) and any
  human edits, and change only what the new input or a derived-from document requires. Adding a
  new requirement must not renumber the existing ones.
- **(b) Review** it. The review has two parts, **in this order**:
  - **First — read the previous features' documents as a reference.** Look across the other
    feature folders (`specs/<NNN-*>/`) and read **both** their thinking documents (`brd.md`,
    `design.md`, `bdd.md`, `tdd.md`, `dod.md`) **and** their SpecKit documents (`spec.md`,
    `plan.md`, `tasks.md`). Use them **only** to reuse existing names, interfaces, and contracts
    (avoid needless divergence) and to **spot conflicts and duplication**. They are a
    **reference, not a source of truth**: never adopt a prior decision unless *this* feature's
    BRD/design justifies it, and if a prior document **conflicts with this feature**, record it
    as a finding and **flag it — do not silently conform** (blindly matching prior docs
    propagates their mistakes forward).
  - **Then — review the current document** for **bugs and gaps**: anything missing, unclear,
    contradictory, not measurable, not testable, not numbered, not traceable to its source, or
    where an edge case is forgotten. List each problem as a concrete finding.
- **(c) Fix** the document using the review findings (both the consistency findings and the
  bug/gap findings).
- **(d) Loop b → c → b** again. Keep repeating until a review returns **no bugs, no gaps, and
  no conflicts** with the previous features' documents. Do this **at least 3 times**, even if
  early reviews look clean — later passes catch what early ones miss.

Only move to the next document when the current one passes a clean review.

---

## Write the documents (in order, each with the review loop)

Throughout this section, "write" means **create-or-update**: if the document already exists,
update it in place (keep correct content, IDs, and human edits) instead of regenerating it from
scratch — see the review loop's step (a) above.

Each entry below lists **what the document must contain** — the output contract, not a script
to copy. Produce a document that satisfies it; how you word it is up to you. And in **every**
document, anything the input doesn't state but a complete document needs is filled from
**researched best practice** and prefixed **[Research best practice]** with a short reason (see
the BRD note below for an example), so a human can tell given facts from assumptions.

### 1. Business Requirement Document (BRD) — `brd.md`
**What it is.** Says *why* we build the feature and *what* it must do, in business words
(no code). The product owner can read it and say "yes".

**Filling gaps with best practice.** The command takes the information the user gives and
turns it into the BRD. **Where the input is missing something a good BRD needs, research the
best practice** for that kind of feature/domain and fill the gap with it. **Mark every such
item clearly** so the human can tell what came from them and what was assumed, e.g.:

> **[Research best practice]** Default session timeout set to 30 minutes — common for
> account-area apps; confirm with the product owner.

**Must contain** (plain business language, no code; every item numbered with a stable ID so
later documents can trace back to it):
- **Purpose / problem** — why this feature exists and the pain it removes.
- **Goals** (`G1…`) and **Non-goals** — what success is, and what we explicitly will not do.
- **Scope** — in-scope (`S1…`) and out-of-scope, each with a one-line reason.
- **Stakeholders & personas** — who uses it and who signs off.
- **Functional requirements** (`R1…`) — each a single testable "the system MUST…" statement.
- **Business rules & constraints** (`BR1…`) — limits, regulations, policies.
- **Success metrics / KPIs** (`M1…`) — each measurable, with a target and how it is measured.
- **Risks & assumptions** (`A1…`) — with impact and a mitigation.
- **Dependencies** — other teams, systems, or data.
- **Acceptance criteria** — for each `R`, the condition that proves it is met.

### 2. Design Document — `design.md`
**What it is.** Says *how* we build it: turns each numbered business need into a technical
plan, and is the bridge from business to code.
**Must contain** a short solution overview + the BRD IDs it covers, then **for each BRD
requirement** (`R1…`):
- **Components / modules** touched (new and changed), with responsibilities.
- **Data model** — entities, fields, types, ownership, migrations.
- **Contracts** — API endpoints and/or events: method/path or topic, request/response schemas,
  status/error codes, idempotency and versioning.
- **Main flow** — step-by-step (or sequence) of the happy path and the key error paths.
- **Failure handling** — timeouts, retries, fallbacks, behaviour on partial failure.
- **Kill-switch / rollout** — feature flag, safe default, how to turn it off fast.
- **Observability** — the logs, metrics, and traces that prove it works in production.
- **Security & privacy** — authn/authz, data handling, PII, least privilege.
- **Performance & scale** — expected load and the latency/throughput targets.
- **Open questions** — anything `brd.md` leaves unclear.

Plus a **traceability list**: every BRD ID → the design parts that satisfy it, flagging any BRD
ID with no design and any design with no BRD ID.

### 3. Behaviour-Driven Development (BDD) — `bdd.md`
**What it is.** Writes the needs as small, testable examples in **Given / When / Then**.
**Must contain** (Gherkin; sourced from `brd.md`, with `design.md` for technical edges):
- One **Feature** per BRD requirement, tagged with its ID (e.g. `@R3`).
- **Scenarios** covering the happy path, boundary values, each error/validation case, and
  important state/permission variations.
- **Scenario Outline** + **Examples** table where one behaviour runs over several inputs.
- **Negative scenarios** for what must NOT happen.
- Concrete, observable steps only — no vague "works correctly".
- A **coverage list**: each BRD ID → its scenarios, flagging any requirement with no scenario.

### 4. TDD plan + Red/Green — `tdd.md`
**What it is.** Turns each BDD scenario into a concrete *test plan* (a plan, not code) **and**,
for each test, records it as a **Red/Green test** described in words: it starts **Red** (would
fail because the feature is not built) and turns **Green** once the behaviour is built. That
proves the tests are meaningful — a test that can never fail is useless. It stays **Markdown**;
the real test code comes later in `/speckit.flowing` → `/speckit.implement`.
**Must contain** (Markdown, no source code in any language; sourced from `bdd.md`). For **each
BDD scenario**, a test entry with:
- a **test ID** and descriptive **name**;
- the **type** (unit / integration / contract / e2e) — prefer the smallest that proves it;
- the **component or function under test**;
- the **target test file path** (matching this project's test layout and naming);
- **preconditions / fixtures / mocks**, the **inputs**, the **expected result / assertions**,
  the **run command**;
- **links** back to the BDD scenario tag and the BRD ID;
- its **Red/Green state** — **RED** (must fail now: feature not built) or **GREEN** — with the
  **assertion in plain words** (Arrange → Act → Assert), the **exact reason it is Red now**
  (e.g. "endpoint returns 404 — not implemented"), and the **condition that flips it to Green**.

Plus a **coverage list** (each BDD scenario → its test(s), flagging any with no test) and a
**Red vs Green summary count**.

### 5. Definition of Done (DoD) — `dod.md`
**What it is.** The list that says "we can ship it now". Each item is a check linked to a
need. "Done" means the list is done — not a feeling.
**Must contain** a table of **gates** (sourced from `brd.md`). For each gate:
- a **gate ID** and a **clear, checkable description**;
- the **BRD ID(s)** it traces to — every gate MUST trace to a measurable need; flag any that
  doesn't;
- a **category** — *blocks-merge* / *for-production* / *needs-sign-off*;
- **how to verify** it (the test, metric, or review that proves it);
- the **owner** and a **status**.

Cover at least: all acceptance criteria met; tests written and passing; code review done;
security/privacy checks; performance targets; observability in place; documentation updated;
and a working rollback / kill-switch.

When all five documents pass a clean review, **print a message to the screen** telling the
human what to do next, for example:

> Done. All five documents are ready in `specs/<NNN-feature-slug>/`. Next, run
> **`/speckit.flowing <feature-slug>`**.

This command **never runs `/speckit.flowing` itself** — it only writes the suggestion to the
screen. Starting the flow is the human's decision.

---

## Answering questions

The **answering job** — answering `/speckit.flowing`'s clarification questions from these
documents, and logging them in `questions.md` — runs as its **own command, `/speckit.answering`**,
documented in **`spec-driven-development-answering.md`**. It uses the same feature folder and the
same documents this command writes (and calls `/speckit.thinking` in **targeted mode** to write a
human answer back into a document).

---

## Things to remember

1. **Documents first, then code.** Each document comes from the one before it.
2. **Every document gets the review loop** — create → review → fix → repeat, at least 3 times,
   until a review finds no bugs and no gaps.
3. **Missing input → research best practice**, fill the gap, and mark it **[Research best practice]**.
4. **The Red/Green tests live in `tdd.md` (with the test plan) as Markdown**, not as code in any language.
5. **Create-or-update** — if a document or the feature folder already exists, update it in place
   (keep correct content, stable IDs, and human edits); never overwrite from scratch.
6. **Answering questions is a separate job** — see `spec-driven-development-answering.md`.
