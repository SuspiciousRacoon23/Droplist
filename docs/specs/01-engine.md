# 01 — Engine (`src/engine/**`)

> Subsystem 1 of DROPLIST. Read `docs/BRIEF.md` first; nothing here relitigates
> its "Settled decisions" or "Non-negotiables". Founder decisions D-A…D-P
> (2026-09-18) are applied throughout and are not reopened.
> Every item is **MUST** unless marked SHOULD / COULD. The MUST set is sized for
> two agents over about two days (§15). Every number in a worked example was
> produced by a script (§13), not by hand.

## 0. Decisions in one page

| # | Decision |
|---|---|
| D1 | All tree arithmetic is **integer**: marks ×100, p in per-mille; one mark = 100 000 units. Exact ties, no epsilon in the tree. |
| D2 | A counted paper has **every row confirmed** (structure and every leaf mapping). A leaf is `mapped` (1+ topics) or `none` (explicit `none_of_these`); anything else keeps the paper out of the count with a visible reason. No provisional-mapping machinery (D-E). |
| D3 | Multi-topic leaf: available only if **no** mapped topic is dropped; expected value uses **min** of the topics' p. |
| D4 | Each paper gets its **own whole-mark target** from subsystem 2's `marksOnPaper` with `T = max(stated max, attainable)`; one rounding (D-O). Papers are ranked against each other in percent of `T`. |
| D5 | Headline = the **lowest** expected-marks backtest across counted papers, labelled "hardest paper on file", always above the full per-paper table (D-C). Mean is a tie-break and a footer. |
| D6 | **One objective, one constraint set** (D-B): minimise minutes subject to hours, the per-paper targets, and an availability loss of at most `G` marks on every paper for the counted drops. Default `G = 0`: the drop list holds only structurally-safe drops. |
| D7 | **Gambles** are a separate list: topics the plan still studies but could drop while keeping the target on expected marks, priced by what they would have cost on a paper on file (§7.6). |
| D8 | A topic already at its target (`pAfter == pNow`) is its own class, "already at your stated standing": not a drop, not a gamble, never counted against `G` (D-L). |
| D9 | A topic on no counted paper is listed under "No evidence either way", never struck through (D-N). |
| D10 | Exact solver = flat enumeration to 16 free topics (D-K). Branch-and-bound to 30 is SHOULD; greedy is SHOULD and labelled approximate. Pins are the MUST escape hatch. |
| D11 | No recency weighting, smoothing, or probability of appearance: those are predictions wearing a backtest's clothes. |
| D12 | A secured target produces **no drop list** (D-M). |

## 1. Files

Extensionless relative imports (`./tree`, not `./tree.ts`); run only via `node --import tsx --test`; no enums, no parameter properties. No runtime import from outside `src/engine/`; `import type` from `src/config/settings.ts` is allowed. `expected.ts` imports `marksOnPaper` from `./grade` (subsystem 2, same directory); until it lands, stub it behind the same signature.

```
src/engine/
  types.ts          every exported type in §3; no logic
  units.ts          MARK_SCALE, P_SCALE, pMille, markUnits, unitsToMarks, keyOfPct
  tree.ts           walk, leavesOf, ancestorsOf, attainable, isCompulsory, contextOf, checkTree
  value.ts          value, valueWithTrace              (reference implementation, used for explanations)
  compile.ts        compilePaper, evalCompiled          (hot-loop implementation)
  select.ts         selectPapers
  availability.ts   paperLoss, setLoss, safeSplit, gambleCandidates, topicOnPaper
  expected.ts       paperReach, targetsFor
  planner.ts        plan, toPlanInput, solveFlat; solveBranchBound (SHOULD); solveGreedy (SHOULD); evaluateStudySet (SHOULD)
  recurrence.ts     recurrence (SHOULD)
  index.ts          barrel: plan, toPlanInput, setLoss, checkTree, selectPapers, types
  testkit.ts        leaf()/group()/paper() builders, mulberry32, randTree, randInstance, exampleCourse
  *.test.ts         next to the file under test; also imports.test.ts (greps every engine import line)
```

## 2. Units

```ts
export const MARK_SCALE = 100;            // marks stored to 2 decimals
export const P_SCALE = 1000;              // p in per-mille
export const UNITS_PER_MARK = 100_000;
export const pMille = (p: number) => Math.round(Math.min(1, Math.max(0, p)) * P_SCALE);
export const markUnits = (marks: number) => Math.round(marks * MARK_SCALE);
export const unitsToMarks = (u: number) => u / UNITS_PER_MARK;
export const keyOfPct = (pct: number) => Math.round(pct * 1_000_000);   // integer ordering key
```

Every value inside a tree evaluation is an integer number of units (`Int32Array` is safe to 21 474 marks; `checkTree` rejects a paper above 10 000). p is quantised once, at the boundary; the error is at most 0.05 marks on a 100-mark paper and the settings help text says so. Feasibility is an integer comparison (§8.2); ordering between candidate plans uses `keyOfPct` integers, so it is exactly transitive. Marks and percentages become floats only in the result object.

## 3. Types

