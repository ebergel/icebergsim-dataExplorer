# Build Plan — icebergsim-dataExplorer

**A generic, local-first data workbench: import CSV / Excel / JSON / R (RDS, RData) files, explore them with the classic PivotTable.js drag-and-drop UI, and keep everything — datasets, named views, and your session — in the browser's own database.**

Handle: `ISM_DEX_R` · repo: `icebergsim-dataExplorer` · suggested subdomain when deployed: `data.icebergsim.com`.

**Profile note.** This plan is the suite's first **medium-rigor profile**, deliberately lighter than the GRT/pivot plans: no statistical oracle, no golden files, no monorepo, single-page app. What it keeps from the framework: spec written before code, a canonical hand-specified fixture, two test tiers, the PostToolUse loop, phase gates, and a (short) inviolable rule. Rigor is spent where this product can be *silently* wrong — file conversion fidelity and persistence round-trips — and nowhere else.

---

## Decisions locked in

1. **Pivot engine/UI: `react-pivottable` as an npm dependency** (not vendored), table-family renderers only — never install plotly. React **18** (known-good with the widget); `npm install --legacy-peer-deps` acceptable; `patch-package` if a small fix is unavoidable. A ~30-line local `.d.ts` shim types the untyped package.
2. **Local-first, no backend.** Static Vite build, deployable to GitHub Pages. Persistence is **IndexedDB via Dexie**. Cloud sync is explicitly out of scope; the portability story is workspace export/import (2.5).
3. **Generic by design.** No domain defaults, no Mozambique anything. One small inline sample dataset so the app isn't empty on first load.
4. **R file support via WebR, lazy-loaded.** WebR (R compiled to WebAssembly) is dynamically imported only when the user opens an `.rds`/`.RData` file, so the base bundle stays light. WebR also runs under Node, so R-conversion tests run in Vitest (marked `slow`; WASM cached in CI).
5. **Stack:** Vite + React 18 + TypeScript (strict) + Vitest + React Testing Library + fast-check + fake-indexeddb (unit) + a small Playwright suite (real-browser persistence). ESLint defaults. Coverage gate ≥ 80 % on `src/lib/**` only (ingest + store); UI is covered by component/e2e tests, not a percentage.
6. **Caps:** `ROWS_MAX = 500,000` (hard, `E_TOO_LARGE`); advisory banner ≥ 250,000 rows.
7. Component extraction for reuse in other suite apps is a *later* refactor if wanted; v1 is one app.

---

## Part 0 — How to use this document

Create an empty repo `icebergsim-dataExplorer`, drop this file in as `BUILD_PLAN.md`, open Claude Code rooted in it (verify with `pwd`), and paste the kickoff prompt (Part 6). The agent writes `spec/SPEC.md` + `spec/cases.yaml` + the fixture files **before** feature code; the hook re-runs the fast suite after every edit; phases end at gates. Sections below marked **[→ file]** are materialized as real files.

---

## Part 1 — Product and architecture

**The app in one paragraph.** A single-page workbench. Left sidebar: the **dataset library** (every imported dataset, persisted locally, with name / source / rows×cols / date; rename, delete) and the **views list** (named saved pivot configurations per dataset). Main area: the PivotTable.js UI on the active dataset, with Save view / Save as / a dirty-state indicator, plus an Import button (file picker + drag-drop) and workspace Export/Import. The session (active dataset, current pivot state, active view) autosaves continuously and restores on next launch. Everything runs and stays in the browser.

**Modules (`src/`):**
- `lib/ingest/` — pure: `csv.ts`, `excel.ts`, `json.ts`, `rlang.ts` (WebR), shared `infer.ts` + `Dataset` model. Every importer returns `Result<Dataset, StructuredError>`.
- `lib/store/` — Dexie schema, dataset/view/session repositories, workspace export/import, storage-persistence request.
- `lib/pivot/` — the **PivotState whitelist** (2.4) and (de)serialization helpers.
- `components/` + `App.tsx` — sidebar, workbench, import dialog, dialogs. Components hold no parsing or persistence logic beyond calling `lib/`.

