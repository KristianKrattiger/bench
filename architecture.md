# Architecture

Bench is the human port of the portfolio. This file is the canonical
overview-level design — nothing here is implemented yet (see
[README.md](README.md) for status). It exists so implementation starts from
a design that has already been argued through, corrected against the sibling
repos, and scoped phase by phase.

Phase 1 is specified in enough detail to build. Phases 2–5 are overview
sketches: purpose, records, and boundaries, not implementation plans.

## Portfolio fit and hard boundaries

Every other piece here is an interface between automated things, and each
holds its epistemic guarantees because no human hand is in the loop at
epistemic write time:

| Piece | Interface | What it may write |
|---|---|---|
| GIN | machine↔corpus / federation | Bookkeeper admits edges; Curator appends labels (not edges) |
| Assay / Receipts | machine↔span / web | Ledger or typed Refusal; pinned snapshots |
| claim-record | local Assay field instance | Same Assay outcomes from uploaded piles |
| **Bench** | **human↔record** | **Only Bench’s own append-only notes/labels** — never Receipts or GIN epistemic state |

Bench is downstream of Receipts specifically, not GIN. It reads on-disk
report JSON that a Receipts checkout already produced, lets a person work
with it, and stores what that person concluded. GIN federation is a named
future extension (Phase 5), not a Phase 1 dependency. claim-record is the
local HTTP/UI pattern reference, not a runtime dependency.

**Hard boundaries (all phases):**

1. **No epistemic logic.** Bench never decides row status, never re-runs
   `assay()`, never fetches. It renders state that already exists and
   captures judgment about that state.
2. **Read-only against `receipts/`.** `reports/` and `snapshots/` are
   inputs. Bench never mutates a Receipts checkout. `cache/proposals/` is
   Assay-internal and is not read by Bench.
3. **Append-only judgment.** Notes and labels are added or superseded;
   never edited in place. Disagreement between labelers is data, not noise
   to collapse.
4. **Proposes, never writes** (toward the live pipeline). Where Bench
   eventually feeds something back (a candidate `SourcePlan` override in
   Phase 2), it produces a proposal a human or the existing Receipts CLI
   chooses to run — the same relationship Cartographer has to the
   Bookkeeper.
5. **DTO boundary.** Bench maps on-disk Receipts JSON into Bench-owned
   types. There is no compile-time import from the receipts or assay
   packages.
6. **Canonical split.** The `.jsonl` logs are Bench’s source of truth for
   human judgment. SQLite is a rebuildable cache. Bench has no *epistemic*
   canonical state; it *does* own the human judgment log.

```
receipts checkout (read-only)
  reports/*.json ──► ingest ──► Bench DTOs ──► UI
  snapshots/*   ──►      │
                         ▼
                   append .jsonl ──► per-labeler fold ──► sqlite cache ──► UI
```

## Design principles

1. **No epistemic logic.** The moment Bench starts typing relations or
   adjudicating admission, the property that makes the rest of the stack
   defensible breaks — in the component with the least test coverage and
   the most UI surface area.
2. **Append-only, never collapsed.** A note is added, never edited in
   place. Two labelers disagreeing is the product working.
3. **Proposes, never writes** toward Receipts/GIN canonical state.
4. **Read-only against `receipts/`.**
5. **Local-first, mergeable by construction.** No shared note server and
   no accounts in Phase 1. Multi-seat sync is git. The record shape is
   chosen so a shared write endpoint later is a transport change, not a
   schema change.
6. **Functional over polished in Phase 1.** The workstation must complete
   the loop (list → report → row note → project note). Visual fidelity to
   published Receipts HTML ledgers is deferred.

## System overview

```
receipts/ (git checkout, read-only from Bench's side)
├─ reports/*.json          ledger-shaped Report, or Refusal — what Bench renders
├─ snapshots/              pinned bytes a row's quote/anchor resolves into
└─ cache/proposals/        not read by Bench (Assay-internal)

bench/ (this repo)
├─ src/
│   ├─ ingest/             load report JSON + resolve snapshots → Bench DTOs; assign rowId
│   ├─ store/              append-only note/label log + per-labeler fold + SQLite rebuild
│   ├─ server/             loopback HTTP (claim-record twin)
│   ├─ web/                minimal static UI
│   └─ types.ts            Bench DTOs + note records only (no receipts imports)
└─ data/
    ├─ notes.<labeler>.jsonl   one file per analyst, append-only, git-committed
    └─ bench.sqlite            gitignored, rebuilt from the .jsonl logs by folding
```

**Environment (Phase 1):**

- `BENCH_RECEIPTS_PATH` — absolute path to a Receipts checkout
- `BENCH_LABELER` — writer identity for this process (stamped on every
  append). All `notes.*.jsonl` files are read; only this labeler’s file is
  written.

