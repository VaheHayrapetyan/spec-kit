---
description: Answer /speckit.flowing's clarification questions ONLY from the thinking documents; log every round in questions.md; escalate NEEDS-HUMAN by asking the human directly.
argument-hint: <NNN-feature-slug> + questions (or the latest PENDING round)
allowed-tools: Read, Write, Edit, Grep, Glob, Task, SlashCommand, AskUserQuestion
---

## User Input

```text
$ARGUMENTS
```

The input is a **feature slug** and a **list of questions** (usually the latest `PENDING` round
in `questions.md`, or a `QUESTIONS:` block, one per line). If the slug is missing, infer it from
the active feature; if you cannot, return `NEEDS-HUMAN` for the whole request and say the slug is
unknown.

## Role

You are the **`/speckit.answering`** command (the answering job of `/speckit.thinking`).
`/speckit.flowing` calls you whenever `analyze`/`clarify` raises questions. You answer them
**only from the documents** — never from memory — so the human does not have to. Your final
message **is** the answer `/speckit.flowing` will use; make it clean and self-contained.

Companion human-readable guide: `spec-driven-development-answering.md`.

## Sources (priority order — read ALL before answering)

1. **This feature's documents** — `brd.md`, `design.md`, `bdd.md`, `tdd.md`, `dod.md`,
   `questions.md` (earlier rounds may already answer it). **First and strongest.**
2. This feature's SpecKit docs if present — `spec.md`, `plan.md`, `tasks.md`.
3. Previous features' docs (`specs/<NNN-*>/`) — a **reference for naming/consistency and
   conflict-spotting only**, never a source of answers. If a prior doc conflicts, flag it; never
   lift its decision.

Do not answer from the first file that looks close. General knowledge may help you *read* the
docs but must **not** substitute for a decision the docs should record. No answer in the docs →
`NEEDS-HUMAN`.

## How to answer each question
1. Find the answer and give a **source tag** for each, e.g. `brd.md R3`, `design.md §Contracts`,
   `bdd.md @R3 scenario 2`. One question may need several sources — list them all.
2. **Documents conflict** → report both sources, mark **`CONFLICT / NEEDS-HUMAN`** (a conflict is
   a document bug); never pick silently.
3. **Only partly answered** → give the supported part, mark the rest **`NEEDS-HUMAN`**.
4. **Not answered at all** → mark the whole question **`NEEDS-HUMAN`** and say what is missing and
   which document should hold it.
5. **Never invent** scope, rules, numbers, or decisions. Answering is **read-only by default** —
   you only **append** to `questions.md`, and change the thinking docs through the escalation path
   below.

## The questions.md mailbox

`/speckit.flowing` writes a `PENDING` round; you append a matching `ANSWERED` round. Fixed header:

```
## Round <n> — <date> — status: PENDING|ANSWERED — asked-by|by: <command>
```

Use **today's date** for `<date>`. `ANSWERED` rounds are tagged **`by: thinking`** — answering is
`/speckit.thinking`'s answering job, so the `by:` field names that job, not this command.

**Append** your `ANSWERED` round to `specs/<NNN-feature-slug>/questions.md` (same round number,
`A#` aligned to each `Q#`), then **return that round** as your final message so flowing can feed
it into `/speckit.clarify` — **unless** it contains `NEEDS-HUMAN` items that you go on to resolve
in the same run (see *Escalating* below), in which case return the **latest** round (the
resolution `N+1`). Never overwrite earlier rounds — the file grows downward and is the
authoritative record. Example:

```markdown
## Round 2 — <date> — status: ANSWERED — by: thinking

A1: <answer>  [source: brd.md R2]
A2: <answer>  [source: design.md §Data, bdd.md @R5]
A3: NEEDS-HUMAN — the docs don't state the retry limit; it belongs in design.md §Failure handling.
A4: CONFLICT / NEEDS-HUMAN — brd.md R7 says 30 min, design.md §Auth says 15 min.

Summary: answered 2 of 4; 2 need a human / a document fix.
```

## Escalating NEEDS-HUMAN — ask the human directly in Claude Code

Asking the human needs an **interactive session**, so you must be invoked in the **main thread**
(via the `SlashCommand` tool), **not** as a background `Task` subagent.

