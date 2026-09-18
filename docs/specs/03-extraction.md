# 03 — Extraction (`src/ingest/**`, `src/extract/**`, `src/lib/llm.ts`, fixtures, eval)

> Subsystem 3 of DROPLIST. Read `docs/BRIEF.md` first. Founder decisions D-A…D-P
> (2026-09-18) are settled and are applied here without discussion.
> Every number in this spec was measured in a spike (§2). Nothing is estimated.
> MUST / SHOULD / COULD per D-H; the MUST set is about two agent-days (§16).

## 0. Decisions in one page

| # | Decision | Evidence |
|---|---|---|
| X1 | **PDF tooling = `pdfjs-dist` (legacy build) + `@napi-rs/canvas`.** Pure npm, Apache-2.0 + MIT, works on Node 25.9 and inside a Next 15.5 route handler with `serverExternalPackages`. No poppler, no mupdf. | §2 rows 1–4. mupdf is 2× faster but AGPL-3.0; poppler is an external binary and slower per call. |
| X2 | Text layer is primary. A page goes to the vision model only when its text layer is **unusable** (§4) *and* its pixels carry text ink. Blank, ruled and rough-work pages are skipped before any model call. | 4/4 real papers and 5/5 synthetic reconcile to their printed maximum through the text path alone (§2 row 5). |
| X3 | One intermediate representation, the **ROW** (§5). Both paths emit it; one deterministic assembler consumes it. | The brief's pipeline, made concrete. |
| X4 | The pick rule is resolved by **regex first**; the text model is asked only for sentences the regex cannot read or that carry a cross-group constraint. | Regex 19/22, model 18/20, and the failures are disjoint (§2 rows 8–9). |
| X5 | Vision: **110 dpi**, one page per call, `temperature 0`, `num_ctx 8192`, JSON-schema `format`, 240 s timeout, retry once, then flag the page. | 150 dpi costs 2616 prompt tokens and 95 s of prompt eval against 1 638 tokens / 33–53 s at 110 dpi, with no gain (§2 row 6). |
| X6 | Topic mapping: batches of 8 leaves, enum-constrained `topic` with `none_of_these`, case-study **stem passed as `context`**. | 23/24 with context vs 20/24 without; every miss without context is a case-study part (§2 row 10). |
| X7 | `llm.ts` has three providers: `ollama`, `mock`, and a **recorded cache** wrapper keyed by `(model, promptVersion, sha256(request))`. The demo replays; tests never call a model. | D-I. |
| X8 | One synthetic course, **DBS-204 Database Systems** at the fictional *Fixture Institute of Technology*: 10 topics, three 100-mark papers (2022, 2023, 2024). With need 58 % the only structurally safe drops are `nosql` and `security`; the only cheap gamble is `recovery` at 5 marks. | Enumerated over all 1 024 drop sets (§13.4). |
| X9 | Every measured number lives in `docs/eval/extraction.latest.json`, written by `npm run eval:extraction`. Docs and UI copy quote from it or not at all. | Brief non-negotiable 4, applied to ourselves. |

## 1. Files

```
src/ingest/
  pdf.ts         openPdf, pageLines, renderPage           MUST
  textlayer.ts   textLayerStats, textLayerUsable          MUST
  pixels.ts      inkStats, isInkless                      MUST
  route.ts       routePage                                MUST
  hash.ts        sha256File, sha256Json                   MUST
src/extract/
  rows.ts        Row types, zod schema, JSON schema       MUST
  parse-text.ts  parseTextRows                            MUST
  pick-rule.ts   parsePickRule (regex), resolvePickRule   MUST
  vision.ts      visionRows, VISION_PROMPT_V1             MUST (path) / SHOULD (eval)
  assemble.ts    assemble                                 MUST
  validate.ts    validate, ExtractionFlag                 MUST
  topics.ts      mapTopics, TOPICS_PROMPT_V1              MUST
  ids.ts         nodeId, nodeLabel                        MUST
  pipeline.ts    extractPaper, ExtractionResult           MUST
  *.test.ts      beside each file; node --import tsx --test "src/{ingest,extract,lib}/**/*.test.ts"
src/lib/llm.ts   LlmProvider, ollama, mock, recorded      MUST
fixtures/synthetic/
  course.ts      topics, question bank, paper plans       MUST
  generate.ts    -> dbs204-{2022,2023,2024}.pdf + .truth.json + manifest.json + demo-scenario.json   MUST
  scan.ts        -> dbs204-YYYY.scan.pdf (rasterised)     SHOULD
  recorded/      llm cache for the fixtures (committed)   MUST (text model) / SHOULD (vision)
scripts/
  gen-fixtures.ts     npm run fixtures:gen
  eval-extraction.ts  npm run eval:extraction [--live] [--real <dir>]
docs/eval/extraction.latest.{json,md}   generated, committed
```

Imports: `src/extract/**` may import `src/engine/{types,tree}` (`checkTree`, `attainable`) and `src/lib/llm.ts`; nothing here imports the DB. Subsystem 4 persists what `extractPaper` returns (§12.4).

## 2. Measured evidence (spikes, M4 16 GB, Node 25.9, Ollama local)

| # | What | Result |
|---|---|---|
| 1 | `pdfjs-dist` 6.3.289 + `@napi-rs/canvas` 1.0.9, 26-page real paper | text 147 ms, PNG at 110 dpi 1 402 ms (54 ms/page), 0 errors; 11-page paper 917 ms total |
| 2 | `mupdf` 1.28.1, same paper | text 88 ms, PNG 703 ms — faster, but AGPL-3.0-or-later; `asText` puts margin marks on their own line |
| 3 | `pdftotext -layout` via `execFileSync` | 191–777 ms per paper; same row counts as pdfjs on all 9 papers |
| 4 | Next 15.5.25 route handler, `serverExternalPackages: ["pdfjs-dist","@napi-rs/canvas"]` | `next build` passes; first call 6.2 s (compile), second 224 ms |
| 5 | Text path (§6 + §8) on 4 real IB mock papers and 5 synthetic | attainable = printed max on **9/9** (50, 60, 80, 30; 80 ×5); parse 6–9 ms; lines 87–186 ms per paper |
| 6 | `qwen3-vl:8b-instruct`, one page per call, JSON `format` | real page @110 dpi: 68–108 s (1 638–1 656 prompt tok, 250–363 out tok); real cover @110: 98 s; D-I cover with a terser prompt: 50.6 s; same page @150 dpi: 161 s (2 616 prompt tok); dense synthetic page (28 rows) @150: 201–223 s (1 050–1 107 out tok); rough-work page: 47 s. Output ≈ 6.5 tok/s |
| 7 | Vision accuracy on those pages | cover: max mark 50 + both section rules right; `(b)(i)` fused label read as one row; marks `[0]` invented on a page printing no per-part marks |
| 8 | Regex pick rule, 22 sentences (16 invented + 6 from real papers) | 19/22; misses: "Question 1 and not more than four other", "three in all, at least one from each section", "FOUR, choosing TWO from each section" |
| 9 | `qwen2.5:7b-instruct` pick rule, 20 sentences | 18/20; 3.1–3.6 s per sentence warm, 14.4 s cold; misses: `out_of` null for "remaining six"; "not more than four other" → 5 |
| 10 | Topic mapping, 24 leaves of one fixture, batch 8 | with stem context **23/24**, 15–22 s per batch (400–712 prompt tok, 111–118 out tok); without context 20/24 (the three extra misses are all case-study parts) |
| 11 | Text-layer stats | born-digital content pages: alnum ≥ 59; blank / ruled-only real pages: alnum = 2 (the page number); rasterised fixture: alnum = 0; real maths pages carry up to 3 % private-use glyphs — nonLatin is **not** a rejection criterion |
| 12 | Pixel text-ink (% of page area, 4 % border excluded, runs ≥ 40 px removed) | blank 0.006, ruled-only 0.006, rough-work 0.28, cover 1.9, question pages 1.8–3.7, continued-question page with mostly ruled lines 0.995; 6–36 ms per page |

