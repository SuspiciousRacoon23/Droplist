# 04 — Data + app skeleton (`src/db/**`, `src/contracts/**`, server surface, scripts, build plan)

> Subsystem 4 of DROPLIST. Read `docs/BRIEF.md` first; its "Settled decisions" and
> "Non-negotiables" are fixed, as are founder decisions D-A…D-P (2026-09-18).
> Aligned to `01-engine.md`, `02-inputs.md` and `05-experience.md` as on disk; `03` was not
> on disk, so what this spec needs from extraction is stated as a contract in §13.
> **MUST** = the demo path and the honesty contract, sized for the two-day parallel build.
> **SHOULD** / **COULD** are one-line bullets.

## 0. Decisions

| # | Decision | Why |
|---|---|---|
| D1 | Meridian's data pattern, verbatim: one SQLite file, `better-sqlite3` + Drizzle for typed queries, DDL written by hand in `migrate.ts` (idempotent, no `drizzle-kit`), `node --import tsx --test`. | Proven on this machine on Node 25.9 (§1). One fewer tool; a parity test (§11) keeps schema and DDL honest. |
| D2 | The choice tree is stored as an **adjacency list** (`pattern_nodes.parent_id`, `ord`), one row per node, loaded into the engine's `Paper` in two queries. | Round-trips every fixture tree exactly; 0.9 ms per paper; row-level edits and per-row confirm flags need rows, not a JSON blob (§1, spike D1). |
| D3 | **Confirm flags live on the node row**: `structure_confirmed` on every node, `mapping_confirmed` on leaves. `leaf_topics` carries the topic ids and the model's confidence only. | A `none_of_these` leaf has no `leaf_topics` row and must still be confirmable. One row = one place the confirm UI writes. |
| D4 | A paper **counts** iff every node is structure-confirmed, every leaf is mapping-confirmed, and no fatal flag is open (D-E). `Paper.structureConfirmed` handed to the engine is that derived boolean; nothing else is derived. | One rule, computed in one function (`paperCounts`), read by the status feed, the loaders and the e2e spec. |
| D5 | Every mutating action re-runs the validators for that paper inside the same transaction and returns the fresh bundle; the client never patches state. | 05-experience D8 and §3.2 require it; flags then clear the moment their condition is false. |
| D6 | `plans` is an **append-only cache keyed by `(input_hash, engine_version)`**. `/` computes `PlanInput`, hashes it, reuses a stored result or runs `plan()` and stores it. | The plan the student saw is on record; repeated loads cost one lookup; the engine stays pure. |
| D7 | Long-running extraction = **a job row + an in-process runner + polling**. One run at a time, one page at a time, progress written to `extraction_runs`, polled by `GET /api/papers/status` every second. | Simplest reliable thing for a single-user local app. A dead server is detected by heartbeat and the run is marked `failed: interrupted`, never silently lost. |
| D8 | The model cache (`llm_cache`) is a table owned here and used by 3's router through a two-method interface. `seed:demo` loads 3's recorded responses into it, so the demo's third paper extracts with no model and no network (D-F, D-I). | Recorded extraction is real pipeline output replayed, not ground truth pasted in. |
| D9 | Uploaded PDFs are never served to the browser and never passed to a native binary. pdfjs (pure JS, `isEvalSupported: false`) reads them; only PNGs leave the server. | An untrusted file gets the smallest surface available. |
| D10 | Ids are validated by regex before any path or query; paths are built with `safeJoin` and asserted under the data directory. | Path safety by construction, not by review. |

## 1. Spikes (evidence)

All under `$SCRATCH/spikes/`. Nothing was copied into the repo.

| # | What ran | Result |
|---|---|---|
| D1 | `data/roundtrip.ts`: the §4 DDL in an in-memory SQLite; save each of 3's five fixture truth trees as rows; load back; deep-equal; edits; cascades. | 5 papers, 151 nodes, **0 mismatches**; save all five 67 ms (unbatched inserts); load one paper **0.92 ms**; `UPDATE marks` then reload shows the edit; deleting a group cascades its children (0 left); deleting a paper removes its nodes and `leaf_topics` (95 → 85); a second root row is refused by the partial unique index (`SQLITE_CONSTRAINT_UNIQUE`). SQLite 3.49.2. The fixture root node had no `page`; the loader defaults it to 1 (the cover). |
| D2 | `data/src/**/*.test.ts` run as `node --import tsx --test "src/**/*.test.ts"` on Node 25.9. | The quoted glob is resolved by Node itself; nested `.test.ts` files under two directories were found; 2/2 pass in 208 ms. |
| — | Reused: Meridian on Node 25.9 (`better-sqlite3` 11.10.0 loads; SQLite 3.49.2). 3's `nextspike`: Next 15.5.25 + React 19.3.0 + `pdfjs-dist` 6.3.289 + `@napi-rs/canvas` 1.0.9 with `serverExternalPackages` built (`next build` clean) and served a PDF text-and-render route (first hit 6.2 s cold compile, then 224 ms). 5's spike: Playwright 1.63.0 with Chromium and WebKit already downloaded on this machine. 3's vision log: a real page through `qwen3-vl:8b-instruct` takes 46–79 s; D-I's cover page 50.6 s. |

## 2. File layout (MUST)

```
droplist/
  package.json  tsconfig.json  next.config.mjs  postcss.config.mjs  playwright.config.ts  .nvmrc  .gitignore
  src/
    instrumentation.ts        register(): migrate(), sweep stale runs, start the job runner (Next looks here when src/app exists)
    engine/**                 1 (+ 2's grade/standing/probe/hours/inputs)      pure
    config/settings.ts        2
    ingest/**  extract/**  lib/llm.ts   3
    contracts/                4  (§6)   zod schemas + the action/loader types; imports engine types, never redefines them
    db/                       4  (§4–5) client, schema, migrate, ids, clock, paths, tree, edits, validate, queries, plan, llm-cache, jobs, testkit
    server/                   4  (§7)   loaders/ (one per route), status, upload, storage
    app/_actions/*.ts         4  (§7)   'use server' files, one per domain
    app/_adapters/*.ts        5  (scaffold ships confirm + papers adapters; 5 owns them from I1)
    app/api/papers/status/route.ts                   4
    app/api/papers/[paperId]/pages/[n]/route.ts      4
    app/**  components/**  copy/**                   5
  scripts/                    reset.ts  seed-demo.ts  (4)   fixtures-build.ts  eval-extraction.ts  (3)
  fixtures/demo/              3: manifest.json, <paperId>.pdf, <paperId>.truth.json, recorded/<paperId>.{extracted,llm}.json
  e2e/demo.spec.ts            4 owns the harness; 5 owns the assertions
  data/                       gitignored: droplist.db(+wal,shm), papers/<paperId>/original.pdf, papers/<paperId>/pages/<n>.png
  docs/
```

Boundary rule: `src/db/**` and `src/server/**` import the engine and the contracts; nothing under `src/app/**` or `src/components/**` imports `src/db/**` or `src/server/**` except `_actions`, the page files that call `src/server/loaders`, and the two route handlers. Components receive view-models only.

## 3. Stack and configuration (MUST)

`package.json` — versions are exact, every one verified on this machine on Node 25.9 (§1):

