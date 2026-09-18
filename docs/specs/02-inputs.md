# 02 — Inputs: marks needed, the Probe, hours, the settings registry

> Subsystem 2 of DROPLIST. Read `docs/BRIEF.md` first. The founder decisions of 2026-09-18
> (D-A … D-P in the build brief) are applied here and are not reopened.
> Everything in this subsystem is deterministic, pure TypeScript: no model call, no DB handle,
> no clock, no I/O inside `src/engine/**`.
> Tags: **MUST** (the demo path or the honesty contract breaks without it), **SHOULD** (after
> the MUST set is green), **COULD** (not before the presentation). MUST is about 1.5 agent-days (§10).

## 0. Decisions in one screen

| # | Decision | Tag |
|---|---|---|
| D1 | The student types **raw marks** (`17` out of `20`); a percentage is `outOf = 100` with `enteredAs: "percent"`. `markPercent` is the only place a raw mark becomes a percentage. | MUST |
| D2 | Outcomes: `cannot_simulate / already_secured / secured_if_estimates_hold / reachable / beyond_reach`. `already_secured` fires on **returned marks only**; with an estimate involved the kind is `secured_if_estimates_hold` and the copy names the estimate. "Every topic can be dropped" appears nowhere. | MUST |
| D3 | Weights that do not sum to 100 (tolerance 0.05) **block the figure**. Nothing is scaled. | MUST |
| D4 | The final is the only unmarked component. Any other unmarked component with weight > 0 blocks the figure until the student types an **estimate** (`provenance: "estimate"`), named in every sentence that rests on it. Droplist never invents a coursework mark. | MUST |
| D5 | One rounding: `marksNeeded = ceil(needPct/100 × T − 1e-9)`; `planTarget = min(T, ceil((needPct + marginPct)/100 × T − 1e-9))`; `marginMarks = planTarget − marksNeeded`. The epsilon is required (§8). Every past paper gets its own target from the same function. | MUST |
| D6 | Probe evidence is **marks-weighted**: `answerable = (marksYes + c·marksPartly) / marksShown`, counts kept beside it. | MUST |
| D7 | `pNow = answerable × p_target`, `pAfter = p_target`. A question the student says they could answer counts at the revised-topic rate, not 100%. `p_target` does two jobs — revision yield and a haircut on self-report — and its help says so. | MUST |
| D8 | `partly` credit `c = 0.5`, a setting. It rarely decides whether a plan exists (1–2% of simulated scenarios) but changes **which** topics are dropped in about a quarter (§8). | MUST |
| D9 | `TopicStanding` is `no_questions_on_file / not_probed / probed`, integers only. Planner numbers come from `planningP` alone, each with a `basis`, an `assumedShare` and the settings it rests on. Self-report is never called "tested" or "measured". | MUST |
| D10 | Thin (`n < 3`): the missing `3 − n` answers are filled with the unprobed assumption `a0`; `assumedShare = (3 − n)/3` is reported and drawn in pencil. | MUST |
| D11 | Unprobed default `a0 = 50%`; `none_of_these` questions use their own setting `no_topic_answerable_pct` (40%). Both unconfirmed until set. | MUST |
| D12 | Hours: one course-wide `hours_per_topic_cold` (3 h, unconfirmed) plus per-topic own hours. No scaling by standing. Every sentence says whether a figure is the student's or Droplist's placeholder. | MUST |
| D13 | `p_target` 80%, buffer 5%, gamble budget 0 marks: unconfirmed until set, first on `/settings`, always named on the plan. The label is **Buffer above the marks needed**, never "safety margin". | MUST |
| D14 | Registry declared once, typed, `/settings` generated from it, saved **one row at a time**; a stored value that fails validation is refused and quoted back, never clamped. Confirming is deliberate and is not evidence: the plan footer always lists every figure the plan rests on, set or not. | MUST |
| D15 | Self-rating (three words) is an **assumption selector**: it replaces `a0` for a topic, is shown as a word, never overrides a non-thin probe. | SHOULD |

**Inputs vs settings.** An *input* is a fact about this student and this exam: no default; it blocks (or degrades, stated) until typed — components, marks, target, paper total, hours, own hours, self-ratings, probe answers. A *setting* is an assumption the engine needs: it has a default and is `unconfirmed` until set.

## 1. Files

