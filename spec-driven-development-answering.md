# `/speckit.answering` — Answer questions from the documents

> This is the **answering job of `/speckit.thinking`**, run as its own command
> **`/speckit.answering`** (the first job, writing the documents, is in
> `spec-driven-development-thinking.md`).
> `/speckit.flowing` calls it when SpecKit's `analyze`/`clarify` raises a question.
> You answer **only from the thinking documents** — never from memory — log every round in
> `questions.md`, and hand anything the documents can't answer (`NEEDS-HUMAN`) straight to the
> human, asked **directly in Claude Code**.

---

## When this runs

`/speckit.flowing` calls `/speckit.answering` (in the **main thread**, via the `SlashCommand`
tool — it may need to ask the human) whenever SpecKit's `analyze` or `clarify` raises questions
about the feature. You answer them **from the documents**, so the human does not have to. Your
final message **is** the answer that `/speckit.flowing` will use, so make it clean and
self-contained.

**Keep every question and answer in `questions.md`.** Before you finish, write each question,
its answer, the source(s), and the status into `specs/<NNN-feature-slug>/questions.md` as a
**running log** — append a new dated round each time you are called, never overwrite the
earlier rounds. `questions.md` is the permanent record of what was asked and how it was
resolved; your returned message is a copy of that round for `/speckit.flowing` to use.

---

## What you receive

A **feature slug** (which folder to use) and a **list of questions** — usually the latest
`PENDING` round in `questions.md`, or a `QUESTIONS:` block, one question per line. If the slug
is missing, infer it from the active feature; if you cannot, return `NEEDS-HUMAN` for the whole
request and say the slug is unknown.

---

## What you may read (your sources, in priority order)

1. **This feature's documents** — `brd.md`, `design.md`, `bdd.md`, `tdd.md`, `dod.md`,
   `questions.md`. These are the **first and strongest** source (`questions.md` holds earlier
   rounds — a question may already be answered there).
2. **This feature's SpecKit documents if they exist** — `spec.md`, `plan.md`, `tasks.md`.
3. **Previous features' documents** (`specs/<NNN-*>/`, thinking **and** SpecKit docs) — a
   **reference for naming/contract consistency and conflict-spotting only, not a source of
   answers.** An answer must come from *this* feature's documents; never lift a decision from
   another feature, and if a prior document conflicts with this feature, flag it rather than
   adopt it.

Read **all** of them before answering — do not answer from the first file that looks close.
Base every **answer** on the documents — they are the team's actual decisions. General
knowledge can help you *read and interpret* them, but **don't substitute it for a decision the
documents should record**: it may be outdated or not match this project. If the documents don't
answer it, mark `NEEDS-HUMAN` rather than filling from general knowledge.

---

## How to answer each question

1. **Find the answer in the documents** and quote/point to the exact place. Give a **source
   tag** for every answer, e.g. `brd.md R3`, `design.md §Contracts`, `bdd.md @R3 scenario 2`.
2. **One question can need several sources** — list them all.
3. **If the documents conflict**, do not pick silently: report the conflict, name both
   sources, and mark it **`CONFLICT / NEEDS-HUMAN`** (a conflict is a document bug).
4. **If the documents only partly answer**, give the part you can support and mark the
   missing part **`NEEDS-HUMAN`** — never fill the rest with a guess or best practice.
5. **If the documents do not answer at all**, mark the whole question **`NEEDS-HUMAN`** and say
   exactly what is missing and which document should hold it.
6. **Never invent** scope, rules, numbers, or decisions. Answering is read-only by default.

Every item you mark **`NEEDS-HUMAN`** or **`CONFLICT / NEEDS-HUMAN`** must be **asked directly to
the human in Claude Code** — see *Escalating NEEDS-HUMAN to the human* below.

---

## When the documents are wrong or incomplete

If answering reveals a document is **wrong, outdated, or contradictory**, say so clearly. Do
**not** quietly patch it mid-answer. Recommend fixing the affected document (and every
document below it in the chain) with the **review loop** (see the writing doc) first, then
re-answering. Only make the fix in the same run if `/speckit.flowing` explicitly asked you to;
otherwise hand the recommendation back so the human decides.

---

## Escalating NEEDS-HUMAN to the human

A `NEEDS-HUMAN` item is a question the documents cannot answer — it needs a real person. Ask it
**directly in Claude Code**, feed the reply back into the documents, and only then hand it to
flowing. No external service is involved.

Asking the human needs an **interactive session**, so `/speckit.answering` must run in the **main
thread** (invoked with the `SlashCommand` tool), **not** as a background `Task` subagent. Handle
every `NEEDS-HUMAN` / `CONFLICT / NEEDS-HUMAN` item in **two phases, across two rounds:**

**Phase 1 — log the gap (round `N`).** The `ANSWERED` round you just wrote already marks the item
`NEEDS-HUMAN`; that is the honest state. **If you cannot reach the human** (you are running as a
non-interactive subagent), **stop here** — do **not** invent an answer. Return the `NEEDS-HUMAN`
items to `/speckit.flowing`, which asks the human in the main session and re-invokes you with the
reply to do Phase 2.