```json
{ "name": "droplist", "private": true, "version": "0.1.0", "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "next dev -p 4320",              "build": "next build",         "start": "next start -p 4320",
    "typecheck": "tsc --noEmit",
    "test": "node --import tsx --test \"src/**/*.test.ts\" \"scripts/**/*.test.ts\"",
    "test:e2e": "playwright test",          "check": "npm run typecheck && npm test",
    "migrate": "node --import tsx src/db/migrate.ts",
    "reset": "node --import tsx scripts/reset.ts",
    "seed:demo": "node --import tsx scripts/seed-demo.ts",
    "demo:reset": "npm run reset && npm run seed:demo",
    "fixtures:build": "node --import tsx scripts/fixtures-build.ts",
    "eval:extraction": "node --import tsx scripts/eval-extraction.ts" },
  "dependencies": {
    "next": "15.5.25", "react": "19.3.0", "react-dom": "19.3.0",
    "better-sqlite3": "11.10.0", "drizzle-orm": "0.36.4", "zod": "3.25.76",
    "pdfjs-dist": "6.3.289", "@napi-rs/canvas": "1.0.9" },
  "devDependencies": {
    "typescript": "5.9.3", "tsx": "4.23.1", "@types/node": "22.20.1", "@types/better-sqlite3": "7.6.13",
    "@types/react": "19.2.17", "@types/react-dom": "19.2.3",
    "tailwindcss": "4.3.3", "@tailwindcss/postcss": "4.3.3",
    "@playwright/test": "1.63.0", "pdfkit": "0.20.2", "@types/pdfkit": "0.17.6" } }
```

- `pdfkit` is 3's fixture generator (spiked); `@types/pdfkit` 0.17.6 is the one pin only resolved against the registry, not run — if its types fight `pdfkit` 0.20.2, drop it and `// @ts-expect-error` the import.
- `npm run demo:reset -- --full` passes `--full` through to `seed-demo.ts` (05 §13 fallback 1).
- `.nvmrc`: `25.9.0`. `.gitignore` adds to the existing file: `data/`, `test-results/`, `playwright-report/`, `e2e/.state/`, `*.tsbuildinfo`, `next-env.d.ts`.
- `next.config.mjs`: `serverExternalPackages: ["better-sqlite3", "pdfjs-dist", "@napi-rs/canvas"]`, `experimental: { serverActions: { bodySizeLimit: "128mb" } }` (five 25 MB uploads), `images: { unoptimized: true }`.
- `tsconfig.json`: Meridian's plus `"noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true` (02 verified its code under both), `"types": ["node"]`, `include` adds `scripts/**/*.ts`, `e2e/**/*.ts`. No ESLint this week: `next lint` would pull an unverified toolchain.
- `playwright.config.ts`: `testDir: "e2e"`, Chromium only, `viewport: { width: 1440, height: 900 }`, `webServer: { command: "npm run demo:reset && npm run dev", url: "http://localhost:4320", timeout: 180_000, reuseExistingServer: !process.env.CI }`, `use.video: "retain-on-failure"`.
- Environment: `DROPLIST_DATA_DIR` (default `<cwd>/data`), `DROPLIST_DB` (default `<data>/droplist.db`; `:memory:` is legal and used by tests), `DROPLIST_LLM_PROVIDER` (overrides the `llm_provider` setting; e2e sets `mock`). No other env.

## 4. Schema (MUST)

`src/db/schema.ts` in Drizzle; `src/db/migrate.ts` holds the equivalent DDL with `CREATE … IF NOT EXISTS` plus a `schema_meta(key, value)` row `version = 1`. Conventions: text primary keys, ISO-8601 UTC text timestamps (`answered_at` is string-compared by 02 §3.2), booleans as `integer({ mode: "boolean" })`, JSON as `text` columns named `*_json`. `PRAGMA journal_mode = WAL; PRAGMA foreign_keys = ON` on every open.