Multi-seat is a Phase 1 requirement: N labelers, every current note visible
side by side, sync via `git pull`. There is no product requirement for a
shared server until a later transport phase.

## On-disk contract (Receipts → Bench DTO)

Bench does not import Receipts TypeScript. It documents and implements a
**file contract**: JSON on disk in, Bench DTOs out. Shapes below describe
what ingest must accept; field names match what Receipts writes today so
fixtures and live checkouts both work.

**Inputs**

| Path | Role |
|---|---|
| `reports/*.json` | Primary object: ledger-shaped report or typed refusal |
| `snapshots/{sha256}.json` | Content-addressed page bytes for span verification |

`cache/proposals/` is not an input.

**`reportId`** = report basename without `.json` (e.g. `reports/tesla-fsd.json`
→ `tesla-fsd`). Reports have no separate id field on disk; the filename is
the stable key.

**Ledger-shaped report (accepted shapes).** Fresh ledgers may carry
`outcome: "ledger"`. Legacy reports may omit `outcome` but still expose
`subject`, `rows`, `docs`, and optionally `audit`. Ingest treats both as
renderable ledgers when `rows` is present. A structural refusal
(`outcome: "refusal"`, or the refusal shape Receipts writes) is a first-class
result, not an error.

**Row statuses (five).** Bench renders exactly the statuses Assay admits:

- `divergent`
- `disputed` — soft contradiction; labeled, not collapsed into `divergent`
- `corroborated`
- `context_unverified`
- `unverified`

Unknown status strings fail ingest loudly for that report (typed refusal in
the UI), rather than being silently remapped.

**Sides and pins.** Each ledger row has `sides[]` with
`{ docId, start, end, text, tag }`. There is no `chunkId` on a ledger row.
Docs carry optional `pin` (`permalink` | `snapshot` | `hash`) with
`sha256`. Ingest resolves `snapshot` (and permalink pins that name a local
snapshot blob) to `snapshots/{sha256}.json` and verifies that
`text === content.slice(start, end)` when bytes are present.

**`rowId`.** Assigned at ingest, stable across reloads:

```
rowId = hex(sha256(
  reportId + "\0" + topic + "\0" + statement + "\0" +
  sides.map(s => `${s.docId}:${s.start}:${s.end}`).join("\0")
))
```

Sides are ordered as stored on the row. The same report bytes always yield
the same `rowId`s. Notes key off `rowId`, not off transient UI indices.

**Audit.** Render whatever audit object is present (`claimantCovered`,
`independentDocsAdmitted`, denial tallies, `disputed` / `contextUnverified`
counts when present). Legacy reports may carry a thinner audit. Bench never
recomputes Assay’s audit; missing fields are absent in the UI, not filled
with zeros invented by Bench.

**Broken pin.** If snapshot bytes are missing or the slice does not match
the stored quote, the row still renders from the ledger’s stored `text`,
flagged `unverifiable_against_snapshot`. It does not fail the whole report
and does not silently pretend the pin verified.