```ts
// types.ts
export type PaperId = string; export type NodeId = string; export type TopicId = string;
export type Pick = number | "ALL";

// ---- choice tree ----
export type LeafMapping =
  | { state: "mapped"; topicIds: TopicId[]; confirmed: boolean }   // topicIds: length >= 1, unique
  | { state: "none"; confirmed: boolean }                          // explicit none_of_these
  | { state: "unknown" };                                          // not mapped yet, or mapping failed
export interface Leaf {
  kind: "leaf"; id: NodeId; label: string;   // "4(b)(ii)"
  marks: number;                             // >= 0, at most 2 decimals
  mapping: LeafMapping;
  excerpt: string;                           // pass-through; never read by the engine
  page: number;                              // 1-based PDF page where the marks are printed
}
export interface Group {
  kind: "group"; id: NodeId; label: string;  // "Section B", "Q5", "" for the root
  pick: Pick; children: TreeNode[];
  page: number; instruction: string | null; statedMarks: number | null;
}
export type TreeNode = Leaf | Group;
export interface Paper {
  id: PaperId; label: string; order: number;   // order: sortable sitting order, e.g. 202405
  root: Group;
  statedMaxMarks: number | null; statedMaxPage: number | null;
  structureConfirmed: boolean;
  excluded: { reason: string } | null;         // student excluded it ("syllabus changed in 2021")
}

// ---- plan input (per-topic numbers arrive RESOLVED from subsystem 2; the engine never recomputes them) ----
export interface TopicInput {
  id: TopicId; name: string;
  pNow: number; pAfter: number;      // 0..1; the engine uses max(pAfter, pNow)
  minutes: number;                   // whole minutes to revise, >= 0
  pBasis: Basis[]; minutesBasis: Basis[];
  thin: boolean;                     // fewer than 3 probe answers behind pNow
  pin: "study" | "drop" | null;
}
export type Need =
  | { kind: "blocked"; detail: string }
  | { kind: "secured"; onEstimates: string[] }                    // component titles; [] = official marks only
  | { kind: "needs"; needPct: number; marginPct: number; basis: Basis[] };   // 0 <= needPct <= 100, marginPct >= 0
export interface SolverOptions { maxExactTopics: number }
export const DEFAULT_SOLVER: SolverOptions = { maxExactTopics: 16 };
export interface PlanInput {
  papers: Paper[]; topics: TopicInput[]; need: Need;
  minutesAvailable: number | null;            // null = hours not stated: run without a budget, report the minutes needed
  pNoTopic: { value: number; basis: Basis[] };      // p for `none` leaves; setting no_topic_answerable_pct x p_target
  gambleBudget: { marks: number; basis: Basis[] };  // G >= 0, whole marks; setting gamble_budget_marks
  solver?: SolverOptions;
}

// ---- citations ----
export interface Cite { paperId: PaperId; page: number; nodeId: NodeId; label: string; scope: "node" | "whole-paper" }
export type Basis =
  | { kind: "pages"; cites: Cite[] }
  | { kind: "stated"; what: "standing"; source: "probe" | "self_rating"; topicId: TopicId; n: number | null }
  | { kind: "stated"; what: "minutes" | "hours" | "need"; topicId: TopicId | null }
  | { kind: "assumption"; key: string };      // a settings key; the UI reads value and `confirmed` from the registry
/** Any number the UI may print on its own. basis.length >= 1 always. */
export interface Figure { value: number; unit: "marks" | "pct" | "minutes" | "count"; basis: Basis[] }

// ---- selection and validation ----
export type ExclusionReason = "user-excluded" | "structure-unconfirmed" | "mappings-unconfirmed" | "broken-tree" | "no-marks";
export interface PaperSelection {
  counted: PaperId[];                                              // sorted by (order, id); every per-paper array below follows this order
  excluded: { paperId: PaperId; reason: ExclusionReason; note: string | null }[];
  thin: boolean;                                                   // counted.length < 3
}
export type TreeIssueCode =
  | "pick-exceeds-children" | "pick-zero" | "pick-invalid" | "empty-group" | "zero-mark-leaf" | "invalid-marks"
  | "duplicate-node-id" | "total-mismatch" | "group-marks-mismatch" | "dangling-topic" | "total-too-large";
export interface TreeIssue { code: TreeIssueCode; cite: Cite; detail: string; fatal: boolean }
export interface NodeTrace { nodeId: NodeId; units: number; chosen: NodeId[]; skipped: NodeId[] }
export interface Traced { units: number; byNode: Map<NodeId, NodeTrace>; selectedLeaves: Leaf[] }

// ---- backtests ----
export interface PaperReach {                 // expected marks, one paper
  paperId: PaperId; attainableMarks: number; statedMaxMarks: number | null;
  denominatorMarks: number;                   // T = max(stated, attainable)
  reachMarks: Figure; reachPct: Figure;       // native marks on this paper; percent of T
  needMarks: number; marginMarks: number; targetMarks: number; marginClipped: boolean;   // §8.2, whole marks
  gapMarks: number;                           // reachMarks - targetMarks (negative = short)
  clears: boolean;                            // integer comparison
  issues: TreeIssue[];
}
export interface Absorption {
  leaf: Cite; marks: number; topicIds: TopicId[];
  at: { cite: Cite; label: string; pick: number; of: number; slack: number };   // the choice group where the loss vanished
  deadChildren: number;                                                          // children of `at` that lost value
}
export interface PaperLoss { paperId: PaperId; loss: Figure; lost: Cite[]; absorptions: Absorption[] }
export interface SetLoss {
  topicIds: TopicId[]; perPaper: PaperLoss[];
  safe: boolean;                                            // loss = 0 on every counted paper
  worst: { paperId: PaperId; lossMarks: number } | null;   // largest loss; ties -> earliest paper; null when safe
}
export type LeafOnPaper = Cite & { marks: number; compulsory: boolean; context: string };   // context = nearest choice group's label, or "compulsory"
export interface TopicPaperDetail {
  paperId: PaperId;
  status: "not-on-paper" | "absorbed" | "lost";   // judged for safeDrops ∪ {topic}
  leaves: LeafOnPaper[]; absorptions: Absorption[];
}

// ---- result items ----
interface TopicBase {
  topicId: TopicId; name: string; minutes: number;
  pNow: number; pBasis: Basis[]; thin: boolean; pinned: boolean;
  appearedIn: Figure;              // count of counted papers the topic is on; cites = its leaves, or whole-paper cites when 0
}
export interface StudyItem extends TopicBase {
  pAfter: number; minutesBasis: Basis[];
  noEvidence: boolean;             // pinned study on a topic no counted paper has
  alsoGamble: boolean;             // appears in `gambles` with dropped = false
}
export interface DropItem extends TopicBase {         // structurally safe drops only
  alwaysOptional: boolean; perPaper: TopicPaperDetail[];
}
export interface GambleItem extends TopicBase {
  dropped: boolean;                // true only when G > 0 or kind = "reachable-with-gambles"
  alwaysOptional: boolean;
  cost: Figure;                    // marks: max over papers of loss(safeDrops ∪ {topic}); cites from that paper
  alone: number;                   // marks: max over papers of loss({topic})
  costPerPaper: PaperLoss[];       // setLoss(safeDrops ∪ {topic}).perPaper
  perPaper: TopicPaperDetail[];
}
export interface AtStandingItem extends TopicBase { pAfter: number }
export interface NoEvidenceItem extends TopicBase { papersOnFile: number }
export interface Headline {
  statistic: "hardest-paper-on-file"; paperId: PaperId;
  reachMarks: Figure; reachPct: Figure; denominatorMarks: number;
  needMarks: number; marginMarks: number; targetMarks: number; needPct: number; marginPct: number;
}
export interface Shortfall {
  paperId: PaperId; marks: number;         // largest (targetMarks - reachMarks) over counted papers, and where
  binding: "hours" | "ceiling";            // ceiling = even unlimited hours do not reach the target
  ceilingPct: number; meetsNeedWithoutMargin: boolean;
  minutesForFeasible: number | null;       // SHOULD; null in MUST
}
export interface PlanOutcome {
  kind: "reachable" | "reachable-with-gambles" | "beyond-reach";
  method: "exact" | "approximate";
  study: StudyItem[]; drop: DropItem[]; gambles: GambleItem[];        // each sorted as §9.6 says
  atStanding: AtStandingItem[]; noEvidence: NoEvidenceItem[];
  dropSet: SetLoss;                        // every counted drop (drop ∪ dropped gambles): the table's last column
  splitExact: boolean;                     // false only when the safe split fell back to greedy (§7.5)
  gambleBudget: { marks: number; usedMarks: number };   // usedMarks = dropSet.worst?.lossMarks ?? 0
  minutesPlanned: number; minutesAvailable: number | null; spareMinutes: number | null; pinsExceedHours: boolean;
  headline: Headline; perPaper: PaperReach[]; meanReachPct: number;
  shortfall: Shortfall | null;             // non-null iff kind = "beyond-reach"
  papers: PaperSelection; thin: boolean;
  assumptionsUsed: string[];               // sorted union of every `assumption` key behind the result
  stats: { liveTopics: number; freeTopics: number; subsetsTotal: number; subsetsEvaluated: number; paperEvals: number };
}
export type PlanResult =
  | { kind: "cannot-plan"; reason: "no-target" | "no-counted-papers" | "no-topics" | "too-many-topics" | "invalid-input"; detail: string; papers: PaperSelection }
  | { kind: "secured"; onEstimates: string[]; papers: PaperSelection }     // no lists of any kind (D-M)
  | PlanOutcome;
```

## 4. How a leaf is valued

A counted paper contains only `mapped` and `none` leaves with `confirmed: true` (§6). `D` is the set of counted dropped topics.

| Leaf | Availability | Expected marks |
|---|---|---|
| mapped `t` | `marks` if `t ∉ D`, else 0 | `marks × p(t)` |
| mapped `t1..tj` | `marks` if **no** `ti ∈ D`, else 0 | `marks × min p(ti)` |
| `none` | `marks` | `marks × pNoTopic` |