```ts
export const courses = sqliteTable("courses", {
  id: text("id").primaryKey(),                          // always "main" (one course, D-F); COURSE_ID constant
  title: text("title").notNull(),
  isSyntheticDemo: integer("is_synthetic_demo", { mode: "boolean" }).notNull().default(false),
  targetPct: real("target_pct"),                        // input, no default (02 §1)
  finalPaperTotal: real("final_paper_total"),
  finalPaperTotalConfirmed: integer("final_paper_total_confirmed", { mode: "boolean" }).notNull().default(false),
  finalTotalCiteJson: text("final_total_cite_json"),    // {paperId,page} of the prefill, or null
  examDate: text("exam_date"),                          // YYYY-MM-DD
  hoursMode: text("hours_mode", { enum: ["total", "per_day"] }).notNull().default("total"),
  totalHours: real("total_hours"), days: integer("days"), hoursPerDay: real("hours_per_day"),
  createdAt: text("created_at").notNull() });

export const gradeComponents = sqliteTable("grade_components", {                     // 02 §11 field list
  id: text("id").primaryKey(), courseId: text("course_id").notNull().references(() => courses.id, { onDelete: "cascade" }),
  ord: integer("ord").notNull(), title: text("title").notNull(), weightPct: real("weight_pct").notNull(),
  isFinal: integer("is_final", { mode: "boolean" }).notNull().default(false),
  scored: real("scored"), outOf: real("out_of"),
  enteredAs: text("entered_as", { enum: ["raw", "percent"] }), provenance: text("provenance", { enum: ["official", "estimate"] }) },
  (t) => ({ byCourse: index("gc_course_idx").on(t.courseId, t.ord) }));

export const topics = sqliteTable("topics", {
  id: text("id").primaryKey(), courseId: text("course_id").notNull().references(() => courses.id, { onDelete: "cascade" }),
  ord: integer("ord").notNull(), name: text("name").notNull(),
  ownHours: real("own_hours"), selfRating: text("self_rating", { enum: ["solid", "shaky", "not_started"] }),
  pin: text("pin", { enum: ["study", "drop"] }), createdAt: text("created_at").notNull() },
  (t) => ({ nameUq: uniqueIndex("topics_name_uq").on(t.courseId, t.name) }));   // DDL adds COLLATE NOCASE

export const papers = sqliteTable("papers", {
  id: text("id").primaryKey(), courseId: text("course_id").notNull().references(() => courses.id, { onDelete: "cascade" }),
  label: text("label").notNull(), ord: integer("ord").notNull(),      // engine Paper.label / Paper.order
  filename: text("filename").notNull(),                               // display only; never used in a path
  contentSha256: text("content_sha256").notNull(), byteSize: integer("byte_size").notNull(), pageCount: integer("page_count").notNull(),
  statedMaxMarks: real("stated_max_marks"), statedMaxPage: integer("stated_max_page"),
  excludedReason: text("excluded_reason"), excludedNote: text("excluded_note"),   // reason non-null = engine excluded {reason}
  isSynthetic: integer("is_synthetic", { mode: "boolean" }).notNull().default(false),
  createdAt: text("created_at").notNull() },
  (t) => ({ shaUq: uniqueIndex("papers_sha_uq").on(t.contentSha256), byOrd: index("papers_ord_idx").on(t.courseId, t.ord, t.id) }));

export const paperPages = sqliteTable("paper_pages", {
  paperId: text("paper_id").notNull().references(() => papers.id, { onDelete: "cascade" }), page: integer("page").notNull(),   // 1-based
  widthPx: integer("width_px"), heightPx: integer("height_px"), textChars: integer("text_chars"),
  hasTextLayer: integer("has_text_layer", { mode: "boolean" }), readMode: text("read_mode", { enum: ["text", "vision"] }),
  renderedAt: text("rendered_at") },                                  // null until <n>.png exists
  (t) => ({ pk: primaryKey({ columns: [t.paperId, t.page] }) }));

export const patternNodes = sqliteTable("pattern_nodes", {
  id: text("id").primaryKey(), paperId: text("paper_id").notNull().references(() => papers.id, { onDelete: "cascade" }),
  parentId: text("parent_id").references((): AnySQLiteColumn => patternNodes.id, { onDelete: "cascade" }),   // null = root
  ord: integer("ord").notNull(),                                      // index among siblings, 0-based, dense
  kind: text("kind", { enum: ["group", "leaf"] }).notNull(), label: text("label").notNull(), page: integer("page").notNull(),
  pick: integer("pick"),                                              // group only; NULL = "ALL"
  marks: real("marks"),                                               // leaf only; 2 decimals
  statedMarks: real("stated_marks"), instruction: text("instruction"),   // group only
  excerpt: text("excerpt").notNull().default(""),                     // leaf; ≤ 500 chars
  bboxJson: text("bbox_json"),                                        // {x,y,w,h} fractions of the page, optional
  source: text("source", { enum: ["extracted", "user"] }).notNull().default("extracted"),
  structureConfirmed: integer("structure_confirmed", { mode: "boolean" }).notNull().default(false),
  mappingState: text("mapping_state", { enum: ["pending", "mapped", "none", "unknown"] }),   // leaf only
  mappingConfirmed: integer("mapping_confirmed", { mode: "boolean" }).notNull().default(false),
  mappingSource: text("mapping_source", { enum: ["extracted", "user"] }) },
  (t) => ({ byParent: index("pn_parent_idx").on(t.paperId, t.parentId, t.ord), byKind: index("pn_kind_idx").on(t.paperId, t.kind) }));
// DDL adds: CREATE UNIQUE INDEX pn_root_uq ON pattern_nodes(paper_id) WHERE parent_id IS NULL;

export const leafTopics = sqliteTable("leaf_topics", {
  nodeId: text("node_id").notNull().references(() => patternNodes.id, { onDelete: "cascade" }),
  topicId: text("topic_id").notNull().references(() => topics.id, { onDelete: "cascade" }),
  ord: integer("ord").notNull().default(0), confidence: text("confidence", { enum: ["high", "medium", "low"] }),   // null for user mappings
  source: text("source", { enum: ["extracted", "user"] }).notNull().default("extracted") },
  (t) => ({ pk: primaryKey({ columns: [t.nodeId, t.topicId] }), byTopic: index("lt_topic_idx").on(t.topicId) }));

export const validatorFlags = sqliteTable("validator_flags", {
  id: text("id").primaryKey(), paperId: text("paper_id").notNull().references(() => papers.id, { onDelete: "cascade" }),
  code: text("code").notNull(), fatal: integer("fatal", { mode: "boolean" }).notNull(), scope: text("scope", { enum: ["paper", "row"] }).notNull(),
  nodeIdsJson: text("node_ids_json").notNull(), expected: real("expected"), actual: real("actual"),
  detail: text("detail").notNull(), page: integer("page"), computedAt: text("computed_at").notNull() },
  (t) => ({ byPaper: index("vf_paper_idx").on(t.paperId) }));

export const probeResponses = sqliteTable("probe_responses", {                        // append-only (02 §3.2)
  id: text("id").primaryKey(), leafId: text("leaf_id").notNull().references(() => patternNodes.id, { onDelete: "cascade" }),
  grade: text("grade", { enum: ["yes", "partly", "no"] }).notNull(), answeredAt: text("answered_at").notNull() },
  (t) => ({ byLeaf: index("pr_leaf_idx").on(t.leafId, t.answeredAt) }));

export const settings = sqliteTable("settings", {                                     // 02 §6: a row = confirmed
  key: text("key").primaryKey(), value: text("value").notNull(), setAt: text("set_at").notNull() });

export const plans = sqliteTable("plans", {                                           // immutable
  id: text("id").primaryKey(), createdAt: text("created_at").notNull(), engineVersion: text("engine_version").notNull(),
  inputHash: text("input_hash").notNull(), inputJson: text("input_json").notNull(), resultJson: text("result_json").notNull(),
  ms: integer("ms").notNull() },
  (t) => ({ byHash: index("plans_hash_idx").on(t.inputHash, t.engineVersion) }));

export const extractionRuns = sqliteTable("extraction_runs", {
  id: text("id").primaryKey(), paperId: text("paper_id").notNull().references(() => papers.id, { onDelete: "cascade" }),
  status: text("status", { enum: ["queued", "running", "done", "failed"] }).notNull(),
  stage: text("stage", { enum: ["reading", "assembling", "mapping"] }), page: integer("page"), pageCount: integer("page_count"),
  mode: text("mode", { enum: ["text", "vision"] }), done: integer("done"), of: integer("of"),
  provider: text("provider").notNull(), textModel: text("text_model").notNull(), visionModel: text("vision_model").notNull(),
  fromCache: integer("from_cache", { mode: "boolean" }).notNull().default(false), cacheHits: integer("cache_hits").notNull().default(0),
  cacheMisses: integer("cache_misses").notNull().default(0), visionPagesJson: text("vision_pages_json").notNull().default("[]"),
  discardedRows: integer("discarded_rows").notNull().default(0), error: text("error"),
  queuedAt: text("queued_at").notNull(), startedAt: text("started_at"), heartbeatAt: text("heartbeat_at"), finishedAt: text("finished_at") },
  (t) => ({ byPaper: index("er_paper_idx").on(t.paperId, t.queuedAt), byStatus: index("er_status_idx").on(t.status) }));

export const llmCache = sqliteTable("llm_cache", {
  key: text("key").primaryKey(),                                      // sha256 computed by 3's router (§13)
  provider: text("provider").notNull(), model: text("model").notNull(),
  kind: text("kind", { enum: ["vision_page", "text_rows", "instruction", "topic_map"] }).notNull(),
  requestJson: text("request_json").notNull(), responseJson: text("response_json").notNull(),
  recorded: integer("recorded", { mode: "boolean" }).notNull().default(false),   // loaded by seed:demo, not produced live
  ms: integer("ms"), createdAt: text("created_at").notNull() });

export const nodeEdits = sqliteTable("node_edits", {                                  // audit; feeds 3's eval (model vs student)
  id: text("id").primaryKey(), paperId: text("paper_id").notNull().references(() => papers.id, { onDelete: "cascade" }),
  nodeId: text("node_id").notNull(), op: text("op").notNull(), beforeJson: text("before_json"), afterJson: text("after_json"),
  at: text("at").notNull() },
  (t) => ({ byPaper: index("ne_paper_idx").on(t.paperId, t.at) }));
```

**Cascades.** `courses` → `grade_components`, `topics`, `papers`. `papers` → `paper_pages`, `pattern_nodes`, `validator_flags`, `extraction_runs`, `node_edits`. `pattern_nodes` → child nodes (self-FK), `leaf_topics`, `probe_responses`. `topics` → `leaf_topics`; after a topic delete, `deleteTopic` sets the orphaned leaves to `mapping_state = 'unknown', mapping_confirmed = 0` in the same transaction so the paper stops counting until re-mapped. `plans`, `llm_cache`, `settings` have no FKs: plans are snapshots, cache is content-addressed, settings are global.

**Ids** (`src/db/ids.ts`): `newId(prefix)` = `prefix + "_" + 12` chars of base32 from `crypto.randomBytes` (`p_`, `n_`, `t_`, `g_`, `f_`, `r_`, `x_`, `e_`). Accepted shapes, used by every zod schema and before every path: `PaperId /^[a-z0-9][a-z0-9_-]{2,63}$/`, `NodeId /^[A-Za-z0-9][A-Za-z0-9_:.-]{0,79}$/`, `TopicId /^[a-z0-9][a-z0-9_-]{0,39}$/`. Fixture ids (`dbs204-2022`, `sql`) satisfy them. `COURSE_ID = "main"`.