| Path | Owns | Tag |
|---|---|---|
| `src/engine/grade.ts` | `markPercent`, `assessTarget`, `marksOnPaper`, `planNeed` | MUST |
| `src/engine/standing.ts` | `TopicStanding`, `buildStanding`, `answerable`, `planningP` | MUST |
| `src/engine/probe.ts` | `selectProbeItems`, probe copy constants | MUST |
| `src/engine/hours.ts` | `hoursAvailable`, `topicHours`, `studyDaysBetween` | MUST |
| `src/engine/inputs.ts` | `buildPlannerInputs`, `yieldCeiling`, `assumptionFooter` — the hand-off to subsystem 1 | MUST |
| `src/config/settings.ts` | registry, `resolveSettings`, `validateSetting`, `engineSettingsFrom`, `assumptionNotes`, `fmtSetting`, `defOf` | MUST |
| `src/contracts/standing.ts` | zod schemas for `TopicStanding`, `ProbeResponse`, `GradeComponent` (subsystem 4's directory; content specified here) | MUST |
| `src/engine/{grade,standing,probe,hours,inputs}.test.ts`, `src/config/settings.test.ts` | colocated tests, `node --import tsx --test` | MUST |

`src/engine/**` may `import type` from `src/config/settings.ts` and nothing else outside `src/engine/`; extensionless relative imports, run via tsx. The app layer calls `engineSettingsFrom(resolveSettings(stored))` and passes the result in. Setting keys are identifiers, not UI words.

## 2. `grade.ts` — marks needed

### 2.1 Types

```ts
export const WEIGHT_TOLERANCE = 0.05;   // also a locked registry row
export const EPS = 1e-9;

export type MarkProvenance = "official" | "estimate";
export type ComponentMark =
  | { kind: "unmarked" }
  | { kind: "marked"; scored: number; outOf: number; enteredAs: "raw" | "percent"; provenance: MarkProvenance };
export type GradeComponent = { id: string; title: string; weightPct: number; isFinal: boolean; mark: ComponentMark };
export type GradeInput = { components: GradeComponent[]; targetPct: number };

export type WeightCoverage =
  | { state: "complete" } | { state: "no_weights" }
  | { state: "short"; declared: number; missing: number }
  | { state: "over"; declared: number; excess: number };

export type GradeStanding = {
  declaredWeight: number;
  bankedPts: number; bankedOfficialPts: number; bankedEstimatePts: number;   // points of 100
  finalWeightPct: number | null;
  pending: { id: string; title: string; weightPct: number }[];               // unmarked, not final, weight > 0
  estimated: { id: string; title: string; weightPct: number; pct: number }[];
  coverage: WeightCoverage;
  cautions: string[];                                                        // one sentence per estimate in use
};

export type CannotReason =
  | "no_components" | "invalid_target" | "invalid_weight" | "invalid_mark"
  | "no_final" | "multiple_finals" | "final_already_marked" | "final_weight_zero"
  | "weights_short" | "weights_over" | "pending_components";

export type TargetOutcome =
  | { kind: "cannot_simulate"; reason: CannotReason; explanation: string }
  | { kind: "already_secured"; headroomPts: number; explanation: string }              // returned marks alone
  | { kind: "secured_if_estimates_hold"; headroomPts: number; officialPts: number;
      estimates: { title: string; pct: number }[]; explanation: string }
  | { kind: "reachable"; needPct: number; explanation: string }                        // 0 < needPct ≤ 100
  | { kind: "beyond_reach"; needPct: number; bestPossiblePct: number; shortfallPts: number; explanation: string };

export type GradeAssessment = { standing: GradeStanding; outcome: TargetOutcome };

export function markPercent(mark: ComponentMark): number | null;   // scored / outOf × 100, or null
export function assessTarget(input: GradeInput): GradeAssessment;
```

Form (D1): per component a title, a weight, and either nothing or `scored` + `outOf` with a `marks | %` toggle (`%` stores `{ scored: pct, outOf: 100, enteredAs: "percent" }`); on non-final rows `returned mark | my estimate` → `provenance`. The form rejects bad marks at entry; the engine defends anyway.

### 2.2 `assessTarget` — order of checks, first failure wins

1. No components → `no_components`. 2. `targetPct` not finite or outside `[0, 100]` → `invalid_target`.
3. Per component: weight not finite or `< 0` → `invalid_weight`; a marked component with non-finite values, `outOf ≤ 0`, `scored < 0` or `scored > outOf` → `invalid_mark`. The sentence names the component and quotes the value. Nothing is clamped or silently dropped.
4. Standing: `declared = Σ weight`; marked → `pts = weight × scored / outOf` into the official or the estimate bucket; unmarked, non-final, weight > 0 → `pending` (weight 0 is ignored).
5. Exactly one `isFinal`, else `no_final` / `multiple_finals`; it must be unmarked (`final_already_marked`) with weight > 0 (`final_weight_zero`).
6. Coverage: `|declared − 100| ≤ 0.05` → complete; over → `weights_over`; short or no weights → `weights_short`.
7. `pending.length > 0` → `pending_components`.
8. `target − bankedOfficialPts ≤ EPS` → `already_secured`, `headroomPts = bankedOfficialPts − target`.
9. Else `target − bankedPts ≤ EPS` → `secured_if_estimates_hold`, `headroomPts = bankedPts − target`, `officialPts = bankedOfficialPts`.
10. `needPct = (target − bankedPts) / finalWeight × 100`. `needPct > 100 + EPS` → `beyond_reach`, `bestPossiblePct = bankedPts + finalWeight`, `shortfallPts = target − bestPossiblePct`. Else `reachable`, `needPct` clamped to `≤ 100`.

Explanations (exact):

- reachable: `41 of the 70 points you want are banked. The other 29 have to come from "Final", which carries 50: that is 58% of the paper.` When estimates are in use the first sentence splits the banked figure — `34 of the 65 points you want are banked: 20 from returned marks and 14 from your estimates.` — and the explanation ends `This rests on your estimate for "IA".` (one clause per estimate).
- already_secured: `Your returned marks already add up to 45 of the 40 points you want, so this target does not depend on the final. There is nothing to triage against it: set the target you actually want. Droplist does not know about pass-the-final rules or grade boundaries. Check yours before you stop revising.` When `headroomPts < 1`, insert after the first sentence: `That is within one point: a rounding on one returned mark could move it.`
- secured_if_estimates_hold: `You reach 45 only if your estimate for "IA" (95%) holds. On returned marks alone you have 8 of the 45 points you want. This is not secured. Lower the estimate to the least you are sure of and a plan appears.`
- weights_short: `The weights add up to 90, not 100. 10 points of the grade are not recorded anywhere. Add the missing component or correct a weight. Nothing is scaled to fit.`
- pending_components: `"IA" (weight 20) has no mark yet. Enter its returned mark, or your estimate, to get a figure. Droplist will not guess it.`
- beyond_reach: `Even full marks on "Final" (50 points) would take you to 70, not 75. This target is 5 points beyond reach on this paper.`

Not done, stated in the UI help: grade boundaries (the student types the percentage their grade needs), hurdle rules, best-of-N, half marks, projection from past performance. Multi-paper finals (IB Paper 1, Paper 2, IA): one paper is `isFinal`; the others need estimates (D4).

### 2.3 Percent → marks on a paper of total T

```ts
export type PaperNeed = {
  paperTotal: number; needPct: number; marginPct: number;
  exactMarks: number;       // needPct/100 × T, unrounded, for the trace
  marksNeeded: number;      // max(0, ceil(exactMarks − EPS))
  planTarget: number;       // min(T, ceil((needPct + marginPct)/100 × T − EPS))
  marginMarks: number;      // planTarget − marksNeeded
  marginClipped: boolean;   // the unclipped target exceeded T
};
/** Throws RangeError unless 0 ≤ needPct ≤ 100, paperTotal > 0 and finite, marginPct ≥ 0. Callers validate first. */
export function marksOnPaper(needPct: number, paperTotal: number, marginPct: number): PaperNeed;

export type PlanNeed =
  | { kind: "blocked"; reason: CannotReason; explanation: string }                     // planner does not run
  | { kind: "secured"; basis: "official" | "estimates"; explanation: string }           // planner does not run
  | { kind: "needs"; needPct: number; marginPct: number; onFinal: PaperNeed | null; explanation: string }
  | { kind: "beyond"; needPct: number; bestPossiblePct: number; shortfallPts: number; explanation: string };
export function planNeed(outcome: TargetOutcome, finalPaperTotal: number | null, marginPct: number): PlanNeed;
```

- Whole marks only; half-mark papers accept the small strictness (D-D).
- **Subsystem 1 never compares a backtest against `onFinal.planTarget`**: for each paper on file it calls `marksOnPaper(needPct, T_paper, marginPct).planTarget`.
- `finalPaperTotal` is an input, prefilled from the most recent counted paper's printed maximum with its citation, `confirmed: false` until accepted. Before acceptance the headline is `You need 58% of the final. The 2024 paper was out of 100 (p.1). If this one is too, that is 58 marks. Confirm the total.` With nothing to prefill: `You need 58% of the final. Say what the paper is out of to see that in marks.`
- `beyond`: subsystem 1 maximises the hardest-paper backtest and reports the shortfall; `needPct` is carried so targets are computed at 100%.

### 2.4 Worked examples (each is a test)

**A — the demo shape (D-F).** Coursework 17/20 weight 20 → 17 pts. Midterm 24/30 weight 30 → 24. Final weight 50. Target 70. `banked = 41`, `needPct = 58`. `T = 100`: 58 needed, plan target `ceil(63) = 63`, buffer 5. `T = 80`: `46.4 → 47`; `ceil(50.4) = 51`; buffer 4. `T = 50`: 29; `ceil(31.5) = 32`; buffer 3.

**B.** 30/40 weight 20 → 15; 25/40 weight 30 → 18.75; final 50; target 70. `banked = 33.75`, `needPct = 72.5`. `T = 80` → 58 needed, target 62. (At the 80% yield this leaves 2.5 points of room: seed the demo from A, not B.)

**C — the float guard.** Midterm 27/60 weight 30 → 13.5. Final 70. Target 72.5. `T = 70`: the exact need is 59 marks; naive `Math.ceil(needPct/100 × 70)` returns **60**; `marksOnPaper` returns **59**. Target `ceil(62.5) = 63`.

**D — pending, then an estimate.** Quiz 8/10 weight 10 → 8. Midterm 30/50 weight 20 → 12. IA weight 20 unmarked. Final 50. Target 65 → `pending_components`. IA estimate 70/100 → 14 pts; `banked = 34`; `needPct = 62`; `T = 80` → 50 needed, target `ceil(53.6) = 54`. Explanation ends `This rests on your estimate for "IA".`

**E — secured on an estimate.** Quiz 8/10 weight 10 → 8 (official). IA weight 40, estimate 95/100 → 38. Final 50. Target 45. `bankedOfficial = 8`, `banked = 46` → `secured_if_estimates_hold`, headroom 1. Estimate lowered to 90 → banked 44, `needPct = 2`, reachable.

**F — boundaries.** Official 45, target 40 → `already_secured`, headroom 5. Official 40, target 40 → `already_secured`, headroom 0, "within one point" clause present. Official 20 over weight 50, final 50, target 75 → `beyond_reach`: need 110%, best possible 70, shortfall 5. Official 25, target 75, final 50 → reachable at exactly 100%; on `T = 50` with buffer 5% the target is 50, `marginMarks = 0`, `marginClipped = true`.

**G — weights short.** 10 + 30 + 50 = 90 → `weights_short` with the sentence above.

**Headline (demo step 2).** `You need 58 of 100 on the final.` Sub-line: `58% of the paper · plans aim for 63 (5% buffer — a default, no data behind it)`; once `margin_pct` is set: `(5% buffer, set by you)`. `assessTarget` is pure; the form imports it client-side and the figure updates as the student types.

## 3. Standing and the Probe

### 3.1 Types

```ts
export const THIN_BELOW = 3;            // brief non-negotiable 4; also a locked registry row

export type ProbeGrade = "yes" | "partly" | "no";
export type ProbeResponse = { leafId: string; grade: ProbeGrade; answeredAt: string /* ISO-8601 UTC */ };

/** Supplied by subsystem 4 from the choice trees. One per Leaf with a topic. */
export type ProbeCandidate = {
  leafId: string; paperId: string; paperLabel: string;   // "2024"
  paperOrder: number;          // larger = more recent
  page: number; contextPages: number[];                  // enclosing question's first page … page, max 4
  marks: number; excerpt: string;
  topicId: string | null;      // null and none_of_these leaves are never candidates
  counted: boolean;            // the paper counts for planning (D-E: every row confirmed, not excluded, no fatal flag)
};
export type Citation = { paperId: string; paperLabel: string; page: number };

export type ProbeEvidence = {
  n: number; yes: number; partly: number; no: number;
  marksShown: number; marksYes: number; marksPartly: number; marksNo: number;
  lastAnsweredAt: string; citations: Citation[];          // one per answered leaf
};

export type TopicStanding =
  | { kind: "no_questions_on_file"; papersCounted: number; onPapersNotCounted: number }
  | { kind: "not_probed"; questionsOnFile: number }
  | { kind: "probed"; questionsOnFile: number; thin: boolean;
      thinBecause: "few_answers" | "few_questions" | null; evidence: ProbeEvidence };
```

No number is constructible for an unprobed topic: (1) the variant has no numeric field but counts; (2) a fraction exists only as the return of `answerable(evidence, c)`, which needs a `ProbeEvidence`; (3) the zod schema is `.strict()` on every arm, so `{ kind: "not_probed", questionsOnFile: 6, answerable: 0.5 }` fails to parse (tested).

`contextPages` exists because real papers put a 2-mark part under a case study that starts on the previous page. MUST: show the leaf's page image. SHOULD: the context strip.

### 3.2 `buildStanding(topicId, candidates, responses, papersCounted): TopicStanding`

1. `mine` = candidates with this `topicId`; `onFile` = `mine` with `counted`.
2. `onFile` empty → `no_questions_on_file { papersCounted, onPapersNotCounted: mine.length }`.
3. Latest response per `leafId`: greatest `answeredAt` (string compare); tie → later array element. Responses belong to a **question**: a re-mapped leaf takes its answer with it; a leaf on a paper that stops counting stops counting. The table is append-only.
4. No answered leaf in `onFile` → `not_probed { questionsOnFile }`.
5. Else `probed`: tally counts and marks; `thin = n < 3`; `thinBecause = "few_questions"` when `questionsOnFile < 3` (only another paper can fix it), else `"few_answers"`. Citations sorted by `paperLabel`, then page.

```ts
/** Share of the marks shown that the student says they could answer. */
export function answerable(ev: ProbeEvidence, partlyCredit: number): number;
// marksShown > 0 ? (marksYes + c·marksPartly) / marksShown : (yes + c·partly) / n
```

Why marks-weighted: the backtest multiplies `p` by leaf marks, so `p` must mean "share of this topic's marks"; a student who can do the 2-mark "state" parts but not the 10-mark "evaluate" part is at 2/12, not 1/2. In simulation (§8) count-weighting ran optimistic in every cell; marks-weighting leaned neither way consistently. Why 0.5: a convention, not a finding. Why × `p_target`: an unrevised "could answer" topic can no longer out-rank a revised one, "yes to everything ⇒ 100% in reach" becomes impossible, and it is the only haircut on self-report in the product (§8, spike 2c).

### 3.3 `planningP` — the only source of per-topic numbers for the planner

```ts
export type SelfRating = "solid" | "shaky" | "not_started";                   // SHOULD
export type PBasis = "probe" | "probe_thin" | "self_rating" | "assumption";
export type StandingSettings = Pick<EngineSettings, "yield" | "untestedAnswerable" | "partlyCredit" | "thinBelow" | "ratingAnswerable">;
export type PlanningP = {
  topicId: string;
  answerablePlan: number;   // 0..1
  pNow: number;             // answerablePlan × yield
  pAfter: number;           // yield
  basis: PBasis;
  assumedShare: number;     // 0 probe · (3 − n)/3 probe_thin · 1 otherwise
  probeN: number;           // answers behind the figure; 0 unless probed
  sentence: string;         // exact, §3.5
  short: string;            // exact, §3.5 — the row text
  restsOn: SettingKey[];    // every setting this number used
};
export function planningP(topicId: string, standing: TopicStanding, rating: SelfRating | null, s: StandingSettings): PlanningP;
```

```
a0 = rating ? s.ratingAnswerable[rating] : s.untestedAnswerable
probed, n ≥ 3 :  a = answerable(ev, c)                        basis "probe",      assumedShare 0
probed, n < 3 :  a = (n·answerable(ev, c) + (3 − n)·a0) / 3   basis "probe_thin", assumedShare (3 − n)/3
otherwise     :  a = a0                                       basis rating ? "self_rating" : "assumption", assumedShare 1
pNow = a × yield ; pAfter = yield
restsOn = [p_target_pct] + (ev.partly > 0 ? [probe_partly_credit_pct] : []) + (a0 used ? [rating key or untested_answerable_pct] : [])
```

Worked, `yield 0.8`, `c 0.5`, `a0 0.5`:

| Case | Answers (marks) | `answerable` | `answerablePlan` | `pNow` | basis · assumedShare |
|---|---|---|---|---|---|
| Probed | yes[6] yes[2] partly[4] no[10] | 10/22 = 0.4545 (count-based would be 0.625) | 0.4545 | 0.3636 | probe · 0 |
| Thin | yes[4] yes[6] | 1.0 | (2·1 + 1·0.5)/3 = 0.8333 | 0.6667 | probe_thin · 1/3 |
| Thin, 1 on file | no[10] | 0 | (0 + 2·0.5)/3 = 0.3333 | 0.2667 | probe_thin · 2/3 |
| Not probed | — | undefined | 0.5 | 0.4 | assumption · 1 |
| Not probed, rated solid | — | undefined | 0.75 | 0.6 | self_rating · 1 |

`pNow ≤ pAfter` always: studying never lowers `p`.

### 3.4 `selectProbeItems` — which real questions to show

```ts
export type ProbeItem = Pick<ProbeCandidate, "leafId" | "paperId" | "paperLabel" | "page" | "contextPages" | "marks" | "excerpt">;
export type ProbeSelection =
  | { kind: "none_on_file"; onPapersNotCounted: number }
  | { kind: "items"; items: ProbeItem[]; questionsOnFile: number; alreadyAnswered: number; canClearThin: boolean /* questionsOnFile ≥ 3 */ };
export function selectProbeItems(topicId: string, candidates: ProbeCandidate[], responses: ProbeResponse[], k: number, includeAnswered?: boolean): ProbeSelection;
```

Deterministic, no randomness. 1. `onFile` as in §3.2; empty → `none_on_file`. 2. `pool` = `onFile` minus answered leaves, unless `includeAnswered` (re-probe; new answers supersede). 3. Group by paper; papers by `paperOrder` desc then `paperId`; within a paper by `marks` desc, `page` asc, `leafId`. 4. Round-robin: round 0 takes each paper's **first** remaining leaf (longest), round 1 each paper's **last** (shortest), alternating, until `k` items or every paper is empty. 5. Fewer than `k` → all of them; fewer than 3 on file → the probe still runs with `canClearThin = false` and the UI says first: `Only 2 questions on this topic are on file, so the result will be thin however you answer.`

Spreading across papers makes three answers speak for three papers; longest-then-shortest spans mark sizes, which marks-weighting needs. `k = probe_questions_per_topic` (4: the thin bar plus a spare).

Example (6 counted leaves): 2024 `[10] p.9`, `[6] p.3`, `[2] p.2`; 2023 `[4] p.4`, `[2] p.5`; 2022 `[6] p.6`. `k = 4` → `2024[10], 2023[4], 2022[6], 2024[2]`. After answering the first two: `2024[6], 2023[2], 2022[6], 2024[2]`.

Probe copy (exact; layout is subsystem 5's), exported from `probe.ts`:

- Prompt: **`Closed book, right now, no notes: how much of this could you get down? Judge the topic, not your memory of this question.`**
- Buttons (`yes` / `partly` / `no` / skip): `Nearly all of it — at least three-quarters of the marks` · `About half` · `Little or none` · `Skip — can't judge from this page`. A skip stores nothing.
- Each item shows paper label, page, marks as `[6]`, the excerpt and the page image. The Probe shows only pages of papers on file; it never shows model-written text. Action link on an unprobed topic: `Probe it — {min(k, questionsOnFile)} past questions.`

### 3.5 Sentences and the three inks

Exact strings from `planningP` (numbers from the §3.3 table; percents are the settings as whole numbers; marks to one decimal):

- **probe:** `You said you could answer 2 of the 4 past questions shown (8 marks) and partly answer 1 (4 marks), out of 22 marks. Stated by you against real questions, not measured. The plan counts "about half" at 50% and anything you could answer at 80%, so it uses 8.0 of the 22 marks.` When `partly = 0` the "and partly …" clause is omitted and the last sentence reads `The plan counts anything you could answer at 80%, so it uses …`.
- **probe_thin:** `You said you could answer 2 of the 2 past questions shown (10 marks), out of 10 marks. That is thin: fewer than 3 answers, so 1 of the 3 parts of this figure is the 50% assumption for an unprobed topic. Counting anything you could answer at 80%, the plan uses 6.7 of the 10 marks.` With a rating the middle clause ends `… is your own rating of "solid" (75%).`
- **assumption (not probed):** `Not probed: you have not judged any past question on this topic. The plan still needs a figure, so it counts this topic at 40% of its marks — the 50% assumption for an unprobed topic × 80% for a revised topic. Both are assumptions, not results.`
- **self_rating:** `Not probed: you have not judged any past question on this topic. The plan uses your own rating of "solid", counted as 75% × 80% = 60% of its marks. That is an assumption, not evidence.`
- **no_questions_on_file (D-N, verbatim):** `Never appeared in the 3 papers on file. That is silence, not safety.` The topic is listed under its own heading `No evidence either way`, is never struck through, and its `restsOn` is empty because the planner does not use its number. When `onPapersNotCounted > 0` the UI adds: `3 questions are mapped to it on papers that do not count yet. Confirm those papers to probe it.`

`short` (the row text): probe `you said: 8 of 22 marks (+ about half of 4) · 4 questions`; probe_thin `you said: 10 of 10 marks · 2 questions · thin: 1 of 3 parts assumed`; assumption `Not probed`; self_rating `you said: "solid" — an assumption`; none `No past question on file`.

Display rules (D-P), owned here, rendered by subsystem 5:

- **Three inks.** Page-cited paper facts in ink. Self-report in the student's colour, always prefixed `you said:`. Assumptions in pencil with a dashed underline. `probe_thin` is pencil with the chip `thin · 1 of 3 parts assumed`; only its `you said:` part is in the student's colour.
- **No percentage in a topic row.** Evidence is a fraction of marks with its count. An assumption's number appears only inside the opened sentence, the plan trace and `/settings`.
- **Words.** `probed` / `not probed`, never `tested` / `measured` (except inside `not measured`). A topic whose number rests on `assumption`, `self_rating` or `probe_thin` carries the tag `on an assumption` wherever it is listed as a drop, not only in the footer.

### 3.6 SHOULD / COULD

- SHOULD — self-rating: `topics.self_rating TEXT NULL`; one row per topic, three words, rendered as a word in pencil labelled `assumption, not evidence`; settings `self_rating_*_pct` (§6). Help: `One tap per topic. This is an assumption, so a rough answer is fine — a probe replaces it.`
- SHOULD — `ProbeCandidate.paperWorked: boolean` (`I have already worked through this paper`, per paper, subsystem 4); the probe sentence appends `2 of these 4 questions are from papers you have already worked through, so this may run high.`
- SHOULD — answer age on the row: `answered 6 days ago — re-probe if you have revised it since.`
- SHOULD — `ProbeCandidate.hiddenPages: number[]` (pages the student marked as answer space or mark scheme); selection drops a candidate whose page or `contextPages` hit one; copy `Hidden: this page is marked as a mark scheme.`
- COULD — `Mark as revised`: `pNow = pAfter = yield`, basis `"marked_revised"`; the engine lists it as `already at your stated standing` (D-L). De-duplicate identical excerpts across papers.

## 4. Hours

```ts
export type HoursInput =
  | { mode: "total"; totalHours: number | null }
  | { mode: "per_day"; days: number | null; hoursPerDay: number | null };
export type HoursAvailable =
  | { kind: "not_stated"; sentence: string } | { kind: "invalid"; sentence: string }
  | { kind: "stated"; hours: number; sentence: string };
export function hoursAvailable(input: HoursInput): HoursAvailable;
/** Whole days from `today` up to but not including `examDate` ('YYYY-MM-DD'); today counts; never negative; null if unparseable. */
export function studyDaysBetween(today: string, examDate: string): number | null;

export type TopicHours = { topicId: string; hours: number; basis: "own_figure" | "course_figure"; sentence: string; restsOn: SettingKey[] };
export function topicHours(topicId: string, ownHours: number | null, course: { hours: number; confirmed: boolean }): TopicHours;
```

- Bounds: `totalHours` 0–400; `days` integer 0–60; `hoursPerDay` 0–16; outside → `invalid` quoting the value. `days = 0` is valid → 0 h.
- `studyDaysBetween` only prefills the days field (`today` injected by the app layer); the student can overwrite it.
- `not_stated` is not an error: `You have not said how many hours you have. The plan shows the hours it needs, and cannot say whether they fit.` Subsystem 1 runs unconstrained.
- `ownHours` null, non-finite or `≤ 0` → the course figure (`topics.own_hours REAL NULL`).
- Sentences: `22.5 h: 9 days × 2.5 h a day, as you stated.` · own `5 h: your own figure for this topic.` · course, confirmed `3 h: the one course-wide figure you set. There is no data behind it.` · course, unconfirmed `3 h: Droplist's placeholder for every topic. You have not set it and there is no data behind it.` `confirmed = !settings.unconfirmed.has("hours_per_topic_cold")`. No sentence in this subsystem says "you" about a value the student has not typed or set.
- Hours page header, verbatim: **`Droplist has no data on how long revision takes you. These hours are yours. One figure covers every topic unless you give a topic its own.`**

No scaling by standing (D12): `hours = base × (1 − answerable)` is an invented multiplier, and unnecessary — with flat hours the planner already prefers cold topics. COULD: `hours_standing_relief_pct` (0 = off), `hours_realism_pct` (100).

## 5. Hand-off to subsystem 1 — `src/engine/inputs.ts`

```ts
export type PlannerTopicInput = {
  topicId: string; name: string;
  pNow: number; pAfter: number; hours: number;
  pBasis: PBasis; probeN: number; thin: boolean; assumedShare: number;
  hoursBasis: TopicHours["basis"];
  noLeaves: boolean;                 // standing.kind === "no_questions_on_file": never a drop, never safe (D-N)
  restsOn: SettingKey[];
  sentence: string; short: string; hoursSentence: string;
};
export type YieldCeiling =
  | { kind: "ok" }
  | { kind: "margin_does_not_fit"; paperLabel: string; sentence: string }
  | { kind: "target_above_yield"; paperLabel: string; sentence: string };
/** Per paper, in whole marks: planTarget > yield × T + EPS → the buffer does not fit; marksNeeded > yield × T + EPS → the target itself is above the ceiling. Reports the paper with the largest excess (ties → first). */
export function yieldCeiling(needPct: number, marginPct: number, yieldFraction: number, papers: { label: string; total: number }[]): YieldCeiling;

export type PlannerInputs = {
  need: PlanNeed;
  ceiling: YieldCeiling | null;              // null unless need.kind === "needs"
  topics: PlannerTopicInput[];
  hours: HoursAvailable;
  pNoTopic: { value: number; restsOn: SettingKey[] };   // noTopicAnswerable × yield = 0.32 at defaults; keys [no_topic_answerable_pct, p_target_pct]
  gambleBudgetMarks: number;                 // the engine's G; 0 at default (D-B)
  restsOn: SettingKey[];                     // union, registry order; always contains p_target_pct
  unconfirmed: SettingKey[];                 // restsOn ∩ settings.unconfirmed
};
export function buildPlannerInputs(args: {
  assessment: GradeAssessment; finalPaperTotal: number | null;
  papers: { label: string; total: number }[];                       // counted papers, for the ceiling
  topics: { topicId: string; name: string; standing: TopicStanding; rating: SelfRating | null; ownHours: number | null }[];
  hours: HoursInput; settings: EngineSettings;
}): PlannerInputs;
export function assumptionFooter(notes: AssumptionNote[]): string;
```

`yieldCeiling` sentences (need 86% / 76%, buffer 5, a 50-mark paper; the ceiling is `0.8 × 50 = 40`):

- target_above_yield: `On the 2022 paper your target needs 43 of 50 marks, but the plan counts a revised topic at 80% of its marks, so no plan reaches more than 40. Raising the 80% makes the plan look reachable without making you better prepared: raise it only if marked work of yours has come in above 80% on topics you had revised. Otherwise the honest reading is that this target is out of reach on this paper.`
- margin_does_not_fit: `On the 2022 paper you need 38 of 50 marks and the plan aims for 41 with the 5% buffer, but a revised topic is counted at 80% of its marks, so no plan reaches more than 40. The buffer does not fit on that paper: any plan will show as short there until the buffer is lowered, unless the 80% is too low for you.` (With need 75% the single-ceil target is 40 and fits; the old double ceil said 41.)

The planner still runs (maximise mode); this is advisory copy, the `yield_ceiling` caveat.

**Plan footer** (`assumptionFooter`, MUST, never absent while `restsOn` is non-empty): `This plan rests on 4 figures with no data behind them. Set by you: Buffer above the marks needed (5%), Hours to revise one topic from cold (3 h). Still Droplist's defaults: Marks from a revised topic (80%), Assumption for an unprobed topic (50%).` Either clause is omitted when empty. Matches the seed in §6.3.

SHOULD — what-if line: the app re-scores the chosen study set with the engine's `evaluateStudySet` at `p_target − what_if_step_pct`: `Counted at 70% instead of 80%, this plan's hardest paper on file (2022) comes out 5 marks under the 63 it aims for.` In simulation every feasible plan fell short at −10 points, by 4.8 marks on average (§8); one evaluation, no search.

## 6. `src/config/settings.ts`

### 6.1 Types

```ts
export type SettingGroup = "plan" | "standing" | "hours" | "fixed" | "models";
type Base = { key: string; label: string; help: string; group: SettingGroup; unconfirmedUntilSet: boolean;
  locked?: boolean;      // fixed by design: read-only, stored row ignored
  order: number };
export type NumericSettingDef = Base & { type: "percent" | "number" | "integer"; default: number; min: number; max: number; step: number; unit: string };
export type EnumSettingDef   = Base & { type: "enum"; default: string; options: readonly { value: string; label: string }[] };   // SHOULD
export type StringSettingDef = Base & { type: "string"; default: string; maxLength: number; pattern?: string };            // SHOULD
export type SettingDef = NumericSettingDef | EnumSettingDef | StringSettingDef;

export const SETTINGS = [ /* §6.2 */ ] as const satisfies readonly SettingDef[];
export type SettingKey = (typeof SETTINGS)[number]["key"];       // unknown key = compile error
export type SettingValue<K extends SettingKey> = Extract<(typeof SETTINGS)[number], { key: K }>["type"] extends "enum" | "string" ? string : number;
export type SettingValues = { [K in SettingKey]: SettingValue<K> };

export type InvalidSetting = { key: SettingKey; raw: string; reason: string };
export type ResolvedSettings = { values: SettingValues;
  unconfirmed: SettingKey[];      // default in force AND def.unconfirmedUntilSet
  invalid: InvalidSetting[] };    // stored value refused; default used; counts as unconfirmed

export function validateSetting(def: SettingDef, raw: string): string | null;   // reason or null; never clamps
export function resolveSettings(stored: Record<string, string>): ResolvedSettings;
export function defOf(key: SettingKey): SettingDef;
export function fmtSetting(def: SettingDef, value: string | number): string;    // "80%", "3 h", "0 marks"

export type EngineSettings = {
  yield: number; marginPct: number; untestedAnswerable: number; noTopicAnswerable: number; partlyCredit: number;
  probeK: number; hoursPerTopicCold: number; thinBelow: number; gambleBudgetMarks: number;
  ratingAnswerable: Record<SelfRating, number>;      // the defaults when the SHOULD rows are not built
  unconfirmed: ReadonlySet<SettingKey>;
};
/** The one place whole-number percents become fractions. marginPct stays in points. */
export function engineSettingsFrom(r: ResolvedSettings): EngineSettings;

export type AssumptionNote = { key: SettingKey; label: string; display: string; unconfirmed: boolean };
export function assumptionNotes(r: ResolvedSettings, keys: Iterable<SettingKey>): AssumptionNote[];   // unconfirmed first, then registry order
```

`resolveSettings`, per def: locked → default, stored row ignored. No row → default, `unconfirmed` if the def says so. Row → `validateSetting`; a reason → `invalid`, default used, unconfirmed; else the parsed value. Numeric validation: non-empty, finite, integer when `type: "integer"`, within `[min, max]`. Percents are stored as whole numbers.

**Confirmed = a row exists.** `Keep 80%` writes `"80"` and clears the flag; `Reset` deletes the row. So `/settings` is **one form per row** (`Save`, `Keep <default>`, `Reset`); one shared form would confirm every default in a click. Server actions (subsystem 4): `saveSetting(key, raw)` → `validateSetting`, a reason is returned as a form error and nothing is written; `resetSetting(key)`. Table `settings(key TEXT PRIMARY KEY, value TEXT NOT NULL, set_at TEXT NOT NULL)`; one course, so settings are global. Rendering: numeric → number input with min/max/step/unit; enum → select; string → text; locked → read-only `fixed by design`; unconfirmed rows carry `unconfirmed` and the help suffix `This is a default standing in for a figure only you know.`; invalid rows show `Stored value "140" was refused: must be between 40 and 100. Using 80.`

### 6.2 Every setting in the product

Owner = the subsystem whose code reads it. † = proposed default for that owner to confirm.

| key | group·order | type | default | min–max (step) unit | unconfirmed until set | tag | owner | label / help |
|---|---|---|---|---|---|---|---|---|
| `p_target_pct` | plan·1 | percent | 80 | 40–100 (5) | yes | MUST | 2→1 | **Marks from a revised topic.** Share of a question's marks the plan counts for a topic you have revised — and for anything you say you could already answer, which counts at this rate, not 100%. It does two jobs: what revision yields, and a haircut on your own judgement. No data behind the default. |
| `margin_pct` | plan·2 | percent | 5 | 0–20 (1) | yes | MUST | 2→1 | **Buffer above the marks needed.** Plans aim this far above the marks you need, as a share of the paper. A cushion, not a guarantee: no data shows 5% is enough, and a plan that clears it on past papers can still fall short on the real one. |
| `gamble_budget_marks` | plan·3 | integer | 0 | 0–50 (1) marks | yes | MUST | 2→1 | **Gamble budget.** The most marks a set of drops may have cost on any paper on file and still sit on the Drop List. At 0 the list holds only drops that cost nothing on every paper on file; everything else is listed apart, as a gamble, with its cost. |
| `untested_answerable_pct` | standing·1 | percent | 50 | 0–100 (5) | yes | MUST | 2 | **Assumption for an unprobed topic.** Share of a topic's marks the plan assumes you could answer when you have not probed it. Never shown as a result; probing replaces it. It rarely decides whether a plan exists, but changed which topics were dropped in half to three-quarters of simulated courses. |
| `no_topic_answerable_pct` | standing·2 | percent | 40 | 0–100 (5) | yes | MUST | 2→1 | **Assumption for a question on no listed topic.** Share of the marks of a "none of these" question the plan assumes you could answer. No topic's standing applies to it. |
| `probe_partly_credit_pct` | standing·3 | percent | 50 | 0–100 (5) | yes | MUST | 2 | **Credit for "about half".** How much of a question's marks an "about half" counts for. "Nearly all" (three-quarters or more) counts in full before the revised-topic rate; "little or none" counts nothing. 50 is a convention, not a finding. |
| `probe_questions_per_topic` | standing·4 | integer | 4 | 3–8 (1) | no | MUST | 2 | **Questions per probe.** Three is the least that is not thin; the fourth is a spare. |
| `self_rating_solid_pct` | standing·5 | percent | 75 | 0–100 (5) | yes | SHOULD | 2 | **"Solid" counts as.** Used only where a topic has no probe result. Not 100%: Droplist assumes, without data, that a rating made with no question in front of you runs high. |
| `self_rating_shaky_pct` | standing·6 | percent | 40 | 0–100 (5) | yes | SHOULD | 2 | **"Shaky" counts as.** |
| `self_rating_not_started_pct` | standing·7 | percent | 10 | 0–100 (5) | yes | SHOULD | 2 | **"Not started" counts as.** |
| `hours_per_topic_cold` | hours·1 | number | 3 | 0.5–20 (0.5) h | yes | MUST | 2 | **Hours to revise one topic from cold.** One figure for the course. No data source: you state it. Give a topic its own hours if it is bigger or smaller. |
| `thin_below` | fixed·1 | integer | 3 | locked | no | MUST | 2 | **Thin evidence.** Fewer answers than this on a topic is thin. A thin probe is padded to 3 answers with the assumption for an unprobed topic. |
| `min_papers_for_recurrence` | fixed·2 | integer | 3 | locked | no | MUST | 1 | **Papers needed to rank recurrence.** Below this there is no ranking and the backtest is flagged thin. |
| `exact_optimiser_max_topics` | fixed·3 | integer | 16 | locked | no | MUST | 1 | **Exact search up to.** Above this many topics the plan is approximate and says so. |
| `weight_tolerance` | fixed·4 | number | 0.05 | locked | no | MUST | 2 | **Weights count as 100 within.** |

SHOULD rows, none unconfirmed: `what_if_step_pct` (fixed·5, locked 10, owner 2: **What-if step**); `llm_provider` (models·1, enum `ollama | mock`); `ollama_base_url` (models·2, string, default `http://127.0.0.1:11434`, pattern `^http://(127\.0\.0\.1|\[::1\])(:\d{2,5})?$` — `localhost` is refused because it can be remapped; help: **Ollama address.** Loopback only. Droplist sends pages to whatever listens there; it cannot check that the listener is local.); `vision_model` / `text_model` (models·3/4, string ≤ 80 chars, defaults `qwen3-vl:8b-instruct` / `qwen2.5:7b-instruct`); `llm_timeout_s` † (models·5, 180, 10–900 s); `text_layer_min_chars` † (models·6, 200, 0–5000: below this a page goes to the vision model). COULD: `vision_render_dpi` † (150, 72–300); `hours_standing_relief_pct` (0) and `hours_realism_pct` (100), both unconfirmed until set. Model rows are owned by subsystem 3.

The registry is the inventory of every constant that changes a number the student sees; numeric guards (`EPS`) and input bounds live in §2 and §4. A new constant that changes an output is a new row. No `EvidencePolicy` rows: under D-E a counted paper has every row confirmed, so the engine's policy is constant (`DEFAULT_POLICY`).

**Not settings (inputs):** target %, components and marks, final paper total, exam date, hours mode / total / days / hours per day, own hours, self-ratings, probe answers.

### 6.3 Seeding (`npm run seed:demo`, subsystem 4)

Write rows for `margin_pct` (`"5"`) and `hours_per_topic_cold` (`"3"`); leave `p_target_pct`, `untested_answerable_pct`, `no_topic_answerable_pct` and `gamble_budget_marks` unwritten, so the demo shows both a set figure and the `unconfirmed` flag; leave one or two topics unprobed so the assumption sentence is on screen.

## 7. Edge cases (each is a test)

Grade: no components · target −1, 101, NaN · weight −5, NaN · `21/20`, `5/0`, `−1/10` · no final · two finals · final already marked · final weight 0 · weights 90 · 130 · 33.3 + 33.3 + 33.4 (complete) · pending weight-0 component does not block · only the final exists (need = target) · official equals target (headroom 0, "within one point") · secured on official with an estimate present (`already_secured`, not the estimates kind) · secured only with the estimate (E) · need exactly 100% (margin clipped) · example C · `marksOnPaper` throws on `T = 0`, `T = NaN`, need 101, margin −1 · `needPct = 0` → 0 needed, target = the buffer alone · `planNeed` with `T = null` for every outcome · ceiling cases (need 75 / 76 / 86, buffer 5, T 50).

Standing: no candidates · candidates only on uncounted papers (their answers do not count; `onPapersNotCounted`) · topic `null` · re-answered (latest wins; same timestamp → later element) · leaf re-mapped (answer moves) · 1 and 2 on file (`few_questions`) · 3 on file, 2 answered (`few_answers`) · all leaves 0 marks (count fallback) · `partly = 0` (no credit key in `restsOn`, no "partly" clause) · rating + non-thin probe (probe wins; rating key absent) · rating + thin probe (rating is the fill; sentence names it) · `assumedShare` ∈ {0, 1/3, 2/3, 1} · `pNow ≤ pAfter` everywhere · strict schema rejects a number on `not_probed` · `no_questions_on_file` sentence verbatim, `restsOn = []`.

Selection: input order irrelevant · `k` > pool · all answered → empty, `alreadyAnswered = questionsOnFile` · `includeAnswered` · single paper alternates longest/shortest · equal marks → page, then id.

Hours: null → `not_stated` · 2.5 days → `invalid` · −1 h · 0 days → 0 h · exam date before today → 0 · unparseable → null · own hours 0 → course figure · course figure confirmed vs unconfirmed sentences.

Settings: keys unique · every default passes its own validation · every def has label and help · fresh store → exactly the `unconfirmedUntilSet` keys, in registry order · storing the default confirms it · `"140"`, `"abc"`, `""` refused, default used, reported, unconfirmed · locked row ignores a stored value · `localhost` and non-loopback URLs refused, `[::1]` accepted · `engineSettingsFrom` converts percents once; `pNoTopic = 0.32` at defaults · `assumptionNotes` orders unconfirmed first · `assumptionFooter` with both clauses, one clause, and the seed's rows · `// @ts-expect-error` on an unknown `SettingKey`.

## 8. Evidence — simulations of stated synthetic models, no student data

All under `$SCRATCH/spikes/inputs/`, fixed LCG seeds. Read the direction, not the magnitude: every "student" is a formula.

| # | File | What ran | Result |
|---|---|---|---|
| 1 | `spike1-ceil.mjs`, `spike1b-single-ceil.mjs` | 264,959 / 264,790 random reachable cases from realistic typed inputs vs BigInt rationals | Naive `Math.ceil` for marks needed wrong in 1,143 (0.43%), always one too high where the exact answer is whole. `ceil(x − 1e-9)`: 0 wrong for the need, 0 wrong for the single-ceil target (naive: 833). The old double ceil exceeded the exact target in **28.4%** of cases, always by 1. Smallest genuine fraction above an integer 9.4e-5, four orders above the epsilon. → D5. |
| 2 | `spike2-weighting.mjs`, `spike2c-d8-estimator.mjs` | Synthetic student whose true share falls with question length; 40,000 trials per cell; n = 3–4; held-out target | Per-topic MAE **0.12–0.20 in every cell** with perfect self-grading (to 0.24 with over-rating): a 4-question probe is a rough sort. Count-weighting biased +0.04 to +0.09 everywhere; marks-weighting −0.03 to +0.04. The shipped `× 0.8` estimator: **−0.07 to −0.10** with perfect self-grading, **+0.01 to +0.04** with a student who over-rates by 0.15. Over 10 topics that student's marks in reach exceeded the truth by more than the 5% buffer in **36.6%** of runs (93.8% undiscounted); with perfect self-grading 0.3% (13.3%). → D6, D7: the discount is a haircut on self-report, not a calibration; the buffer is a cushion, not a guarantee. |
| 3 | `spike3-sensitivity.mjs`, `spike3b-setsize.mjs` | 300 scenarios, 10 topics, 3 synthetic choice papers, exact enumeration, one setting at a time | Feasibility flipped / **study set changed**: partly 0.4–0.6 → 1–2% / 24–27% (0.3–0.7 → 2% / 45–50%); untested 0.25–0.75 → 2–4% / 47–76%; `p_target` 0.7–0.9 → 25–35% / 72–88%; buffer 0–10% → 16–20% / 66–75%. About two topics switch side when a set changes. What-if at `p_target − 10` on the same set: **0 of 199** feasible plans still cleared every paper; mean shortfall 4.8 marks (max 7.2); 0.009 ms each. → D8, D11, D13, §5. |
| 4 | `src/` (reference implementation) | `tsc --strict` clean, 10/10 tests | Predates this revision: its sentences, double ceil and `tested` vocabulary are superseded. The spec is complete without it. |

## 9. Test list

`grade.test.ts` — worked A–G; every `CannotReason` once; tolerance; weight-0 pending; estimate labelling and split; both secured kinds; `marksOnPaper` range errors; `planNeed` for every outcome and `T = null`; 1,000 seeded cases asserting `marksNeeded − 1 < exactMarks + 1e-9 ≤ marksNeeded + 1e-9` and the same for the unclipped `planTarget` against `(needPct + marginPct)/100 × T`.
`standing.test.ts` — §7 standing list; the §3.3 table to 1e-9; every §3.5 sentence and `short` verbatim; **vocabulary sweep** over every output string: none contains `tested`, `measured` (except inside `not measured`), `safety margin`, `worst case`, `every topic can be dropped`, `weak`, `behind`, `failing`, `will appear`, `predict`, `guarantee`, `likely`, `probably`, `should come up`, `won't come up`, `expected to`, `\bscore\b`, or `\bsafe\b` outside a string containing `papers on file`.
`probe.test.ts` — §7 selection list; the §3.4 example; determinism under shuffle. `hours.test.ts` — §7 hours list.
`inputs.test.ts` — `yieldCeiling` three kinds with the T = 50 cases; `buildPlannerInputs` unions `restsOn`, computes `unconfirmed`, passes `blocked` through with no topic dropped, sets `noLeaves` and `pNoTopic = 0.32`; `assumptionFooter` cases; the vocabulary sweep again. `settings.test.ts` — §7 settings list.
Playwright (subsystem 4's harness), MUST: type example A into the marks form → `You need 58 of 100 on the final`; open an unprobed topic → its row contains no `%`, the opened sentence contains `40%` and `assumption`. SHOULD: `/settings` → `Keep 80%` removes `unconfirmed` on that row only.

## 10. Build order (about 1.5 agent-days for the MUST set)

1. `settings.ts` MUST rows + tests (everything depends on `SettingKey`). 2. `grade.ts` + tests. 3. `standing.ts`, `probe.ts` + tests. 4. `hours.ts`, `inputs.ts` + tests. 5. Marks form, probe screen, hours form, `/settings` (with subsystem 5).

SHOULD, in order: models rows → self-rating → what-if line → `paperWorked` → answer age → `hiddenPages` → `targetProvenance` (`published | guess`; when a guess the headline appends `— if 70% is the boundary for the grade you want. That figure is your guess.` and the footer lists it) → the need bracket for `weights_short` / `pending` (`If the missing 10 points are a separate piece of work and the final really carries 50: between 54% and 74% of the final. If the final's own weight is what is wrong, neither figure holds.`).

Cut, not deferred: the one-click "same percentage on every unmarked paper" fill, which wrote a back-solved number as the student's estimate (against D4). If ever built it is a live `same_as_this_paper` provenance, never stored as a mark.

## 11. Cross-subsystem

**Subsystem 1 (engine).** Consumes `PlannerInputs` only; never recomputes a standing or an hours figure. Divergences from `01-engine.md`, founder decisions winning: (a) §8.2's `marksNeeded + ceil(margin × T)` is replaced by the single ceil of D5 — its 50- and 60-mark examples happen to give the same 32 and 38; (b) `pNoTopic` is `no_topic_answerable_pct × yield` (0.32 at defaults), not the unprobed topic's `pNow` (answers 01 §18.3); (c) the adapter gap in 01 §17 is closed: `PlannerTopicInput` carries `name`, `probeN`, `thin`, `noLeaves`; `PlannerInputs` carries `pNoTopic`, `gambleBudgetMarks`, and `needPct` on `beyond`; (d) `finalPaperTotal = null` means percent only — the engine's `*OnFinal` fields should be nullable, not fall back to the latest paper's total (an unlabelled assumption); (e) `marksOnPaper` throws on out-of-range input; `buildPlannerInputs` guarantees the range and 01 §9.1's validation runs first; (f) `exact_optimiser_max_topics` is 16 (D-K); (g) no `EvidencePolicy` settings; (h) a `noLeaves` topic is listed under `No evidence either way` with the D-N sentence and never struck through; a topic with `pNow = pAfter` is `already at your stated standing` (D-L), reachable only via the COULD `Mark as revised`.

**Subsystems 3 / 4.** Every Leaf has a stable `leafId`, `page`, `marks`, `excerpt`, its paper's label and order, the enclosing question's first page (for `contextPages`), and `counted` (D-E). Page images are addressable by `(paperId, page)`. Tables: `grade_components(id, title, weight_pct, is_final, scored NULL, out_of NULL, entered_as NULL, provenance NULL)`; course inputs `target_pct`, `final_paper_total` + `final_paper_total_confirmed`, `exam_date`, `hours_mode`, `total_hours`, `days`, `hours_per_day` (all NULL-able); `topics.own_hours NULL`, `topics.self_rating NULL`; append-only `probe_responses(id, leaf_id, grade, answered_at)`; `settings(key, value, set_at)`. Server actions `saveSetting`, `resetSetting`, `recordProbeAnswer`; `today` injected at the edge. Demo fixture papers print a maximum mark of 100; the seed follows §6.3 (D-F).

**Subsystem 5 (experience).** Renders every sentence in §2–§5 verbatim and follows §3.5. Changes against `05-experience.md`: `StandingVM.kind` becomes `probed | not_probed | no_questions_on_file` and `StandingVM.short` is `PlanningP.short` (with its `you said:` prefix); `S.probeAnchor` and the button labels are superseded by §3.4; the `secured` layout needs two headlines (`already_secured` vs `secured_if_estimates_hold`) and "every topic can be dropped" is deleted (D-M); the needed sub-line reads `(5% buffer — a default, no data behind it)` and links to `/settings#margin_pct` (key renamed from `safety_margin_pct`); the demo headline is `You need 58 of 100 on the final` (D-F), not `58 of 80`; the assumptions note is always present with its two clauses (D-G); 05's "blue pen = the student's hand" is the student's colour of D-P, so its four pens and the three inks here are one rule; 05 §7's vocabulary sweep gains the words in §9.

## 12. Open questions

- Q1. Refusing on weights ≠ 100 (D3) is stricter than Nexus. If it annoys in real use, the fallback is to treat the missing weight as one more pending component and ask for an estimate.
- Q2. The probe anchor ("at least three-quarters of the marks" / "about half") has not been tried on a real student. It is the bar the whole standing rests on.