Handle every `NEEDS-HUMAN` / `CONFLICT / NEEDS-HUMAN` item in **two phases, across two rounds**:

**Phase 1 — log the gap (round `N`).** You have already written round `N` with the item marked
`NEEDS-HUMAN`; that is the honest state. **If you cannot reach the human** (you are a
non-interactive subagent), **stop here** — do **not** fabricate an answer. Return the
`NEEDS-HUMAN` items to `/speckit.flowing`, which will ask the human in the main session and
re-invoke you with the reply to do Phase 2.

**Phase 2 — resolve it (new round `N+1`).** When you can reach the human (main thread) **or** the
human's reply is already in your input (a re-invoke):
1. **Ask the human with the `AskUserQuestion` tool — never as free text that ends your turn**
   (a bare `?` question with no tool call stops the turn and the next reply is misattributed).
   **Skip this step if the reply is already provided.** **Batch up to 4** open `NEEDS-HUMAN` /
   `CONFLICT` items into a **single** `AskUserQuestion` call (loop in batches of 4 if there are
   more); do **not** stop the turn between items. Each question's text carries the feature slug,
   round + question ID, the question, why it needs a human, and which document should hold the
   answer; give **2–4 options** — for a `CONFLICT`, the conflicting values; for an **open** question,
   your best-guess candidate answers **as suggestions, not the only choices** (lead with the one
   you'd recommend and a one-line why, as `/speckit.clarify` does). **Never invent scope, numbers,
   or decisions just to fill option slots** — the tool always adds a free-form "Other", which is
   where the human types the real answer when none of your suggestions fit. The submitted selection
   is the answer. No external service. Example (one question in the batch):

   ```
   AskUserQuestion → question:
     "SpecKit needs a human — feature 003-account-delete · round 2 · Q4.
      CONFLICT: brd.md R7 says 30 min, design.md §Auth says 15 min. Which is correct?
      (Answer lands in brd.md R7.)"
   options: ["30 min (brd.md R7)", "15 min (design.md §Auth)"]   ("Other" is added automatically)
   ```
2. The human's **submitted selection is the answer** (one question = one answer = one resolution;
   no re-asking).
3. **Update the documents first:** run **`/speckit.thinking TARGETED <slug>/<doc>: <the Q&A>`**
   (include the feature **slug** so it edits the right `specs/<slug>/` folder) to write the
   decision into the right document **and its downstream dependents** via the review loop, so the
   docs now contain it — **not** a full five-document regeneration. Name the **topmost** document
   in the chain (`brd → design → bdd → tdd → dod`) that must change, so the downstream cascade
   reaches every affected doc — e.g. resolve a `brd.md`↔`design.md` **conflict** by targeting
   **`brd.md`** (not `design.md`), so the correction flows down to both.
4. **Then log the resolution:** append **one new round `N+1`** (status `ANSWERED`) that resolves
   **all** of round `N`'s `NEEDS-HUMAN` items together — one `A#` per item, each **referencing the
   original round + question ID** it resolves (e.g. "resolves round <N> Q4") with a source tag
   pointing at the now-updated document. Do **not** open a separate round per item. **Return round
   `N+1`** to `/speckit.flowing`. The docs and the answer never disagree.

## When a document is wrong or incomplete
If answering reveals a document is wrong/outdated/contradictory, say so — do **not** quietly
patch it mid-answer. Recommend fixing it (and everything below it in the chain) via the review
loop first, then re-answering. Only fix in the same run if `/speckit.flowing` explicitly asked.

## Remember
1. Answer from the documents, never a guess. Unknown → `NEEDS-HUMAN`.
2. Answer from *this* feature's docs; prior features are reference only.
3. A conflict between documents is a bug → `CONFLICT / NEEDS-HUMAN`, never pick silently.
4. Never patch a document mid-answer — recommend the fix, let the review loop handle it.
5. Every round is logged in `questions.md` — append, never overwrite; the file is the truth.
6. Every `NEEDS-HUMAN` is asked directly to the human via the `AskUserQuestion` submit picker
   (batched, never a turn-ending free-text `?`); the reply is written back into the documents
   first, then handed to flowing.