**Design rules (short list, into CLAUDE.md):** `lib/**` is pure and DOM-free; errors are `Result` values with codes; nothing non-whitelisted is ever persisted; the widget's own aggregation is trusted as-is (we wrap it, we don't reimplement it); WebR is only ever loaded via dynamic import.

---

## Part 2 — Contracts **[→ file: `spec/SPEC.md`]**

### 2.1 The Dataset model

```ts
type ColType = "number" | "string" | "boolean" | "date";
type Dataset = {
  id: string;                       // uuid
  name: string;                     // user-editable, unique-ified on collision
  source: { filename: string; format: "csv"|"tsv"|"xlsx"|"json"|"rds"|"rdata"; sheet?: string };
  columns: { name: string; type: ColType; warnings?: string[] }[];
  rows: Record<string, unknown>[];  // values: number | string | boolean | null
  importedAt: string; rowCount: number; colCount: number;
};
```

**Null tokens** (all formats, case-insensitive, trimmed): `""`, `NA`, `N/A`, `null` → `null`. **Type inference** per column over non-null values: all numeric → `number`; all `true/false` → `boolean`; all ISO-8601 → `date` (stored as ISO strings; lexicographic = chronological); else `string`; mixed numeric/string → `string` + a column warning, never silent coercion. Duplicate column names → `E_DUPLICATE_COLUMNS`.

### 2.2 Format-specific rules

- **CSV/TSV** — papaparse, `header: true`, chunked parsing (no double-buffered 100 MB strings), delimiter auto with TSV honored by extension.
- **Excel** — SheetJS with `cellDates: true`; dates → ISO strings; empty cells → null; header = first row; multi-sheet workbooks open a **sheet picker** and `source.sheet` records the choice.
- **JSON** — array-of-objects or `{col: values[]}` column form; anything else → `E_NOT_TABULAR`.
- **R (`.rds`, `.RData`)** — via WebR: write the uploaded bytes into WebR's virtual FS, `readRDS()` / `load()`, then convert. **Type mapping (normative):** factor → `string` (the label, not the integer code); character → `string`; numeric/integer → `number`; logical → `boolean`; `Date`/`POSIXct` → `date` as ISO string; `NA` of any type → `null`. Accepted objects: `data.frame` (or tibble); a matrix is converted with its column names (or `V1…Vn`); anything else (list, model, function) → `E_NOT_TABULAR` with the object's class in the message. `.RData` containing several objects opens a picker like the sheet picker. The WebR call sequence is implementation (verify against the pinned WebR version); this mapping table is the spec.

### 2.3 Canonical R fixture (human-specified; the seed of Tier A)

`fixtures/make_fixtures.ts` (runs WebR in Node) creates and commits `sample.rds`: a 6-row data.frame —

| id (int) | group (factor: a,b) | score (num) | ok (logical) | when (Date) | note (chr) |
|---|---|---|---|---|---|
| 1 | a | 1.5 | TRUE | 2024-01-01 | x |
| 2 | b | 2.5 | FALSE | 2024-01-02 | y |
| 3 | a | NA | TRUE | 2024-01-03 | z |
| 4 | b | 4.0 | NA | NA | NA |
| 5 | a | 5.5 | FALSE | 2024-01-05 | w |
| 6 | b | 6.0 | TRUE | 2024-01-06 | v |

The imported Dataset must be exactly: types `number, string, number, boolean, date, string`; `group` values the *labels* `a/b`; every `NA` → `null`; `when` values the ISO strings shown; 6 rows × 6 cols. The same script writes `sample.xlsx`, `sample.csv`, `sample.json`, and a small `.RData` (two objects, to exercise the picker) with stated expectations. These fixtures and expectations freeze at the end of Phase 1.

### 2.4 PivotState whitelist (the persistence gotcha, made law)

react-pivottable's `onChange` state echoes back non-serializable members (the data array, aggregator/renderer functions, sorters). **Persisted state is exactly this whitelist and nothing else:** `rows, cols, vals, aggregatorName, rendererName, valueFilter, rowOrder, colOrder, hiddenAttributes, hiddenFromAggregators, hiddenFromDragDrop`. `serializePivotState()` drops everything not listed; `deserialize` merges onto defaults. A saved **View** is `{ id, name, datasetId, state: WhitelistedPivotState, createdAt, updatedAt }`.

