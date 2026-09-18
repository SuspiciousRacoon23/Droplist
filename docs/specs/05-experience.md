# 05 — Experience: screens, states, copy, and the "Marked Paper" design system

Owns: `src/app/**`, `src/components/**`, `src/copy/**`, `src/app/globals.css`, `src/app/fonts/**`, `e2e/demo.spec.ts`.
Reads: `docs/BRIEF.md`, the founder decisions of 2026-09-18 (D-A … D-P, applied throughout and not reopened), `01-engine.md` and `02-inputs.md` (aligned to their types; every divergence is in section 15).
Priority: **MUST** = the demo path plus the honesty contract, sized for one agent's share of a two-day parallel build (section 14). **SHOULD** = after MUST is green. **COULD** = idle time only.

---

## 0. Decisions made in this spec

| # | Decision | Why |
|---|---|---|
| D1 | Six routes: `/` (the Drop List), `/setup`, `/papers`, `/papers/[paperId]` (confirm), `/standing`, `/settings`. The result is the home page. | No dashboard, wizard, course list, topic pages or chart. |
| D2 | Card order on `/`: headline + ledger + plan footer, drops, the three labelled lists (gambles, at standing, no evidence), keep, backtest. | Spike S6: the struck rows sit above the fold at 1440×800. |
| D3 | Three inks (D-P). **Ink** = a fact cited to a page of a counted paper. **Blue pen** = the student's hand: every self-report, prefixed `you said:`, and every control. **Pencil** = an assumption or unconfirmed model output, always dashed and labelled. **Red pen** = the tool's judgement: strikes, flags, shortfall, a gamble's cost. | One rule a builder applies without asking. Pencil is never a colour alone. |
| D4 | The strike is an SVG stroke as a CSS `background-image` with `box-decoration-break: clone`. A covered drop gets a solid stroke; a gamble that entered the drop list under a raised budget gets a broken stroke. | Spike S3: wraps across lines in Chromium and WebKit, animates, falls back in forced-colors. |
| D5 | Fonts: STIX Two Text (paper), Atkinson Hyperlegible Next (chrome), DM Mono (marks), self-hosted with `next/font/local`. Never `next/font/google`. | The demo runs offline. Spike S2. |
| D6 | No boxed cards. A block is a numbered exam question with its short answer in the right margin, `[2 of 10]`. A faint red margin rule runs the sheet's height. | Stops it reading as a dashboard. |
| D7 | The UI never classifies a topic. It renders the engine's five classes: **drop** (covered by choice), **gamble** (a separate list at budget 0; struck with a broken stroke only when the student raised the budget), **already at your stated standing**, **no evidence either way**, **keep**. `d` in "Stop studying d of n" counts drops only. | D-B, D-L, D-N. One owner for the verdict. |
| D8 | A row on the confirm screen has one flag, `confirmed`. A paper counts only when every row is confirmed and no fatal flag is open. The screen has one bulk action, `Confirm the N rows with no flag`, plus per-row edits that confirm the row they touch. | D-E. No provisional-mapping tier. |
| D9 | All strings live in `src/copy/`. Sentences owned by 01 and 02 are rendered verbatim, except the secured states (D-M), which this spec owns. Grammar this spec owns is pure functions with unit tests, and one sweep bans the founder's forbidden vocabulary on every rendered route. | Copy is product behaviour. |
| D10 | Tokens use `@theme static` and Tailwind's palette is removed (`--color-*: initial`). | Spike S4: plain `@theme` drops tokens no utility uses. |

---

## 1. Spikes (evidence, `$SCRATCH/spikes/experience/`, throwaway)

S1 `contrast.mjs`: every text pair ≥ 4.83:1 in light; input borders ≥ 3.3:1 after darkening `rule-strong` to `#858073`. S2 font renders: STIX reads as a typeset paper; DM Mono holds at 12 px on a 1× projector; six woff2 files, 116 KB, all OFL. S3 `strike.html`/`anim.html`/`dash.html`: one wobbly stroke per line box in both engines; `background-size` animates as a pen; forced-colors falls back to `text-decoration`; `stroke-dasharray` + `vector-effect: non-scaling-stroke` gives even dashes. S4 Tailwind 4.3.3 build: `@theme static` emits unused tokens, plain `@theme` does not. S5 brute force on a throwaway course: marginal losses neither add up nor identify the harmless drops, so the verdict has one owner (the engine). S6 static mocks at 1440×800: headline, ledger, both struck rows and one open why-panel fit in 770 px; the confirm layout shows 9 rows beside a 44 % page pane. S7 `pdftoppm -r 150`: 1275×1650 px, 145 KB, 0.23 s a page.

Real papers (read, never quoted): the maximum mark and section rules are on p.1; marks print inline at the end of a part; most of a page is ruled answer lines, so a part is unreadable without its page image.

---

## 2. Information architecture

### 2.1 Routes (MUST)

| Route | Nav | File | Purpose |
|---|---|---|---|
| `/` | `4 The drop list` | `src/app/page.tsx` | The result, or exactly what is missing. |
| `/setup` | `1 Course` | `src/app/setup/page.tsx` | Topics and hours; grade and target; hours available. |
| `/papers` | `2 Papers` | `src/app/papers/page.tsx` | Upload, extraction status, papers on file. |
| `/papers/[paperId]` | (under Papers) | `src/app/papers/[paperId]/page.tsx` | Confirm: page image left, rows right. |
| `/standing` | `3 Standing` | `src/app/standing/page.tsx` | The Probe. |
| `/settings` | `Assumptions` | `src/app/settings/page.tsx` | Generated from the registry. |

Single course. Pages are server components, `export const dynamic = "force-dynamic"`. Client boundaries: `ConfirmWorkspace` (and everything under `components/confirm/`, imported only from it), `DropZone`, `ExtractionProgress`, `CiteLayer`, `GradeForm`. Everything else is a form with a server action or a native `<details>`.

### 2.2 Shell (MUST)

```
+------------------------------------------------------- sheet (max 1040px, paper) -+
| DROPLIST                                                   exam in 9 days         |
| {course.title} - final examination            {N} papers on file · {counted} count |
|==== 2px ink rule ================================================================|
| 1 Course   2 Papers   3 Standing   4 The drop list                  Assumptions  |
|---------------------------------------------------------------+------------------|
| 1. What can I stop studying?                                   |        [2 of 10] |
|                                                    red rule ->  |  margin 112px   |
|----------------------------------------------------------------------------------|
| Demo course · synthetic papers ...           Local only · nothing leaves this machine |
```

The sheet lies on a `desk` ground; no sidebar; nav items are numbered because the flow is linear. The confirm screen uses the wide sheet (1360 px) without the margin rule. Footer: left = `S.syntheticFooter` when `course.isSyntheticDemo`, else the data path; right = `S.localOnly`. SHOULD: nav state in mono (`2 Papers [3]`).

### 2.3 The card rule (MUST)

Every numbered block is a `QuestionCard`: a number, a question in the student's voice, an optional margin answer. Three unnumbered **labelled lists** follow card 2 on `/`, each rendered only when non-empty: `Gambles: not in your drop list` · `Already at your stated standing` · `No evidence either way`.

| Screen | Questions |
|---|---|
| `/` | 1. What can I stop studying? · 2. What am I dropping, and what does each drop risk? · 3. What do I still revise? · 4. How would this plan have done on each paper on file? |
| `/setup` | 1. What is on this exam? · 2. What do I need on the final? · 3. How much time do I have? |
| `/papers` | 1. Which past papers are on file? |
| `/papers/[id]` | Does this match the {label} paper? |
| `/standing` | 1. Could I answer what has been asked before? |
| `/settings` | one per registry group: `plan` What does the plan assume? · `standing` What counts for a topic I have not probed? · `hours` How long does revision take me? · `fixed` Which numbers are fixed by design? · `models` Which models read my papers? · plus Where does the data live? |

In a `beyond_reach` plan card 2 reads `2. What do the hours not cover, and what does each omission risk?`

### 2.4 The plan footer and the attention tier (MUST)

**Plan footer** (D-G): under the ledger of card 1, a `Note` with tag `PENCILLED IN`, lead `This plan rests on {m} {assumption / assumptions} you have not confirmed:` and one line per setting, `{label} — {display} · unconfirmed`, each linking to `/settings#{key}`. Always rendered when `m > 0`, never folded, never summarised into a count. It is what 01 and 02 call the plan footer; it sits above the fold because 02's spike found those numbers flip feasibility in a quarter of scenarios.

**Attention notes** render only when something is wrong: on `/` under the plan footer, elsewhere at the top. At most 3 shown; the rest behind `<details>` `{n} more notes`. Priority: `papers_excluded`, `yield_ceiling`, `thin_papers`, `tree_issues`, `no_topic_leaves`, `approximate`, `hours_not_stated`. Each names one action. Thin standing, not-probed topics and course-figure hours are normal and are drawn in pencil inside their row, never as a note.

---

## 3. Contracts

### 3.1 Rendered as given

