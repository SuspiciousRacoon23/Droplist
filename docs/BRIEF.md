# DROPLIST — design brief

> The single source every design and build agent reads first. Decisions under
> "Settled" are not up for debate; spend your effort on "Open".
> Written 2026-09-17. Presentation is within 7 days. Coursework, not a product.

## 1. What it is

A standalone, local-first revision **triage** tool for the last 7–10 days before
one exam. Its primary output is not a study plan. It is the list of topics the
student can **stop studying** and still reach their target mark — and, for each
one, the evidence for why that is safe or how much of a gamble it is.

One scenario only: a student is nine days out, part of the grade is already
banked, and they need a specific number on the final.

**The insight it is built on.** Choice-based papers ("attempt any 5 of 7",
"Section A all, Section B answer one", "Q5 (a) OR (b)") have slack built into
their structure. That slack can only be found by reading *this institution's*
past papers. A general chatbot has never seen the pattern.

## 2. Non-negotiables — the honesty contract

These are product behaviour, not tone. Code and copy must both obey them.

1. **It never predicts the paper.** It says "appeared in 4 of the 5 papers on
   file". It never says "will appear". Every paper-derived claim is a *backtest
   against papers on file*.
2. **Every claim cites a page.** No extracted fact may be rendered without
   `(paper, page)`. If a fact has no citation it is not shown.
3. **Extraction is a proposal until confirmed.** Everything a model extracts is
   stored `confirmed: false` and is visibly provisional until the student
   confirms it. The planner runs on confirmed data only (or loudly flags what is
   unconfirmed).
4. **Unmeasured is not weak.** A topic with no evidence gets no accuracy number.
   Where the planner needs a number it uses a *stated, labelled assumption*.
   Fewer than 3 data points is "thin" and is shown as thin.
5. **No single readiness score.** Output is "marks in reach vs marks needed",
   both traceable to inputs.
6. **Fewer than 3 papers on file → no recurrence ranking**, and the backtest is
   flagged thin.
7. **Every assumption is a visible, editable setting** flagged `unconfirmed`
   until the student sets it. No hidden constants in an engine.
8. **It writes no answers.** No model-generated exam answers, essays or
   assignment prose. It only decides what to study. This is the academic
   integrity line.
9. **Deterministic where deterministic is more reliable.** The model does
   extraction and topic mapping only. All arithmetic, the optimiser and every
   validator are plain tested code.
10. **Nothing leaves the machine.** Local models only. No telemetry, no cloud.

## 3. Settled decisions