### 2.5 Storage and workspace

Dexie DB `ism-dataexplorer`, schema v1: tables `datasets`, `views`, `session` (single row: `{activeDatasetId, activeViewId?, liveState, savedAt}`), `meta`. Schema changes only via Dexie versioned migrations. Session autosaves debounced (500 ms) and on `visibilitychange`; restored silently on launch. Deleting a dataset lists and deletes its views after one confirm. On first import, request `navigator.storage.persist()` and surface the answer in the UI footer ("storage: persistent / best-effort").

**Workspace export/import:** one JSON file, `schema: "ism.dataexplorer.workspace/1"`, containing datasets + views + session. Import merges: id collisions get fresh ids; name collisions get ` (imported)`. Round-trip identity is a Tier-P property.

### 2.6 Errors (Result codes)

`E_PARSE` (with row/col or sheet context) · `E_UNSUPPORTED_FORMAT` · `E_NOT_TABULAR` · `E_EMPTY_DATA` · `E_DUPLICATE_COLUMNS` · `E_TOO_LARGE` · `E_STORE` (IndexedDB failure, with a retry affordance) · `E_WEBR` (runtime load/eval failure, with "try again / export CSV from R instead" guidance).

---

## Part 3 — Acceptance **[→ files: `spec/cases.yaml`, `tests/`, `fixtures/`]**

Two tiers, written in Phase 1 and red-for-the-right-reason before implementation.

**Tier A — exact fixtures.**
- **A1 — R worked fixture.** Importing `sample.rds` yields exactly the 2.3 table: types, factor labels, ISO dates, every `NA` → null. `slow`-marked (WebR in Node). The `.RData` fixture exercises the object picker path and the same mapping.
- **A2 — per-format exactness.** `sample.csv` / `sample.xlsx` / `sample.json` each import to their stated Dataset (types, nulls, warnings); the messy variants pin: null tokens, mixed-type → string + warning, duplicate headers → `E_DUPLICATE_COLUMNS`, non-tabular JSON → `E_NOT_TABULAR`, multi-sheet → picker + `source.sheet`.
- **A3 — whitelist enforcement.** Serializing a pivot state containing `data`, function-valued members, and unknown keys produces exactly the whitelist; deserializing merges onto defaults; a saved View never contains a `data` key (regression pin).
- **A4 — store CRUD + cascade.** Dataset save/load identity (fake-indexeddb); view save/load identity; deleting a dataset deletes exactly its views; session write/read identity.
- **A5 — error codes.** One case per 2.6 row.

**Tier P — round-trip properties (fast-check, seeded).**
- **P1** dataset → store → load → deep-equal (arbitrary small generated datasets).
- **P2** arbitrary whitelisted pivot state → view save → load → deep-equal.
- **P3** workspace export → import into empty DB → datasets/views deep-equal modulo regenerated ids.
- **P4** CSV writer-less round-trip: generated rows → CSV text → import → equal modulo type inference rules (numbers/booleans/nulls as specified).
- **P5** whitelist idempotence: `serialize(serialize(s)) ≡ serialize(s)`; unknown keys never survive.

**e2e (Playwright, small and real-browser only where it must be):** load app → sample visible → drag a field → save view "V1" → **reload page** → session restored → open "V1" → import `sample.csv` → delete it (confirm cascade). One extra spec, local/nightly rather than per-push if CI time hurts: import `sample.rds` end-to-end (real WebR download).

**Inviolable rule (short form).** `spec/SPEC.md`, `spec/cases.yaml`, and `fixtures/**` freeze at the Phase-1 gate. The 2.3 table is canonical as written here. No expected value, fixture, or tolerance changes to make code pass; a wrong-looking case is an escalation. (No goldens, no oracle — that machinery is deliberately absent from this profile.)

---

## Part 4 — Scaffold and Part 5 — Phases