**Pattern references (not imports):** Receipts assay types and snapshot
layout; claim-record’s loopback server; GIN Curator `Store.read_log` /
fold discipline; GIN `trust_gate` for Phase 2 shape. See [Related
documents](#related-documents).

## Data model (Phase 1)

Two record kinds, both append-only, both shaped like GIN Curator’s
`LabelRecord`: an id, a timestamp, a labeler identity, an optional
`supersedes` pointing at the record it overrides, and otherwise immutable.

**`SpanRef`** — optional pointer into a cited side (not a Curator chunk id):

```ts
{
  docId: string
  start: number
  end: number
}
```

**`RowNote`** — a note or judgment attached to one ledger row, or a specific
cited span within it:

```ts
{
  kind: "row_note"
  id: string
  reportId: string
  rowId: string
  spanRef?: SpanRef
  labeler: string
  ts: string                 // UTC ISO-8601
  body: string
  agreement?: "agree" | "disagree" | "uncertain"
  supersedes?: string
}
```

**`ProjectNote`** — a freestanding note or write-up, not tied to one row:

```ts
{
  kind: "project_note"
  id: string
  projectId: string
  labeler: string
  ts: string
  body: string
  cites: { reportId: string, rowId?: string, spanRef?: SpanRef }[]
  supersedes?: string
}
```

Citations link to evidence; they never inline quote text. That keeps the
note grounded without duplicating bytes Receipts already pinned.

### Fold semantics

Curator’s `fold_current` collapses to one current label per unordered chunk
pair globally. Bench needs a different product property: **every labeler’s
current note stays visible**.

Discipline borrowed from Curator:

- Append-only JSONL; one record per line
- Malformed line → loud failure with file path and line number (never skip)
- Latest `(ts, idx)` wins for “current” within a fold key
- Full log retained; nothing discarded
- `supersedes` is recorded for history UI and human intent; the fold
  algorithm itself is latest-`(ts, idx)` per key, not a pointer walk
  (same as Curator)

Bench fold key (deliberate extension):

```
(noteKind, entityKey, labeler)
```

- For `RowNote`: `entityKey = reportId + "\0" + rowId + "\0" + spanKey`,
  where `spanKey` is `"row"` when `spanRef` is absent, otherwise
  `${docId}:${start}:${end}`. Row-scoped and span-scoped notes are distinct
  fold entities.
- For `ProjectNote`: `entityKey = projectId`

**`projectId` (Phase 1):** the report’s `subject` string as written on disk
(e.g. whatever Receipts stamped). Reports that share a subject share a
project-notes space. An empty or missing subject uses the sentinel
`project:unspecified` so notes still have a stable key.

Read every `notes.<labeler>.jsonl`, fold as above, surface **all** labelers’
current records for an entity side by side. Global latest-wins across
labelers is forbidden: that would erase disagreement. Agreement rate across
labelers is a query over the multi-current fold, not a collapse step.

Per-labeler files (`notes.<labeler>.jsonl`) are deliberate. Curator’s default
is often one shared `labels.jsonl`; Bench uses one file per writer so git
authorship and sync stay obvious, and so `BENCH_LABELER` only appends to one
path.

## Storage and multi-seat sync

Local-first: no shared note server, no accounts, in Phase 1. Each analyst’s
Bench process appends to its own `notes.<labeler>.jsonl`, committed to git
the same way Receipts commits `snapshots/` and `reports/*.json`. Multi-seat
sync is `git pull`. Merging is safe by construction: concatenation-then-fold
does not depend on which file a line came from (the same property GIN’s
`curator_merge_check` relies on), provided fold keys include `labeler`.

A local SQLite database, gitignored and fully rebuildable from the `.jsonl`
logs, is the query engine behind the UI — fast row lookup, filters, and
(Phase 3) the graph view. SQLite is a cache. The `.jsonl` logs are the
source of truth for human judgment.

This shape is transport-agnostic: a later phase can add a shared endpoint
that appends the same record shape without changing the schema.

## Phase 1 workflow and API surface

Minimal workstation — functional, not polished.

1. `npm start` serves Bench on `127.0.0.1:8787` (port overridable via
   `PORT`, same convention as claim-record). The home view lists reports
   under `BENCH_RECEIPTS_PATH/reports/`, grouped by `subject` when present.
2. Opening a report renders the ledger as ingested — rows grouped by the
   five statuses, claimant and independent excerpts shown with quotes
   verified against snapshot bytes when pins resolve. Audit fields that
   exist on the report surface directly; Bench does not recompute them.
3. Selecting a row opens a note panel: append a `RowNote` as
   `BENCH_LABELER`, see every labeler’s **current** notes on that row side
   by side, and walk history via `supersedes`.
4. A project-notes view lets the analyst write a `ProjectNote` with a
   citation picker that adds `cites` entries (report / row / span), never
   pasted quote bodies.

**HTTP surface (claim-record-shaped: raw `node:http`, static UI, JSON):**

| Method | Path | Behavior |
|---|---|---|
| `GET` | `/` | Minimal static UI (`src/web/`) |
| `GET` | `/api/reports` | List ingested reports (id, subject, outcome/kind, row counts) |
| `GET` | `/api/reports/:reportId` | One report DTO + resolved span verification flags + folded notes for its rows |
| `GET` | `/api/projects/:projectId/notes` | Folded `ProjectNote`s for a project (all labelers’ currents + history access) |
| `POST` | `/api/notes` | Append one `RowNote` or `ProjectNote` for `BENCH_LABELER` only; body must not spoof another labeler |

`POST /api/notes` is append-only. There is no update or delete route; override
is a new record with `supersedes`. Writes always target
`data/notes.<BENCH_LABELER>.jsonl`. After a successful append, the store
updates its fold / SQLite cache before responding.

Errors return JSON `{ error: string }` with 4xx for client/contract failures
and 5xx for unexpected failures — same plain style as claim-record.

## Error handling

- Missing or malformed report JSON → typed refusal in the UI (and a 4xx on
  the report API), not a silent empty ledger.
- Unknown row status → ingest refusal for that report.
- Broken or mismatched snapshot pin → row renders with
  `unverifiable_against_snapshot`; report still loads.
- Malformed `.jsonl` line → loud failure with file and line number at read
  / fold time.
- `POST /api/notes` with a `labeler` other than `BENCH_LABELER`, or missing
  required fields → 400.
- Two labelers disagreeing is not an error.

## Testing

- **Ingest:** pure function tests against fixture reports and snapshots. No
  network, no live Receipts checkout. Cover: five statuses including
  `disputed`; legacy thin audit vs fuller audit; `rowId` stability for the
  same bytes; broken pin → `unverifiable_against_snapshot`; refusal-shaped
  files; unknown status → loud ingest failure.
- **Store/fold:** Curator-inspired cases — latest-wins-for-current within a
  labeler, nothing discarded, `supersedes` chains, stable ordering
  independent of which file a record came from — plus the Bench-specific
  case: two labelers’ currents on the same `rowId` both survive fold (global
  collapse must fail the test).
- **UI/server:** smoke/integration only in Phase 1 (list → open → append →
  see note). Correctness lives in ingest and store.

## Phased roadmap

### Phase 1 — Ledger + notes

Everything above. Multi-seat via git from day one. Thin end-to-end loop:
open a report, read the evidence, write down what you think, keep every
labeler’s take.

### Phase 2 — Source-trust curation

Adds a `SourceTrust` record:

```ts
{
  kind: "source_trust"
  id: string
  domain: string
  labeler: string
  ts: string
  weight: number | "block" | "trust"
  rationale: string
  supersedes?: string
}
```

…and a gate function shaped like GIN’s
`trust_gate.is_trusted(peer_domains, peer_weights, threshold)` — same
“absence of information never gates; only a known-bad weight does”
invariant. This does **not** plug into Receipts’ fetch path. Bench’s fold of
trust records produces a *proposed* `SourcePlan` override that a human (or
the existing `receipts` CLI) chooses to run. Bench still only proposes.

### Phase 3 — Graph view

A read-only visualization over the same SQLite fold: nodes are sources,
ledger rows, and notes; edges are citations (`ProjectNote.cites`) and
row-to-source anchors. No new canonical data — presentation only over data
Phases 1 and 2 already collected.

### Phase 4 — Narrator as research assistant

Receipts / claim-record already have an optional Narrator (off by default)
that restates admitted rows with citations and never invents quotes. Bench
extends that role — summarizing across a project’s notes and ledgers,
surfacing patterns, helping navigate — under the same constraint: every
Narrator output lands as a *suggested* `ProjectNote` a human accepts, edits,
or discards. A human’s save action appends the record. Narrator never
writes one on its own. That is what keeps Phase 4 from becoming epistemic
logic inside Bench.

### Phase 5 — GIN / federation-shaped extension point (not built now)

`labeler` (and `SourceTrust.labeler`) remains a string, so a trust weight’s
origin can later be a federation peer’s node id instead of a human name —
the same fold absorbs it, no schema change. A peer’s domain-trust summary
becomes another labeler in the same store, not a special case.

If GIN-level labeling (Cartographer’s typed edges, `issue_frame` divergence)
becomes a second surface Bench serves, it gets its own design pass. A ledger
row and a corpus chunk-pair are different units of judgment from different
source systems; collapsing them into one data model prematurely would be the
same mistake this document avoids elsewhere.

## What this is not

- Not a fetcher. Bench never calls Solari, never runs a live fetch.
- Not an admission gate. It never decides `divergent` vs `disputed` vs
  `corroborated` — only renders what Assay already decided.
- Not a database of record for Receipts. `receipts/` stays read-only from
  Bench’s side.
- Not a compile-time consumer of receipts or assay packages. On-disk JSON
  in; Bench DTOs out.
- Not multi-tenant shared-server infrastructure in Phase 1. Multi-seat is
  real (N labelers, git sync); a hosted write path is a later transport
  decision, not a Phase 1 requirement.
- Not GIN-integrated yet. See Phase 5.
- Not a polished ledger publication surface in Phase 1. Functional
  workstation first.

## Related documents

- [README.md](README.md) — status, quickstart, what a note means.
- [../receipts/architecture.md](../receipts/architecture.md) — the ledger
  Bench renders, fetch/pin/replay.
- [../receipts/src/assay/types.ts](../receipts/src/assay/types.ts) —
  on-disk row/status/audit shapes ingest must accept (reference only).
- [../claim-record/src/server/index.ts](../claim-record/src/server/index.ts)
  — loopback HTTP + static UI pattern Phase 1 mirrors.
- [../gin/GIN/gin/curator/](../gin/GIN/gin/curator/) — `LabelRecord` /
  `Store.read_log` / fold discipline (Bench extends fold keys with
  `labeler`).
- [../gin/GIN/scripts/curator_merge_check.py](../gin/GIN/scripts/curator_merge_check.py)
  — concat-then-fold merge safety; per-curator agreement as data.
- [../gin/GIN/gin/federation/trust_gate.py](../gin/GIN/gin/federation/trust_gate.py)
  — trust-gate shape Phase 2’s `SourceTrust` gate is modeled on.