Spike sources: `$SCRATCH/spikes/extraction/{pdfspike,nextspike,fixturespike,textmodel,vision,real-text,real-png,tuning}`.

## 3. PDF → pages (`src/ingest/pdf.ts`) — MUST

```ts
export interface TextItem { s: string; x: number; y: number; w: number; h: number }   // PDF points, origin bottom-left
export interface Line { text: string; bbox: BBox; items: TextItem[] }
export interface BBox { x: number; y: number; w: number; h: number }                  // fractions of the page box, origin top-left
export interface PageText { page: number; width: number; height: number; lines: Line[] }
export interface PdfHandle { pageCount: number; pageLines(n: number): Promise<PageText>; renderPage(n: number, dpi: number): Promise<Buffer /* PNG */>; close(): Promise<void> }
export async function openPdf(bytes: Uint8Array): Promise<PdfHandle>;   // throws PdfError { code: "not-pdf" | "encrypted" | "no-pages" | "too-many-pages" }
export const MAX_PAGES = 60;
```

`pageLines` — line clustering (verified on 9 papers):

1. Keep items with non-empty `str`; `h = |transform[3]| || height || 10`.
2. `med` = median `h`. Items with `h ≥ 0.8·med` are *body*; the rest (sub/superscripts, fraction parts) are *small*.
3. Body items sorted by `y` desc then `x` asc; an item joins the last line whose `|y − line.y| ≤ 0.35·max(h, line.h)`, else starts a line.
4. Each small item attaches to the nearest line within `0.9·line.h`, else becomes its own line.
5. Within a line, sort by `x`; between consecutive items insert four spaces if the gap `> 2.5·h`, one space if `> 0.12·h`, else nothing. Trim the right end.
6. `bbox` = union of the line's items, converted to fractions with `y` flipped (`1 − (y + h)/pageHeight`).

`renderPage`: `getViewport({ scale: dpi/72 })`, white-filled canvas, `page.render({canvasContext, viewport, canvas})`, `toBuffer("image/png")`. Load with `useSystemFonts: true`, `standardFontDataUrl`, `cMapUrl` + `cMapPacked`, `verbosity: 0`. Page images for the viewer are rendered at **150 dpi** once per paper into `data/pages/<paperId>/<n>.png` (subsystem 4 serves them); vision input is a separate 110 dpi render, made only for pages routed to vision.

## 4. Routing a page (`textlayer.ts`, `pixels.ts`, `route.ts`) — MUST

```ts
export interface TextLayerStats { alnum: number; words: number; vowelWordRatio: number; badRatio: number }
export function textLayerStats(lines: string[]): TextLayerStats;
export function textLayerUsable(s: TextLayerStats, minAlnum: number /* setting text_layer_min_chars */): boolean;
export interface InkStats { textInkPct: number; longRunFrac: number }
export function inkStats(png: Buffer): Promise<InkStats>;
export const INKLESS_BELOW_PCT = 0.35;
export type PageRoute = "text" | "vision" | "skip";
export async function routePage(pdf: PdfHandle, n: number, minAlnum: number): Promise<{ route: PageRoute; stats: TextLayerStats; ink: InkStats | null }>;
```