| Area | Decision |
|---|---|
| Repo | `~/Code/Droplist`, its own git repo. Real exam papers are never committed (`papers-real/` is gitignored). |
| Shape | Local-first web app on `localhost:4320`. No auth, single user, one SQLite file in `data/`. |
| Stack | Next.js 15 (App Router) + React 19 + TypeScript strict + Tailwind v4 + zod. SQLite via `better-sqlite3` + Drizzle (verified working on this machine's Node 25.9). Tests: `node --import tsx --test` for engines (the Meridian pattern), Playwright for the demo path. |
| Engines | Pure functions. No DB handle, no clock, no I/O inside `src/engine/**`. The instant is injected. |
| Vision model | `qwen3-vl:8b-instruct` via Ollama (6.1 GB) — for scanned pages and pages with no usable text layer. |
| Text model | `qwen2.5:7b-instruct` via Ollama (already installed) — topic mapping and instruction parsing. Instruct, not reasoning: measured earlier at 8–20 s vs 90–150 s. |
| Model I/O | One `src/lib/llm.ts` router with a provider interface (`ollama`, `mock`). Ollama structured outputs (`format` = JSON schema) on every call; zod-validate every response; topic field is an enum of the course's topics plus an explicit `none_of_these`. `temperature: 0`. |
| PDF | Text layer first: most papers are born-digital Word PDFs. Vision only when a page has no usable text layer, or as a cross-check. `pdftoppm`/`pdftotext` (poppler) are installed; a pure-npm path is preferred if it is reliable on Node 25 — designer's call, with a spike. |
| Optimiser | Exact enumeration over topic subsets for n ≤ 20 (2^20 × a handful of papers is cheap). Above that: greedy + local search, and the UI says the result is approximate. |
| Hardware | MacBook, M4, 16 GB RAM, ~14 GB free disk after the model. One model loaded at a time. |
| Look | "The Marked Paper": the UI borrows from a real question paper and an examiner's red pen. Paper-white ground, ink text, marks set in a right-hand margin in mono like `[12]`, dropped topics **struck through in red pen**. Light theme first. Not the Baxtage brand, not Meridian's Night Survey. |

### Out of scope this week

Multi-user, auth, cloud, spaced repetition, model-generated questions, desktop
wrapper, syllabus-PDF parsing (topics are typed or pasted), mobile layout beyond
"doesn't break".

## 4. Core concepts

### 4.1 The paper pattern is a choice tree

```
Paper      = Group(pick: ALL, children: Section[])
Section B  = Group(pick: 5,   children: Question[7])      "attempt any five"
Question 5 = Group(pick: 1,   children: [Part a, Part b])  "(a) OR (b)"
Question 3 = Group(pick: ALL, children: [Part a, Part b, Part c])
Part       = Leaf(marks, topicId | null, text excerpt, page)
```

One recursive node type — `Group { pick: k | ALL }` or `Leaf { marks, topic }` —
covers compulsory questions, section-level choice, internal OR, and
"any 3 of 5 short notes". Each node carries its page citation.

### 4.2 Two backtests, per past paper

Given a set of dropped topics `D` and a per-topic probability `p(t)`:

- **Availability** (binary): leaf value = `marks` if `topic ∉ D` else `0`; a
  `Group(pick k)` is worth the sum of its top-k children. `loss(D, paper) =
  total(paper) − value(paper)`. A drop set is **structurally safe** iff
  `loss = 0` on every paper on file. Otherwise it is a **gamble**, reported as
  "would have cost 12 marks on the 2022 paper (p.4)".
- **Expected marks**: leaf value = `marks × p(topic)`; same top-k rule (the
  student answers what they are best at). This is "marks in reach".

Per-topic attribution inside a drop set uses marginal loss:
`loss(D) − loss(D \ {t})`. Two topics can each be safe alone and unsafe
together (they lean on the same slack) — the set-level figure is the truth and
the UI must not imply otherwise.

### 4.3 The plan

Choose the set `S` of topics to study. Studying `t` costs `hours(t)` and moves
`p(t)` from `p_now(t)` to `p_target`. Primary objective: **minimum hours such
that the worst-case expected-marks backtest ≥ marks needed + margin**, within
the hours available. If infeasible: maximise the worst-case backtest within the
hours and report the shortfall plainly (mirror `BeyondReach`). Ties: higher
mean backtest, then fewer gamble drops.

### 4.4 Inputs

- **Marks needed**: grade components with weights and marks so far → the
  percentage needed on the final → marks on a paper of that total. Four
  outcomes: cannot simulate / already secured / reachable / beyond reach.
- **Standing per topic — the Probe**: the student is shown *real past questions*
  for a topic and self-grades "could answer / partly / no". `p_now` = (yes +
  ½·partly) / n. It is *stated by the student against real questions* and is
  labelled so. `n < 3` is thin. Untested → no number shown; the planner's
  assumption for it is a visible setting.
- **Hours**: hours available before the exam, and hours to revise a topic. There
  is no data source for the second — the student states it. Be open about this
  in the UI.

### 4.5 Extraction pipeline

```
PDF ─► pages ─► text layer usable? ─yes─► deterministic row parser ─┐
                       └─no─► vision model ─► rows (JSON schema) ────┤
                                                                      ▼
   rows: header | instruction | question | part | or-separator | marks
                                                                      ▼
   deterministic assembler ─► choice tree ─► VALIDATORS ─► confirm UI
                                                                      ▼
                          topic mapping (text model, enum-constrained)
```

Validators are deterministic and are the safety net for the model:
- the tree's attainable total must equal the paper's stated maximum mark
  (real papers print it: "The maximum mark for this examination paper is
  [50 marks]");
- part marks must sum to their question's marks where both are printed;
- a `pick k` must have at least `k` children;
- question numbering must be contiguous.

A failed validator does not block — it becomes a visible flag on the paper.

## 5. The 90-second demo path (must not fail, must work offline)

1. Drop three past papers in → the extracted pattern appears, each line linked to
   its page, the page image beside it.
2. Enter marks so far and a target → "you need 58 on the final".
3. The Drop List appears. Two units go to red strikethrough.
4. Click a struck-through unit → "Safe: Section B lets you skip 2 of 7, and this
   unit has never appeared outside Section B. 2023 p.2 · 2024 p.3."

Because a live 8B vision model on a laptop is slow and can fail on stage, the
repo ships a **seeded demo course built from synthetic fixture papers** with
extraction already cached. Live extraction of one page is a bonus, not a
dependency.

**Fixture papers are synthetic and must say so on every page.** They are
generated by a script in the repo with known ground truth, which is also what
makes extraction accuracy *measurable*.

## 6. Subsystems

| # | Subsystem | Owns |
|---|---|---|
| 1 | Engine | `src/engine/**` — tree, backtests, planner, attribution, recurrence |
| 2 | Inputs | `src/engine/grade.ts`, probe/standing, hours, `src/config/settings.ts` |
| 3 | Extraction | `src/ingest/**`, `src/extract/**`, `src/lib/llm.ts`, fixtures, eval harness |
| 4 | Data + app | `src/db/**`, `src/contracts/**`, server actions/routes, file layout, test strategy |
| 5 | Experience | `src/app/**`, `src/components/**`, design tokens, copy |

## 7. Reference code on this machine (read, do not import)

- `~/Code/Meridian/src/engine/grade.ts` — needed-average maths to port.
- `~/Code/Meridian/src/engine/queue.ts`, `attention.ts`, `src/config/settings.ts`
  — capacity cut line; attention-only-when-wrong; settings declared once and the
  UI generated from the defs.
- `~/Code/Nexus/crates/nexus-core/src/campus.rs` (module doc + `TargetOutcome`)
  and `arena.rs` (`TopicStanding`, `Readiness`, `is_thin`) — the honesty idioms
  this tool inherits: absence is not zero; no top-level readiness percent.
- `~/Code/SoftSol/src/lib/llm.ts` — an existing local-Qwen router.
- Known Ollama trap: with reasoning models `think:true` keeps reasoning out of
  `content`, `think:false` can spill it untagged, and non-reasoning models 400 on
  the param. We use instruct models and do not send `think`.

## 8. Real test documents (local only — never copy into the repo)

`$SCRATCH/real/` holds four of the founder's own IB mock papers:
`bm-hl-p2.pdf` (26 pp, "Section A: answer all. Section B: answer one question.",
marks as `[2]`, max mark printed), `econ-hl-p3.pdf`, `math-sl-p1.pdf`,
`eng-p2.pdf`. They are born-digital Word PDFs with a text layer, and they
include answer-box pages that are noise. Use them to ground the design in how
real papers are actually laid out. They are copyrighted: read them, never commit
them, never quote them at length in docs.

`$SCRATCH` = `/private/tmp/claude-501/-Users-adi-Desktop-Baxtage-com/93fd81b5-2636-463e-9804-edb7ada2a4e1/scratchpad`