**Phase 2 — resolve it (new round `N+1`).** When you can reach the human (main thread), or the
reply is already in your input (a re-invoke):

1. **Ask — with the `AskUserQuestion` tool, never turn-ending free text.** A bare `?` question
   with no tool call stops the turn and the next reply is misattributed to the wrong question, so
   ask through the `AskUserQuestion` submit picker instead. **Batch up to 4** open items into one
   call (loop in batches of 4 if there are more) — skip this if the reply is already provided. Each
   question's text carries the feature slug, the round and question ID, the question, the reason it
   needs a human, and which document should hold the answer; give 2–4 options — for a `CONFLICT`,
   the conflicting values; for an **open** question, your best-guess candidates **as suggestions**
   (lead with the one you'd recommend, as `/speckit.clarify` does). **Never invent scope or numbers
   just to fill slots** — the auto-added free-form "Other" is where the human types the real answer
   when none fit.

   ```
   AskUserQuestion → question:
     "SpecKit needs a human — feature 003-account-delete · round 2 · Q4.
      CONFLICT: brd.md R7 says 30 min, design.md §Auth says 15 min. Which is correct?
      (Answer lands in brd.md R7.)"
   options: ["30 min (brd.md R7)", "15 min (design.md §Auth)"]   ("Other" is added automatically)
   ```
2. **Answer — the submitted selection is the answer** (one question = one resolution; no
   re-asking).
3. **Apply — update the documents first.** Run **`/speckit.thinking` in targeted mode** —
   `TARGETED <slug>/<doc>: <the Q&A>` — to write the decision into the right document **and its
   downstream dependents** via the review loop (not a full five-document regeneration). Name the
   **topmost** document in the chain (`brd → design → bdd → tdd → dod`) that must change, so the
   cascade reaches every affected doc — e.g. resolve a `brd.md`↔`design.md` conflict by targeting
   **`brd.md`**, so the correction flows down to both.
4. **Then log the resolution.** Append **one new round `N+1`** (status `ANSWERED`) that resolves
   **all** of round `N`'s `NEEDS-HUMAN` items together — one `A#` per item, each referencing the
   original round + question ID it resolves (e.g. "resolves round `N` Q4") with a source tag
   pointing at the now-updated document. **Return round `N+1`** to `/speckit.flowing`.

So the human only ever decides things the documents truly don't cover, that decision is written
back into the thinking documents **first**, and only then handed to flowing — so the answer and
the documents never disagree.

---

## The `questions.md` mailbox

`/speckit.flowing` writes its questions as a **`PENDING`** round and you answer with a matching
**`ANSWERED`** round. The round header is fixed so both sides parse it the same way:

```
## Round <n> — <date> — status: PENDING|ANSWERED — asked-by|by: <command>
```

A `PENDING` round (written by `/speckit.flowing`) holds numbered questions `Q1, Q2…`:

```markdown
## Round 2 — 2026-06-29 — status: PENDING — asked-by: flowing
Q1: What is the session retry limit?
Q2: Which roles can delete an account?
```

---

## Output format

First **append your `ANSWERED` round to `specs/<NNN-feature-slug>/questions.md`** (same round
number, `A#` aligned to the round's `Q#`), then **return that round** as your final message so
`/speckit.flowing` can put it into `/speckit.clarify` — **unless** it holds `NEEDS-HUMAN` items you
go on to resolve in the same run (see *Escalating*), in which case return the **latest** round (the
resolution `N+1`):

```markdown
## Round 2 — 2026-06-29 — status: ANSWERED — by: thinking

A1: <answer>  [source: brd.md R2]
A2: <answer>  [source: design.md §Data, bdd.md @R5]
A3: NEEDS-HUMAN — the documents don't state the retry limit; it belongs in design.md §Failure handling.
A4: CONFLICT / NEEDS-HUMAN — brd.md R7 says 30 min, design.md §Auth says 15 min.

Summary: answered 2 of 4; 2 need a human / a document fix.
```

Keep every earlier round untouched — `questions.md` grows downward, one section per call, and
it (not the returned text) is the authoritative record.

---

## Things to remember

1. **Answer from the documents, never from a guess.** No answer in the docs → `NEEDS-HUMAN`.
2. **Answer from *this* feature's docs.** Read prior features' docs only as a naming/contract
   reference and to spot conflicts — never lift an answer from them. Prefer the documents over
   general knowledge (which may be outdated); unknown → `NEEDS-HUMAN`.
3. **A conflict between documents is a bug** — mark `CONFLICT / NEEDS-HUMAN`, never pick silently.
4. **Never patch a document mid-answer** — recommend the fix and let the review loop handle it.
5. **Every round is logged in `questions.md`** — append, never overwrite; the file is the truth.
6. **Every `NEEDS-HUMAN` is asked directly to the human via the `AskUserQuestion` submit picker**
   — batched (up to 4 per call), never a turn-ending free-text `?`; the reply is written back into
   the documents first, then handed to flowing.