- `none`: the student confirmed the question belongs to no listed topic, so dropping listed topics cannot make it unavailable. No standing applies, so its p is the labelled input `pNoTopic` = `no_topic_answerable_pct × p_target` (its own registry row, D-E). It is a constant: no plan changes it, so it shifts every plan equally and distorts only substitution inside a choice group, which is why its basis is carried.
- Multi-topic: availability is all-required. For expected marks the combiner must give 1 at (1,…,1), 0 when any argument is 0 (so availability stays the exact special case, P6) and be monotone (the solvers need it). `min` reads as "as answerable as its weakest required topic"; product would assume independence. Per-leaf topic weights are COULD.

## 5. `value()`

```ts
export type LeafValueFn = (leaf: Leaf) => number;   // integer units, finite, >= 0
export function value(node: TreeNode, leafValue: LeafValueFn): number;
export function valueWithTrace(node: TreeNode, leafValue: LeafValueFn): Traced;
export function attainable(node: TreeNode): number;  // value(node, l => markUnits(l.marks) * P_SCALE)
```

```
value(leaf)  = sanitise(leafValue(leaf))                        // non-finite or < 0 -> 0
value(group) = v[i] = value(child i); m = children.length
               k = pick == "ALL" ? m : clamp(floor(pick), 0, m)   (NaN -> 0)
               order = indices by (v[i] DESC, i ASC); chosen = first k, re-sorted ASC
               return Σ v[i] over chosen, in ascending i
```

`valueWithTrace` records `chosen` / `skipped` per group and `selectedLeaves` (leaves reachable through chosen children, document order). A chosen child may be worth 0 and is still reported as chosen.

Top-k is optimal because selections inside sibling children are independent and leaf values are constants for a fixed `D` or `p`, so the group's best score is the best `k` of the children's best scores, and with non-negative values "at most k" equals "exactly k". Brute force agreed on 9 000 random tree × valuation pairs (`$SCRATCH/spikes/engine/props.ts`). Not covered, by design: cross-group rubrics ("five in all, at least two from each section") — the confirm UI sets `paper.excluded = { reason: "cross-section rule not supported" }`; a choice made after seeing the paper, for which the figure is a lower bound; time pressure, dependent parts, negative marking.

### 5.1 Edge cases (`checkTree`)

| Input | Behaviour | Issue | Fatal |
|---|---|---|---|
| `pick > children` | `k = m` | `pick-exceeds-children` | no |
| `pick = 0` | group worth 0 | `pick-zero` | no |
| `pick` negative / non-integer / NaN | clamp / floor / 0 | `pick-invalid` | no |
| empty group | worth 0 | `empty-group` | no |
| zero-mark leaf | worth 0; never "dead" in §7 | `zero-mark-leaf` | no |
| marks NaN or negative | treated as 0 | `invalid-marks` | **yes** |
| marks with > 2 decimals | rounded | `invalid-marks` | no |
| attainable ≠ stated max | both kept; `T` = the larger | `total-mismatch` | no |
| group attainable ≠ `statedMarks` | none | `group-marks-mismatch` | no |
| mapped topic id not in the course | paper not counted (§6) | `dangling-topic` | no |
| duplicate node id | | `duplicate-node-id` | **yes** |
| attainable > 10 000 marks | | `total-too-large` | **yes** |

No engine function throws on a flagged tree; issues are data. Determinism: the value is independent of tie-breaking; the trace is fixed by "value descending, index ascending"; topics are sorted by `id` (plain `<`) and papers by `(order, id)` on entry to every public function, so results are invariant under input permutation (P8).

## 6. Which papers count

```ts
export function selectPapers(papers: Paper[], knownTopics: ReadonlySet<TopicId>): PaperSelection;
```

First failing rule gives the reason: `excluded !== null` → `user-excluded`; `!structureConfirmed` → `structure-unconfirmed`; any leaf whose mapping is `unknown`, or has `confirmed: false`, or names a topic outside `knownTopics` → `mappings-unconfirmed` with `note = "<n> rows"`; a fatal `checkTree` issue → `broken-tree`; `attainable = 0` → `no-marks`. Excluded papers are always returned with reason and note; the UI lists them under the table. `thin = counted.length < 3` does not block the plan. All counted papers weigh the same; a paper from another syllabus is handled by exclusion, which is visible, not by a weight, which is not.

## 7. Availability backtest

### 7.1 Loss and its explanation

```ts
export function paperLoss(paper: Paper, drop: ReadonlySet<TopicId>): PaperLoss;
export function setLoss(papers: Paper[], drop: TopicId[]): SetLoss;
```

`loss(D, paper) = attainable − value(paper, availD)`, `loss(∅) = 0`, monotone in `D` (checked on 3 000 random trees, 0 violations). Explaining it, per paper:

```
base = valueWithTrace(root, full marks); after = valueWithTrace(root, availD)
nodeLoss(n) = base.byNode[n].units − after.byNode[n].units         // >= 0
dead = leaves with marks > 0 whose topics intersect D
for each dead leaf: walk up; the first ancestor with nodeLoss = 0 absorbs it
    -> Absorption { at = that group (always k < m), slack = of − pick, deadChildren = its children with nodeLoss > 0 }
    no such ancestor -> the leaf is `lost`
PaperLoss.loss.value = nodeLoss(root) in marks
```

The figure is always `nodeLoss(root)`, never the sum of lost leaves (two 20-mark questions with one dead 2-mark part and one dead 4-mark part lose 2, not 6). **Cites of the loss figure:** when `loss > 0`, the `lost` leaves (non-empty whenever the root loses, because a losing path from the root always ends at a dead leaf); when `loss = 0` and the set has leaves on the paper, every `absorptions[].at.cite` plus the dead leaves; when the set has no leaf on the paper, the root with `scope: "whole-paper"`.

### 7.2 Set-level safety

`D` is **structurally safe** iff `loss(D, paper) = 0` on every counted paper; safe sets are downward closed (P7). Two topics can each be safe alone and unsafe together: Section A compulsory 30 marks; Section B answer one of Q4 (20, Marketing) or Q5 (20, Decision trees); `loss{Mkt} = loss{DT} = 0`, `loss{Mkt, DT} = 20`. The set figure is the truth; per-topic figures are never totalled.

### 7.3 Per-topic attribution

For a gamble `t`: `alone = loss({t})` and `cost = loss(safeDrops ∪ {t})`, each maximised over papers. For a safe drop both are 0 by downward closure (no computation). `marginalInFullSet = loss(D) − loss(D \ {t})` and Shapley values are SHOULD / COULD: marginals are not additive (a compulsory 10-mark leaf mapped to `{u, v}` has `loss{u, v} = 10` and both marginals 0).

### 7.4 Topic on paper

```ts
export function topicOnPaper(paper: Paper, topicId: TopicId): LeafOnPaper[];   // document order; compulsory = every ancestor has k >= m
```

`appearedIn.value` = number of counted papers with at least one such leaf; cites = all of them, or one whole-paper cite per counted paper when there are none. `alwaysOptional` = at least one appearance and no compulsory leaf anywhere. `context` = label of the nearest ancestor with `k < m`, or `"compulsory"`.

### 7.5 The safe split

```ts
export function safeSplit(papers: Paper[], counted: TopicId[], minutes: ReadonlyMap<TopicId, number>): { safe: TopicId[]; gambles: TopicId[]; exact: boolean };
```