- `textLayerStats`: join lines; strip runs of `[_.\-–—…·]{4,}` (ruled answer lines); `alnum` = count of `\p{L}|\p{N}`; `words` = tokens matching `^[A-Za-z]{3,}$`; `vowelWordRatio` = share of those containing a vowel; `badRatio` = share of non-space chars that are U+FFFD, C0 controls or private-use.
- `textLayerUsable` = `alnum ≥ minAlnum && words ≥ 4 && vowelWordRatio ≥ 0.6 && badRatio ≤ 0.05`. Default `text_layer_min_chars` = **20** (§17: subsystem 2's proposed 200 would send real continued-question pages with 59 alnum chars to a two-minute vision call).
- `inkStats`: luminance `< 128` = dark; ignore a 4 % border; horizontal dark runs `≥ 40 px` are rules, not text; `textInkPct = 100·(dark − runPixels)/area`.
- `routePage`: usable text → `text`. Otherwise render at 110 dpi, and `textInkPct < INKLESS_BELOW_PCT` → `skip` (blank, ruled, rough work) else `vision`. A usable text layer never goes to vision.

## 5. The ROW (`src/extract/rows.ts`) — MUST

```ts
export const ROW_KINDS = ["meta","header","instruction","question","part","or","continuation","marks","text","noise"] as const;
export type RowKind = (typeof ROW_KINDS)[number];
export interface Row {
  id: string;                       // `${page}:${seq}`, seq from 1 per page
  page: number;                     // 1-based
  kind: RowKind;
  label: string | null;             // header "A"; instruction: section letter or null; question "4"; part "b" | "ii" (lower case, no brackets); continuation: question number or null
  level: 1 | 2 | null;              // part only: 1 letter, 2 roman
  text: string;                     // ≤ 400 chars, may be ""
  marks: number | null;             // integer 0..200
  marksKind: "leaf" | "question_total" | "paper_max" | "each" | null;
  source: "text" | "vision";
  confidence: number;               // text 1; vision 0.7, or 0.5 when a vision part/question has marks null
  bbox: BBox | null;                // text rows only
  afterOr: boolean;                 // this part/question row directly follows an `or` row
  scopeHint: "question" | null;     // an instruction printed on a question's own line ("Write short notes on any three")
}
```

Kinds: `meta` = a line stating the paper's maximum (`marksKind: "paper_max"`); `header` = "Section B"; `instruction` = an "answer N" sentence; `question` / `part` = starts of numbered / lettered items; `or` = a line that is only "OR"; `continuation` = "(Question 4 continued)" / "(This question continues…)"; `marks` = marks printed alone on a line; `text` = stem/body text (first paragraph of a stimulus kept, ≤ 400 chars); `noise` = recognised and ignored (answer rules, page numbers, stamps, "turn over"). The text parser does not emit `noise`; vision maps anything unknown to it. The assembler ignores `noise` and `text` outside a question.

zod (the runtime truth):

```ts
export const BBoxSchema = z.object({ x: z.number().min(0).max(1), y: z.number().min(0).max(1), w: z.number().min(0).max(1), h: z.number().min(0).max(1) });
export const RowSchema = z.object({
  id: z.string().regex(/^\d+:\d+$/), page: z.number().int().min(1), kind: z.enum(ROW_KINDS),
  label: z.string().max(4).nullable(), level: z.union([z.literal(1), z.literal(2)]).nullable(),
  text: z.string().max(400), marks: z.number().int().min(0).max(200).nullable(),
  marksKind: z.enum(["leaf","question_total","paper_max","each"]).nullable(),
  source: z.enum(["text","vision"]), confidence: z.number().min(0).max(1),
  bbox: BBoxSchema.nullable(), afterOr: z.boolean(), scopeHint: z.literal("question").nullable(),
}).strict();
export const RowsSchema = z.array(RowSchema);
```

JSON schema (`ROW_JSON_SCHEMA`, exported for docs and for the cache file validator): `{ "type":"object", "additionalProperties":false, "required":[all 12 keys], "properties": { "id":{"type":"string","pattern":"^\\d+:\\d+$"}, "page":{"type":"integer","minimum":1}, "kind":{"enum":[…ROW_KINDS]}, "label":{"type":["string","null"],"maxLength":4}, "level":{"enum":[1,2,null]}, "text":{"type":"string","maxLength":400}, "marks":{"type":["integer","null"],"minimum":0,"maximum":200}, "marksKind":{"enum":["leaf","question_total","paper_max","each",null]}, "source":{"enum":["text","vision"]}, "confidence":{"type":"number","minimum":0,"maximum":1}, "bbox":{"oneOf":[{"type":"null"},{bbox object, 4 numbers 0..1}]}, "afterOr":{"type":"boolean"}, "scopeHint":{"enum":["question",null]} } }`. A test feeds 30 recorded rows and 10 malformed ones through both and asserts identical accept/reject.

## 6. Text path — MUST

### 6.1 `parseTextRows(pages: PageText[], opts): { rows: Row[]; bareMode: boolean; pageStats: PageReadStats[] }`

Pre-pass over the whole paper: `hasBracketMarks` = at least 3 line-ends match `[\[\(]\s*\d{1,3}\s*(marks?)?\s*[\]\)]\s*$`; `bareMode` = the text contains `figures? (to|on|in) the right` **or** (`!hasBracketMarks` and ≥ 5 lines end in `\s{3,}\d{1,2}\s*$`). In bare mode a right-margin number is a mark; otherwise it never is.

Regexes (`RE`, exported for tests; all applied to a line NFKC-normalised with NBSP → space, leading bullets stripped):

| name | pattern (source) | note |
|---|---|---|
| answerLine | `^[\s_.\-–—…·]{6,}$` | drop |
| pageNo | `^\s*(?:[-–~]\s*\d{1,3}\s*[-–~]\|\d{1,3}\|page\s+\d+(\s+of\s+\d+)?)\s*$` (i) | drop |
| stamp | `SYNTHETIC FIXTURE` (i) | drop |
| turnOver | `^\s*(turn over\|p\.?t\.?o\.?\|please turn over)\s*$` (i) | drop |
| paperMax | `\b(?:maximum\|max\.?\|total\|full)\s+marks?\b[^\d\[\(]{0,40}[\[\(]?\s*(\d{2,3})\b` (i) | only before the first question |
| qMax | `\[\s*(?:maximum\|max\.?\|total)\s+marks?\s*:?\s*(\d{1,3})\s*\]` (i) | `[Maximum mark: 14]` |
| section | `^\s*(section\|part\|group)\s*[-–:]?\s*([A-H]\|[IVX]{1,4}\|\d)\b\s*[:.\-–]?\s*(.*)$` (i) | header when rest ≤ 40 chars; instruction when rest matches `instruction` |
| instruction | `\b(answer\|attempt\|solve\|write\|choose\|do)\b[^.]{0,60}?\b(all\|any\|both\|either\|only\|NUM)\b\|\b(is\|are)\s+compulsory\b` (i) | NUM = `\d{1,2}\|one…twelve` |
| continuation | `^\s*\(?\s*(?:this\s+)?question\s*(\d{1,2})?\s*(?:is\s+)?continue[sd]?\b` (i) | |
| or | `^\s*[-–—(*]*\s*OR\s*[-–—)*]*\s*$` | case-sensitive |
| question | `^\s*(?:Q(?:uestion\|ues)?\s*\.?\s*(?:No\.?\s*)?)?(\d{1,2})\s*[.):](?!\d)\s*` (i) and `^\s*Q\s*\.?\s*(\d{1,2})\b\s*[.):]?\s*` (i) | accepted only if `n === lastQ + 1`, or `n === lastQ` right after an OR |
| partLetter | `^\s*\(([a-h])\)\s*` and `^\s*([a-h])[.)]\s+(?=\S)` | level 1 |
| partRoman | `^\s*\((x\|ix\|iv\|vi{0,3}\|i{1,3})\)\s*` (i) and `^\s*(x\|ix\|iv\|vi{0,3}\|i{1,3})[.)]\s+(?=\S)` | level 2; `(i)` is a letter only when the previous letter part was `(h)` |
| marksBracket | `[\[\(]\s*(\d{1,3})\s*(?:marks?\|m\|pts?\|points?)?\s*[\]\)]\s*\.?\s*$` (i) | `[2]`, `(5 marks)`, `[2 marks]` |
| marksWord | `(?:^\|\s)(\d{1,3})\s*marks?\s*\.?\s*$` (i) | `… 10 marks` |
| marksBare | `\s{3,}(\d{1,2})\s*$` | bare mode only |
| each | `each\s+(?:\w+\s+){0,3}?carr(?:y\|ies)\s+(\d{1,2})\s+marks?\|(\d{1,2})\s+marks?\s+each` (i) | `marksKind: "each"` |

Per line, first match wins, in this order: drop-patterns → `paperMax` (before the first question; not on a `[Maximum mark: n]` line) → `continuation` → `or` (sets `afterOr` for the next question/part) → `section` → `question` (monotonic filter; `qMax` on the same line → `question_total`; `each` → an `each` marks row; an `instruction` match in the rest → an extra `instruction` row with `scopeHint: "question"`; a part label on the same line → the question keeps text "" and parsing continues on the rest) → `part` (up to two labels on one line: `(b)(i)` → part `b` with text "" then part `i`) → `instruction` (line < 220 chars) → `marks` alone on a line → otherwise **continuation text**: appended to the open question/part (`open` = the last question/part row that has no marks yet; a trailing mark on the continuation line closes it), else appended to the last `text` row on the same page, else a new `text` row.

Row `confidence` is 1. `bbox` is the line's bbox (union of the lines when a row spans several). Marks on their own line (`marks` row) bind in the assembler (§8), not here.

### 6.2 Pick rule (`pick-rule.ts`)

```ts
export interface PickRule { pick: number | "ALL"; of: number | null; compulsory: string[]; unit: "questions" | "parts";
  sectionLabel: string | null; extraConstraint: boolean; resolvedBy: "regex" | "model"; text: string; page: number }
export function parsePickRule(text: string): Omit<PickRule, "text"|"page"> | null;          // deterministic; null = not a rule
export const CONSTRAINT_WORDS = /\b(at least|from each|not more than|choosing|selecting|remaining|in all)\b/i;
export async function resolvePickRule(row: Row, llm: LlmProvider, model: string): Promise<PickRule | null>;
```

`parsePickRule` (spike-verified 19/22): `sectionLabel` from `\bsection\s+([A-H])\b`; `compulsory` from every `Q(uestion)?\s*\.?\s*(No\.?\s*)?(\d{1,2})\s+is\s+compulsory`; `any` = `\b(any|only|exactly|at least)\s+NUM\b(…\b(out of|of the( following)?|from( the)?( remaining)?)\s+NUM\b)?`; `verbNum` = `\b(answer|attempt|solve|choose|do|write)\s+(short notes on\s+)?NUM\s+(question|part|of|from|note)`. Then: `(all|both)` without `any`/`verbNum` → `ALL` (`both` → `of: 2`); `any` → `pick`, `of`; `verbNum` → `pick`; `either` → 1; compulsory only → `ALL`; else null. `unit` = `parts` when the text has `short notes`, `part(s)` or `of the following`. `extraConstraint = CONSTRAINT_WORDS.test(text)`.

`resolvePickRule`: `r = parsePickRule(row.text)`. If `r && !r.extraConstraint` → return it (`resolvedBy: "regex"`). Otherwise call the text model with `PICKRULE_PROMPT_V1` (the spike's system prompt verbatim, with its five examples) and schema

```json
{"type":"object","required":["kind","count","out_of","compulsory","extra_constraint"],"properties":{
 "kind":{"type":"string","enum":["answer_all","answer_count","not_a_rule"]},
 "count":{"type":["integer","null"],"minimum":1,"maximum":30},"out_of":{"type":["integer","null"],"minimum":1,"maximum":40},
 "compulsory":{"type":"array","items":{"type":"string"},"maxItems":6},"extra_constraint":{"type":"boolean"}}}
```

`options { temperature: 0, num_ctx: 2048, num_predict: 120, seed: 7 }`, timeout `text_timeout_s` (60). `not_a_rule` → null. `answer_all` → `ALL`; `answer_count` → `count`. `resolvedBy: "model"`. On error/timeout after one retry → return `r` if it exists (with `extraConstraint`) else null. A rule with `extraConstraint: true` is applied as if the constraint were absent **and** raises `cross-group-rule` (§9), because the engine cannot represent it (01-engine §5.3).

## 7. Vision path (`vision.ts`) — MUST (the eval of it is SHOULD)

```ts
export const VISION_PROMPT_VERSION = "vision.v1";
export async function visionRows(png: Buffer, page: number, llm: LlmProvider, model: string, timeoutMs: number): Promise<{ rows: Row[]; pageType: VisionPageType; stats: LlmStats } | { failed: true; reason: string }>;
```

Exact system prompt (`VISION_PROMPT_V1`) — the spike-C prompt with two changes (12-word texts; no invented marks):

```
You transcribe the STRUCTURE of one page of an examination question paper into rows. You never answer questions, never solve anything, and never add content that is not printed on the page.

page_type: "cover" (title page and/or general instructions), "questions" (contains question or part text), "answer_space" (only ruled lines, boxes or empty space for answers or rough work), "blank" (nothing printed; a page number alone counts as blank), "reference" (formula sheet, data booklet, tables only), "other".

Row kinds, in reading order from top to bottom:
- "meta": a line stating the maximum or total marks of the whole paper. Put the number in "marks".
- "header": a section heading that literally starts with Section, Part or Group. label = the letter or number only. Table and figure captions are NOT headers.
- "instruction": a sentence telling candidates how many questions or parts to answer. Copy it exactly into "text". If it names a section ("Section B: answer one question") label = that letter, else null.
- "question": the start of a numbered question ("1.", "Q3", "Q.5", "5)"). label = the number only. text = the first 12 words after the number (may be empty). marks = the marks printed at the end of the question's text if it has no lettered parts, else null.
- "part": a lettered or roman sub-part ("(a)", "b.", "c)", "(ii)"). label = the letter or numeral only, lower case, no brackets. text = the first 12 words. marks = the marks printed at the end of that part, else null. If a line starts with two labels such as "(b)(i)", output two rows: a part row for "b" with empty text, then a part row for "i".
- "or": a line that contains only the word OR between two alternatives.
- "continuation": "(Question 3 continued)" or "(This question continues on the following page)". label = the question number if printed, else null.
- "text": the first paragraph of a case study or stimulus only. text = its first 12 words. Skip every other body paragraph, table and caption.

Marks appear as "[2]", "[2 marks]", "(5 marks)", "10 marks", or a bare number such as "07" alone in the right-hand margin. Report a mark only where one is printed for that row. Never guess, add up or compute marks; if none is printed, marks = null.
Do not output rows for ruled answer lines, page numbers, running headers or footers, logos or graphs.
```

User message: `Transcribe the structure of this page.` with `images: [base64 png]`. `format`:

```json
{"type":"object","required":["page_type","rows"],"properties":{
 "page_type":{"type":"string","enum":["cover","questions","answer_space","blank","reference","other"]},
 "rows":{"type":"array","maxItems":60,"items":{"type":"object","required":["kind","label","text","marks"],"properties":{
   "kind":{"type":"string","enum":["meta","header","instruction","question","part","or","continuation","text"]},
   "label":{"type":["string","null"]},"text":{"type":"string"},"marks":{"type":["integer","null"]}}}}}}
```

`options { temperature: 0, num_ctx: 8192, num_predict: 1500, seed: 7 }`, `keep_alive: "10m"`, **one page per call**, timeout `vision_timeout_s` (default 240; measured 47–223 s). Retry **once** on network error, HTTP ≥ 500, non-JSON content, zod failure or `done_reason: "length"`; then return `failed` and the pipeline raises `vision-page-failed` for that page and continues. Mapping to `Row`: `page_type ∈ {answer_space, blank}` → no rows; `meta` → `marksKind: "paper_max"`; `part` label → `level` 2 if roman (and not a letter by the §6.1 rule) else 1; `afterOr` set when the previous row was `or`; unknown kind → `noise`; `bbox: null`; `confidence` 0.7, or 0.5 for a question/part with `marks: null`. Vision rows are never mixed with text rows on the same page; a paper may mix pages.

## 8. Assembler (`assemble.ts`) — MUST

```ts
export interface AssembledPaper { root: Group; statedMaxMarks: number | null; statedMaxPage: number | null; meta: Map<NodeId, NodeMeta>; flags: ExtractionFlag[] }
export interface NodeMeta { rowIds: string[]; source: "text" | "vision" | "mixed"; confidence: number; bbox: BBox | null; stem: string | null; marksInferred: boolean; rule: PickRule | null }
export function assemble(rows: Row[], rules: Map<string /* row.id */, PickRule | null>): AssembledPaper;
```

Works on a mutable draft node `{ role: "paper"|"section"|"question"|"part"|"or"|"block"|"choice", label, page, pick, children, marks, printedTotal, eachMarks, text, stem, rule, implicit }`, then finalises into the engine's `Group`/`Leaf`.

**State**: `sec`, `q`, `part`, `sub` (current section / question / level-1 / level-2 draft or null), `pendingOr`, `lastMarkable`, `labelledRules: Map<letter, PickRule>`, `paperRule`, `statedMax`. Transitions, one per row kind, in row order:

| row | action |
|---|---|
| `meta/paper_max` | first one → `statedMax = {marks, page}`; a different later value → `conflicting-stated-max` |
| `header` | `sec = section(label, page)`; push to paper; `q = part = sub = null`; `pendingOr = false` |
| `instruction` | `rule = rules.get(row.id)`; null → `instruction-unparsed`, continue. Bind: `scopeHint === "question" && q` → `q.rule`; else `row.label ?? rule.sectionLabel` → `labelledRules[label] ??= rule`; else `sec && !sec.implicit && sec.children.length === 0` → `sec.rule`; else `!sec && !q` → `paperRule ??= rule`; else `q && rule.unit === "parts" && q.children.length === 0` → `q.rule`; else `instruction-unbound` |
| `question` | ensure a section (implicit `_` if none); `node = question(label, page, {text, marks: marksKind==="leaf" ? marks : null, printedTotal: marksKind==="question_total" ? marks : null, eachMarks})`; `pendingOr && prev` → `orJoin(sec, prev, node)` else push; `q = node; part = sub = null; lastMarkable = node` |
| `part` level 1 | no `q` → make `question("?")` + `orphan-part`. `sibs = q.children`, `prev = last sib`. If `pendingOr && prev`: when `row.label` equals the first sibling's label and there is more than one sibling → **block alternative** (`(a)(b) OR (a)(b)`): move existing sibs into `block alt1`, start `block alt2`, wrap both in `or(pick 1)`; otherwise `orJoin(q, prev, node)`. Else if `q._altBlock` push there, else push. `part = node; sub = null; lastMarkable = node` |
| `part` level 2 | `parent = part ?? q` (no `part` → `subpart-without-part`); `pendingOr && prev` → `orJoin(parent, prev, node)` else push; `sub = node; lastMarkable = node` |
| `or` | `pendingOr = true` |
| `marks` | `lastMarkable` with no marks and no children → set its marks; else create `part("?", marks)` under `sub ? part : part ? q : q ?? sec` + `orphan-marks` |
| `continuation` | `row.label && q && row.label !== q.label` → `continuation-mismatch` (info; the real 2024 BM paper prints "Question 4 continued" inside Q5) |
| `text` | `q` → `q.stem += text` (≤ 600 chars) else ignored |

`orJoin(parent, prev, next)`: if `prev.role === "or"` push `next` into it; else replace `prev` in `parent.children` with `or(label: prev.label, page: prev.page, pick 1, children [prev, next])`.

**Finalise** (`toTree`, bottom-up) in this order:

- F1 *shared-mark hoist*: a non-section group with no own marks and no `printedTotal`, all children leaves, and exactly the **last** child carrying marks → one leaf with those marks and the joined texts; `shared-marks-hoisted` (info). (Maths papers: `(a)(i)… (ii)… [4]`.)
- F2 *own marks kept as part*: a group with own marks, marked children, and a first child whose label is not `a`/`i` → the own line becomes a leaf labelled `inline` inserted first; `own-marks-kept-as-part` (info). (Real BM Q5(c): "(i) …, and (ii) … [2]" then "(iii) … [2]".)
- F3 a group with own marks and no marked children → a leaf with the own marks and joined texts. A group with own marks **and** marked children → `printedTotal = marks`.
- F4 `rule` on a node → `pick = rule.pick`; `eachMarks` fills every unmarked leaf child (`marksKind: "each"` from "Each note carries 5 marks").
- F5 section rules: `rule = s.rule ?? labelledRules[s.label] ?? (s.implicit ? paperRule : null)`. A section holding a single question when the rule is per-parts or `pick > 1` → the rule moves onto that question (conflict → `conflicting-rules`). `compulsory` non-empty with `pick ≠ ALL` → `[compulsory questions…, choice(pick, rest)]`. Otherwise `s.pick = rule.pick`. `rule.of` ≠ child count → `pool-mismatch` (info). No rule and > 1 child → `pick = ALL` + `no-instruction-default-all` (warn; this is the brief's "default pick ALL + flag").
- F6 *marks inference* (essay papers): `statedMax` known, exactly one section, every child a leaf without marks, `pick ≠ ALL`, `statedMax / pick` an integer → each leaf gets it, `marksInferred: true`, `marks-inferred` (warn). Verified on the real English paper (30 / pick 1 → 30 each).

**Engine shape and ids** (`ids.ts`): root `Group { id: "root", label: "", pick: "ALL", page: 1, instruction: null, statedMarks: null }`. Section → `Group { id: "s:B", label: "Section B", pick, instruction: rule.text ?? null }`; an implicit section is skipped (its children hang from the root) unless it carries a rule. Question → `Group { id: "q:4", label: "Q4", statedMarks: printedTotal }` or `Leaf { id: "q:4", label: "Q4" }`. Part → `q:4.b`, label `4(b)`; sub-part `q:4.b.ii`, label `4(b)(ii)`; OR group → `q:4.or` label `Q4` (or `q:4.b.or` label `4(b)`), `instruction: "(a) OR (b)"`; blocks `q:4.alt1` / `q:4.alt2` label `4 (first alternative)`; choice → `s:B.rest`, label `Section B (the rest)`; `?`-labelled nodes → `q:?.<seq>`. `Leaf.page` = the page of the row that carried the marks; `Leaf.excerpt` = text ≤ 160 chars; `Leaf.mapping = { state: "unknown" }` until §10 runs; `Leaf.marks` = 0 when still null (the validator flags it). `NodeMeta.bbox` = union of the node's rows' bboxes (SHOULD; enables the confirm screen's highlight band).

**Multi-page questions**: the state machine carries `q`/`part` across pages, so `(Question 2 continued)` pages simply keep appending; a `continuation` row never opens or closes anything. Verified: the real BM paper's Q2 spans pp. 4–7 and Q4 pp. 11–14 and both reconcile.

## 9. Validators (`validate.ts`) — MUST

```ts
export interface ExtractionFlag { code: string; fatal: boolean; scope: "paper" | "row"; nodeIds: NodeId[]; expected: number | null; actual: number | null; detail: string; cite: Cite | null }
export function validate(a: AssembledPaper, paperId: PaperId, knownTopics: ReadonlySet<TopicId>): ExtractionFlag[];
```

Exact definitions (`attainable` and `checkTree` are the engine's):

| code | fatal | scope | condition | expected / actual |
|---|---|---|---|---|
| `total-mismatch` | no | paper | `statedMaxMarks !== null && attainable(root)/UNITS_PER_MARK !== statedMaxMarks` | stated / attainable; cite = stated-max page |
| `no-stated-total` | no | paper | `statedMaxMarks === null` | null / attainable |
| `group-marks-mismatch` | no | row | group with `statedMarks !== null` and `attainable(group) ≠ statedMarks` | statedMarks / attainable |
| `pick-exceeds-children` | no | row | `pick` numeric `> children.length` | pick / count |
| `numbering-gap` | no | row | question labels (numeric, first occurrence order) are not `1..n` contiguous; one flag per gap, `nodeIds[0]` = the question after the gap | expected number / found |
| `zero-mark-leaf` | no | row | leaf with marks 0 (unset) | null / 0 |
| `rows-without-page` | no | paper | rows with `page < 1` (only possible from a corrupt cache) | null / count |
| `vision-page-failed` | no | paper | pages routed to vision that failed twice; `detail` lists them | null / count |
| `no-instruction-default-all`, `instruction-unparsed`, `instruction-unbound`, `cross-group-rule`, `marks-inferred`, `orphan-part`, `orphan-marks`, `subpart-without-part`, `conflicting-rules`, `conflicting-stated-max`, `pool-mismatch`, `continuation-mismatch`, `shared-marks-hoisted`, `own-marks-kept-as-part` | no | row (paper for the first three when unbound) | raised by the assembler as listed in §8 | as noted |
| everything else from `checkTree` (`pick-zero`, `pick-invalid`, `empty-group`, `invalid-marks`, `duplicate-node-id`, `dangling-topic`, `total-too-large`) | as the engine says | row | pass-through, de-duplicated by `(code, nodeId)` against the rows above | null |

No validator blocks. Flags are data on the paper (brief §4.5). Codes not in the experience spec's table render there through its generic branch (code in caps + `detail`). `detail` sentences are written for that branch: e.g. `no-instruction-default-all` → `Section B: no "answer N" instruction was found, so every question is treated as compulsory. Set the choice if the paper offers one.`

## 10. Topic mapping (`topics.ts`) — MUST

```ts
export const TOPICS_PROMPT_VERSION = "topics.v1";
export interface TopicProposal { nodeId: NodeId; mapping: LeafMapping; confidence: "high" | "medium" | "low" | null; failed: boolean }
export async function mapTopics(leaves: { leaf: Leaf; stem: string | null }[], topics: { id: TopicId; name: string }[], llm: LlmProvider, model: string, timeoutMs: number, onProgress?: (done: number, of: number) => void): Promise<TopicProposal[]>;
```

- Leaves are sent in **batches of 8** in document order; each item is `{ id: "i1".."i8", text: excerpt ≤ 300 chars, context?: stem ≤ 400 chars }`. `context` is present only for leaves under a question whose `NodeMeta.stem` is non-empty (case studies): the part "Explain one disadvantage." is unmappable without it (§2 row 10). Short-note titles ("Bitmap indexes") map at 100 % without context.
- System prompt `TOPICS_PROMPT_V1` (spike verbatim): the topic list as `- id: name` lines plus `- none_of_these: the item is not about any topic in the list`; confidence definitions high / medium / low; the context rule; "Return exactly one result per item, in the same order, using the same ids." User message: `JSON.stringify({ items })`.
- `format`: `{"type":"object","required":["results"],"properties":{"results":{"type":"array","minItems":n,"maxItems":n,"items":{"type":"object","required":["id","topic","confidence"],"properties":{"id":{"type":"string"},"topic":{"type":"string","enum":[...topicIds,"none_of_these"]},"confidence":{"type":"string","enum":["high","medium","low"]}}}}}}` — the enum is rebuilt per course, so the model cannot name a topic that does not exist. `options { temperature: 0, num_ctx: 4096, num_predict: 60·n + 40, seed: 7 }`, timeout `text_timeout_s`.
- Result → `mapping`: topic id → `{ state: "mapped", topicIds: [id], confirmed: false }`; `none_of_these` → `{ state: "none", confirmed: false }`; missing or unknown `id` → `{ state: "unknown" }`, `failed: true`. A batch that fails twice → all its leaves `unknown`; the pipeline raises `mapping-incomplete` (paper scope, `actual` = count). No leaf is ever mapped to a topic outside the enum (zod re-checks).
- Storage (subsystem 4): the `LeafMapping` on the leaf; `confidence` in `NodeMeta`. The confirm screen shows `mapping…` while `pending` and the pencil select afterwards; D-E's "Confirm the N rows with no flag" confirms structure and mapping together.

## 11. `src/lib/llm.ts` — MUST

```ts
export interface ChatRequest { model: string; system: string; user: string; images?: Buffer[]; format: object; options: { temperature: 0; num_ctx: number; num_predict: number; seed: number }; keepAlive?: string; timeoutMs: number; promptVersion: string }
export interface LlmStats { totalMs: number; loadMs: number; promptTokens: number; promptMs: number; outputTokens: number; outputMs: number; doneReason: string; fromCache: boolean }
export type ChatResult = { ok: true; content: string; stats: LlmStats } | { ok: false; error: "timeout" | "network" | "http" | "empty"; status?: number; detail: string };
export interface LlmProvider { readonly name: "ollama" | "mock" | "recorded"; chat(req: ChatRequest): Promise<ChatResult>; status(): Promise<{ reachable: boolean; models: string[] }> }
export async function chatJson<T>(p: LlmProvider, req: ChatRequest, schema: z.ZodType<T>): Promise<{ ok: true; value: T; stats: LlmStats } | { ok: false; error: string }>;   // one retry on any failure, then error
export function ollamaProvider(baseUrl: string): LlmProvider;           // POST {baseUrl}/api/chat, stream:false, messages [system,user{images}], format, options, keep_alive; never sends `think`
export function mockProvider(script: Record<string, string | ((req: ChatRequest) => string)>): LlmProvider;   // key = promptVersion; used by every unit test
export function recordedProvider(inner: LlmProvider, dirs: { readOnly: string[]; writable: string }): LlmProvider;
export function requestKey(req: ChatRequest): string;   // `${model}/${promptVersion}/${sha256(canonical JSON of {system,user,images: sha256 each,format,options})}`
export function providerFromSettings(s: { llm_provider: string; ollama_base_url: string }): LlmProvider;   // ollama|mock, always wrapped in recordedProvider([fixtures/synthetic/recorded], data/cache/llm)
```

`recordedProvider.chat`: look up `<dir>/<model>/<promptVersion>/<hash>.json` in each read-only dir then the writable one; a hit returns `{ content, stats: {…, fromCache: true, totalMs: 0} }` without touching `inner`; a miss calls `inner` and, on `ok`, writes `{ request: {model, promptVersion, system, user, imageHashes, format, options}, response: content, stats, recordedAt }` to the writable dir. Cache files are validated with zod on read; a corrupt file is ignored and overwritten. `ollamaProvider` uses `AbortSignal.timeout(req.timeoutMs)`; loopback only is enforced by subsystem 2's `ollama_base_url` pattern. `mock` fails on a missing key so a test cannot silently hit a model. `status()` = `GET /api/tags` with a 5 s timeout.

## 12. Pipeline (`pipeline.ts`) — MUST

```ts
export interface ExtractOptions { paperId: PaperId; topics: { id: TopicId; name: string }[]; settings: { text_layer_min_chars: number; vision_render_dpi: number; vision_timeout_s: number; text_timeout_s: number; vision_model: string; text_model: string }; llm: LlmProvider; pagesDir: string | null /* 150 dpi PNGs written here */ }
export type Progress = { kind: "reading"; page: number; pageCount: number; mode: PageRoute } | { kind: "assembling" } | { kind: "mapping_topics"; done: number; of: number } | { kind: "done" } | { kind: "failed"; reason: string };
export interface ExtractionResult { version: 1; paperId: PaperId; contentHash: string; pageCount: number; isSynthetic: boolean;
  routes: { page: number; route: PageRoute; ms: number }[]; visionPages: number[]; failedPages: number[];
  rows: Row[]; root: Group; statedMaxMarks: number | null; statedMaxPage: number | null; meta: Record<NodeId, NodeMeta>;
  proposals: TopicProposal[]; flags: ExtractionFlag[]; timings: { pdfMs: number; textMs: number; visionMs: number; assembleMs: number; mappingMs: number; totalMs: number }; llmCalls: LlmStats[] }
export async function extractPaper(bytes: Uint8Array, opts: ExtractOptions, onProgress: (p: Progress) => void): Promise<ExtractionResult>;
```

Order: hash → `openPdf` → for each page: `routePage`, then `parseTextRows` (text pages are parsed together at the end, since bare-mode is a paper-level decision) or `visionRows` (one call, sequential) or skip; render the 150 dpi viewer PNG for every page; `assemble` with rules from `resolvePickRule` on every `instruction` row (regex needs no model); `validate`; `mapTopics` over every leaf with `marks > 0` in document order; `done`. `isSynthetic` = page 1 text contains the stamp **or** `contentHash` is listed in `fixtures/synthetic/manifest.json`. Any thrown `PdfError` → `failed` with its code as the reason. Time budget on the demo path (fixture, all text, mapping from cache): under 2 s.

### 12.4 What subsystem 4 stores

`ExtractionResult` whole, as `data/cache/extract/<contentHash>.json` (so a re-upload of the same bytes is instant — the experience spec's "appears in about a second"), plus, into its tables: the `root` as the paper's tree with `structureConfirmed: false`, `statedMaxMarks/Page`, each `TopicProposal.mapping` onto its leaf, `meta` as a node side-table (`bbox`, `source`, `confidence`, `stem`), `flags`, `visionPages`, `isSynthetic`. Nothing here writes to the DB. Re-running validators after a student edit is subsystem 4 calling `validate()` on the stored tree (it takes an `AssembledPaper`; a 10-line adapter rebuilds one from the stored tree, `meta` and `statedMax`).

## 13. Synthetic fixtures (D-F) — MUST (born-digital) / SHOULD (scanned)

### 13.1 The course

`DBS-204 Database Systems`, *Fixture Institute of Technology (fictional)*. Ten topics, ids fixed:

| id | name | | id | name |
|---|---|---|---|---|
| `er` | ER modelling | | `recovery` | Recovery and logging |
| `relalg` | Relational model and algebra | | `index` | Indexing and hashing |
| `sql` | SQL | | `qopt` | Query processing and optimisation |
| `norm` | Functional dependencies and normalisation | | `nosql` | NoSQL and distributed databases |
| `txn` | Transactions and concurrency control | | `security` | Database security and authorisation |

The question bank (`course.ts`: per topic 3 short, 3 long, 1 split pair, 2 note titles; two case studies with a stem and four parts) is the spike's `bank.mjs`, copied. It is invented text about databases; nothing is quoted from any real paper.

### 13.2 The pattern (identical on all three papers; 100 marks)

```
Section A  Q1 compulsory: 5 parts (a)–(e), 5 marks each              25
Section B  "any FIVE of seven" Q2–Q8, 12 marks each                    60
           one of them is "(a) OR (b)" (12 + 12, pick 1)
           one is a split (a) 5 + (b) 7; one is a case study 3+4+3+2
Section C  Q9 "short notes on any THREE of the following", 5 notes × 5  15
```

| paper | Q1 (a)–(e) | Section B Q2…Q8 | Q9 notes | style |
|---|---|---|---|---|
| 2022 | er, relalg, sql, norm, txn | er · sql (split) · **norm OR txn** · index (case) · qopt · nosql · security | recovery, qopt, nosql, index, er | `[n]`, "1.", "(a)", instructions under headers |
| 2023 | sql, norm, txn, index, **recovery** | er · sql (split) · **norm OR qopt** · index (case) · relalg · nosql · security | recovery, txn, er, sql, qopt | `(n marks)`, "Q.n", "(a)", instructions on the cover *and* under headers |
| 2024 | er, relalg, sql, qopt, index | norm · txn (case) · **sql OR er** · index · nosql · security · qopt | recovery, norm, txn, nosql, relalg | bare right-margin `07` + "Figures to the right indicate full marks", "Qn.", "a)", `[Maximum mark: n]` per question |

Layout rules the generator enforces (and asserts): A4, Times 11 pt, marks in a 70 pt right margin; page 1 = cover with the printed maximum (`The maximum mark for this examination paper is [100 marks].` or `Maximum marks: 100`); Section A starts page 2; the case study's stem is placed so that its question crosses a page (`(This question continues on the following page)` / `(Question 5 continued)`); Section C on its own page; a last page `Space for rough work` with 26 ruled lines; **every page** prints `SYNTHETIC FIXTURE — not a real examination paper` at top and bottom in 8 pt with the paper id and `Page n of N`; the cover title says `(FICTIONAL)`; PDF `Title` = `dbs204-2023 (synthetic fixture)`, `CreationDate` fixed at 2026-01-01 so bytes and hashes are reproducible. 5–6 pages per paper. Generation ≈ 1 s per paper with `pdfkit` (MIT).

### 13.3 Files written by `npm run fixtures:gen`

`fixtures/synthetic/dbs204-YYYY.pdf`; `dbs204-YYYY.truth.json` = `{ paperId, course, year, maxMarks: 100, synthetic: true, pages, root: Group /* engine shape, ids and labels exactly as §8 would assign, every leaf with topicId, page and marks */, rules: { "s:B": {pick:5, of:7}, "q:9": {pick:3}, "q:4.or": {pick:1} } }`; `manifest.json` = `{ papers: [{ id, file, sha256, pages, truth }] , generatedBy: "fixtures/synthetic/generate.ts@<git sha>" }`; `demo-scenario.json` (§13.5). SHOULD: `dbs204-YYYY.scan.pdf` = each page rendered at 150 dpi, rotated by a seeded ±0.3–1.5°, offset ±5 px, off-white ground, Gaussian noise σ = 9, JPEG q62, re-embedded full-page — no text layer, so every page routes to vision.

### 13.4 The tuning (D-F) — arithmetic, verified by `$SCRATCH/spikes/extraction/tuning/tune.mjs`

Availability loss with the top-k rule, over all 1 024 drop sets on the three papers: the structurally safe drop sets are exactly `{nosql}`, `{security}`, `{nosql, security}`. Why: neither topic ever appears in Q1; per paper they occupy at most 2 of Section B's 7 slots (slack 2) and at most 2 of Section C's 5 notes (slack 2). Every other topic sits in Q1 of at least one paper, so dropping it loses ≥ 5 marks there.

Cost of one more drop on top of the safe core `{nosql, security}` (loss on 2022 / 2023 / 2024; cost = max):

| topic | loss | cost | why |
|---|---|---|---|
| **recovery** | 0 / **5** / 0 | **5** | only Q1(e) of 2023 (p.2) is compulsory; elsewhere it is a short note the slack absorbs |
| relalg, norm, txn | 5/12/5 · 5/5/12 · 5/5/12 | 12 | one Q1 part plus a third dropped Section B question |
| er, sql, index, qopt | 17/12/5 · 17/17/5 · 12/17/17 · 12/0/17 | 17 | Q1 part + Section B overflow on the same paper |

So there are **two safe drops and one 5-mark gamble**; with `gamble_budget_marks` raised to 5 the plan drops `recovery` too.

Expected marks with the §13.5 standing (`p_target` 80 %, dropped topics at their stated standing): study the other eight → **80 of 100 on every paper** (Q1 25×0.8 = 20; Section B best five of seven at 9.6 = 48; Section C best three at 4 = 12). Need 58 → `marksOnPaper(57.5, 100, 5)` = 58 needed, plan target 63 (D-O). Nothing studied → 40. Everything studied → 80. Min-hours plan under G = 0: 8 topics, **26 of 40 h**; under G = 5: 7 topics, 24 h, lowest paper 78. Every one of these figures is regenerated by `scripts/eval-extraction.ts --plan` from the truth files through the engine and written to `docs/eval/extraction.latest.json`; the experience spec's view-model fixtures read from there, never from this table.

### 13.5 `demo-scenario.json` (consumed by subsystem 4's `npm run seed:demo`)

```json
{ "course": "DBS-204 Database Systems", "topics": ["er","relalg","sql","norm","txn","recovery","index","qopt","nosql","security"],
  "grade": { "target": 70, "components": [ {"name":"Coursework","weight":25,"scored":34,"outOf":40}, {"name":"Midterm","weight":25,"scored":32,"outOf":40}, {"name":"Final","weight":50,"final":true,"total":100} ] },
  "hoursAvailable": 40, "ownHours": { "er":3,"relalg":3,"sql":4,"norm":4,"txn":3,"recovery":2,"index":3,"qopt":4,"nosql":3,"security":2 },
  "probe": { "answerableTarget": { "er":0.5,"relalg":0.5,"sql":0.5,"norm":0.5,"txn":0.5,"recovery":0.5,"index":0.5,"qopt":0.5,"security":0.17 }, "untested": ["nosql"], "responses": [ { "leafId": "q:1.a", "paperId": "dbs204-2022", "grade": "yes" }, "…generated: four equal-mark leaves per topic graded yes/no/partly/partly, three Section B leaves for security graded partly/no/no" ] },
  "settingsWritten": { "hours_per_topic_cold": "3", "safety_margin_pct": "5" }, "papersConfirmedAtSeed": ["dbs204-2022","dbs204-2023"] }
```

Grade arithmetic: 34/40 × 25 + 32/40 × 25 = 21.25 + 20 = 41.25; 70 − 41.25 = 28.75 of the final's 50 → **57.5 % → 58 of 100**. `p_target_pct` and `untested_answerable_pct` are left unwritten so the plan footer names them (D-G, 02-inputs §6.3).

## 14. Eval harness (`scripts/eval-extraction.ts`) — MUST (text) / SHOULD (vision)

Per fixture PDF (born-digital ×3; scanned ×3 with `--vision`), run `extractPaper` with `providerFromSettings` (recorded cache; `--live` permits misses to call Ollama), compare with `.truth.json`, and print one table:

| metric | definition |
|---|---|
| leaves | truth leaves / extracted leaves / matched by node id |
| marks exact | matched leaves with equal `marks` ÷ truth leaves |
| pick rules | groups (by id) with equal `pick` ÷ truth groups with `pick ≠ ALL`, plus root/section `ALL` groups |
| reconciles | `attainable(root) === 100` and `statedMaxMarks === 100` |
| topic mapping | matched leaves with `proposal.topicIds[0] === truth.topicId` (or `none`) ÷ matched leaves; also split by model confidence |
| flags | count by code |
| timing | ms per page by route; model calls with tokens and ms (`fromCache` marked) |

Writes `docs/eval/extraction.latest.json` `{ runAt, gitSha, node, ollama: { models }, perPaper: [...], totals, plan: {…§13.4 figures via the engine…} }` and renders `extraction.latest.md` from it. `--real <dir>` runs the real papers (never committed; no truth) and prints reconciliation and timings only, to the console. The `README` badge line and every "measured" sentence in the UI quote this file; a test fails if `docs/eval/extraction.latest.json` is older than `fixtures/synthetic/manifest.json`.

## 15. Tests (`node --import tsx --test`)

MUST:
1. `textlayer`: born-digital line set → usable; 2-char page → not; garbage-glyph page (badRatio 0.2) → not; maths PUA glyphs → usable.
2. `pixels`: three tiny synthetic PNGs (blank, ruled only, text) → inkless / inkless / not.
3. `rows`: zod and JSON schema agree on 30 recorded rows + 10 malformed.
4. `parse-text` (fixtures = page line arrays checked in as `.json`): every regex in the table against 3 positives + 2 negatives; `(b)(i)` fusion; `1.Sai Seat Covers` (no space); marks wrapped to the next line; `[Maximum mark: 14]`; bare mode on/off; `each` marks; `(This question continues…)`; a question number that repeats after OR.
5. `pick-rule`: the 22 regex sentences with expected results; `CONSTRAINT_WORDS` routing; `resolvePickRule` with `mock` returning each schema variant, and with a failing mock (falls back to regex/null).
6. `assemble`: from row fixtures → the three truth trees (deep-equal after stripping `meta`); block alternative; orphan marks; shared-mark hoist; own-marks-kept; essay inference (30 / pick 1); multi-page question; unbound instruction; no instruction → ALL + flag.
7. `validate`: each code in §9 provoked once; `checkTree` pass-through de-duplication; a 101-mark tree → `total-mismatch` with expected 100 / actual 101.
8. `topics`: `mock` script returning a valid batch, a batch with a missing id, a batch naming a topic outside the enum (zod rejects → unknown), a timeout → `failed`.
9. `llm`: `requestKey` stable under key order and image bytes; `recordedProvider` hit/miss/write/corrupt-file; `mock` throws on a missing key; `ollamaProvider` against a local `http` stub for 200 / 500 / timeout, and never sends `think`.
10. `pipeline` end-to-end on the three fixtures with the committed recorded cache: reconciles 3/3, 0 model calls (`fromCache` on every stat), < 2 s each; `isSynthetic` true; a 2-page PDF with no text and blank pixels → 0 rows, `skip` ×2, no flag but `no-stated-total`.
11. `fixtures`: `npm run fixtures:gen` is idempotent (identical sha256 on the second run); every page of every PDF contains the stamp; truth trees pass `checkTree` with no fatal issue and attainable = 100.
12. `tuning` (engine as a dependency): from the truth files, the safe drop sets are exactly the three of §13.4 and `recovery`'s cost is 5 on `dbs204-2023`.

SHOULD: vision row mapping from recorded responses (real-page shapes: fused `(b)(i)`, invented `[0]` marks → confidence 0.5, `answer_space` → no rows); scanned fixtures through the recorded vision cache reconcile ≥ 2/3; bbox present on every text-path node; `--real` smoke on the founder's papers = 4/4 reconcile (local only).

## 16. MUST / SHOULD / COULD and build order (D-H)

**MUST (≈ 2 agent-days):** §3 pdf.ts (2 h) → §4 routing (1.5 h) → §5 rows (1 h) → §6 parser + pick rule (4 h; port the spike, then the test list) → §8 assembler + §9 validators (4 h) → §11 llm.ts (2 h) → §10 topics (1.5 h) → §7 vision call + row mapping (1.5 h) → §12 pipeline (2 h) → §13 generator, truth, manifest, scenario, recorded text-model cache (3 h) → §14 harness, text metrics, `extraction.latest.*` (2 h). Total ≈ 25 h.

**SHOULD:** scanned variants + recorded vision cache + vision metrics in the harness; `bbox` on nodes; `--real` mode; block-alternative OR; `each`-marks distribution; the experience spec's `Re-read this page live` endpoint (`POST /api/papers/{id}/pages/{n}/vision`, returns rows only, never writes).

**COULD:** cross-group rules encoded as the engine's `Group(pick 1)` over legal splits; multi-topic leaves from the mapper (`topicIds.length > 1`); per-page cross-check of text rows by vision; syllabus-PDF topic extraction (explicitly out of scope this week).

## 17. Cross-subsystem

- **01-engine.** Trees are emitted in its §3 shape exactly (`Leaf.page` = marks page; OR → `Group(pick 1)`; `statedMarks` from `[Maximum mark: n]`; mapping states `mapped | none | unknown`). `validate` calls `checkTree(paper, knownTopics)` and `attainable`; if `checkTree`'s real signature differs, the adapter is one line. Numbering contiguity stays here (`numbering-gap`), as its §17 asks. Its `testkit.exampleCourse()` (U1–U7) is a unit-test fixture, not a demo course, so D-F's "no other demo course" holds; it must never be seeded.
- **02-inputs.** Registry rows owned here, confirmed or changed: `llm_provider`, `ollama_base_url`, `vision_model`, `text_model` as proposed. **Diverge:** `llm_timeout_s` (180) is replaced by two rows, `vision_timeout_s` default **240** (measured 47–223 s per page) and `text_timeout_s` default **60**; `text_layer_min_chars` default **20**, not 200, with help text `Alphanumeric characters (ruled lines removed) a page needs before its text layer is trusted; below this the page is checked for ink and may go to the vision model.`; `vision_render_dpi` default **110**, not 150 (§2 row 6). `no_topic_answerable_pct` (D-E) is subsystem 2's row; extraction only produces the `none` state it applies to. None of these five is `unconfirmedUntilSet` — they change speed, not results.
- **04-data.** Owns `npm run seed:demo` / `demo:reset`: copy `fixtures/synthetic/recorded/**` into the read-only cache path (or point `providerFromSettings` at it), load the three PDFs through `extractPaper`, apply `demo-scenario.json` (grade, hours, own hours, probe responses, the two written settings), confirm every row of 2022 and 2023, leave 2024 provisional (`--full` confirms all three). Stores what §12.4 lists. Serves `data/pages/<paperId>/<n>.png`. `updateGroup`/`updateLeaf` re-run `validate`.
- **05-experience.** Its Appendix A economics course is superseded by §13 (it invited this); the headline becomes `Stop studying 2 of 10 topics.`, the ledger `58 of 100`, `80 of 100`, `26 of 40 h`, the safe drops *NoSQL and distributed databases* and *Database security and authorisation*, the gamble *Recovery and logging* (`would have cost 5 marks on the 2023 paper (p.2)`), regenerated from `docs/eval/extraction.latest.json`. Its Q6 (bbox) is answered: text-path rows carry one (SHOULD), vision rows do not. Its `visionPages`, `fromCache`, `isSynthetic` and `PaperStatus` progress map 1:1 to `ExtractionResult` and `Progress`. Flag codes outside its table use its generic branch. Its page route serves 150 dpi as it asks; vision uses 110 dpi internally.
- **Copy rules honoured here (D-P):** flag details say `probed / not probed` never `tested`; nothing in this subsystem prints a probability; the only words about the model are `read by the vision model` and `proposed topic`.

## 18. Open questions

1. `checkTree`'s exact signature (paper vs root + known topics) — trivial adapter either way; flagging so subsystem 1 states it.
2. The gamble list under D-B: this fixture guarantees exactly one topic with cost 5 on top of the safe core. Whether the engine surfaces gambles as "cost of one more drop on top of the safe core" (what §13.4 computes) or as the G = null plan's extra drops changes nothing in the fixture but should be one sentence in 01-engine.
3. Real scanned papers were not available; the vision path's accuracy is measured only on rasterised fixtures and four real born-digital pages rendered to PNG. `extraction.latest.md` says so until a real scan is on file.