```
icebergsim-dataExplorer/
├── BUILD_PLAN.md · CLAUDE.md · README.md · package.json · tsconfig.json
├── .claude/settings.json          # PostToolUse: npm run check:fast | tail -40
├── .github/workflows/ci.yml       # typecheck+lint+vitest(+slow weekly)+build · playwright · pages
├── spec/ SPEC.md · cases.yaml
├── fixtures/ make_fixtures.ts · sample.{csv,xlsx,json,rds} · sample_multi.{xlsx,RData} · messy.csv
├── src/ lib/{ingest,store,pivot}/ · components/ · App.tsx · types/react-pivottable.d.ts
└── tests/ cases.spec.ts · unit per module · properties/ · e2e/
```

| Phase | Builds | Gate |
|---|---|---|
| 0 — Scaffold | Vite+React18+TS strict, deps (`--legacy-peer-deps` if needed), hook, CI skeleton; the prototype's inline-sample pivot renders | `check:fast` runs; dev build shows a working pivot on the inline sample |
| 1 — Spec & fixtures | SPEC.md, cases.yaml, `make_fixtures.ts` run (fixtures committed, incl. WebR-generated `.rds`/`.RData`) | cases present, red for the right reason; fixtures frozen |
| 2 — Ingest (JS formats) | csv/excel/json + infer + Dataset | A2 A5(subset) P4 green |
| 3 — Store & workspace | Dexie schema, repos, session, export/import, persist() | A3 A4 P1 P2 P3 P5 green (fake-indexeddb) |
| 4 — Workbench UI | sidebar (datasets/views), pivot area, save/dirty, import dialog, autosave restore | component tests green; Playwright core spec green |
| 5 — R support | `rlang.ts` via lazy WebR (browser) + Node path (tests) | **A1 green**; bundle check: WebR absent from the main chunk |
| 6 — Polish & ship | advisory banner, storage-status footer, empty/error states, README, Pages deploy | full suite + e2e green; **DoD** |

**Definition of Done:** all Tier A/P green (incl. `slow` A1); Playwright core flow green against the production build; whitelist regression (A3) holds; the app survives a hard reload with session, datasets, and views intact; WebR loads only on demand; static build deploys to Pages; README states scope honestly (local-first, per-browser storage, export = backup).

---

## Part 6 — Loop config and kickoff

**CLAUDE.md (paste verbatim):** the Part-1 design rules, the whitelist law (2.4), the R type-mapping table pointer, the inviolable short form, commands (`npm run check:fast`, `npm test`, `npm run test:slow`, `npm run e2e`, `npm run build`, `npx tsx fixtures/make_fixtures.ts`), and: *work phase by phase; never advance past a red gate; commit once per green gate; observe library behavior before assuming it (record surprises in ENGINE_NOTES.md, which is scratch, not spec).*

**Escalation triggers (only these):** WebR cannot realize a row of the 2.2 R type-mapping table · react-pivottable is unusable under the pinned React even with `patch-package` · IndexedDB behavior makes a Tier-A/P case unimplementable as written · a frozen fixture/case looks wrong. Everything else proceeds.

**Kickoff prompt:**

> You are building **icebergsim-dataExplorer** in this repo per `BUILD_PLAN.md` — read it fully first. This is a medium-rigor build: spec and fixtures before features, two test tiers, gates — but no oracle and no goldens; do not add heavyweight machinery the plan doesn't ask for, and equally do not skip what it does ask for. Phase 1 freezes `spec/`, `cases.yaml`, and `fixtures/**` (the R worked table in SPEC 2.3 is canonical as written); after that, never edit them to make code pass — escalate instead, but only on the four triggers in Part 6. The persistence whitelist (SPEC 2.4) is law: nothing outside it is ever stored. WebR is loaded only by dynamic import, and R conversion is tested in Node with the committed fixtures. Work Phases 0→6 in order, never advancing past a red gate; read the hook output after every edit; commit once per green gate. Done = the Definition of Done in Part 5. Start with Phase 0 and confirm the Part-5 table back to me as your todo list.