**Clock** (`src/db/clock.ts`): `type Clock = () => string` (ISO UTC). Every data function takes `deps: { db; clock }`; the app passes `systemClock`; tests pass a fixed one. `todayLocal(clock)` gives `YYYY-MM-DD` for 02's `studyDaysBetween`, computed once at the loader edge.

## 5. Tree ↔ rows, edits, validators, counting (MUST)

`src/db/tree.ts`:

```ts
export type NodeRow = typeof patternNodes.$inferSelect;
export type PaperBundle = { meta: typeof papers.$inferSelect; paper: Paper /* engine */; rows: NodeRow[] /* pre-order */;
  topicsByLeaf: Map<NodeId, { topicId: TopicId; confidence: Confidence | null }[]>; flags: ValidatorFlag[];
  pages: (typeof paperPages.$inferSelect)[]; counts: boolean; rowsToReview: number };
export function loadPaper(deps, paperId: PaperId): PaperBundle | null;
export function loadPapers(deps): PaperBundle[];                       // course order (ord, id)
export function savePaper(deps, x: ExtractedPaper, opts: { replace: boolean }): PaperBundle;   // §13 shape; one transaction
export function paperCounts(rows: NodeRow[], flags: ValidatorFlag[]): boolean;
```

Load: `SELECT * FROM pattern_nodes WHERE paper_id = ? ORDER BY ord`, `SELECT lt.* … JOIN pattern_nodes … WHERE paper_id = ?`, then assemble children by `parent_id` sorted by `ord`. Leaf mapping: `mapped` with ≥ 1 `leaf_topics` row → `{ state: "mapped", topicIds, confirmed }`; `none` → `{ state: "none", confirmed }`; `pending`, `unknown`, or `mapped` with zero rows → `{ state: "unknown" }`. Group `pick`: `NULL → "ALL"`. `Paper.structureConfirmed = rows.every(structure_confirmed) && leaves.every(mapping_confirmed)`; `Paper.excluded = excluded_reason ? { reason } : null`; `order = ord`. `counts = structureConfirmed && !flags.some(fatal)`. `rowsToReview` = rows not fully confirmed (05 `isFullyConfirmed`: structure, and mapping for a leaf unless `pending`).

Save: flatten pre-order (parents before children satisfies the self-FK), `ord` = sibling index, rows with `structure_confirmed = 0`, leaves `mapping_confirmed = 0`. `replace: true` deletes the paper's existing nodes first (cascading its probe answers) and is refused when any row is confirmed (`ConflictError`); re-extraction is only for unconfirmed papers.

**Edits** (`src/db/edits.ts`). Every op runs in one transaction: apply → `node_edits` row (`before_json` / `after_json` = the affected row(s), a deleted subtree in full) → `recomputeFlags` → return `loadPaper`. An edit is the student's statement, so it confirms what it touched (05 §5.3); `source` flips to `'user'` and stays there.

| op | Input (zod, `src/contracts/edits.ts`) | Effect |
|---|---|---|
| `set-marks` | `{ nodeId, marks: int 0..200 }`, leaf | `marks`, `source='user'`, `structure_confirmed=1` |
| `set-pick` | `{ nodeId, pick: int 1..50 \| "ALL" }`, group | `pick` (NULL for ALL), `source='user'`, `structure_confirmed=1` |
| `set-topic` | `{ nodeId, topicId: TopicId \| null }`, leaf | delete `leaf_topics` for the node; `topicId` → one row `{ord 0, confidence null, source 'user'}`, `mapping_state='mapped'`; null → `'none'`; `mapping_confirmed=1`, `mapping_source='user'` |
| `confirm` | `{ nodeIds: NodeId[] (1..400) }` | `structure_confirmed=1`; for leaves with `mapping_state != 'pending'` also `mapping_confirmed=1` (05 D8: one keystroke, both flags) |
| `unconfirm` | `{ nodeId }` | both flags 0 |
| `delete` | `{ nodeId }`, not the root | delete row (cascade); renumber the siblings' `ord` densely |
| `reparent` | `{ nodeId, parentId, index }` | parent must be a group of the same paper and not a descendant of `nodeId`; move, renumber both sibling lists; `source='user'` on the moved node |
| `add-leaf` (SHOULD) | `{ parentId, index, label, marks, page }` | new `n_` row, `source='user'`, both flags 1, `mapping_state='unknown'` |

Bulk confirm of "the N rows with no flag" is `confirm` with the ids the client computed from `bulkConfirmable(rows)`; the server recomputes the set and refuses (`{ error }`) if any id is flagged now.

**Validators** (`src/db/validate.ts`): `recomputeFlags(deps, paperId)` deletes the paper's flags and inserts: (a) engine `checkTree(paper)` issues mapped 1:1 (`code`, `fatal`, `scope = cite.scope === "whole-paper" ? "paper" : "row"`, `nodeIds = [cite.nodeId]`, `page = cite.page`, `expected`/`actual` parsed from `detail` where the engine gives them, else null); (b) 3's `extractionFlags(bundle)` from `@/extract/validators` (`numbering-gap`, `no-stated-total`; the scaffold stub returns `[]`); (c) `rows-without-page` when the latest run's `discarded_rows > 0`. Runs after every edit and at the end of every extraction. Flags are never dismissed; they disappear when recomputed clean.

**Counting** is D4. `Paper.structureConfirmed` is derived, never stored, so the engine's own `selectPapers` gives `structure-unconfirmed` for a half-reviewed paper and `broken-tree` for a fatal flag, and the status feed agrees by construction.

## 6. `src/contracts` (MUST)

One module per concern; every module exports zod schemas and `z.infer` types. Engine and inputs types are **imported** and pinned with `satisfies` / `z.ZodType<T>` so the schema cannot drift from the type it validates.