Input: the counted drop set `D_counted` (§9.1). If `setLoss(D_counted).safe`, `safe = D_counted` with no search (the `G = 0` case). Otherwise candidates = topics of `D_counted` with `alone = 0` on every paper (P7). With ≤ 12 candidates, enumerate every subset with the compiled availability evaluator (§9.4) and take the safe one with the largest Σ minutes, then largest size, then lexicographically smallest id list; `exact: true`. With more, add candidates in (minutes desc, id) order while the set stays safe; `exact: false`, and the UI folds it into its "approximate" note. `gambles = D_counted \ safe`.

### 7.6 Gamble candidates

```ts
export function gambleCandidates(ctx: PlanCtx, S: TopicId[], safe: TopicId[]): TopicId[];
```

Computed once, after solving, for `kind ∈ {reachable, reachable-with-gambles}`: for each `t ∈ S` with `pin ≠ "study"` and `minutes > 0`, `t` is a gamble candidate iff `S \ {t}` still clears every paper's target on expected marks (one compiled evaluation per paper). Its `cost = max_i loss(safe ∪ {t}, paper_i)` with the cites of that paper's loss figure; `dropped = false`. By optimality `cost > 0` for every candidate (a loss-free, feasible, cheaper set would have been the plan) — asserted in U-P. Dropped gambles from §7.5 use the same `cost` formula with `dropped = true`. Gambles sort by `cost` ascending, `minutesSaved` descending, id.

### 7.7 Classes and what the UI may say