| From | Used for |
|---|---|
| 01 `PlanResult` and the section-10 wording rules | Everything on `/`: "hardest paper on file", never "worst case"; past conditional; marks first, percent second; marginals never totalled. |
| 02 `GradeAssessment`, `PlanNeed`, `PaperNeed` (with D-O's `planTarget` and `marginMarks`), explanations | `/setup` card 2, the ledger. Verbatim except the secured states. |
| 02 `TopicStanding`, `PlanningP.sentence`, display rules | Evidence as marks with a count, never a bare percent; an assumption's number only inside `Trace` and on `/settings`. |
| 02 `HoursAvailable.sentence`, `TopicHours.sentence`, hours header | `/setup` cards 1 and 3. |
| 02 `SETTINGS`, `ResolvedSettings`, `assumptionNotes`, `fmtSetting`, `validateSetting` | `/settings` and the plan footer. |

### 3.2 View-models (`src/components/view-models.ts`)

Adapters in `src/app/_adapters/*.ts` build these. They add paper labels to cites, convert minutes to hours, fold absorptions by group label, and map 02's `untested` to `not_probed`. **No adapter or component computes a mark, a probability, a verdict, a class or a feasibility.**

```ts
export type NonEmpty<T> = [T, ...T[]];

/** Non-negotiable 2: a paper-derived fact without a Cite cannot be constructed. */
export type Cite = {
  paperId: string; paperLabel: string;   // label WITHOUT "paper": "2023"; copy adds the word
  page: number; nodeId: string | null; label: string | null;   // "Q4(b)"
  scope: "node" | "whole-paper";         // whole-paper = a claim about absence; opens p.1
  bbox?: { x: number; y: number; w: number; h: number };       // fractions of the page; SHOULD
};

export type CourseHeadVM = { courseId: string; title: string; daysToExam: number | null;
  papersOnFile: number; papersCounted: number; isSyntheticDemo: boolean; dataPath: string };

export type StandingVM = {
  kind: "probed" | "not_probed" | "no_questions_on_file";
  basis: "probe" | "probe_thin" | "assumption" | "self_rating";
  thin: boolean;
  youSaid: string | null;   // probe: "could answer 10 of 22 marks · 4 questions"; self-rating: "solid"; else null
  fallback: string;         // "Not probed" | "No past question on file"; used when youSaid is null
  sentence: string;         // verbatim 02 planningP sentence, shown inside Trace
  restsOn: string[];        // setting keys
};

export type GroupFold = { label: string; pick: number; of: number; slack: number; deadChildren: number; cites: NonEmpty<Cite> };
export type DropPaperVM = { paperId: string; paperLabel: string; status: "not-on-paper" | "absorbed" | "lost";
  alone: number; withDrops: number; leaves: (Cite & { marks: number; compulsory: boolean })[]; groups: GroupFold[] };

export type DropRowVM = {                  // in the drop list
  topicId: string; name: string;
  classification: "safe" | "gamble";       // gamble only when gambleBudget.marks > 0 (SHOULD path)
  hoursSaved: number; standing: StandingVM;
  appearedIn: { count: number; of: number; cites: NonEmpty<Cite> };
  alwaysOptional: boolean;
  uniformGroup: GroupFold | null;          // every absorption on every paper shares label/pick/of
  cost: { marks: number; paperLabel: string; cites: NonEmpty<Cite> } | null;   // gamble only
  perPaper: DropPaperVM[];                 // SHOULD lines
};
export type GambleRowVM = {                // not in the drop list; revised in this plan
  topicId: string; name: string; hoursSaved: number; standing: StandingVM;
  cost: { marks: number; paperLabel: string; cites: NonEmpty<Cite> };   // max over papers of loss(drops ∪ {t})
  maxAlone: number; appearedIn: { count: number; of: number; cites: NonEmpty<Cite> };
};
export type AtStandingRowVM = { topicId: string; name: string; standing: StandingVM };
export type NoEvidenceRowVM = { topicId: string; name: string; papersOnFile: number; cites: NonEmpty<Cite> };  // one whole-paper cite per counted paper

export type KeepRowVM = { topicId: string; name: string; hours: number; hoursBasis: "own_figure" | "course_figure";
  hoursSentence: string; standing: StandingVM; pinned: boolean;
  gainMarks: number | null;               // SHOULD; on the headline paper
  appearedIn: { count: number; of: number; cites: NonEmpty<Cite> } | null };

export type BacktestRowVM = { paperId: string; paperLabel: string; total: number; totalCite: Cite;
  reach: number; reachPct: number; need: number; target: number; gap: number;
  clears: boolean; isHeadline: boolean; lossIfDropsScoreZero: number; issues: number };

export type ShortfallVM = { marks: number; paperLabel: string; binding: "hours" | "ceiling";
  meetsNeedWithoutMargin: boolean; hoursForFeasible: number | null };

export type UnconfirmedVM = { key: string; label: string; display: string };   // from 02 assumptionNotes, unconfirmed only

export type CaveatVM =
  | { kind: "papers_excluded"; papers: { paperId: string; paperLabel: string;
        reason: "user-excluded" | "structure-unconfirmed" | "broken-tree" | "no-marks"; note: string | null }[] }
  | { kind: "yield_ceiling"; sentence: string }
  | { kind: "thin_papers"; papersCounted: number }
  | { kind: "tree_issues"; papers: { paperId: string; paperLabel: string; issues: number }[] }
  | { kind: "no_topic_leaves"; count: number; marks: number; pct: number; confirmed: boolean; cites: NonEmpty<Cite> }
  | { kind: "approximate"; topicCount: number }
  | { kind: "hours_not_stated"; sentence: string };

export type NeedVM = { needPct: number; marginPct: number; marginConfirmed: boolean; marginKey: string;
  onFinal: { marksNeeded: number; planTarget: number; marginMarks: number; paperTotal: number } | null;
  estimates: string[];      // component titles whose marks are estimates
  explanation: string };

export type PlanVM = {
  kind: "reachable" | "beyond_reach"; method: "exact" | "approximate";
  splitEstimated: boolean;                 // D-K: the safe/gamble split was greedy; only when gambleBudget.marks > 0
  topicCount: number;                      // drop + keep + atStanding + noEvidence
  hoursPlanned: number; hoursAvailable: number | null; hoursSaved: number;
  gambleBudget: { marks: number; confirmed: boolean; key: string };
  headline: { paperLabel: string; reach: number; total: number; reachPct: number; need: number; target: number };
  shortfall: ShortfallVM | null;           // non-null iff beyond_reach
  drop: DropRowVM[]; gambles: GambleRowVM[]; atStanding: AtStandingRowVM[]; noEvidence: NoEvidenceRowVM[];
  keep: KeepRowVM[];                       // includes the gamble topics: the plan revises them
  backtest: NonEmpty<BacktestRowVM>;
  unconfirmed: UnconfirmedVM[];            // the plan footer
};

export type Blocker =
  | { kind: "no-topics" } | { kind: "no-target"; explanation: string }
  | { kind: "no-counted-papers"; papersOnFile: number; excluded: { paperId: string; paperLabel: string; reason: string }[] }
  | { kind: "too-many-topics"; detail: string } | { kind: "invalid-input"; detail: string };

export type DropListVM =
  | { kind: "cannot_say"; course: CourseHeadVM; blockers: NonEmpty<Blocker> }
  | { kind: "secured"; course: CourseHeadVM; targetPct: number }                              // returned marks only
  | { kind: "secured_if_estimates_hold"; course: CourseHeadVM; targetPct: number; estimates: NonEmpty<string> }
  | { kind: "plan"; course: CourseHeadVM; need: NeedVM; gradeBeyond: { explanation: string } | null;
      plan: PlanVM; caveats: CaveatVM[] };

// ---------- papers + confirm ----------
export type PaperStatus =
  | { kind: "queued" }
  | { kind: "reading"; page: number; pageCount: number; mode: "text" | "vision" | "mapping"; elapsedS: number }
  | { kind: "needs_review"; rows: number; confirmed: number; flags: number; mappingPending: number }
  | { kind: "counted"; rows: number; flags: number }
  | { kind: "excluded"; reason: "user-excluded" | "broken-tree" | "no-marks"; note: string | null }
  | { kind: "failed"; reason: string };
export type PaperRowVM = { paperId: string; paperLabel: string; filename: string; pageCount: number;
  status: PaperStatus; fromCache: { seconds: number } | null; visionPages: number[]; isSynthetic: boolean };
export type UploadResultVM = { kind: "accepted"; paperId: string }
  | { kind: "duplicate"; paperId: string; paperLabel: string; counted: boolean }
  | { kind: "rejected"; filename: string; reason: string };

export type ConfirmRowVM = {
  nodeId: string; parentId: string | null; depth: 0 | 1 | 2 | 3; kind: "group" | "leaf";
  label: string; text: string;            // leaf: excerpt ≤ 160 chars; group: the instruction as printed
  cite: Cite;                             // required; rows without a page are dropped by the adapter and counted in a flag
  marks: number | null;                                                       // leaf; halves allowed
  topicIds: string[]; mappingState: "pending" | "mapped" | "none" | "unknown";   // leaf; none = none_of_these
  pick: number | "ALL" | null; childCount: number | null; attainableMarks: number | null;   // group
  confirmed: boolean; flagIds: string[];
};
export type FlagVM = { id: string;
  code: "total-mismatch" | "group-marks-mismatch" | "pick-exceeds-children" | "pick-zero" | "pick-invalid" | "empty-group"
      | "zero-mark-leaf" | "invalid-marks" | "duplicate-node-id" | "dangling-topic" | "total-too-large"
      | "numbering-gap" | "no-stated-total" | "rows-without-page";
  fatal: boolean; scope: "paper" | "row"; nodeIds: string[];
  expected: number | null; actual: number | null; detail: string; cite: Cite | null };
export type ConfirmVM = { course: CourseHeadVM;
  paper: PaperRowVM & { statedTotal: number | null; statedTotalCite: Cite | null; attainableTotal: number };
  rows: ConfirmRowVM[]; flags: FlagVM[]; topics: { id: string; name: string }[] };   // rows in pre-order

export type ProbeItemVM = { leafId: string; leafLabel: string; marks: number; excerpt: string;
  cite: Cite; contextPages: number[]; grade: "yes" | "partly" | "no" | null };
```

Server actions (subsystem 4). Confirm actions return the fresh `ConfirmVM`; the client replaces its state. Validators re-run inside every mutating action; the UI never runs one.

```ts
confirmRows(paperId: string, nodeIds: string[]): Promise<ConfirmVM>       // refuses a leaf whose mapping is pending
updateLeaf(paperId: string, nodeId: string, patch: { marks?: number; topicId?: string | null }): Promise<ConfirmVM>   // also confirms the row
unconfirmRow(paperId, nodeId): Promise<ConfirmVM>                          // SHOULD
updateGroup(paperId, nodeId, patch: { pick: number | "ALL" }): Promise<ConfirmVM>   // SHOULD
uploadPapers(form: FormData): Promise<UploadResultVM[]>
recordProbeAnswer(form: FormData): Promise<void>                          // 02; append-only
saveSetting(key: string, raw: string): Promise<{ error: string | null }>; resetSetting(key: string): Promise<void>
saveTopics / saveGrade / saveHours (form: FormData): Promise<void>
```

Routes: `GET /api/papers/status` → `PaperRowVM[]`, polled every 1 s while any paper is `queued | reading`; `GET /api/papers/{paperId}/pages/{n}` → a pre-rendered PNG (150 dpi) with immutable cache headers. No PDF tool runs on a request.

---

## 4. The Drop List (`/`) — MUST

### 4.1 Layout

```
1. What can I stop studying?                                              [2 of 10]
   Stop studying 2 of 10 topics.                                <- h1, serif 42px
   Revising the other 7 topics takes 21 of your 30 hours. On the hardest of the
   3 papers on file (2023), that would have put 66.4 of 100 in reach. You need 58;
   the plan aims for 63. Both drops are covered by choice on every paper on file.
   1 more is listed below as a gamble: dropping it would have cost marks on a
   paper on file.
   ---------------------------------------------------------------
   58 of 100                66.4 of 100                   21 of 30 h
   needed · aims for 63     in reach, hardest paper on file   revision planned
   from your marks so far   2023 paper                        your hours
   | PENCILLED IN  This plan rests on 3 assumptions you have not confirmed:
   |   Marks from a revised topic — 80% · unconfirmed   ...

2. What am I dropping, and what does each drop risk?                    [saves 8 h]
   ~~Coastal processes~~  [✓ COVERED BY CHOICE]                                  [0]
   Only ever inside Section B (answer 5 of 7) on the 3 papers on file.     saves 4 h
   2022 p.4 · 2023 p.4 · 2024 p.5
   > Why is this drop covered on the papers on file?          <- native <details>
   ~~Glacial systems~~ ...

   GAMBLES: NOT IN YOUR DROP LIST
   Plate tectonics                                                            [−12]
   Revised in this plan. If you dropped it and could answer nothing on it, it would
   have cost up to 12 marks on the 2022 paper. ...                        would save 4 h

   NO EVIDENCE EITHER WAY
   Fieldwork methods                                                          [—]
   Never appeared in the 3 papers on file. That is silence, not safety.

3. What do I still revise?                                                   [21 h]
   Rivers                                                                    [3 h]
   you said: could answer 10 of 22 marks · 4 questions · on 3 of the 3 papers on file
   2022 p.2 · 2023 p.3 · 2024 p.2

4. How would this plan have done on each paper on file?                 [3 papers]
   PAPER            IN REACH          NEEDED  AIMS FOR  GAP   IF EVERY DROP SCORED ZERO
   2023 [HARDEST]   66.4 of 100 (66%)   58      63      +3.4              0
   The 2 covered drops would have cost 0 marks on every paper on file.
   This is a backtest against the 3 papers on file. It is not a prediction of the next paper.
```

The figures above are grammar illustrations, not the demo course (section 13).

**Ledger.** Needed: `{marksNeeded} of {paperTotal}` / `needed · aims for {planTarget}` / `from your marks so far` + pencil ` · {n} estimated` when `need.estimates` is non-empty; links `/setup#grade`. `aims for {planTarget}` is pencil-dashed and links `/settings#{need.marginKey}` when `marginConfirmed` is false. In reach: `{fmtReach(reach)} of {total}` / `in reach, hardest paper on file` / `{paperLabel} paper`; links `#backtest`. Hours: `{hoursPlanned} of {hoursAvailable} h` (or `{hoursPlanned} h` when not stated) / `revision planned` / `your hours`; links `/setup#hours`. Two traceable numbers, no score.

### 4.2 Number formatting (`src/copy/format.ts`)

Display never feeds a comparison; display rounds against the student.

```ts
const floor1 = (x: number) => Math.floor(x * 10 + 1e-6) / 10;   // 16.4 * 10 === 164.00000000000003
const ceil1  = (x: number) => Math.ceil(x * 10 - 1e-6) / 10;
const strip  = (x: number) => String(x).replace(/\.0$/, "");
fmtReach(x)  = strip(floor1(x))            // 66.44 -> "66.4", 79.99999999999999 -> "80"
fmtShort(x)  = strip(ceil1(x))             // 7.71 -> "7.8"
fmtGap(x)    = x >= 0 ? "+" + fmtReach(x) : "−" + fmtShort(-x)      // U+2212
fmtLoss(x)   = x === 0 ? "0" : "−" + fmtShort(x)
fmtHours(x)  // one decimal at most, ".0" dropped
fmtPct(x)    // Math.round(x) + "%"
paperScope(N, label) = N === 1 ? `the only paper on file (${label})`
                     : N === 2 ? `the harder of the 2 papers on file (${label})`
                     : `the hardest of the ${N} papers on file (${label})`
plural(n, one, many); list(["a","b","c"]) // "a, b and c"
```

`need`, `target`, `cost` are whole marks and print as they are. A non-finite number anywhere in a view-model makes `/` render the `cannot_say` layout with `S.internalFigureError`; `NaN` is never printed.

### 4.3 The headline — exact grammar (`src/copy/headline.ts`)

```ts
export type Headline = { h1: string; sub: string[] };   // sub sentences joined with a space
export function headline(vm: DropListVM): Headline;
```

`n` = topicCount, `d` = drop.length, `k` = keep.length, `g` = gambles.length, `a` = atStanding.length, `e` = noEvidence.length, `N` = papersCounted, `H` = plan.headline.

- `HOURS(k)` = `Revising the other {k} {topic / topics} takes {hoursPlanned} of your {hoursAvailable} hours.`; hours not stated: `… takes {hoursPlanned} hours.`
- `REACH` = `On {paperScope(N, H.paperLabel)}, that would have put {fmtReach(H.reach)} of {H.total} in reach.`
- `NEED` = `You need {H.need}; the plan aims for {H.target}.`; when `H.target === H.need`: `You need {H.need}.`
- `COVERED(d)` = d=1 `The drop is covered by choice on every paper on file.` · d=2 `Both drops are covered by choice on every paper on file.` · d≥3 `All {d} drops are covered by choice on every paper on file.` (SHOULD, budget > 0 with `c` covered and `g'` gambles struck: `{c} {is / are} covered by choice on every paper on file; {g'} {is a gamble / are gambles} within the {G}-mark budget you set.`)
- `GAMBLES(g)`, g > 0 = `{g} more {is / are} listed below as a gamble: dropping {it / them} would have cost marks on a paper on file.`
- `AT(a)`, a > 0 = `{a} {topic is / topics are} already at your stated standing; revising {it / them} would not move a figure on this page.`
- `NOEV(e)`, e > 0 = `{e} more {topic / topics} never appeared in the {N} papers on file: silence, not safety.`

| Case | h1 | sub |
|---|---|---|
| plan, reachable, d > 0, k > 0 | `Stop studying {d} of {n} topics.` | HOURS REACH NEED COVERED GAMBLES AT NOEV |
| plan, reachable, d = 0, g = 0 | `Nothing can be dropped.` | `Every topic with evidence is needed. Revising all {k} takes {hoursPlanned} of your {hoursAvailable} hours.` REACH NEED AT NOEV |
| plan, reachable, d = 0, g > 0 | `Nothing can be dropped without a gamble.` | `Every drop that would have cost 0 marks on every paper on file is already taken: none. Revising all {k} takes {hoursPlanned} of your {hoursAvailable} hours.` REACH NEED GAMBLES AT NOEV |
| plan, reachable, k = 0 | `No revision is planned.` | `On {paperScope}, what you said you could answer would already have put {reach} of {total} in reach.` NEED `This rests entirely on what you said you could answer.` COVERED AT NOEV |
| beyond_reach, binding hours | `With {hoursAvailable} hours, the hardest paper on file would have come {fmtShort(shortfall.marks)} marks short.` | `The best use of {hoursAvailable} hours is the {k} {topic / topics} below.` REACH NEED + (`hoursForFeasible`: `Reaching it would have taken {hoursForFeasible} hours.`) + (`meetsNeedWithoutMargin`: `It would have reached the {H.need} you need but not the {H.target} the plan aims for.`) |
| beyond_reach, binding ceiling, gradeBeyond null | `This target is beyond reach on the papers on file.` | `Even revising every topic, {paperScope} would have had {reach} of {total} in reach.` NEED + the `yield_ceiling` sentence if any |
| gradeBeyond non-null | `{targetPct}% overall is beyond reach.` | `gradeBeyond.explanation` verbatim, then `If you sit it anyway, the best use of your hours is below.` |
| secured | `Your target is already secured.` | `On your returned marks, even 0 on the final leaves you at or above {targetPct}% overall. There is nothing to plan at this target.` Action `Raise the target`. |
| secured_if_estimates_hold | `Your target is secured if your estimates hold.` | one `This rests on your estimate for "{title}".` per estimate, then `There is nothing to plan at this target.` Action `Raise the target`. |
| cannot_say | `Cannot say yet.` | `Droplist needs {b} more {thing / things} before it can say anything.` then the blockers (4.6). |

Chips above the h1: `PENCILLED IN · {m} unconfirmed` (pencil, links to the plan footer) when `plan.unconfirmed` is non-empty; `APPROXIMATE` (pencil) when `method = "approximate"`.

**Grammar fixtures** (`src/components/__fixtures__/droplist.ts`) are hand-typed and frozen: they exercise the grammar and never change with the engine or the demo course.

1. `G_TWO_ONE_GAMBLE`: n 10, d 2, k 7 (one of them the gamble), g 1, a 0, e 0, N 3, H {2023, 66.4 of 100, need 58, target 63}, 21 of 30 h → the h1 and sub of 4.1.
2. `G_ALL_LISTS`: as 1 with a 1 and e 1 (n 11). Sub ends `1 topic is already at your stated standing; revising it would not move a figure on this page. 1 more topic never appeared in the 3 papers on file: silence, not safety.`
3. `G_SHORTFALL`: 12 h, k 4, reach 55.2 of 100, need 58, target 63, shortfall 7.8, binding hours, hoursForFeasible 21 → `With 12 hours, the hardest paper on file would have come 7.8 marks short.` / `The best use of 12 hours is the 4 topics below. On the hardest of the 3 papers on file (2023), that would have put 55.2 of 100 in reach. You need 58; the plan aims for 63. Reaching it would have taken 21 hours.`
4. `G_SECURED` (target 70) and `G_SECURED_EST` (estimates `["IA"]`) → the two secured rows.
5. `G_GRADE_BEYOND`: 02 example F (need 110 %, best possible 70 %) → `75% overall is beyond reach.`
6. `G_NOTHING`: d 0, g 0 → `Nothing can be dropped.`

### 4.4 The drop card and the three labelled lists

**Drop row** (`DropRow`): `<s class="strike">name</s>` preceded by `<span class="sr-only">Dropped: </span>`, chip `COVERED BY CHOICE` + check (ink outline), margin `[0]`, sub-line `saves {h} h`; one-line reason (`dropReason(row, N)`): with `uniformGroup` → `Only ever inside {label} (answer {pick} of {of}) on the {N} papers on file.`; otherwise → `Every question on this topic sits inside a choice that can be answered without it, on all {N} papers on file.`; then `CiteList` (max 3, the rest inside the why-panel). Card margin `[saves {hoursSaved} h]`.

SHOULD (budget > 0): a `gamble` row in the drop list is struck with `.strike-gamble`, sr-only `Dropped, a gamble: `, chip `GAMBLE · {G}-mark budget, set by you` (red on red-wash), margin `[−{cost.marks}]` red, reason `Within the gamble budget you set. If you could answer nothing on it, it would have cost up to {cost.marks} marks on the {cost.paperLabel} paper.` + cites. `splitEstimated` adds a pencil chip `SPLIT ESTIMATED` on the card and the line `Every combination of {n} topics was checked. With more than 12 candidate drops, the split between covered drops and gambles was estimated, not enumerated; a drop shown as a gamble may be covered.`

**Why panel** (`WhyPanel`, native `<details>`, closed; summary `Why is this drop covered on the papers on file?`):

```ts
export function whySummary(row: DropRowVM, N: number, c: number): string;
export function whyLines(row: DropRowVM): { text: string; cite: Cite }[];   // SHOULD
```

- uniform: `{label} lets you skip {slack} of {of}, and on the {N} papers on file this topic has never appeared outside {label}. With the {c} covered drops, {of − deadChildren} {label} questions stay answerable on every paper.`
- mixed: `On all {N} papers on file, every question on this topic sits inside a choice, and each choice keeps enough answerable questions with the covered drops.`
- then `CiteList` of every cite (MUST). SHOULD per-paper lines: absorbed `{paperLabel} paper: {leaf.label} [{marks}] sits in {group.label} (answer {pick} of {of}); {of − deadChildren} stay answerable.` · lost, compulsory `… was compulsory.` · lost, in a choice `… but {deadChildren} of its {of} questions are already dropped.` · not-on-paper `{paperLabel} paper: this topic does not appear.` (whole-paper cite).

"up to" is deliberate wherever a cost appears: the per-topic figure is an upper bound that is not additive. The set-level figure is the last column of card 4.

**Gambles: not in your drop list** (`GambleList`, label in pencil caps). Row: name unstruck (serif), red margin `[−{cost.marks}]` with sub-line `would save {hoursSaved} h`; reason: `Revised in this plan. If you dropped it and could answer nothing on it, it would have cost up to {cost.marks} marks on the {cost.paperLabel} paper. Dropping it is a bet that the next paper leaves a way around it.` + `CiteList cost.cites max 3`; when `maxAlone = 0`: ` Alone it is covered by choice; next to the covered drops it is not.`; then a pencil link line `Raise the gamble budget to {cost.marks} marks to add it to the drop list.` → `/settings#{gambleBudget.key}`. The topic also appears in card 3 with its hours, because the plan revises it.

**Already at your stated standing** (`AtStandingList`). Row: name, margin `[—]`, line `you said: {youSaid}` in blue pen, then `No revision is planned: at what you said you could answer, revising would not move a figure on this page. It is not a drop.`

**No evidence either way** (`NoEvidenceList`). Row: name, never struck, margin `[—]` pencil, `S.noEvidence(N)` = `Never appeared in the {N} {paper / papers} on file. That is silence, not safety.` + `CiteList` of the whole-paper cites (`2022 (whole paper) · 2023 (whole paper)`). Not part of `d`.

COULD: pins (`Keep anyway` / `Drop anyway` → engine `pin`; chip `PINNED BY YOU`). SHOULD: set caveat (`SetCaveat`, budget > 0 only): `interaction` = `No single one of these {d} drops would have cost more than {maxAlone} marks on the {paperLabel} paper. Together, if you scored nothing on any of them, they would have cost {setLoss}. Drops that are each harmless alone can use up the same choice.`; `slack_exhausted` = `Together the {c} covered drops use all of {label}'s slack (skip {slack} of {of}) on the {list(paperLabels)} {paper / papers}. Any further drop that appears in {label} would have cost marks.`

### 4.5 Keep card and backtest card

**Keep row**: name (serif 19 px); margin `[{hours} h]`. Second line, sans 13.5 px, ` · ` joined: standing — `probed`: blue pen `you said: {youSaid}` + pencil chip `THIN` when thin; `self_rating`: blue pen `you said: {youSaid}` + pencil chip `NOT PROBED`; `assumption`: pencil-dashed `{fallback} · assumed` linking `/settings#{restsOn[0]}`; then pencil `hours assumed` when `hoursBasis = "course_figure"`; then `on {count} of the {of} papers on file` + `CiteList cites max 3` when `appearedIn` is non-null (never "often", never a rank). `Trace` (`<details>`) holds `standing.sentence` and `hoursSentence` verbatim: the only place an assumption's number appears here. Order: course order (MUST). SHOULD: when every row has `gainMarks`, order by gain per hour, render pencil `would have gained +{gain}` with the basis in `Trace`, foot `Ordered by marks that would have been gained per hour on the hardest paper on file.` Gains are never totalled. Card foot, pencil: `S.keepFoot`.

**Backtest table**: Paper (serif; `HARDEST` chip on `isHeadline`; red `{issues} FLAG{S}` chip linking to the confirm screen) · In reach `{fmtReach} of {total} ({fmtPct})` · Needed · Aims for · Gap (`fmtGap`; red + `SHORT` chip when `clears` is false) · If every drop scored zero (`fmtLoss`, red when non-zero). The paper name links to `/papers/{paperId}`; the total is a `CiteLink` to `totalCite`. Under the table: `S.coveredDropsCostNothing(d)` when `d > 0`, then always `S.backtestNotPrediction(N)`. `N < 3`: caption `{N} {paper / papers} on file — thin`. The headline row is the first row.

### 4.6 States

| State | Renders |
|---|---|
| `cannot_say` | Card 1 only: h1, one `Blocker` line per blocker with one action. `no-topics` `No topics are listed for this exam.` → `List the topics` · `no-target` explanation verbatim → `Enter your marks so far` · `no-counted-papers`, none on file `No past papers are on file.` → `Add past papers`; some on file `{m} {paper / papers} on file, none counted. Extraction is a proposal until you confirm every row.` → `Review the {paperLabel} paper` · `too-many-topics` / `invalid-input` `detail` verbatim → `Open the course`. |
| `secured`, `secured_if_estimates_hold` | Card 1 only, with the action. |
| `plan` | All four cards, the labelled lists, the plan footer and notes. |
| loading (SHOULD) | `loading.tsx`: shell + pencil `Checking every combination of {n} topics…` |
| error | `error.tsx`: `Something failed while building this page. Your data is untouched: {dataPath}.` + `Try again`. |

Edge cases (each a fixture): a name that wraps (the strike wraps); 10+ drops (why-panels closed); one counted paper; papers with different totals (the sentence names the headline paper; the table carries the rest); d = 0; k = 0; hours not stated; margin 0; e > 0; a > 0; g = 0.

---

## 5. The confirm screen (`/papers/[paperId]`) — MUST

### 5.1 Layout

```
Does this match the 2024 paper?
| ! TOTALS DO NOT MATCH  The rows below add up to 102 marks. The paper states a maximum
|   of 100 (p.1). Start with question 3: its parts add up to 7 and the question is
|   marked 5.                                                        [Go to question 3]
+-- page pane (sticky, 44%) ------------+-- rows --------------------------------------------+
| 2024 paper · p.2 of 8    [ prev next ]| 14 of 27 rows confirmed · 1 flag                   |
| +-----------------------------------+ |        [Confirm the 12 rows with no flag]          |
| |  page image, 150 dpi              | |----------------------------------------------------|
| |  ░ highlight band (SHOULD) ░      | | 1     Answer all parts        [20] p.2 ✓ CONFIRMED |
| +-----------------------------------+ | ┃ 1(a) Define …  [Rivers   v] [ 2] p.2 ✓ CONFIRMED |
| Synthetic fixture page. Not a real    | ┇ 3(a) Outline … [Coastal  v] [ 2] p.3  [Confirm]   | <- focused
| exam paper.                           | ┃ 3(b) Explain … [Coastal  v] [ 5] p.3  ! CHECK MARKS|
|                                       | B     Answer any 5 of 7      [60] p.4  [Confirm]   |
```

Row grid `label 54px | excerpt minmax(240px,1fr) | topic 170px | marks 64px | cite 56px | state 104px`. Below 1000 px the pane stacks above the rows ("doesn't break").

### 5.2 Provisional versus confirmed — four signals

| | Provisional | Confirmed |
|---|---|---|
| Text | `pencil` | `ink` |
| Left rule | 3 px dashed pencil | 3 px solid blue pen |
| Inputs | dashed border, pencil text | solid `rule-strong` border, ink |
| State cell | `Confirm` button | check icon + `CONFIRMED` in blue pen |

A leaf whose mapping is `pending` shows pencil `mapping…` in its topic cell and its `Confirm` button disabled with title `Waiting for a topic`. A flagged row swaps the rule for 3 px solid red, shows the warning icon and the flag's short label in the state cell, and outlines the offending input in red. Once, under the bar: `S.provisionalHelp`.

### 5.3 Reviewing 20+ rows

- Rows are `<li tabIndex={0}>` inside `role="list"`. Clicking or focusing a row sets the page pane to `row.cite.page` (and the band, if `bbox` ships). `ArrowDown` / `ArrowUp` on a focused row move focus. On load, focus goes to the first unconfirmed row.
- `Confirm the {n} rows with no flag` (primary). `n = bulkConfirmable(rows).length` = rows not confirmed, with no flag, and (leaves) not `pending`. Pencil under it: `S.confirmAllHelp`. Disabled at `n = 0`. No shortcut: a deliberate click or Tab + Enter.
- Per-row `Confirm` button calls `confirmRows(paperId, [nodeId])`. Editing marks or topic calls `updateLeaf`, which validates and confirms the row (the edit is the student's statement). Marks accept multiples of 0.5 in `0..max`, `max = paper.statedTotal ?? paper.attainableTotal`; anything else shows `S.marksInputError(max)` and makes no call; the action validates again server-side.
- Topic select: course topics in course order, then `None of these`. A multi-topic leaf shows `{first} +{n}` read-only.
- When `paper.status.kind === "counted"` the bar becomes the done banner (blue-wash): `All {rows} rows confirmed. The {paperLabel} paper now counts.` + `Back to papers` · `See the drop list`. While any mapping is pending the bar's count line adds `{m} rows are still being mapped to topics. This paper counts once every row is confirmed.`
- State is `useState` over `{ vm, focusId, panePage, pending }`; the pure helpers `nextToReview(rows, fromId)`, `bulkConfirmable(rows)` and `flagsFor(row, flags)` live in `src/components/confirm/confirm-helpers.ts` and are unit-tested.
- SHOULD: `Enter` confirms the focused row and moves to the next unconfirmed one; `e`/`t` focus marks/topic; `u` unconfirms; `[`/`]` page without moving focus; `updateGroup` pick editing; an `aria-live` region.

### 5.4 Validator flags

Paper-scope flags render as `FlagNote`s above the workspace; row-scope flags on the row and in the bar's count. `flagMessage(flag, rows)` in `src/copy/flags.ts` → `{ tag, body, short, jumpLabel }`. A `fatal` flag prefixes `THIS PAPER CANNOT COUNT YET · `.

| code | tag | body | short |
|---|---|---|---|
| `total-mismatch` | `TOTALS DO NOT MATCH` | `The rows below add up to {actual} marks. The paper states a maximum of {expected} ({cite}).` + hint | — |
| `group-marks-mismatch` | `PARTS DO NOT ADD UP` | `Question {label}: its parts add up to {actual} and the question is marked {expected} ({cite}).` | `CHECK MARKS` |
| any other | the code in caps, hyphens as spaces | `detail` verbatim + ` ({cite})` | `CHECK ROW` |

Hint: if a `group-marks-mismatch` exists ` Start with question {label}: its parts add up to {actual} and the question is marked {expected}.`; else if `visionPages` is non-empty ` Start with the pages read by the vision model: {pages}.`; else ` Check the marks column against the page.` SHOULD: dedicated templates for `pick-exceeds-children`, `numbering-gap`, `zero-mark-leaf`/`invalid-marks`, `no-stated-total`, `rows-without-page`.

Each note has `Go to {label}` focusing `nodeIds[0]`. Resolution is only "make the rows match the page"; a flag disappears when its condition is false; there is no dismiss. A row with a non-fatal flag can be confirmed; the flag travels to `/papers`, the `tree_issues` note and the backtest table. When totals match, a quiet ink line (`data-testid="totals-check" data-result="pass"`): `Totals check passed: the rows add up to {x} and the paper states {x} ({cite}).`

### 5.5 Other states

Still extracting: the pane works once pages exist; the right side shows `ExtractionProgress` and rows as they arrive. Zero rows: `No questions were found in this PDF.` + reason + link back. Page image fails: pencil `Page image unavailable.`; rows stay usable. Vision pages: pencil `S.visionPages(pages)`. From cache: pencil `S.fromCache(t)`. Unknown id: `not-found.tsx` `No paper with that id is on file.` COULD: `Re-read this page live` with a timer (one page, one call, timed out per D-J).

---

## 6. The other screens

### 6.1 `/papers` — MUST

Card `1. Which past papers are on file?` margin `[{n}]`. `DropZone`: dashed box, serif `Drop past-paper PDFs here`, real button `Choose files` (`<input type="file" accept="application/pdf" multiple>`), pencil `S.uploadPrivacy`. One result line per file: accepted → the row appears; duplicate → `Already on file: the {paperLabel} paper ({"confirmed and counted" | "not yet confirmed"}).`; rejected → red `Could not read {filename}: {reason}. Nothing was added.` `PaperRow`: label, filename + pages (pencil mono), status, margin `[{rows} rows]`.

| status | text | action |
|---|---|---|
| queued | `Waiting for the model` | — |
| reading text / vision / mapping | `Reading p.{page} of {pageCount} from the text layer` / `… with the vision model · {elapsedS} s` / `Mapping questions to topics` | — |
| needs_review | pencil `PROVISIONAL` + `{confirmed} of {rows} rows confirmed` (+ red `{flags} FLAG{S}`) | primary `Review` |
| counted | blue check + `Confirmed · counts · {rows} rows` (+ flags chip) | `Open` |
| excluded | pencil `Left out: {reason in words}` + note | `Open` |
| failed | red `Could not read this file: {reason}` | SHOULD `Remove` |

Empty: `No past papers on file yet. Droplist can say nothing about this exam until one is confirmed, and ranks nothing until there are three.` `thin_papers` note when 1 or 2 count. Synthetic papers carry chip `SYNTHETIC`. SHOULD: rename, `Leave out of the backtest` with a reason, remove.

### 6.2 `/setup` — MUST

**Card 1 `What is on this exam?`** margin `[{n} topics]`. Header verbatim (02 hours header). Table: name, `own hours` (step 0.5, blank = course figure), mapped-questions count (pencil), `TopicHours.sentence`. Textarea `Paste topics, one per line` + `Add topics`; `| 6` at a line's end sets own hours. SHOULD: delete.

**Card 2 `What do I need on the final?`** (`GradeForm`, client; `assessTarget` is pure and imported client-side). Per component: title, weight, `scored`/`out of`, `marks | %` toggle, and on non-final rows `returned mark | my estimate`; radio `this is the exam I am revising for`; `Target overall`; `The final is out of` (prefilled from the latest counted paper's stated maximum, pencil with its cite until accepted). `NeededSentence` (`data-testid="needed-sentence"`, serif 22 px):

| Situation | Sentence | Sub-line (mono) | Trace |
|---|---|---|---|
| `needs`, total known | `You need {marksNeeded} of {paperTotal} on the final.` | `{needPct}% of the paper · plans aim for {planTarget}, a {marginMarks}-mark buffer above the marks needed{ — unconfirmed}` | explanation verbatim |
| `needs`, total unknown | `You need {needPct}% of the final. Say what the paper is out of to see that in marks.` | — | same |
| `secured` (official) | `Your target is already secured.` + the 4.3 sentence | — | — |
| `secured` (estimates) | `Your target is secured if your estimates hold.` + the estimate sentences | — | — |
| `beyond`, `blocked` | explanation verbatim | — | bracket sentence (SHOULD) |

Worked example (02 example A, D-O rounding): 17/20 weight 20 → 17; 24/30 weight 30 → 24; final weight 50; target 70 → need 58 %; total 100, margin 5 → `planTarget = min(100, ceil(0.63 × 100 − 1e-9)) = 63`, `marginMarks = 5` → `You need 58 of 100 on the final.` / `58% of the paper · plans aim for 63, a 5-mark buffer above the marks needed — unconfirmed`. The margin's anchor is `/settings#{marginKey}` read from the registry def.

**Card 3 `How much time do I have?`** Exam date (prefills days), mode `total | per day`, the fields, `HoursAvailable.sentence` verbatim.

### 6.3 `/standing` — MUST (minimal)

Card `1. Could I answer what has been asked before?` margin `[{probed} of {n} probed]`. Intro: `S.noAnswers`, `S.probeAnchor`. One `<details>` per topic: summary = name, the standing line as in the keep row, margin `[10 of 22]` marks for probed topics, `[—]` otherwise; never a bare percent. Inside: `standing.sentence` verbatim, then one `ProbeItem` per item from 02 `selectProbeItems`: `{leafLabel} [{marks}]`, the excerpt, `PageImage` at 520 px (the same component as the viewer; a part cannot be judged without its page), and a server-action form with 02's four buttons `Could answer` · `Partly — some of the marks` · `No` · `Skip — can't judge from this page`; the chosen one is blue-pen filled. No timer, no score. `no_questions_on_file`: sentence only. SHOULD: keys `1 2 3`; `contextPages` strip; self-rating row (`solid / shaky / not started`) rendered `you said: solid` in blue pen with pencil chip `NOT PROBED`.

### 6.4 `/settings` ("Assumptions") — MUST, generated

Header `Every number the planner assumes is on this page. Nothing is hidden in code.` A `QuestionCard` per group, a `SettingRow` per def by `order`, each row **its own form** (one shared form would confirm every default in one click): label, help, control by `type`, unit, `Save`; unconfirmed → pencil chip `UNCONFIRMED`, help suffix `This is a default standing in for a figure only you know.`, button `Keep {fmtSetting(default)}`; confirmed → blue check `Set by you` + `Reset`; locked → read-only, chip `FIXED BY DESIGN`; invalid stored value → red `Stored value "{raw}" was refused: {reason}. Using {default}.`; a refused save shows the reason and writes nothing. Row ids are `#{key}`. The gamble budget (`gamble_budget_marks`) and the no-topic assumption (`no_topic_answerable_pct`) are ordinary rows here. Fixed card `Where does the data live?`: `One SQLite file at {dataPath}. Uploaded PDFs and page images sit beside it. Nothing here talks to a server.`

### 6.5 The page viewer — MUST

`CiteLink` renders `{paperLabel} p.{page}` (or `{paperLabel} (whole paper)`) in mono blue pen, underlined, as `<Link href="?cite={paperId}.{page}" scroll={false}>` (+ `&hl=x,y,w,h` when `bbox` exists). `CiteLayer` (client, once in `layout.tsx` inside `<Suspense>`, reads `useSearchParams`) opens a modal `<dialog>`: title `{paperLabel} paper · p.{page} of {pageCount}`, the image (`max-height: 86vh`), prev/next, close, and for synthetic papers `S.syntheticPage`. `Escape`, close and backdrop call `router.replace(pathname, { scroll: false })`; focus returns to the opening link. SHOULD: the highlight band `.hl-band`.

---

## 7. Honesty copy — exact strings (`src/copy/strings.ts`) — MUST

```ts
export const S = {
  backtestNotPrediction: (N) => `This is a backtest against the ${N} ${plural(N,"paper","papers")} on file. It is not a prediction of the next paper.`,
  thinPapers: (N) => `Only ${N} ${plural(N,"paper counts","papers count")}. With fewer than 3, Droplist does not rank topics by how often they come up, and this backtest is thin: one unusual paper could change every line on this page.`,
  thinPapersAction: "Add another past paper",
  paperExcluded: (label, reason) => `The ${label} paper is left out of every figure on this page: ${reason}.`,
  exclusionReason: { "structure-unconfirmed": "its rows are not all confirmed", "user-excluded": "you left it out",
    "broken-tree": "its structure has an open flag that makes its total unusable", "no-marks": "no marks were found on it" },
  treeIssues: (label, n) => `The ${label} paper has ${n} open ${plural(n,"flag","flags")}. Its figures are used as they stand.`,
  noTopicLeaves: (count, marks, pct, confirmed) => `${count} ${plural(count,"question","questions")} on the counted papers (${marks} marks) ${plural(count,"matches","match")} none of your topics. The plan counts ${plural(count,"it","them")} at ${pct}% answerable${confirmed ? "" : ", a default you have not confirmed"}.`,
  approximate: (n) => `With ${n} topics, Droplist searched for a good plan instead of checking every combination. A better plan may exist.`,
  assumptionsLead: (m) => `This plan rests on ${m} ${plural(m,"assumption","assumptions")} you have not confirmed:`,
  assumptionsAction: "Review assumptions",
  youSaid: "you said:",
  thinChip: "THIN",
  thinHelp: "Fewer than 3 questions probed. The plan fills the gap with the not-probed assumption.",
  notProbedChip: "NOT PROBED",
  hoursAssumed: "hours assumed",
  keepFoot: "Droplist has no data on how long revision takes you. These hours are yours.",
  keepOrder: "Ordered by marks that would have been gained per hour on the hardest paper on file.",
  coveredDropsCostNothing: (d) => `The ${d} covered ${plural(d,"drop","drops")} would have cost 0 marks on every paper on file.`,
  gamblesLabel: "GAMBLES: NOT IN YOUR DROP LIST",
  gambleReason: (marks, label) => `Revised in this plan. If you dropped it and could answer nothing on it, it would have cost up to ${marks} marks on the ${label} paper. Dropping it is a bet that the next paper leaves a way around it.`,
  gambleSameSlack: "Alone it is covered by choice; next to the covered drops it is not.",
  gambleRaise: (marks) => `Raise the gamble budget to ${marks} marks to add it to the drop list.`,
  atStandingLabel: "ALREADY AT YOUR STATED STANDING",
  atStanding: "No revision is planned: at what you said you could answer, revising would not move a figure on this page. It is not a drop.",
  noEvidenceLabel: "NO EVIDENCE EITHER WAY",
  noEvidence: (N) => `Never appeared in the ${N} ${plural(N,"paper","papers")} on file. That is silence, not safety.`,
  provisionalHelp: "Pencil rows were read by a model. Nothing in pencil counts until you confirm it.",
  confirmAllHelp: "Confirming means you have checked these rows against the page. Flagged rows stay provisional.",
  marksInputError: (max) => `Marks are a number from 0 to ${max}, in halves.`,
  syntheticFooter: "Demo course · synthetic papers generated by a script in this repo · not set by any school or exam board",
  syntheticPage: "Synthetic fixture page. Not a real exam paper.",
  syntheticChip: "SYNTHETIC",
  localOnly: "Local only · nothing leaves this machine",
  uploadPrivacy: "Files stay on this machine. They are read by local models only.",
  noAnswers: "Droplist shows you past questions and asks whether you could answer them. It never writes an answer.",
  probeAnchor: "“Could answer” means: closed book, today, for most of its marks.",
  fromCache: (t) => `Read earlier on this machine. Loaded from the extraction cache in ${t.toFixed(1)} s.`,
  visionPages: (pages) => `${plural(pages.length,"Page","Pages")} ${list(pages.map(String))} had no usable text layer and ${plural(pages.length,"was","were")} read by the vision model. Check those rows with extra care.`,
  internalFigureError: "A figure on this page could not be computed, so nothing is shown. Your data is untouched.",
} as const;
```

**Vocabulary sweep**, `src/copy/honesty.test.ts`: renders every string with sample arguments, plus every `headline`, `dropReason`, `whySummary`, `whyLines`, `setCaveatText` and `flagMessage` output for every fixture, and fails on:

- `/\bwill (appear|come up|be asked|be on)\b/i`, `/\blikely\b/i`, `/\bexpect/i`, `/\bguarantee/i`, `/\breadiness\b/i`, `/\bscore\b/i` (`scored`/`scoring` allowed), `/\bworst\b/i`, `/\bpredict(s|ed|ing)?\b/i` (`prediction` allowed only in `backtestNotPrediction`);
- `/every topic can be dropped/i`, `/\bsafety margin\b/i`, `/\b(un)?tested\b/i`, `/\bmeasure(d|ment)?\b/i` after removing the literal phrases `not measured` and `not a measurement`, `/\bcarr(y|ies) (it|them)\b/i`;
- `safe` in any string that does not also contain `papers on file`; `/\b(behind|failing|lazy|weak)\b/i`.

The Playwright spec repeats the sweep on `body.innerText` of every route. That run is the contract for sentences owned by 01 and 02: a failure there is filed against the owning spec, never patched in this copy. Known offenders on disk are listed in section 15.

---

## 8. Design system: "The Marked Paper"

### 8.1 Rules (MUST)

1. **Three inks and the red pen** (D3). A colour is never used outside its meaning: no red buttons, no blue decoration, no grey text that is not `pencil` (provisional) or `ink-2` (secondary).
2. **No boxes.** Questions are separated by whitespace and hairlines. Filled surfaces: the sheet, why-panel and page pane (`paper-2`), a note (`red-wash`), the focused confirm row and the done banner (`blue-wash`).
3. **Margin marks.** A quantity that answers a row goes in the right margin, mono, in brackets. Body text does not repeat it.
4. **Serif** for anything that is or quotes the paper; **sans** for chrome; **mono** for marks, hours, cites, counts.
5. Radius 2–3 px on controls, 0 on the sheet. One shadow (the sheet). One motion (the strike, SHOULD).

### 8.2 `src/app/globals.css` (MUST — copy as written)

```css
@import "tailwindcss";
@theme static {
  --color-*: initial;
  --color-desk: #ece8df; --color-paper: #fcfbf7; --color-paper-2: #f4f1e9;
  --color-rule: #d9d4c7; --color-rule-strong: #858073; --color-margin-rule: #e6b1ab;
  --color-ink: #16181d; --color-ink-2: #3d4148; --color-pencil: #5f636a;
  --color-red-pen: #c3211b; --color-red-wash: #fbe9e6;
  --color-blue-pen: #1d3fa6; --color-blue-wash: #e8ecf8; --color-on-accent: #ffffff;
  --color-highlighter: #ffe45c;
  --font-serif: var(--font-stix), "STIX Two Text", "Times New Roman", Times, serif;
  --font-sans: var(--font-atkinson), system-ui, -apple-system, "Segoe UI", sans-serif;
  --font-mono: var(--font-dm-mono), ui-monospace, Menlo, monospace;
  --text-headline: 42px; --text-headline--line-height: 1.08;
  --text-need: 22px;  --text-row: 19px;  --text-lede: 18px;  --text-q: 17px;  --text-prose: 16.5px;
  --text-body: 14px;  --text-small: 12.5px;  --text-label: 10.5px;  --text-marks: 13px;  --text-figure: 21px;
  --spacing-margin: 112px; --spacing-gutter: 40px; --spacing-indent: 26px; --spacing-qgap: 26px;
  --radius-ctl: 3px; --shadow-sheet: 0 18px 40px -28px rgb(40 30 10 / 0.35);
}
@layer base {
  html { background: var(--color-desk); color-scheme: light; }
  body { background: var(--color-desk); color: var(--color-ink); font: var(--text-body)/1.5 var(--font-sans); -webkit-font-smoothing: antialiased; }
  ::selection { background: color-mix(in oklab, var(--color-highlighter) 60%, transparent); }
  :focus-visible { outline: 2px solid var(--color-blue-pen); outline-offset: 2px; border-radius: 2px; }
  a { color: var(--color-blue-pen); text-underline-offset: 2px; }
}
@layer components {
  .sheet { position: relative; max-width: 1040px; margin: 24px auto 64px; background: var(--color-paper);
           border: 1px solid var(--color-rule); box-shadow: var(--shadow-sheet); }
  .sheet-wide { max-width: 1360px; }
  .sheet-margined::before { content: ""; position: absolute; top: 0; bottom: 0; right: var(--spacing-margin);
           width: 1px; background: var(--color-margin-rule); pointer-events: none; }
  .mrow { display: grid; grid-template-columns: minmax(0, 1fr) var(--spacing-margin); }
  .mrow > .mbody { padding: 0 28px 0 var(--spacing-gutter); }
  .mrow > .mmarks { padding-right: 12px; text-align: right; white-space: nowrap;
           font: 500 var(--text-marks)/1.5 var(--font-mono); font-variant-numeric: tabular-nums; }
  .marks::before { content: "["; } .marks::after { content: "]"; }
  .marks-sub { display: block; font: 400 11.5px/1.3 var(--font-mono); color: var(--color-pencil); }
  .label { font: 700 var(--text-label)/1.2 var(--font-sans); letter-spacing: .1em; text-transform: uppercase; color: var(--color-pencil); }
  /* Red pen through the text: one wobbly stroke per line box (S3). */
  .strike { text-decoration: none; color: var(--color-ink-2);
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 200 20' preserveAspectRatio='none'%3E%3Cpath d='M1 12 C 24 7.5, 50 14.5, 82 10 S 140 13.5, 199 8' fill='none' stroke='%23C3211B' stroke-width='2.2' stroke-linecap='round' vector-effect='non-scaling-stroke'/%3E%3C/svg%3E");
    background-repeat: no-repeat; background-size: 100% 100%; background-position: 0 55%;
    -webkit-box-decoration-break: clone; box-decoration-break: clone; padding: 0 .2em; margin: 0 -.2em; }
  .strike-gamble { background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 200 20' preserveAspectRatio='none'%3E%3Cpath d='M1 12 C 24 7.5, 50 14.5, 82 10 S 140 13.5, 199 8' fill='none' stroke='%23C3211B' stroke-width='2.2' stroke-linecap='round' stroke-dasharray='9 7' vector-effect='non-scaling-stroke'/%3E%3C/svg%3E"); }
  .strike-draw { animation: strike-draw 420ms cubic-bezier(.3,.7,.2,1) both; animation-delay: calc(var(--i, 0) * 120ms); }
  .pencil { color: var(--color-pencil); }
  .pencil-line { color: var(--color-pencil); border-bottom: 1px dashed var(--color-pencil); }
  .you-said { color: var(--color-blue-pen); }
  .edge-pencil { border-left: 3px dashed var(--color-pencil); } .edge-pen { border-left: 3px solid var(--color-blue-pen); }
  .edge-red { border-left: 3px solid var(--color-red-pen); }
  .chip { display: inline-flex; align-items: center; gap: 5px; white-space: nowrap; vertical-align: 3px;
    font: 700 var(--text-label)/1 var(--font-sans); letter-spacing: .1em; text-transform: uppercase;
    border: 1px solid currentColor; border-radius: 2px; padding: 3px 6px 3px 5px; }
  .chip-ink { color: var(--color-ink-2); } .chip-red { color: var(--color-red-pen); background: var(--color-red-wash); }
  .chip-pencil { color: var(--color-pencil); border-style: dashed; } .chip-blue { color: var(--color-blue-pen); background: var(--color-blue-wash); }
  .note { border-left: 3px solid var(--color-red-pen); background: var(--color-red-wash); padding: 9px 14px 10px;
    font: italic 400 16px/1.45 var(--font-serif); color: var(--color-ink); }
  .note > .label { color: var(--color-red-pen); font-style: normal; margin-bottom: 5px; }
  .note-pencil { border-left-color: var(--color-pencil); background: var(--color-paper-2); }
  .btn { display: inline-flex; align-items: center; gap: 6px; min-height: 36px; padding: 0 14px; font: 600 13px/1 var(--font-sans);
    border-radius: var(--radius-ctl); cursor: pointer; border: 1px solid var(--color-blue-pen); background: var(--color-blue-pen); color: var(--color-on-accent); }
  .btn-ghost { background: transparent; color: var(--color-blue-pen); } .btn:disabled { opacity: .45; cursor: not-allowed; }
  .field { font: 13px var(--font-sans); color: inherit; background: var(--color-paper); border: 1px solid var(--color-rule-strong);
    border-radius: var(--radius-ctl); padding: 4px 6px; }
  .field-marks { font: 500 13px var(--font-mono); text-align: right; font-variant-numeric: tabular-nums; }
  .field-pencil { border-style: dashed; color: var(--color-pencil); }
  .cite { font: 500 12px var(--font-mono); color: var(--color-blue-pen); text-decoration: underline; white-space: nowrap; }
  .hl-band { position: absolute; background: var(--color-highlighter); opacity: .45; mix-blend-mode: multiply; pointer-events: none; }
}
@keyframes strike-draw { from { background-size: 0% 100%; } to { background-size: 100% 100%; } }
@media (prefers-reduced-motion: reduce) { .strike-draw { animation: none; } * { scroll-behavior: auto !important; } }
@media (forced-colors: active) { .strike { background-image: none; text-decoration: line-through; text-decoration-thickness: 2px; }
  .strike-gamble { text-decoration-style: dashed; } }
@media (max-width: 720px) { :root { --spacing-margin: 76px; --spacing-gutter: 18px; } }
```

The plan footer uses `.note.note-pencil` (pencil rule, recessed ground): it is an assumption list, not an examiner's mark. COULD: a night theme by overriding the colour tokens under `:root[data-theme="night"]` (button text must be dark on the night accent, S1).

### 8.3 Fonts (MUST)

`src/app/fonts/` holds six OFL-1.1 woff2 files copied from `@fontsource/stix-two-text` (400, 400 italic, 600), `@fontsource-variable/atkinson-hyperlegible-next` (wght) and `@fontsource/dm-mono` (400, 500), plus `OFL.txt`. `layout.tsx` declares them with `next/font/local` as `--font-stix` (fallback `"STIX Two Text", "Times New Roman", serif`), `--font-atkinson` (`weight: "200 800"`, fallback `system-ui`) and `--font-dm-mono` (fallback `ui-monospace, Menlo`), and puts the three variables on `<html>`. No synthesised bold-italic.

### 8.4 Type recipes

Headline `font-serif text-headline tracking-[-0.01em]`; lede `font-serif text-lede text-ink-2 max-w-[41em]`; question header `font-serif text-q font-semibold` with the number in a 26 px hang; row title `font-serif text-row`; why/flag prose `font-serif text-prose`; help and cells `font-sans text-body text-ink-2`; ledger figure `font-mono text-figure font-medium`. Question blocks `mt-qgap`; rows `py-[10px]` with a top hairline. Icons: inline 12×12 SVG in `src/components/ink/icons.tsx` (`IconCheck`, `IconWarn`, `IconQuestion`, `IconPage`, `IconClose`, `IconChevron`); no icon library.

---

## 9. Component inventory (`src/components/`)

- `sheet/`: `Sheet { wide?, margined?, children }`, `RunningHead { course }`, `QuestionNav { current, status? }`, `SheetFooter { course }`.
- `card/`: `QuestionCard { n?, question, margin?, id?, testId?, children }` (a `<section aria-labelledby>` with an `h2`; `n` omitted for a labelled list), `MarginRow { marks?, indent?, children }`.
- `ink/`: `Marks { value, tone?, sub?, srLabel }` (brackets are CSS; `srLabel` is the spoken form), `Strike { variant: "safe"|"gamble", index?, animate?, children }`, `Chip { tone, icon?, href?, children }`, `Pencil { underline?, href?, children }`, `YouSaid { children }`, `Trace { sentences }`, `Note { tag, kind, tone?: "red"|"pencil", action?, children }` (`role="note"`, `data-testid="attention-note"`), `AttentionNotes { caveats }`, `icons.tsx`.
- `cite/`: `CiteLink { cite }`, `CiteList { cites, max? }` (dedupes on `(paperId, page)`, sorts by paper order then page), `CiteLayer` (client) `{ papers }`, `PageImage { paperId, page, bbox?, alt, width? }`.
- `droplist/`: `Headline { vm }`, `Ledger { need, plan }`, `PlanFooter { unconfirmed }`, `BlockerList { blockers }`, `DropCard`, `DropRow { row, index, N, c }`, `WhyPanel { row, N, c }`, `GambleList { rows, budget }`, `AtStandingList { rows }`, `NoEvidenceList { rows, N }`, `KeepCard`, `KeepRow { row }`, `BacktestTable { rows, d }`, `SetCaveat` (SHOULD).
- `papers/`: `DropZone` (client) `{ action }`, `PaperList`, `PaperRow { paper }`, `ExtractionProgress` (client) `{ initial, intervalMs? }` (polls; `router.refresh()` when all settle).
- `confirm/` (all client, imported only from `ConfirmWorkspace { initial, actions }`): `PagePane { paper, page, bbox?, onPage }`, `ConfirmBar { confirmed, total, flags, bulkCount, mappingPending, onConfirmAll, pending }`, `ConfirmRow { row, topics, focused, flags, on: { focus, confirm, marks, topic } }`, `FlagNote { flag, rows, onJump }`, `TotalsCheck { paper, flags }`, `DoneBanner { rows, paperLabel }`, `confirm-helpers.ts`.
- `setup/`: `TopicsForm`, `GradeForm` (client), `NeededSentence`, `HoursForm`. `standing/`: `StandingTopic`, `ProbeItem`. `settings/`: `SettingRow { def, value, unconfirmed, invalid }`.
- Pure: `src/copy/{strings,format,headline,drop-reason,why,flags}.ts` (+ `set-caveat.ts` SHOULD), `src/components/__fixtures__/{droplist,confirm}.ts`. SHOULD: `src/app/dev/states/page.tsx` renders every fixture through the real components.

---

## 10. Accessibility (MUST)

Colour is never the only signal: dropped = strike + sr-only prefix + chip with icon; provisional vs confirmed = four signals; a failing backtest row = `SHORT` chip. Contrast per S1. Struck names stay `ink-2` (9.9:1) under a 2.2 px stroke. One `h1` per page; every `QuestionCard` is a `section` labelled by its `h2`; notes are `role="note"`; `Marks` exposes `srLabel` and hides the bracketed form with `aria-hidden`; the backtest is a `<table>` with `<th scope>`; the viewer is a modal `<dialog>` with `aria-labelledby` and focus return; page images have `alt="{paperLabel} paper, page {n}"`. Buttons and probe choices ≥ 36 px; confirm rows ≥ 40 px. Reduced motion and forced colors per 8.2. The keyboard path through the demo (Tab to `2 Papers`, `Choose files`, `Review`, ArrowDown, Tab to the bulk button, Tab through the grade fields, Tab to the first `Why…` summary, Tab to a cite, Escape) is driven once by the e2e spec. SHOULD: per-row `aria-label`s and an `aria-live` region on the confirm screen.

---

## 11. Test ids and the Playwright spec (MUST)

Ids are static strings; identity goes in data attributes.

| Area | `data-testid` (data attributes) |
|---|---|
| Shell | `running-head`, `nav-setup`, `nav-papers`, `nav-standing`, `nav-droplist`, `nav-settings`, `footer-synthetic`, `footer-local` |
| Cites | `cite-link` (`data-paper-id`, `data-page`, `data-scope`), `page-viewer`, `page-viewer-title`, `page-viewer-image` (`data-page`), `page-viewer-synthetic`, `page-viewer-prev`, `page-viewer-next`, `page-viewer-close`, `hl-band` |
| Drop List | `droplist-headline`, `droplist-sub`, `headline-chip` (`data-kind`), `ledger-needed`, `ledger-in-reach`, `ledger-hours`, `plan-footer`, `footer-setting` (`data-key`), `attention-note` (`data-kind`), `blocker` (`data-kind`), `blocker-action`, `drop-card`, `drop-row` (`data-topic-id`, `data-class`), `drop-row-name`, `drop-row-chip`, `drop-row-cost`, `why-toggle`, `why-panel`, `why-summary`, `why-line`, `gamble-list`, `gamble-row` (`data-topic-id`), `gamble-row-cost`, `at-standing-row`, `no-evidence-row`, `keep-card`, `keep-row` (`data-topic-id`), `keep-row-hours`, `keep-row-standing`, `trace`, `backtest-table`, `backtest-row` (`data-paper-id`, `data-headline`, `data-clears`), `backtest-covered-line`, `backtest-disclaimer` |
| Papers | `dropzone`, `file-input`, `choose-files`, `upload-result` (`data-kind`), `paper-row` (`data-paper-id`, `data-status`), `paper-status`, `paper-review`, `paper-open` |
| Confirm | `confirm-workspace`, `page-pane`, `page-pane-image` (`data-page`), `page-prev`, `page-next`, `confirm-count`, `confirm-all-btn`, `confirm-row` (`data-node-id`, `data-kind`, `data-state="provisional|confirmed"`, `data-flagged`, `data-focused`), `confirm-row-btn`, `row-marks-input`, `row-topic-select`, `row-cite`, `row-state`, `flag-note` (`data-code`, `data-fatal`), `flag-jump`, `totals-check` (`data-result`), `confirm-done` |
| Setup | `topics-textarea`, `topics-add`, `topic-row`, `topic-hours-input`, `component-row`, `component-weight`, `component-scored`, `component-out-of`, `component-provenance`, `component-is-final`, `target-input`, `final-total-input`, `grade-save`, `needed-sentence` (`data-kind`), `needed-sub`, `needed-trace`, `hours-total-input`, `hours-save`, `hours-sentence` |
| Standing | `standing-topic` (`data-topic-id`, `data-kind`, `data-thin`), `standing-short`, `standing-sentence`, `probe-item` (`data-leaf-id`, `data-grade`), `probe-page-image`, `probe-yes`, `probe-partly`, `probe-no`, `probe-skip` |
| Settings | `setting-row` (`data-key`, `data-unconfirmed`, `data-locked`), `setting-input`, `setting-save`, `setting-keep`, `setting-reset`, `setting-error` |

**`e2e/demo.spec.ts`**: state from `npm run seed:demo -- --partial`; `mock` LLM provider; every request not to `localhost:4320` aborted. Every expected figure is read from `gt = fixtures/demo/ground-truth.json` (section 13); no number is typed into the spec. `last = gt.papers[gt.papers.length − 1]`.

1. `/`: `blocker[data-kind=no-target]`, no `attention-note`.
2. `/papers`: set all fixture PDFs on `file-input`; expect `gt.papers.length − 1` `upload-result[data-kind=duplicate]` and one `paper-row[data-status=needs_review]` within 5 s.
3. `paper-review`: `confirm-row` count = `last.rowCount`; every row has `row-cite`; `totals-check[data-result=pass]`; `ArrowDown` → `page-pane-image[data-page]` equals the focused row's `data-page`; click `confirm-all-btn`; `confirm-done` visible.
4. `/setup`: fill `gt.demoGrade`; `needed-sentence` = `You need ${gt.expected.needMarks} of ${last.total} on the final.`; `needed-sub` contains `aims for ${gt.expected.planTarget}`; `grade-save`.
5. `/`: `droplist-headline` = `Stop studying ${gt.expected.safeDrops.length} of ${gt.topics.length} topics.`; `drop-row[data-class=safe]` names = `gt.expected.safeDrops`; each `drop-row-name` contains `s.strike`; exactly one `gamble-row`, outside `drop-card`, named `gt.expected.gamble.name`, with `gamble-row-cost` = `−${gt.expected.gamble.costMarks}` and a `cite-link`; `plan-footer` has one `footer-setting` per key in `gt.expected.unconfirmedSettings`; `backtest-disclaimer` contains `not a prediction`.
6. First `why-toggle`; `why-summary` contains `lets you skip ${gt.expected.why.slack} of ${gt.expected.why.of}`; first `cite-link` in the panel → `page-viewer` visible, `page-viewer-image[data-page]` equals the link's `data-page`, `page-viewer-synthetic` visible; `Escape` closes it and focus is on the link.
7. Honesty sweep on every route (section 7).
8. Every `why-summary`, every `keep-row` containing `papers on file`, every `no-evidence-row` and every `gamble-row` contains a `cite-link`; every `confirm-row` contains `row-cite`.
9. `/settings`: `setting-row[data-key=gamble_budget_marks]` exists with `data-unconfirmed=true`; `setting-keep` on it → `data-unconfirmed=false`.

---

## 12. Unit tests

`node --import tsx --test src/copy/*.test.ts src/components/**/*.test.ts`.

- `headline.test.ts`: the six grammar fixtures verbatim; every case row; every `COVERED` / `GAMBLES` / `AT` / `NOEV` form and singular; `paperScope` N = 1, 2, 3; margin 0 shortens NEED; hours not stated shortens HOURS; `cannot_say` pluralises "thing"; the string `every topic can be dropped` appears in no output.
- `format.test.ts`: `fmtReach(66.44) = "66.4"`, `fmtReach(79.99999999999999) = "80"`, `fmtShort(7.71) = "7.8"`, `fmtLoss(0) = "0"`, `fmtLoss(12) = "−12"`, `fmtGap(3.4) = "+3.4"`, `fmtGap(-7.8) = "−7.8"`, `fmtHours(22.5) = "22.5"`; non-finite throws `FigureError`.
- `drop-reason.test.ts`, `why.test.ts`: uniform and mixed forms verbatim; every summary is followed by at least one cite; gamble reason and `gambleSameSlack`; `noEvidence(3)` equals the D-N sentence exactly; `atStanding` verbatim.
- `flags.test.ts`: the three MUST templates; the fatal prefix; the hint prefers `group-marks-mismatch`, then vision pages, then the generic line; an unknown code falls back to `detail`.
- `honesty.test.ts`: the sweep of section 7, including the whitelist for `not measured` / `not a measurement`.
- `confirm-helpers.test.ts`: `nextToReview` wraps and returns null when none; `bulkConfirmable` excludes confirmed, flagged and `pending` rows; a `pending` leaf cannot be confirmed; marks validation accepts `2.5`, rejects `2.3`, `-1`, `max + 0.5`.
- `adapters.test.ts`: a 01 `PlanOutcome` at budget 0 yields `drop` with `classification: "safe"` only, `gambles` from the engine's gamble candidates, `atStanding` from the engine's flag, `noEvidence` with one whole-paper cite per counted paper, `unconfirmed` from `assumptionNotes` filtered to unconfirmed; no adapter path compares `pNow` to `pAfter` or a loss to a budget.

---

## 13. The 90-second demo

**The course** is subsystem 3's (`fixtures/demo/`): one invented course, synthetic and labelled so on every page, 9–10 topics, three or four 100-mark papers, pattern Q1 compulsory + "any 5 of 7" + one internal OR + "any 3 of 5 short notes", tuned so that at need ≈ 58 % two topics are covered drops and one is a gamble (D-F). No other demo course exists. This spec types none of its numbers; it reads `fixtures/demo/ground-truth.json`:

```
{ synthetic: true, courseTitle, topics: [{ id, name, hours }],
  papers: [{ label, total, rowCount, pageCount, pdf }],
  demoGrade: { components: [{ title, weightPct, scored, outOf, provenance }], targetPct },
  expected: { needPct, needMarks, planTarget, marginMarks, safeDrops: [{ id, name }],
    gamble: { id, name, costMarks, costPaperLabel }, headlinePaperLabel, reachOnHeadline,
    hoursPlanned, hoursAvailable, why: { topicId, groupLabel, slack, of }, unconfirmedSettings: [key] } }
```

The paper facts are written by the generator; the `expected` block is produced by running the engine on the fixture at generation time.

**Prepared state** `npm run seed:demo -- --partial` (subsystem 4): every paper but the last on file, confirmed and counted; the last paper's extraction recorded but the paper not on file; probe answers recorded; hours available stated; grade components with weights and `out of` but no marks and no target; the final total accepted; every page pre-rendered to `data/pages/{hash}/{n}.png`; **no setting confirmed** (D-G). `--full`: all papers counted and the grade entered.

**Runbook** (the day before and again an hour before): `lsof -ti:4320 | xargs kill 2>/dev/null; NEXT_TELEMETRY_DISABLED=1 npm run build && npm run seed:demo -- --partial && npm run start -- -p 4320`; open all six routes once; browser 1440×900 at 100 %, Wi-Fi off, the fixture PDFs in a Finder window. Ollama is not needed. Agents running these steps use `timeout` and the background per D-J.

| Time | On screen | Said |
|---|---|---|
| 0:00 | `/`: `Cannot say yet.`, one blocker (marks so far), footers. | "Nine days out, the useful question is what I can stop studying. Droplist answers it from my school's own past papers. Right now it says nothing, because it doesn't know what I need." |
| 0:10 | `/papers`: drag the PDFs. Duplicates for the earlier papers; the last one: `Read earlier on this machine…`, `PROVISIONAL · 0 of {rowCount} rows confirmed`. | "Three past papers. Two I checked earlier. The third was read on this laptop, no cloud, and everything it read is in pencil until I check it." |
| 0:22 | `Review`. Page left, rows right. `ArrowDown` three times: the pane follows. Point at the `any 5 of 7` group row, then at `Totals check passed…`. Click `Confirm the N rows with no flag`. Pencil turns to ink; the done banner. | "Every row points at its page. It found the structure, and a plain arithmetic check agrees with the maximum printed on page one. I confirm, and only now does this paper count." |
| 0:40 | `/setup` card 2: type the `demoGrade` marks and target. The sentence updates live: `You need {needMarks} of {total} on the final.` and the buffer line. | "My marks so far, and the overall I want. So I need {needMarks} on the final. That is arithmetic, not a model." |
| 0:55 | `/`: `Stop studying 2 of {n} topics.`; two strikes; the ledger; the plan footer listing every unconfirmed default; the gamble list below with its cost. | "Stop studying two topics. The rest take {hoursPlanned} of my {hoursAvailable} hours, and on the hardest paper on file that would have put {reach} in reach against {needMarks} needed. It lists every number in that which is still a default I haven't confirmed. And it shows one more drop it will not make for me: a gamble, with what it would have cost." |
| 1:10 | `Why is this drop covered…` on the first drop; the summary; click a cite; the page opens; `Escape`. | "Why is it covered? {groupLabel} lets me skip {slack} of {of}, and on every paper on file this topic only ever appeared there. There's the page." |
| 1:24 | Card 4; point at the disclaimer. | "It's a backtest against the papers on file, not a prediction. It never writes an answer, and nothing left this machine." |

**Encore** ("what if I took the gamble?"): `/settings`, set `Gamble budget` to `gamble.costMarks`, `Save`. `/`: the gamble moves into the drop list with the broken strike and the chip `GAMBLE · {G}-mark budget, set by you`; the headline reads `Stop studying 3 of {n} topics.` and the last column of card 4 shows the cost. Requires the SHOULD budget-in-drop-list rendering; rehearse it or skip it.

**Fallbacks**: (1) cache miss, live extraction starts: say "live extraction takes about a minute on this laptop; here is one I prepared", run `seed:demo -- --full` from a ready terminal tab, reload, continue from 0:22 by opening the last paper. (2) Drag fails: `Choose files`. (3) The app will not start: play `docs/demo/droplist-demo.webm` (SHOULD, recorded by the e2e run the day before). (4) Only if everything is green and time remains: `Re-read this page live` (COULD).

---

## 14. Priorities and build order

**MUST** (about 16 agent-hours):
1. `globals.css`, fonts, `layout.tsx`, shell and ink primitives, icons. (2.5 h)
2. `src/copy/**` with tests; the grammar and confirm fixtures. (2 h)
3. Drop List: `Headline`, `Ledger`, `PlanFooter`, `AttentionNotes`, `BlockerList`, `DropCard`/`DropRow`/`WhyPanel`, `GambleList`, `AtStandingList`, `NoEvidenceList`, `KeepCard`, `BacktestTable`, every state of 4.6, the adapter and its test. Build against fixtures first. (3.5 h)
4. `CiteLink`, `CiteLayer`, `PageImage`. (1 h)
5. `/papers`: `DropZone`, `PaperList`, `ExtractionProgress`, upload results. (1.5 h)
6. Confirm: workspace, pane, bar, row, the three flag templates, totals line, done banner, helpers and tests, click/ArrowUp/Down focus. (2.5 h)
7. `/setup`: three cards, live `NeededSentence`. (1.5 h)
8. `/standing` minimal. (0.75 h)
9. `/settings` generated. (0.75 h)
10. Test ids, semantics pass, the e2e spec. (1 h)

Cut from MUST to get there: the set caveat; per-paper why-lines; five flag templates; `assembling`/`mapping_topics` statuses; Enter/`e`/`t`/`u`/`[`/`]` keys and the reducer; per-row aria-labels and the live region; `updateGroup`; keep-row gains and ordering; the gamble-in-drop-list rendering (budget > 0); nav state; the strike animation.

**SHOULD** (one line each): budget > 0 rendering (broken strike, chip, `SPLIT ESTIMATED`, `COVERED` mixed form) · set caveat · per-paper why-lines and gamble why-panel · remaining flag templates · confirm keys, `unconfirmRow`, `updateGroup`, aria-live · highlight band · keep gains and ordering · nav state · strike draw-on · `loading.tsx` · `/dev/states` · probe keys `1 2 3`, context strip, self-rating row · rename / leave out / remove a paper; delete a topic · the 02 what-if line under the ledger, verbatim · demo recording · next-page preload in the pane.

**COULD**: night theme · `Re-read this page live` · pins · a red ellipse around `{d} of {n}` · print polish · `#why-{topicId}` deep links · provisional-mapping tier (counting a paper with confirmed structure and pending mappings, with every dependent figure marked) — cut by D-E.

---

## 15. Cross-subsystem: divergences, dependencies, closed questions

**Engine (1)** — 01 on disk predates the founder decisions; this spec consumes the shape below and names where it differs.
- D12's ladder and three incumbents are gone (D-B, D-K): one objective, one constraint set, `gambleBudgetMarks` default 0 from the registry. `gambleBudget.honoured` is not read.
- Needed on `PlanOutcome`: `gambles: GambleItem[]` = topics the plan revises only because of the budget, each with `cost` (max over counted papers of `loss(drops ∪ {t})`, with cites), `maxAlone`, `minutesSaved`, `perPaper`; `atStanding: TopicId[]` (D-L: `pAfter == pNow`; excluded from drops, gambles and the budget); `splitEstimated: boolean` (D-K, greedy split above 12 candidates); `maxExactTopics = 16` in `DEFAULT_SOLVER`. With budget > 0, gambles inside the drop set arrive as `DropItem.classification = "gamble"` after the post-solve split.
- `Provisional` lists on a counted paper are always empty under D-E; the UI does not render them. `pNoTopic` comes from `no_topic_answerable_pct`.
- Its `secured` kind is superseded by the two D-M kinds, decided from 02's provenance.
- The sentence "hardest paper on file" and the past conditional are kept from its section 10; its table column "cost of the safe drops (always 0)" is a sentence under the table (closes Q5).

**Inputs (2)**
- `marksOnPaper` follows D-O: `planTarget = min(T, ceil((needPct + marginPct)/100 × T − 1e-9))`, `marginMarks = planTarget − marksNeeded`; the UI prints `marginMarks`, never a recomputation.
- Registry: `safety_margin_pct` is labelled **Buffer above the marks needed** (the key may stay; the sweep bans the old label); new rows `gamble_budget_marks` (plan, integer marks, default 0) and `no_topic_answerable_pct` (standing, default 40); every default in D-G is `unconfirmedUntilSet: true`, including `probe_partly_credit_pct` and the self-rating rows; the seed confirms nothing (its §6.3 is superseded).
- `TargetOutcome.already_secured` must be distinguishable by provenance: either a `basis: "official" | "estimates"` field or the adapter derives it from `standing.estimated`; the estimate titles feed `secured_if_estimates_hold`. `PlanNeed.secured.explanation` is never rendered; "every topic can be dropped" must leave that spec.
- Sentences rendered verbatim must pass the section-7 sweep. Known offenders on disk: `Not tested.` (two `planningP` sentences → `Not probed.`), the `untested_answerable_pct` label/help (`untested` → `not probed`), the `p_target_pct` help (`you expect` → `counted for you`). The adapter maps `TopicStanding.kind = "untested"` to `not_probed` whatever 02 calls it.
- `SelfRating` renders as `you said: solid` in blue pen with `NOT PROBED`; its number appears only in `Trace` and on `/settings` (D-A).

**Extraction (3)**: the D-F fixture and `fixtures/demo/ground-truth.json` with the keys of section 13; every row has a page; optional `bbox` per row; uploads de-duplicated by content hash; recorded extraction replayed for the fixture PDFs so a drop appears as `fromCache` within a second (D-I); synthetic label on every page; a status feed with page-level progress and the pages read by the vision model.

**Data and app (4)**: loaders adaptable to 3.2; the actions and two routes of 3.2; validators re-run in every mutating action and return `FlagVM`s with node ids; **a paper counts iff every row is confirmed and no fatal flag is open** (one `confirmed` flag per row; `confirmRows` refuses a `pending` leaf); marks validated in halves within `0..max`; `npm run seed:demo -- --partial | --full` producing the states of section 13 and pre-rendering every page PNG so no PDF tool runs during the demo; `today` injected at the loader edge.

**Closed**: Q1 demo shape → D-F. Q2 footer placement → the plan footer is the note under card 1 (D-G). Q3 → one `confirmed` flag per row (D-E). Q5 → sentence under the table (D-B: at budget 0 the column is always 0).

**Open**: Q4 multi-topic leaves are read-only on the confirm screen. Q6 if extraction cannot supply `bbox` this week, the band is cut and the 1:10 beat points at the page by hand.