| Module | Owner | Contents |
|---|---|---|
| `ids.ts` | 4 | `PaperIdSchema`, `NodeIdSchema`, `TopicIdSchema`, `COURSE_ID`, `LIMITS` (§8) |
| `paper.ts` | 4 | `PaperSchema: z.ZodType<Paper>` (recursive via `z.lazy`), `LeafMappingSchema`, `ExtractedPaperSchema`, `ExtractionProgressSchema`, `ValidatorFlagSchema` (05's `FlagVM` minus display fields), `PaperStatusSchema` (05 §3.2 union, verbatim), `Confidence` |
| `course.ts` | 4 | `TopicsFormSchema` (`{ topics: { id?: TopicId; name: 1..80; ownHours?: 0.5..40 }[] (≤ 40); pasted?: string }`), `GradeFormSchema` (`components: { id?, title 1..60, weightPct 0..100, isFinal, scored?, outOf?, enteredAs?, provenance? }[] (≤ 12)`, `targetPct? 0..100`, `finalPaperTotal? 1..1000`, `finalPaperTotalConfirmed`), `HoursFormSchema` (02 `HoursInput` + `examDate?`) |
| `standing.ts` | 2 (content) | `TopicStandingSchema` (`.strict()` arms), `ProbeResponseSchema`, `GradeComponentSchema`, `ProbeAnswerFormSchema` (`{ leafId: NodeId; grade: "yes"\|"partly"\|"no" }`), `SelfRatingSchema` |
| `edits.ts` | 4 | the §5 `NodeEdit` union, `ConfirmRowsSchema` |
| `settings.ts` | 4 | `SaveSettingSchema` (`key` ∈ `SETTINGS.map(k)`, `raw` ≤ 40 chars); re-exports `SettingKey` from `@/config/settings` |
| `fixtures.ts` | 3 + 4 | `DemoManifestSchema` (§10) |
| `plan.ts` | 4 | `ENGINE_VERSION` re-export, `SOLVER_MUST = { maxExactTopics: 16, maxPaperEvals: 6_000_000 }` (D-K), `DropListBundle` (§9), `PlanInputHash` |
| `actions.ts` | 4 | `ActionResult<T> = { ok: true; data: T } \| { ok: false; error: string; field?: string }` and one `Input`/`Output` type per action of §7 |
| `index.ts` | 4 | barrel |

`ENGINE_VERSION` is exported by `src/engine/index.ts` (asked of 1 in §13; the scaffold stub exports `"0"`). It is bumped by hand when `plan()`'s semantics or result shape change; `plans` rows under an older version are never reused.

## 7. Server surface (MUST unless marked)

Server actions live in `src/app/_actions/<domain>.ts` (`'use server'`). Each: parse with the contract schema → on failure return `{ ok: false, error, field }` and write nothing → run the data function in a transaction → `revalidatePath` the affected routes → return. Confirm actions return the fresh `ConfirmVM` through `toConfirmVM(bundle)` in `src/app/_adapters/confirm.ts` (05's signatures, §3.2). `today` and `now` enter here and in the loaders, never lower.

| Action | Input schema | Returns | Data function |
|---|---|---|---|
| `saveTopics(form)` | `TopicsFormSchema` | `ActionResult<{ topicIds }>` | upsert by id, insert `t_` ids for new rows, parse `pasted` lines (`name \| 6` → own hours), keep `ord`; delete is SHOULD `deleteTopic(topicId)` |
| `saveGrade(form)` | `GradeFormSchema` | `ActionResult<void>` | replace the course's components (ids kept where given), set `target_pct`, `final_paper_total(+_confirmed)` |
| `saveHours(form)` | `HoursFormSchema` | `ActionResult<void>` | set the five hours columns and `exam_date` |
| `uploadPapers(form)` | `files: File[] (1..5)` | `UploadResultVM[]` | §8 `storeUpload` per file → paper row + `extraction_runs` queued → `ensureRunner()` |
| `confirmRows(paperId, nodeIds)` | `ConfirmRowsSchema` | `ConfirmVM` | `applyEdit(confirm)` |
| `unconfirmRow(paperId, nodeId)` | ids | `ConfirmVM` | `applyEdit(unconfirm)` |
| `updateLeaf(paperId, nodeId, patch)` | `{ marks?; topicId?: TopicId \| null }` | `ConfirmVM` | `set-marks` and/or `set-topic`, one transaction |
| `updateGroup(paperId, nodeId, patch)` | `{ pick }` | `ConfirmVM` | `set-pick` |
| `deleteRow`, `reparentRow`, `addLeaf` (SHOULD) | §5 | `ConfirmVM` | as named |
| `recordProbeAnswer(form)` | `ProbeAnswerFormSchema` | `ActionResult<void>` | insert `probe_responses` (leaf must exist and be a leaf); never updates |
| `setSelfRating(topicId, rating)` (SHOULD) | `SelfRatingSchema \| null` | `ActionResult<void>` | `topics.self_rating` |
| `saveSetting(key, raw)` | `SaveSettingSchema` | `{ error: string \| null }` | `validateSetting(defOf(key), raw)` → reason returned, nothing written; else upsert `settings` with `set_at` |
| `resetSetting(key)` | key | `void` | delete the row |
| `excludePaper(paperId, note)`, `includePaper`, `renamePaper`, `removePaper`, `retryExtraction` (SHOULD) | ids + ≤ 200-char text | `ActionResult<void>` | `excluded_reason = 'user-excluded'`; delete the paper (cascade) and its directory; requeue a run when the paper has no confirmed rows |
| `setPin(topicId, pin)` (COULD) | | | `topics.pin` |

Route handlers:

- `GET /api/papers/status` → `PaperRowVM[]` (05 §3.2), built by `src/server/status.ts` from `papers`, the latest `extraction_runs` row per paper, row counts and flags: `queued`; `running` + stage `reading` → `reading { page, pageCount, mode, elapsedS = now − startedAt }`; `assembling`; `mapping` → `mapping_topics { done, of }`; `failed { reason }`; `done` → `excluded` if `excluded_reason`, else `counted { rows, flags, mappingsProvisional: 0 }` if `paperCounts`, else `needs_review { rowsToReview, rows, flags }`. `fromCache = { seconds }` when the run's `from_cache` is true. `Cache-Control: no-store`.
- `GET /api/papers/[paperId]/pages/[n]` → `image/png`, `Cache-Control: public, max-age=31536000, immutable`. Validates `paperId` by regex and `n` as an integer in `1..page_count`; serves `data/papers/<id>/pages/<n>.png`; on a miss renders it with 3's `renderPage(pdfPath, n, dpi)` (`vision_render_dpi`, 150), writes the file and `paper_pages.rendered_at`. 404 for unknown paper or page. Never serves the PDF.
- `POST /api/papers/[paperId]/reread/[n]` (COULD): single-page live vision read; returns rows beside the stored ones, writes nothing.

**Loaders** (`src/server/loaders/*.ts`, one per route, called only from server components and passed to 5's adapters): `loadCourseHead(deps, today)`, `loadSetup`, `loadPapersList`, `loadConfirm(paperId)` → `{ head, bundle, topics }`, `loadStanding(today)` → per topic `{ topic, standing: buildStanding(...), items: selectProbeItems(...) }` with `ProbeCandidate`s built from every leaf (02 §3.1: `leafConfirmed = structure_confirmed`, `topicConfirmed = mapping_confirmed`, `contextPages` = pages from the enclosing top-level question's first page to the leaf's page, max 4), `loadSettingsPage` → `resolveSettings(rows)`, `loadDropList(today)` → §9.

**Extraction jobs** (`src/db/jobs.ts`):

```
ensureRunner(deps)        idempotent; the loop lives on globalThis.__droplistRunner so dev HMR does not start a second one
loop:  run = oldest queued  →  status running, started_at, heartbeat_at
       structure = await extractStructure({ paperId, pdfPath, pagesDir, settings, llm: cachedLlm, onProgress, signal })   // 3
       savePaper(structure, { replace: true })  with mapping_state 'pending' on every leaf; stage 'mapping'
       mapping = await mapTopics({ leaves, topics, settings, llm: cachedLlm, onProgress, signal })                          // 3
       apply mappings (leaf_topics + mapping_state mapped|none|unknown, confidence); recomputeFlags; status done, from_cache, counters
       on throw: status failed, error = message (≤ 500 chars); rows already saved stay, provisional
       next queued run, else stop
onProgress: UPDATE extraction_runs SET stage/page/page_count/mode/done/of, heartbeat_at   (throttled to ≤ 4 writes/s)
signal:     AbortSignal.timeout(llm_timeout_s × pages + 60 s) for the whole run; 3 applies per-call timeouts (D-J)
boot sweep (src/instrumentation.ts register()): running rows with heartbeat_at older than 10 min → failed 'interrupted: the server restarted'; queued rows are picked up
```

One run at a time keeps one model loaded (brief, hardware). `retryExtraction` (SHOULD) is a new queued row; the UI's `failed` row offers it.

## 8. File storage and untrusted PDFs (MUST)

`src/server/storage.ts`. Layout: `data/papers/<paperId>/original.pdf`, `data/papers/<paperId>/pages/<n>.png`. `data/` is gitignored; `reset` empties it.

`LIMITS` (in `contracts/ids.ts`): PDF ≤ 25 MB (the four real papers are 0.2–0.9 MB); ≤ 80 pages; ≤ 5 files per upload; ≤ 12 papers on file; ≤ 40 topics; ≤ 400 nodes per paper; excerpt ≤ 500 chars; label ≤ 24; filename stored ≤ 120 chars with control characters stripped.

`storeUpload(deps, file): UploadResultVM` — in order: size check → read into a Buffer → first five bytes must be `%PDF-` → `sha256` → `SELECT id, label FROM papers WHERE content_sha256 = ?` → `duplicate { paperId, paperLabel, counted }`; else `newId("p_")`, write to `data/papers/<id>/original.pdf.part` and rename; open with pdfjs (`isEvalSupported: false`, `stopAtErrors: true`, `verbosity: 0`) under `AbortSignal.timeout(20_000)` to read `numPages` (≤ 80) — any throw → delete the directory, `rejected { filename, reason }` in plain words ("not a PDF that could be opened", "more than 80 pages"); insert `papers` (`label` = a `20\d\d` match in the filename, else the stem; `ord` = the year or `created` order), `paper_pages` skeleton rows, a queued `extraction_runs` row → `accepted { paperId }`. All inside one transaction after the file is on disk; on any DB error the directory is removed.

Path safety: `paperDir(id)` and `pagePath(id, n)` call `safeJoin(DATA_DIR, ...)`, which resolves and asserts the result starts with `DATA_DIR + path.sep`; ids are regex-validated before reaching it; page numbers are integers from zod. Uploaded filenames never touch a path. No shell-out: poppler is not used (the pure-npm path was verified in 3's spike). The PDF is opened only by pdfjs on the server, never streamed to the client.

## 9. The plan pipeline (MUST)

`src/db/plan.ts` (`buildPlanInput`, `getOrComputePlan`) and `src/server/loaders/droplist.ts` (`loadDropList`):

```ts
export type DropListBundle = {
  head: CourseHead; today: string;
  settings: ResolvedSettings; engine: EngineSettings;                          // 02
  assessment: GradeAssessment | null; need: PlanNeed;                          // 02
  plannerInputs: PlannerInputs | null;                                         // 02 §5
  papers: PaperBundle[]; topics: TopicRow[];
  input: PlanInput | null; result: PlanResult | null; planId: string | null;   // 01
  standings: Map<TopicId, { standing: TopicStanding; p: PlanningP; hours: TopicHours }>;
};
export function loadDropList(deps, today: string): DropListBundle;
export function getOrComputePlan(deps, input: PlanInput): { result: PlanResult; planId: string; cached: boolean };
```

Steps: `resolveSettings(settings rows)` → `engineSettingsFrom`; `assessTarget` on the components (if the course has a target and ≥ 1 component, else `need = blocked`); `planNeed(outcome, finalPaperTotal, safetyMarginPct)`; for each topic `buildStanding(topicId, candidates, responses)` → `planningP(topicId, standing, selfRating, s)` → `topicHours`; `buildPlannerInputs`; papers = `loadPapers` (all, the engine selects); `toPlanInput(plannerInputs, papers, pins, DEFAULT_POLICY)` with `pNoTopic = { value: noTopicAnswerable × yield, basis: [assumption no_topic_answerable_pct, assumption p_target_pct] }`, `gambleBudgetMarks` from `gamble_budget_marks`, `solver: SOLVER_MUST`, `finalTotalMarks = course.final_paper_total`. `need.kind = "blocked"` → `result = null` and the loader stops (5 renders `cannot_say`).

`getOrComputePlan`: `inputHash = sha256(canonicalJson(input))` (keys sorted, arrays as given — the engine sorts on entry, so order is already canonical); `SELECT … WHERE input_hash = ? AND engine_version = ?` → reuse; else `plan(input)` timed, insert. Retention: keep the newest 100 rows (SHOULD). The engine's worst case at 16 free topics is well under the request budget (01 §9.6: 3.5 s at n = 20 with the flat solver), so `plan()` runs on the request path; `/` is `force-dynamic`.

## 10. Scripts (MUST)

- `src/db/migrate.ts` — `migrate(deps)`: `exec(DDL)`, insert `schema_meta version`, insert the `courses` row `main` if absent (title `"Untitled course"`). Also runnable: `npm run migrate`.
- `scripts/reset.ts` — delete `<DB>`, `<DB>-wal`, `<DB>-shm`, `data/papers/*`; run `migrate`. Refuses to run when `DROPLIST_DATA_DIR` resolves outside the repo unless `--force`.
- `scripts/seed-demo.ts [--full]` (D-F) — reads `fixtures/demo/manifest.json` (`DemoManifestSchema`):

```ts
export type DemoManifest = {
  course: { title: string; topics: { id: TopicId; name: string; ownHours: number | null }[];
    grade: { components: { title; weightPct; isFinal; scored: number | null; outOf: number | null }[]; targetPct: null; finalPaperTotal: number };
    hours: { mode: "total"; totalHours: number }; examDate: string | null;
    settingsWritten: Partial<Record<SettingKey, string>> };          // 02 §6.3: hours_per_topic_cold + safety_margin_pct only
  papers: { paperId: PaperId; label: string; ord: number; pdf: string; truth: string; recorded: { extracted: string; llm: string };
    demoState: "confirmed" | "cached" }[];                            // "cached": PDF + llm entries only; on file only with --full
  probes: { paperId: PaperId; nodeId: NodeId; grade: "yes" | "partly" | "no" }[];   // against confirmed papers
  expected: { needSentence: string; headlineH1: string; safeDropTopicIds: TopicId[]; gambleTopicIds: TopicId[];
    hardestPaperLabel: string; reach: number; need: number; target: number };   // produced by fixtures:build from the truth via plan()
};
```

  Steps, in one transaction after `migrate`: upsert course (`is_synthetic_demo = 1`, hours, exam date, no target, `final_paper_total` accepted), topics, components, settings rows from `settingsWritten` (`set_at = now`), then for each paper with `demoState: "confirmed"` (or every paper with `--full`): copy the PDF to `data/papers/<id>/original.pdf`, `sha256` it into `papers`, insert `paper_pages` from the recorded page list, `savePaper(recorded.extracted)` then `confirm` every node (`source` stays `'extracted'`: the seed confirms, it does not edit), `recomputeFlags`, insert a `done` run with `from_cache = 1`. For every paper, load `recorded.llm` into `llm_cache` with `recorded = 1` (the third paper's rows are produced at demo time by the real pipeline on cache hits). Insert `probes` with `answered_at` spread one minute apart. Render page PNGs for on-file papers with 3's `renderAllPages` (≈ 0.25 s per page, 05 S7). Finally assert `manifest.expected.safeDropTopicIds.length === 2` against a dry `loadDropList` with the manifest's target applied in memory, and print the demo state. The seed never writes a target or scored marks (05 §13).

- `scripts/fixtures-build.ts`, `scripts/eval-extraction.ts` — 3's; listed here so `npm run` resolves from Step 0 (stubs print "not built").

## 11. Test strategy and definition of done

Unit and integration tests are `node:test` files next to their module, run by `npm test`; DB tests use `src/db/testkit.ts`: `openTestDb()` (`:memory:`, `migrate`, fixed clock `2026-09-18T09:00:00Z`) and `seedExampleCourse(deps)` = 01's `testkit.exampleCourse()` papers written through `savePaper` and confirmed. e2e is Playwright against `demo:reset` state with the `mock` provider and every request not to `localhost:4320` aborted.

| Id | Test |
|---|---|
| T-S1 | Schema/DDL parity: for every Drizzle table, `PRAGMA table_info` columns (name, type, notnull, pk) equal the Drizzle definition; every index in `schema.ts` exists. |
| T-S2 | Cascades of §4, each; the partial unique root index; `topics_name_uq` is case-insensitive; `deleteTopic` sets orphaned leaves to unknown/unconfirmed. |
| T-T1 | Round trip: `savePaper` → `loadPaper` deep-equals 01's `exampleCourse()` papers and every `fixtures/demo/*.truth.json` (as in spike D1); pre-order and `ord` are preserved; `pick` NULL ↔ `"ALL"`. |
| T-T2 | `structureConfirmed` is false with one unconfirmed node or one unconfirmed leaf mapping; true when all confirmed; `paperCounts` false with a fatal flag; `pending` leaves block counting. |
| T-E1 | Each edit op: effect, `source='user'`, the confirm side-effect, the `node_edits` row, flags recomputed; `delete` renumbers; `reparent` refuses a descendant target and a leaf target; `replace: true` refused on a confirmed paper; bulk `confirm` refused when an id is flagged now. |
| T-V1 | `recomputeFlags` maps every `TreeIssueCode` from a minimal tree; `total-mismatch` clears after `set-marks` makes the sum match (the demo's "Totals check passed" line depends on it). |
| T-P1 | Probe answers: append-only (two answers for one leaf → both rows; `buildStanding` sees the latest); an answer for a group id or unknown id is refused. |
| T-G1 | `saveSetting("p_target_pct", "140")` returns the reason and writes nothing; `"80"` writes a row and `resolveSettings` no longer lists it unconfirmed; `resetSetting` deletes. |
| T-L1 | `loadDropList` on the example course with 02's example A grades produces `need.kind = "needs"`, `needPct = 58`, a `PlanResult` with two safe drops (01 §13 row 1, `G = 0`); `getOrComputePlan` hits the cache on the second call and misses after `ENGINE_VERSION` changes; the plan row's `input_json` re-parses with `PaperSchema` etc. |
| T-U1 | `storeUpload`: duplicate by hash; `%PDF-` check; 25 MB and 80-page limits; a corrupt file is rejected and leaves no directory; a valid file yields a queued run; `safeJoin` throws on `..` and absolute parts. |
| T-J1 | Job runner with a stub `extractStructure` / `mapTopics`: progress rows, `from_cache` counters, `failed` on throw with rows kept provisional, boot sweep marks a stale `running` row failed, two `ensureRunner` calls start one loop. |
| T-R1 | `/api/papers/status` derives each `PaperStatus` kind from a seeded state; the page route 404s on a bad id, a non-integer and an out-of-range page, and serves a PNG with immutable headers. |
| T-D1 | `seed-demo` on a temp data dir: the manifest's confirmed papers count, the cached one is absent (present with `--full`), `llm_cache` holds every recorded entry, no target is set, exactly the manifest's `settingsWritten` keys are confirmed. |
| E2E | `e2e/demo.spec.ts`, 05 §11 steps 1–8, expectations read from `manifest.expected`; plus: the status endpoint reaches `needs_review` for the third paper within 5 s with no network. |

**Definition of done (end of the week).** `npm run check` green; `npm run demo:reset` then `npm run dev` on a machine with Wi-Fi off and Ollama stopped completes the 05 §13 script with the fixture's expected figures; `npm run test:e2e` green in Chromium; `data/` empty after `reset`; no row of any table can be rendered without its `page` (every `pattern_nodes.page` is `NOT NULL`, and the confirm adapter drops nothing because nothing lacks a page); `git status` clean of `data/` and `papers-real/`.

## 12. Parallel build plan

**Step 0 — scaffold (one agent, ≈ 4 h, blocks everyone).** Creates the repo state every work package starts from, then tags `scaffold`:

1. `package.json` (§3) and `npm install`; `npx playwright install chromium` (already on disk for 1.63.0); `tsconfig`, `next.config.mjs`, `postcss.config.mjs`, `playwright.config.ts`, `.nvmrc`, `.gitignore`, `src/instrumentation.ts`.
2. **Frozen contracts**: `src/engine/types.ts` copied from 01 §3 verbatim; type-only stubs for `src/engine/{grade,standing,probe,hours,inputs}.ts` and `src/config/settings.ts` carrying 02's exported types, the `SETTINGS` array of 02 §6.2 (plus the two rows of §13) and function bodies `throw notBuilt("…")`; `src/engine/index.ts` exporting `ENGINE_VERSION = "0"` and stubs of `plan`, `evaluateStudySet`, `checkTree`, `selectPapers`, `recurrence`; `src/extract/index.ts` (`extractStructure`, `mapTopics`), `src/extract/validators.ts` (`extractionFlags → []`), `src/ingest/render.ts` (`renderPage`, `renderAllPages`), `src/lib/llm.ts` (`LlmClient`, `LlmCache` interface, a working `mock` provider that returns the recorded response or an empty result); all of `src/contracts/**` complete.
3. `src/db/**` complete except `plan.ts` (`client`, `schema`, `migrate`, `ids`, `clock`, `paths`, `tree`, `edits`, `validate`, `queries`, `llm-cache`, `jobs`, `testkit`) with T-S1, T-S2, T-T1 (on the example course), T-E1 green.
4. Six route pages that render "not built" with the layout, a placeholder `globals.css` (`@import "tailwindcss";` only), `_adapters/confirm.ts` and `_adapters/papers.ts` first versions, the two route handlers, `scripts/*.ts` (real `reset`; the rest print "not built").
5. `npm run check` and `npm run build` green. Commit `scaffold: contracts frozen`. From here a contract change is one commit by the integrator, announced to every package, who rebase; nobody else edits `src/contracts/**` or another package's stub signatures.

**Work packages** (git worktrees off `scaffold`, one agent each, disjoint directories):

| WP | Owns (may write) | May read | Must not touch |
|---|---|---|---|
| WP1 Engine | `src/engine/**` except 2's five files | contracts | DB, app |
| WP2 Inputs | `src/engine/{grade,standing,probe,hours,inputs}.ts` + tests, `src/config/settings.ts`, `src/contracts/standing.ts` | `src/engine/types.ts` | the rest of engine, DB, app |
| WP3 Extraction | `src/ingest/**`, `src/extract/**`, `src/lib/llm.ts`, `fixtures/**`, `scripts/{fixtures-build,eval-extraction}.ts` | `src/db/llm-cache.ts` (interface), contracts, engine `checkTree`/`attainable` | DB writes, app |
| WP4 Data + app | `src/db/**`, `src/server/**`, `src/app/_actions/**`, `src/app/api/**`, `scripts/{reset,seed-demo}.ts`, `e2e/**` harness, `src/instrumentation.ts` | everything | `src/app/**` pages, components, copy, adapters after I1 |
| WP5 Experience | `src/app/**` pages, `layout.tsx`, `globals.css`, `fonts/**`, `src/components/**`, `src/copy/**`, `src/app/_adapters/**`, `e2e/demo.spec.ts` assertions | loaders' types, contracts | DB, actions' internals, engine |

**Integration order** (the integrator = WP4's agent):

- **I1 (end of day 1)**: WP2 then WP1 merge; `npm test` green including 01's U-P on the §13 fixture and 02's reference tests; `ENGINE_VERSION = "1"`. WP4 rebases and finishes `plan.ts` + T-L1 against the real engine (until I1 it works against the stubs and the round-trip tests only).
- **I2 (day 2 morning)**: WP4 merges: actions, loaders, routes, jobs with the stub extractor; T-U1, T-J1, T-R1 green; `seed-demo` runs against WP3's manifest if it exists, else against `seedExampleCourse` in a `--example` mode used only for integration.
- **I3 (day 2 midday)**: WP3 merges: `fixtures:build` produces `fixtures/demo/**` and `manifest.expected`; the runner extracts a fixture PDF end to end on the mock provider; `seed-demo` real; T-D1 green; `eval:extraction` reports accuracy against truth.
- **I4 (day 2 afternoon)**: WP5 merges last against real loaders; the e2e demo spec runs green; the demo recording is made (05 fallback 3).

Until its dependency lands, each package builds against the Step 0 stubs and its own fixtures: WP5 against `src/components/__fixtures__`, WP4 against `seedExampleCourse`, WP3 against its truth files.

**Risks and the mitigation in place.** (1) Contract drift after freeze → single owner of `src/contracts/**`, `satisfies` pins, T-S1/T-T1 fail loudly. (2) `better-sqlite3` native binding breaks after a Node change → `.nvmrc` 25.9.0; `npm rebuild better-sqlite3` is the fix; the DB layer has no other native dependency. (3) Next bundling pdfjs / canvas → `serverExternalPackages` exactly as 3's spike; `runtime = "nodejs"` on both route handlers. (4) Server Action body limit → `bodySizeLimit` set; upload also has an explicit per-file check so the error is ours. (5) Two job loops after HMR → `globalThis` guard, T-J1. (6) Demo machine without Ollama → `DROPLIST_LLM_PROVIDER=mock` for e2e and the recorded cache for the demo; nothing in the demo path calls a model. (7) Fonts or browsers fetched at build time → `next/font/local` (05 D5), Playwright 1.63.0's browser already downloaded. (8) Fixture shape changes in WP3 after `expected` was computed → `fixtures:build` regenerates `expected` and the e2e spec reads it; nothing is typed by hand. (9) Time → the SHOULD list is cut first; MUST for this subsystem is ≈ 14 h of the ≈ 48 h two-day budget.

## 13. Cross-subsystem

**Asked of 1 (engine).** Export `ENGINE_VERSION` from `src/engine/index.ts`. `Paper.structureConfirmed` is supplied as the D4 derived boolean; the provisional-mapping path (01 §4) stays but never fires. `solver` is passed as `SOLVER_MUST` (16, D-K); the registry's `exact_optimiser_max_topics` should say 16 too. Results are stored verbatim as JSON: `plan()` must return plain data (01 §17 already promises no `Map` leaves the engine).

**Asked of 2 (inputs).** Two registry rows: `gamble_budget_marks` (plan·3, integer, default 0, 0–40 step 1, unconfirmed until set, label "Marks you are willing to gamble on the papers on file", D-B) and `no_topic_answerable_pct` (standing·7, percent, default 40, 0–100 step 5, unconfirmed until set, D-E), with `engineSettingsFrom` exposing `gambleBudgetMarks` and `noTopicAnswerable`. `safety_margin_pct`'s label becomes "Buffer above the marks needed" (D-O). `EvidencePolicy` is not a setting this week; `DEFAULT_POLICY` is passed (COULD: two rows). `TargetOutcome` gains `secured_if_estimates_hold` (D-M); the data layer already carries `provenance`.

**Assumed of 3 (extraction; `03` not on disk).** Functions in `src/extract/index.ts`:

```ts
extractStructure(a: { paperId: PaperId; pdfPath: string; pagesDir: string; settings: ModelSettings; llm: LlmClient;
  onProgress: (p: ExtractionProgress) => void; signal: AbortSignal }): Promise<ExtractedPaper>;
mapTopics(a: { paperId; leaves: { id: NodeId; excerpt: string; context: string | null }[]; topics: { id: TopicId; name: string }[];
  settings; llm; onProgress; signal }): Promise<Record<NodeId, { topicIds: TopicId[]; confidence: Confidence } | { none: true; confidence: Confidence } | null>>;  // null = mapping failed → unknown
type ExtractedPaper = { paper: Omit<Paper, "structureConfirmed" | "excluded">;      // every node has a page; ids n_…; leaf mapping {state:"unknown"}
  pages: { page; widthPx; heightPx; textChars; hasTextLayer; readMode }[]; bboxes: Record<NodeId, Bbox>; discardedRows: number;
  run: { visionPages: number[]; cacheHits: number; cacheMisses: number } };
type ExtractionProgress = { stage: "reading"; page; pageCount; mode } | { stage: "assembling" } | { stage: "mapping"; done; of };
```

`ModelSettings` is 3's slice of 02's registry (`llm_provider`, `ollama_base_url`, `vision_model`, `text_model`, `llm_timeout_s`, `text_layer_min_chars`, `vision_render_dpi`), resolved by 4 and passed in. `src/lib/llm.ts` exposes `LlmClient` with a `cache: LlmCache` where `LlmCache = { get(key: string): string | null; put(e: { key; provider; model; kind; requestJson; responseJson; ms }): void }` (implemented by `src/db/llm-cache.ts`); the key is `sha256(provider | model | kind | canonical request)` computed by the router so recorded entries replay byte-for-byte. `renderPage(pdfPath, n, dpi): Promise<Buffer>` and `renderAllPages` in `src/ingest/render.ts`. The fixture bundle is `fixtures/demo/manifest.json` per §10 with `expected` generated by `fixtures:build` through the engine, 3–4 papers of 100 marks (D-F), every page stamped synthetic. Note from spike D1: the truth tree's root needs a `page` (1) to satisfy the engine's shape.

**Deviations from 5.** Adapters `_adapters/confirm.ts` and `_adapters/papers.ts` are written first by the scaffold (near-1:1 projections of rows) and handed to 5 at I1; 5 owns every other adapter from the start. Confirm actions return `ConfirmVM` as 5 asks; every other action returns `ActionResult<T>` rather than `void`, so a refused input carries its reason (05 §6.4 needs it for settings; the others get it free). 5's Q2 (footer vs note) is 5's call; the data is the same `assumptionsUsed`. 5's Q3: one keystroke sets both flags, as §5 `confirm`. 5's Q6: `bbox_json` is stored when 3 supplies it; the highlight band degrades to nothing otherwise.

**Deviation from the task sketch.** `leaf_topics.confirmed` is not a column; confirmation is per leaf on `pattern_nodes.mapping_confirmed` (D3). `confidence` is the model's three-word enum, not a number, matching 3's spike.

## 14. SHOULD / COULD

- SHOULD: `deleteTopic`, `excludePaper`/`includePaper`/`renamePaper`/`removePaper`, `retryExtraction`, `setSelfRating`, `add-leaf`, plans retention, page-image preload of `n+1` on render, `--example` seed mode kept after I3 as a dev seed.
- COULD: `POST …/reread/[n]`, pins, `EvidencePolicy` settings rows, a `data/backups/` copy on `reset`, `drizzle-kit` if a second migration ever appears.

## 15. Open questions

1. Retention of `plans`: keep every plan the student ever saw (audit) or only the newest 100 (disk)? Default here: 100, SHOULD.
2. Re-extraction of a paper with confirmed rows is refused. Should `removePaper` + re-upload be the only path, or is a "discard my confirmations and re-read" action wanted before the presentation?
3. Should `node_edits` feed 3's `eval:extraction` (model proposal vs student correction on real papers) this week, or is truth-based accuracy on fixtures enough? The table exists either way.
4. `bodySizeLimit` of 128 MB assumes at most five 25 MB files; the real papers are under 1 MB. Lower both if the founder prefers a tighter surface.
