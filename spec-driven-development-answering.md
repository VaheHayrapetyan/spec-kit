# `/speckit.thinking` (answering) — Answer questions from the documents

> This is the **second job** of the `/speckit.thinking` command (the first job, writing the
> documents, is in `spec-driven-development-thinking.md`).
> `/speckit.flowing` calls this job when SpecKit's `analyze`/`clarify` raises a question.
> You answer **only from the thinking documents** — never from memory — log every round in
> `questions.md`, and send anything the documents can't answer (`NEEDS-HUMAN`) to the team's
> Slack group.

---

## When this runs

`/speckit.flowing` calls `/speckit.thinking` (usually as a sub-agent) whenever SpecKit's
`analyze` or `clarify` raises questions about the feature. You answer them **from the
documents**, so the human does not have to. Your final message **is** the answer that
`/speckit.flowing` will use, so make it clean and self-contained.

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

Every item you mark **`NEEDS-HUMAN`** or **`CONFLICT / NEEDS-HUMAN`** must also be **sent to the
team's Slack group** — see *Escalating NEEDS-HUMAN to Slack* below.

---

## When the documents are wrong or incomplete

If answering reveals a document is **wrong, outdated, or contradictory**, say so clearly. Do
**not** quietly patch it mid-answer. Recommend fixing the affected document (and every
document below it in the chain) with the **review loop** (see the writing doc) first, then
re-answering. Only make the fix in the same run if `/speckit.flowing` explicitly asked you to;
otherwise hand the recommendation back so the human decides.

---

## Escalating NEEDS-HUMAN to Slack

A `NEEDS-HUMAN` item is a question the documents cannot answer — it needs a real person. Each
one goes to the team's Slack group, the human replies, and the answer is fed back into the
documents so the question can be answered from them next time.

**Setup (once per project).** The user runs **`/speckit.slack`** to initialize the Slack group
— it stores the destination (channel / webhook / token) in the project config, e.g.
`.specify/slack.json`. This is a one-time step; the agents only **read** that config. If it is
not configured, do **not** fail: note in the returned summary that Slack is not set up (the
user should run `/speckit.slack`) and surface the `NEEDS-HUMAN` items to the human directly.

**1. Send — one question = one message.**
After you write the `ANSWERED` round to `questions.md`, take every item marked `NEEDS-HUMAN` or
`CONFLICT / NEEDS-HUMAN` and post **a separate Slack message for each one** (never grouped).
Each message carries the feature slug, the round and question ID, the question, the reason it
needs a human, and which document should hold the answer. Record each message's id/permalink
next to its item in `questions.md` so the reply can be matched back. Answered items are **not**
sent — only the ones that need a human.

```
🟠 SpecKit needs a human — feature 003-account-delete · round 2 · Q4
CONFLICT: brd.md R7 says 30 min, design.md §Auth says 15 min. Which is correct?
Reply to this message with the answer.
```

**2. Answer — the first reply is the answer.**
The **first reply** to a question's message is taken as the human's answer. One message holds
one question, so one reply resolves exactly one item — no ambiguity about which answer maps to
which question.

**3. Apply — update the documents first, then answer flowing.**
When the human replies, the **answering agent** does these **in order**:
1. **Read the first reply** — that is the human's answer (there is no re-asking).
2. **First, run the thinking agent** to write that question and answer into the right document
   (e.g. set the value in `design.md`, resolve the conflict in `brd.md`) using the review loop,
   so the documents now contain the decision.
3. **Only after the documents are updated, return the answer to `/speckit.flowing`** — turn it
   into a normal question+answer (`ANSWERED`) item, write it into the `ANSWERED` round in
   `questions.md`, and return it so flowing can feed it into `/speckit.clarify`.

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
number, `A#` aligned to the round's `Q#`), then **return the same block** as your final message
so `/speckit.flowing` can put it into `/speckit.clarify`:

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
6. **Every `NEEDS-HUMAN` goes to Slack** — the group set up once via `/speckit.slack`; if it
   isn't configured, say so and surface the items to the human instead.