| Class | Rule | Copy (owned by subsystem 5; the facts are the engine's) |
|---|---|---|
| safe drop | in `safe` | Struck through in red. "Covered by choice: 0 marks lost on every paper on file, together with the other struck-through units." |
| gamble | §7.5 / §7.6 | Not struck (unless `dropped`). "Not dropped. Each would save hours, but had you scored nothing on it, it would have cost up to `cost` marks on the `paper` paper (p.N)." |
| already at your stated standing | `pAfter == pNow`, not pinned | "Revising it changes nothing at your stated standing." Never a drop, never a gamble. |
| no evidence either way | on no counted paper | "Never appeared in the N papers on file. That is silence, not safety." Not struck. |

## 8. Expected-marks backtest

### 8.1 Per paper

```ts
export function paperReach(paper: Paper, pByTopic: ReadonlyMap<TopicId, number /* per-mille */>, pNoTopicM: number, target: PaperTarget | null): PaperReach;
```

Leaf value = `markUnits × p` per §4. For a plan `S`: `p(t) = max(pAfterM, pNowM)` if `t ∈ S`, else `pNowM`. A dropped topic keeps `pNow`; it is not zero. The two backtests bracket the truth about a drop: availability asks "what if I score nothing on it", expected marks "what if I score what I said I could". `reachMarks.basis` = `pages` (cites of `selectedLeaves`) plus one `stated` or `assumption` basis per topic touched.

### 8.2 Per-paper whole-mark targets (D-O)

```ts
export interface PaperTarget { needMarks: number; marginMarks: number; planTarget: number; marginClipped: boolean }
export function targetsFor(needPct: number, marginPct: number, T: number): PaperTarget;   // wraps ./grade marksOnPaper
```

```
T_i          = max(statedMaxMarks_i ?? 0, attainableMarks_i)
needMarks_i  = max(0, ceil(needPct/100 × T_i − 1e-9))
planTarget_i = min(T_i, ceil((needPct + marginPct)/100 × T_i − 1e-9));  marginMarks_i = planTarget_i − needMarks_i
marginClipped_i = needPct + marginPct >= 100
required_i   = planTarget_i × UNITS_PER_MARK
clears_i(S)  ⇔ EVunits_i(S) ≥ required_i;   feasible(S) ⇔ clears_i(S) for every counted paper
reachPct_i   = 100 × EVunits_i / (markUnits(T_i) × P_SCALE);  lowest(S) = min_i;  mean(S) = mean_i
```

Feasibility is decided in whole marks paper by paper; percent only ranks papers against each other. Example, need 58 %, margin 5 %: on 50 marks `needMarks = 29`, `planTarget = ceil(31.5) = 32`, `marginMarks = 3`; on 60 marks `35 / ceil(37.8) = 38 / 3`. A plan reaching 31.6 of 50 does not clear even though 63.2 % ≥ 63 %. The table leads with marks ("31.6 in reach · aims for 32"). `max(stated, attainable)` is conservative whichever way extraction is wrong; `total-mismatch` is shown on the row. The setting is labelled "Buffer above the marks needed" (never "safety margin"). Rescaling to the upcoming paper's total is SHOULD (`finalScale`) and never happens inside the engine.

## 9. The planner

```ts
export function plan(input: PlanInput): PlanResult;
```

### 9.1 Preparation

1. Validate: `need.needPct` or `marginPct` non-finite, `needPct ∉ [0, 100]`, `marginPct < 0`, any topic with non-finite or negative minutes, p outside [0, 1], or a duplicate id → `cannot-plan/invalid-input` (checked before `marksOnPaper`, which throws). `need.blocked` → `no-target`. No topics → `no-topics`. `selectPapers` counts nothing → `no-counted-papers`. `need.secured` → `{ kind: "secured", onEstimates, papers }` and stop.
2. Quantise: `pNowM = pMille(pNow)`, `pAfterM = max(pMille(pAfter), pNowM)`, `minutes = round(minutes)`, `pNoTopicM = pMille(pNoTopic.value)`. Budget = `minutesAvailable ?? +∞`.
3. Partition (topics sorted by id). `live` = on ≥ 1 counted paper.
   - `noEvidence`: not live and `pin ≠ "study"` (a pinned drop here stays in this class, `pinned: true`).
   - `forcedStudy`: `pin = "study"` (a non-live one gets `noEvidence: true`, minutes counted), or live free with `minutes = 0`.
   - `forcedDrop`: live, `pin = "drop"`.
   - `atStanding`: live, unpinned, `pAfterM == pNowM`.
   - `free`: the remaining live topics; `n = free.length`.
4. `n > solver.maxExactTopics` and no SHOULD solver → `too-many-topics`, detail "pin N topics as study or drop".
5. `forcedMinutes > budget` → budget = `forcedMinutes`, `pinsExceedHours = true` (only `S = forcedStudy` fits; it is evaluated, not refused).
6. Compile every counted paper (§9.4) over the live topics; compute `targetsFor` per paper.

### 9.2 Objective and order (D-B, D-K)

```
minimise   minutes(S)                                          S = forcedStudy ∪ X, X ⊆ free
subject to minutes(S) ≤ budget
           feasible(S)                                                             (§8.2)
           loss(D_counted, paper_i) ≤ max(G, loss(forcedDrop, paper_i))  for every i   (§7)
where      D_counted = forcedDrop ∪ (free \ X)      -- atStanding and noEvidence are never counted
```

Pinned drops never count against `G`. All three constraints are monotone in `S`. Candidate order, lower is better, all keys integer:

| Incumbent | Key |
|---|---|
| plan (all constraints) | `minutes ↑`, `keyOfPct(lowest) ↓`, `keyOfPct(mean) ↓`, `mask ↑` |
| effort (hours only) | `clears ↓` (1 before 0), `keyOfPct(lowest) ↓`, `minutes ↑`, `keyOfPct(mean) ↓`, `mask ↑` |

`mask` is the integer over `free` in id order (bit 0 = smallest id). Two incumbents, one pass, no ladder.

### 9.3 Outcome

- Plan incumbent exists → `kind: "reachable"`, `S` = that set.
- Else the effort incumbent `E`:
  - `E` clears every target → `kind: "reachable-with-gambles"`, `S = E`. Its `D_counted` violates `G` (otherwise it would be the plan); the split of §7.5 names the dropped gambles and their costs. `shortfall = null`. Required copy: "No plan within your N hours keeps every drop covered by choice. The best use of N hours also drops `t`: had you scored nothing on it, that would have cost up to `cost` marks (`paper` p.N)."
  - Else `kind: "beyond-reach"`, `S = E`, with `Shortfall`: `marks` / `paperId` = the largest `targetMarks_i − reachMarks_i` (ties → earliest paper); `binding = "ceiling"` if `S_all = live \ forcedDrop` still fails a target with the budget ignored, else `"hours"`; `ceilingPct = lowest(S_all)`; `meetsNeedWithoutMargin` = every paper has `reachMarks_i ≥ needMarks_i`. SHOULD `minutesForFeasible`: min minutes over sets meeting targets and `G` with the budget ignored, `null` when binding is ceiling.

The gap is arithmetic, not a judgement; the copy mirrors `BeyondReach`.

### 9.4 Compiled evaluation

`compilePaper(paper, topicIndex)` flattens the tree to post-order typed arrays: `isLeaf`, `markUnits`, `leafTopicStart/End` into `leafTopics` (live-topic indices), `leafNone` (1 for a `none` leaf), `pick` (effective k), `childStart/End` into `children`, `attainableUnits`, `denominatorUnits`, `maxFan`.

```ts
export function evalCompiled(c: Compiled, member: Uint8Array, pLo: Int32Array, pHi: Int32Array, pNone: number, val: Int32Array, scratch: Int32Array): number;
```

One forward pass, no allocation: leaf = `markUnits × (leafNone ? pNone : min over its topics of (member[t] ? pHi[t] : pLo[t]))`; group = sum if `k ≥ m`, max if `k = 1`, else insertion-sort the child values descending into `scratch` and sum the first `k`. Must equal `value()` exactly (U-C1; 0 mismatches in 10 000 spike evals). Two passes share one compilation:

| Pass | `member[t] = 1` for | `pLo`, `pHi` | `pNone` |
|---|---|---|---|
| expected | `forcedStudy ∪ X` | `pNowM`, `pAfterM` | `pNoTopicM` |
| availability | `forcedStudy ∪ atStanding ∪ X` (everything not in `D_counted`) | all 0, all `P_SCALE` | `P_SCALE` |

Forced entries are written once; the solver writes the free bits into both vectors per candidate. Availability constants are **never** the expected-marks constants (a `none` leaf or a pinned-study topic is fully available, not `p × marks`; test U-C2).

### 9.5 `solveFlat`

`minutesOf[mask]` by the lowest-set-bit recurrence (`Int32Array(2^n)`). Order papers by ascending floor reach (expected pass with `X = ∅`) so the likeliest failure is evaluated first. Allowance `A_i = max(G, loss(forcedDrop, paper_i))` in units.

```
for mask in 0 .. 2^n − 1:
  mm = forcedMinutes + minutesOf[mask];  skip if mm > budget
  skip if plan exists and mm > plan.minutes                  // equal minutes must still compete
  write bits; ev_i = expected pass per paper; if plan exists, stop at the first paper that fails
  clears = every ev_i ≥ required_i
  if clears: withinG = every (attainableUnits_i − availability pass_i) ≤ A_i    // only for clearing sets
             if withinG: compare with plan by the plan key
  if no plan yet: compare with effort by the effort key (needs every paper evaluated)
```

`O(2^n · Σ_i N_i)` expected evaluations plus one availability pass per clearing candidate. `method = "exact"`. Measured (M4, Node 25.9, 5 papers × 36 nodes, `$SCRATCH/spikes/engine/timing.ts`): 0.43 µs per paper-eval; n = 16 unpruned **141 ms**; n = 20 unpruned 2.2 s; the object-tree path is ≈ 6 µs per paper-eval and is used only for explanations. The demo course (≤ 10 live topics) plans in single-digit milliseconds.

- SHOULD `solveBranchBound`: DFS in id order using the two monotonicities (minutes grow, reach grows as topics are added); prune when over budget, over the incumbent's minutes, or when `inMask ∪ undecided` fails a target; exact to n = 30 (spike: ≤ 0.13 s at 20, ≤ 4.5 s at 30); P10 must include `G` and pins before the exact boundary moves.
- SHOULD `solveGreedy`: add the best `Δlowest / minutes` topic until feasible, reverse-delete, 1-swap local search; always `method: "approximate"` (spike: optimum in 77.7 % of 400 cases, missed an existing feasible plan 15 times), copy "Approximate: a cheaper plan may exist".
- SHOULD `evaluateStudySet(input, studyIds)`: the same result object for a student-chosen set; `spareMinutes` may be negative.

### 9.6 Result assembly

After solving, once, on the object path: `dropSet = setLoss(D_counted)`; `safeSplit`; `gambleCandidates`; per item `topicOnPaper`, `appearedIn`, `alwaysOptional`; `TopicPaperDetail.status` and absorptions from `paperLoss(safe ∪ {t})`; `paperReach` per paper for `S`; headline = the paper with the lowest `keyOfPct(reachPct)`, ties to the earliest by `(order, id)`; `meanReachPct`; `assumptionsUsed` = sorted union of assumption keys across every live topic's bases, `need.basis`, `gambleBudget.basis`, and `pNoTopic.basis` when any counted paper has a `none` leaf. Sorting: `study`, `drop`, `atStanding`, `noEvidence` by topic id; `gambles` per §7.6; every per-paper array in `papers.counted` order.

## 10. Headline wording (hand to subsystem 5)

The lowest expected-marks backtest across counted papers, always directly above the full table. Minimum, not median (with 3–5 papers the median discards the paper where the structure bit hardest) and not mean (an exam is one paper). Adding a paper can only lower it: new evidence never makes the tool more confident. Its weakness — one odd paper — is visible: the row carries its `issues` and the student can exclude the paper, which is then listed as excluded. Rules: "hardest paper on file" or "lowest backtest", never "worst case", never "prediction"; past conditional only ("On the 2022 paper, 34 of 50 would have been in reach"); marks first, percent second. Headline: "In reach on the hardest paper on file (2022): 34.0 of 50 (68 %). Needed on that paper: 29; the plan aims for 32." Table: paper · in reach · needed · aims for · gap · if every drop scored zero · flags. Under it: "The N covered drops cost 0 marks on every paper on file." `thin`: caption "2 papers on file — thin". UI words are "probed / not probed", never "tested"; self-report is prefixed "you said:"; assumptions are in pencil (D-P).

## 11. Recurrence (SHOULD)

`recurrence(papers, topics): RecurrenceReport` — per topic per paper `leafCount`, marks, `marksPct` of `T`, compulsory vs optional split, `sharedMarks`; `meanMarksPct` over all counted papers; `rank` by `papersAppeared ↓, meanMarksPct ↓, id`, **null when fewer than 3 papers are counted** (`ranked: false`, brief non-negotiable 6); `unmapped` marks on `none` leaves per paper. The MUST demo sentence needs only `appearedIn`, `alwaysOptional`, `context` and cites, which §7.4 provides.

## 12. Citations

Rule: **every number the UI prints is a `Figure`, or lives in an object that carries `cites` / `leaves` / `lost` for it.** Figures: reach (marks, pct), loss, cost, `appearedIn`. Objects with cites: `PaperReach` (`issues`, its `reachMarks` cites), `PaperLoss`, `Absorption`, `TopicPaperDetail`, `Headline` (through `reachMarks`), `Shortfall` (through `perPaper[paperId]`). Bare numbers (`needMarks`, `targetMarks`, `gapMarks`, `minutes`) are arithmetic on student inputs and are labelled by `need.basis` / `minutesBasis`. Test U-X1 walks every result: each `Figure` has `basis.length ≥ 1`; each `pages` basis has ≥ 1 cite; each cite resolves to a real node on that paper with a matching `page`; a whole-paper cite names the root.

- Presence claims cite nodes. Absence claims cite the root with `scope: "whole-paper"` ("2023 (whole paper)").
- Rendering "2023 p.2 · 2024 p.3": dedupe cites on `(paperId, page)`, sort by `(paper.order, page)`.
- The demo's click-through sentence is assembled from data: `DropItem` → "Covered by choice"; `absorptions[].at` → "Section B lets you skip `slack` of `of`"; `alwaysOptional` with one `context` → "this unit has never appeared outside Section B"; `appearedIn` cites → "2023 p.2 · 2024 p.3". SHOULD `describeDrop(item)` so the wording is tested once against the honesty rules.

## 13. Worked example — the engine unit fixture (`testkit.exampleCourse()`)

This is the **unit fixture only**; the demo course is subsystem 3's (§16). Numbers from `$SCRATCH/spikes/engine/rev2.mjs`. Seven topics, `pAfter = 0.8`, no pins.

| Topic | pNow | minutes |
|---|---|---|
| U1 | 0.6 | 180 |
| U2 | 0.5 | 180 |
| U3 | 0.7 | 120 |
| U4 | 0.4 | 120 |
| U5 | 0.3 | 240 |
| U6 | 0.2 | 240 |
| U7 | 0.5 | 120 |

Papers (every leaf 10 marks; Section A compulsory; Section B "any three of five"):

| Paper | Total | Section A | Section B |
|---|---|---|---|
| 2022 | 50 | U1, U2 | U3, U4, U5, U6, U7 |
| 2023 | 60 | U2, U3, U1 | U4, U5, U6, U7, U3 |
| 2024 | 50 | U1, U4 | [(a) U2 OR (b) U5], U3, U6, U7, U5 |

**Nothing studied.** 2022: A = 6 + 5 = 11; B values 7, 4, 3, 2, 5 → 16; total 27 = 54.00 %. 2023: 18 + 16 = 34 of 60 = 56.67 %. 2024: 10 + (max(5, 3) = 5, 7, 2, 5, 3 → 17) = 27 = 54.00 %. Everything studied: 40 / 48 / 40 = 80 % everywhere.

**Row A — need 58 %, margin 5 %, 12 h, G = 0.** Targets 32 / 38 / 32 (need 29 / 35 / 29, margin 3 / 3 / 3). 128 subsets, 93 within budget, **1** satisfies every constraint: `S = {U1, U2, U3, U4, U7}`, 720 min, reach 40 / 48 / 40 (80.00 % each; headline 2022 by earliest), `dropSet = {U5, U6}` with loss 0 / 0 / 0. Absorptions: on 2022 and 2023 both leaves die inside Section B (`pick 3, of 5, slack 2, deadChildren 2`); on 2024 U5's OR leaf is absorbed at Q3 (`pick 1 of 2, slack 1`) and the two Section B leaves at B (`deadChildren 2`). `alwaysOptional` holds for U5, U6, U7; not for U3 (compulsory on 2023). Gamble candidates (each keeps the plan feasible without it):

| Gamble | saves | cost per paper | cost | alone | reach without it |
|---|---|---|---|---|---|
| U1 | 180 | 10 / 10 / 10 | 10 | 10 | 38 / 46 / 38 |
| U2 | 180 | 10 / 10 / 10 | 10 | 10 | 37 / 45 / 37 |
| U4 | 120 | 10 / 10 / 10 | 10 | 10 | 36 / 44 / 36 |
| U7 | 120 | 10 / 10 / 10 | 10 | 0 | 37 / 45 / 37 |
| U3 | 120 | 10 / **20** / 10 | 20 | 10 | 39 / 46 / 39 |

U7 has `alone = 0` (covered by choice on its own) yet costs 10 next to the covered drops: it leans on the same Section B slack. For contrast, with the constraint removed (`G` unbounded) the plan is `{U4, U7}`, 240 min, reach 34 / 41 / 34, dropping U1, U2, U3 for a loss of 30 / 40 / 30 — the brief's objective verbatim, and why `G = 0` is the default.

**Row B — need 70 %, margin 5 %, 12 h, G = 0.** Targets 38 / 45 / 38. Same plan; exactly two gambles: U1 (cost 10, saves 180) and U3 (cost 20, saves 120).

**Row C — reachable with gambles: need 58 % + 5 %, 10 h, G = 0.** No set within 600 min is loss-free (the loss-free drop sets are subsets of {U5, U6, U7} of size ≤ 2, and the cheapest study set they leave is 720 min). Effort = `{U1, U2, U4, U7}`, 600 min, reach 39 / 46 / 39, clears every target → `kind: "reachable-with-gambles"`, `D_counted = {U3, U5, U6}`, loss 10 / 20 / 10, `usedMarks = 20`. Split: safe {U5, U6}; U3 a dropped gamble, cost 10 / 20 / 10, `alone` 0 / 10 / 0. The same result at 11 h.

**Row D — already at standing (D-L).** U1 marked revised (`pNow = 0.8`), need 58 % + 5 %, 12 h, G = 0: U1 is `atStanding`, never counted; plan `{U2, U3, U4, U7}`, 540 min, reach 40 / 48 / 40; drops {U5, U6} safe; four gambles (U2, U4, U7 cost 10; U3 cost 20). `G = 0` stays satisfied even though U1 is compulsory on every paper.

**Row E — pinned drop U3** (need 58 % + 5 %, 12 h, G = 0): allowance 0 / 10 / 0; nothing loss-free fits 12 h; effort `{U1, U2, U4, U7}` clears → `reachable-with-gambles`, safe {U5, U6}, U3 a dropped gamble with `pinned: true`.

**Row F — infeasible by hours:** 6 h, need 67 % + 5 % → targets 36 / 44 / 36 (need 34 / 41 / 34). 25 sets fit, none clears. Effort `{U3, U4, U7}`, 360 min, reach 35 / 43 / 35, lowest 70.00 % (unique at that level). `Shortfall`: 1 mark on 2022 (ties 2024; earliest wins), `binding: "hours"`, `meetsNeedWithoutMargin: true`; split of {U1, U2, U5, U6}: safe {U5, U6}, dropped gambles U1, U2 (cost 10 each). SHOULD `minutesForFeasible = 720` (`{U1, U2, U3, U4, U7}` is the cheapest set meeting the targets **and** `G = 0`; `{U2, U4, U7}` at 420 min meets the targets but loses 20 / 30 / 20).

**Row G — infeasible by ceiling:** need 80 % + 5 % → targets 43 / 51 / 43, 12 h or unlimited. Effort `{U1, U2, U3, U4, U7}`, 40 / 48 / 40, 3 marks short on 2022, `binding: "ceiling"`, `ceilingPct = 80`.

**Row H — pins exceed hours:** U5 and U6 pinned study (480 min) with 6 h: budget becomes 480, `pinsExceedHours: true`, `S = {U5, U6}`, reach 34 / 41 / 34 clears 32 / 38 / 32 → `reachable-with-gambles` (loss 30 / 40 / 20; safe {U7}; dropped gambles U1, U2, U3, U4).

**Mean tie-break (separate two-paper case).** Topics A, B (`pNow 0.5`, 60 min each), Y (`pNow 0.9` → at standing). P1 (20 marks, all compulsory): A 10, B 10. P2 (20): A 6, B 4, Y 10. Need 60 %, margin 0 → targets 12 / 12. `{}` reaches 10 / 14 (infeasible); `{A}` 13 / 15.8 (65 %, 79 %; mean 72); `{B}` 13 / 15.2 (65 %, 76 %; mean 70.5). Equal minutes, equal lowest; the plan is `{A}` by mean.

**Layouts the fixture set must also cover** (from the four real papers, described not quoted): a 50-mark paper with a compulsory section of three multi-part questions and "answer one of two" 20-mark multi-part questions whose parts span topics (the choice is between sums); an 80-mark and a 60-mark paper where everything is compulsory and each question prints its own `statedMarks` — nothing there is ever structurally safe and the engine must say so; a 30-mark essay paper that is a single pick-1-of-4 of 30-mark leaves.

## 14. Tests

Seeded generators in `testkit.ts` (mulberry32; no fast-check). `randTree(seed, { depth ≤ 3, fan ≤ 5 })` includes `pick 0`, `pick > m`, empty groups, zero-mark, multi-topic and `none` leaves. `randInstance(seed)`: n ∈ [3, 8] topics, 1–4 papers of 4–12 leaves, `pNow ∈ {0.2, …, 0.8}`, `pAfter = 0.8` (10 % of topics at standing), minutes ∈ {60, 120, 180, 240}, need ∈ [40, 80], margin ∈ {0, 5}, budget 40–70 % of total minutes, `G ∈ {0, 10}`, one pin in 20 % of instances; the suite asserts ≥ 30 % of instances end `reachable`. Each property runs 300 cases and prints the failing seed. **The MUST suite completes in under 30 s** or U-B fails.

### 14.1 Properties (P1–P9 MUST; P10–P13 SHOULD)

| # | Property |
|---|---|
| P1 | `0 ≤ value(n, lv) ≤ attainable(n)` whenever `0 ≤ lv ≤ full`. |
| P2 | `lv1 ≤ lv2` pointwise ⇒ `value(lv1) ≤ value(lv2)`. |
| P3 | `value(c · lv) = c · value(lv)` for integer `c ≥ 0`. |
| P4 | (SHOULD) `value` = brute force over every legal selection. |
| P5 | `loss(∅) = 0`; `D1 ⊆ D2 ⇒ loss(D1) ≤ loss(D2) ≤ attainable`. |
| P6 | Expected marks with `p ∈ {0, 1}` (0 on `D`, `pNoTopic = 1`) equals the availability value, multi-topic leaves included. |
| P7 | Subsets of a safe set are safe; `safeSplit.safe` is safe, `⊆ D_counted`, and no counted drop can be added to it and stay safe. |
| P8 | Shuffling children leaves `value` unchanged; shuffling `topics` and `papers` leaves `plan()` deep-equal. |
| P9 | `solveFlat` = a naive oracle (recursive `value`, all candidates sorted by the §9.2 keys, both incumbents) for n ≤ 8, pins and `G` included; `evalCompiled` = `value` on random memberships for both passes. |
| P10 | `solveBranchBound` = `solveFlat` on 300 instances. |
| P11 | More hours never hurt: reachable at `H1` ⇒ the identical plan at `H2 ≥ H1`; otherwise `lowest(H2) ≥ lowest(H1)`. |
| P12 | For two runs both `reachable`: raising `needPct` or `marginPct` never lowers `minutesPlanned`; raising `G` never raises it; raising any `pNow` never raises it. |
| P13 | `solveGreedy` is never better than `solveFlat` under the plan key and stays within budget. |

### 14.2 Unit tests (MUST)

- **U-V** value: single leaf; ALL group; pick 1; any 5 of 7; nested OR inside a pick section; pick > children; pick 0; empty group; zero-mark leaf; NaN / negative marks; tie → lower index, `chosen` in document order; 12.5-mark leaves sum exactly.
- **U-T** tree: `attainable` on the §13 papers = 50 / 60 / 50; every `checkTree` code from a minimal tree with its `fatal` flag; `isCompulsory`; `contextOf`.
- **U-S** select: each `ExclusionReason` in rule order, including an `unknown` leaf, an unconfirmed `none` leaf and a dangling topic → `mappings-unconfirmed` with the row count; `thin` at 2 vs 3; stable order.
- **U-A** availability: §13 losses exactly (`{U5} = {U6} = {U5, U6} = 0`, `{U5, U6, U7} = 10` everywhere, `{U3, U5, U6, U7} = 20 / 30 / 20`); Marketing / Decision-trees (0, 0, 20); `{u, v}` (10); partial question loss (parts 2 and 4 dead → 2, both cited); the absorption on 2022 names Section B with `pick 3, of 5, slack 2, deadChildren 2` and the 2024 OR names Q3; zero-loss cites are the absorbing groups; absence cites the root; `safeSplit` on `{U3, U5, U6}` → safe {U5, U6}, exact; three topics on two units of slack → deterministic split; greedy fallback at 13 candidates sets `exact: false`.
- **U-E** expected: §13 reach figures to the unit; `T` with stated 60 / attainable 50 and the reverse; targets 32 / 38 and margins 3 / 3 for 58 % + 5 %; 36 / 44 for 67 % + 5 %; 31.6 of 50 does not clear and 32.0 does; `marginClipped` at 100 %; `min` on a two-topic leaf; `pNoTopic` on a `none` leaf; `pAfter` clamp when `pNow > pAfter`.
- **U-P** planner: rows A–H of §13 exactly, including gamble lists, `alone`, `usedMarks` and `pinsExceedHours`; every gamble's `cost > 0`; the mean tie-break case; `secured` returns no lists; every `cannot-plan` reason; `minutesAvailable: null` runs without a budget; `needPct = 100` with margin ends `beyond-reach` with `marginClipped`; `toPlanInput` on a hand-built `PlannerInputs` (each `PBasis` value, `beyond`, `not_stated` hours); a no-evidence topic is in `study` only when pinned; `plan(structuredClone(x))` deep-equals `plan(x)`; n = 17 without SHOULD solvers → `too-many-topics`; `assumptionsUsed` includes `no_topic_answerable_pct` only when a counted paper has a `none` leaf.
- **U-C1** compile: `evalCompiled` = `value` on the §13 papers for all 128 memberships, both passes. **U-C2** compiled availability = `setLoss()` for every membership on a fixture with a pinned-study topic, a pinned-drop topic, an at-standing topic, a `none` leaf and a multi-topic leaf.
- **U-X1** the citation walker (§12) over every §13 outcome.
- **U-I** `imports.test.ts`: no engine file imports outside `src/engine` except `import type` from `../config/settings`; no `.ts` extension in a specifier.
- **U-B** (SHOULD) `solveFlat` at n = 16 under 1 s.
- **U-D** (MUST, owned by subsystem 4, outside `src/engine`): after `npm run seed:demo`, `plan(toPlanInput(...))` on the demo course is `reachable` with exactly two safe drops, exactly one gamble, `spareMinutes ≥ 15 %` of the budget, and the same two drops under margin ± 2, `p_target` 70–90 and hours − 4 h.

## 15. Priorities and build order

Two agents after `types.ts` and `units.ts` are frozen and exported (first 2 h, one agent). Agent A: `tree`, `value`, `compile`, `solveFlat`. Agent B: `select`, `availability` (loss, absorption, split, candidates, `topicOnPaper`), `expected`. They meet at `plan()` assembly by the end of day 1 and publish a JSON snapshot of the §13 row A result for subsystems 4 and 5.

| Pri | Item | Est. |
|---|---|---|
| MUST | `types.ts`, `units.ts`, `testkit.ts` incl. `exampleCourse` and the two-paper tie case | 2 h |
| MUST | `tree.ts`, `value.ts` + U-V, U-T, P1–P3 | 2.5 h |
| MUST | `select.ts` + U-S; `imports.test.ts` | 1 h |
| MUST | `availability.ts`: loss, absorption, cites, `topicOnPaper`, `safeSplit`, `gambleCandidates` + U-A, P5–P7 | 4.5 h |
| MUST | `expected.ts` + U-E (stub `marksOnPaper` if `grade.ts` is late) | 1.5 h |
| MUST | `compile.ts`, `solveFlat`, two incumbents, outcome kinds, pins, `toPlanInput`, assembly + U-P, U-C1, U-C2, P8, P9 | 8 h |
| MUST | U-X1 walker; snapshot for subsystems 4 / 5 | 1.5 h |
| MUST | integration with the demo fixture (U-D, with subsystem 4) | 1.5 h |
| | **MUST total** | **≈ 22.5 h** |
| SHOULD 1 | `solveBranchBound` + P10 (exact to n = 30) | 2 h |
| SHOULD 2 | `recurrence.ts` (rank, mean, shared, unmapped) | 1.5 h |
| SHOULD 3 | `evaluateStudySet` (what-if); `minutesForFeasible`; `finalScale`; `StudyItem.gain` (`lowest(S) − lowest(S \ {t})`, never totalled); `describeDrop`; P4, P11, P12 | 3 h |
| SHOULD 4 | `solveGreedy` + P13 and the approximate labels | 2 h |
| COULD | `marginalInFullSet`; Shapley for `|D| ≤ 10`; cross-group rubric encoding; per-leaf topic weights; a "papers disagree" flag when max − min reach > 15 points | — |

The demo path needs only MUST items on a course of ≤ 10 live topics with every row confirmed.

## 16. Cross-subsystem

**Inputs (2)** — `toPlanInput` (MUST, `planner.ts`):

```ts
export function toPlanInput(args: {
  inputs: PlannerInputs;                                                   // 02-inputs §5
  papers: Paper[];
  topics: { id: TopicId; name: string; pin: "study" | "drop" | null; probeN: number | null }[];
  settings: { gambleBudgetMarks: number; noTopicAnswerable: number; yield: number };   // fractions from engineSettingsFrom
  solver?: SolverOptions;
}): PlanInput;
```

Mapping: `hours × 60 → minutes` (rounded); `PlanNeed` blocked → `blocked`; secured → `secured` with `onEstimates` (subsystem 2 to add that array to its `secured` variant, from `GradeStanding.estimated` titles — D-M); needs → `needs` with `basis = [stated need] + assumption(safety_margin_pct)`; `beyond` → `needs` with `needPct = 100` (targets clip at `T`; the grade-level explanation is rendered by subsystem 5 from `PlannerInputs` directly); `hours.kind` `not_stated` or `invalid` → `minutesAvailable: null`. Per topic: `pBasis` `probe` / `probe_thin` → `{ stated, standing, source: "probe", n: probeN }` (`thin = probe_thin`); `self_rating` → `{ stated, standing, source: "self_rating", n: null }`; every `restsOn` key → an `assumption` basis; `hoursBasis` `own_figure` → `{ stated, minutes }`, `course_figure` → `assumption(hours_per_topic_cold)`. `pNoTopic.value = noTopicAnswerable × yield`, basis `[no_topic_answerable_pct, p_target_pct]`. `gambleBudget` basis `[gamble_budget_marks]`. Two registry rows are the engine's and belong in subsystem 2's table: `no_topic_answerable_pct` (plan group, percent, default 40, 0–100, unconfirmed until set) and `gamble_budget_marks` (plan group, integer marks, default 0, 0–50, unconfirmed until set, help: "Marks a drop may have cost on a paper on file before it is kept out of the drop list. 0 = only drops covered by choice."). `exact_optimiser_max_topics` becomes 16. Subsystem 2's §2.3 formula for `planTarget` must match D-O exactly (one rounding; `marginMarks = planTarget − marksNeeded`), and its `secured` copy must lose "every topic can be dropped" (D-M). Divergence: this spec's `Need.needs` never carries `needPct > 100`.

**Extraction (3)** produces trees in the §3 shape: `page` on every node, OR as `Group(pick 1)`, `statedMarks` where printed, leaf mapping `mapped` / `none` / `unknown` (`none_of_these` → `none`); it may call `checkTree` / `attainable`; numbering contiguity stays on its side. It owns the one demo course (D-F) and tunes it so that at need ≈ 58 % + 5 % the §9 plan has exactly two safe drops and §7.6 yields exactly one gamble, with the robustness margins of U-D. §13 is not a demo course.

**Data (4)** persists `structureConfirmed`, per-leaf `mapping` with `confirmed`, `excluded.reason`, `pin`; zod-validates before calling the engine; may run `plan()` off the request path (synchronous, pure). Results are plain JSON except `Traced.byNode`, which never leaves the engine. Owns U-D and the snapshot consumers. The confirm action "Confirm the N rows with no flag" sets both flags; 05's `pending` and `unknown` mapping states both arrive here as `unknown`.

**Experience (5)** renders `Figure`s and cited objects only, follows §10, and needs these changes to its view-models: `PlanVM.kind` gains `reachable_with_gambles`; `DropRowVM` is built from `DropItem` (safe only; `classification` "safe"); a new gamble list from `GambleItem` with `dropped`, `cost`, `alone` ("Alone this drop would have cost `alone`; next to the covered drops, up to `cost`"), header "Not dropped. Each would save hours, but had you scored nothing on it, it would have cost marks on a paper on file."; `atStanding` rows in the student's colour, "you said:"; `noEvidence` rows under "No evidence either way" with the D-N sentence. `alone` and `marginalInFullSet` on safe rows are 0 by P7, so `deriveSetCaveat`'s interaction rule cannot fire at `G = 0` and its slack rule works unchanged. `BacktestRowVM.lossIfDropsScoreZero = dropSet.perPaper[i].loss.value`; `lossHi` is gone. `Provisional` is gone: a counted paper has no unconfirmed row. `CaveatVM.approximate.safeCoreExact` → `splitExact`. The `unconfirmed_assumptions` note lists every key in `assumptionsUsed` (D-G). Answers to its questions: Q1 — one demo course, subsystem 3's, tuned as above; Q5 — a sentence under the table, agreed.

## 17. Open questions

1. Under `reachable-with-gambles` the effort rule (max lowest reach within hours) picks the fullest study set, which usually means the fewest and cheapest dropped gambles, but not always the single cheapest gamble. Accepting that keeps one rule and two incumbents; a "smallest gamble first" rule is the ladder D-B removed.
2. Half-mark papers: targets are whole marks (D-D); the engine handles 2-decimal marks exactly. A `mark_step` input is a ten-line change if a real course needs it.
3. Should `no_topic_answerable_pct` default to 40 or to the untested default 50? D-G says 40; the two are separate rows either way.
