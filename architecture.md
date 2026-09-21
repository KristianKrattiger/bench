# Architecture

Bench is the human port of the portfolio. This file describes the intended
design — nothing here is implemented yet (see [README.md](README.md) for
status). It exists so implementation starts from a design that's already
been argued through, not from a blank editor.

## Relationship to the portfolio

Every other piece here is an interface between automated things, and each
holds its epistemic guarantees because no human hand is in the loop at write
time:

| Piece | Interface | Sole writer of canonical state |
|---|---|---|
| GIN | machine-to-corpus | Bookkeeper (admits Cartographer's `EdgeProposal`s) |
| Assay | machine-to-span | the admission gate inside `assay()` (admits proposed quote pairings) |
| Receipts | machine-to-web | — (wraps Assay for live fetch; same gate) |
| Federation | machine-to-machine | — (Bookkeeper again, across a trust boundary) |
| **Bench** | **human-to-record** | **nobody — it has no canonical state to write** |

Bench is downstream of Receipts specifically, not GIN. It reads a
`Report | Ledger | Refusal` that Receipts' `analyzeLive` already produced,
lets a person work with it, and stores what that person concluded. GIN
federation is a named future extension (Phase 5), not a Phase 1 dependency.

## Design principles

1. **No epistemic logic.** Bench never decides `divergent` vs
   `corroborated`, never re-runs `assay()`, never fetches. It renders state
   that already exists and captures judgment about that state. The moment
   Bench starts typing relations or adjudicating admission, the property
   that makes the rest of the stack defensible breaks — and it breaks in the
   component with the least test coverage and the most UI surface area.
2. **Append-only, never collapsed.** A note or label is added, never edited
   in place. Two labelers disagreeing is data, not noise to be voted away.
3. **Proposes, never writes.** Where Bench eventually feeds something back
   toward the live pipeline (a candidate `SourcePlan` override in Phase 2), it
   produces a proposal a human or the existing CLI chooses to run — the same
   relationship Cartographer has to the Bookkeeper.
4. **Read-only against `receipts/`.** Bench never mutates a Receipts
   checkout. `reports/`, `snapshots/`, `cache/proposals/` are inputs.
5. **Local-first, mergeable by construction.** No server, no accounts, no
   sync protocol in Phase 1 — and the record shape is chosen so adding a
   server later (Phase 2+) is a transport change, not a schema change.

## System overview

```
receipts/ (git checkout, read-only from Bench's side)
├─ reports/*.json          Report | Ledger | Refusal — the object Bench renders
├─ snapshots/               pinned bytes a row's quote/anchor resolves into
└─ cache/proposals/          not read by Bench directly (Assay-internal)

bench/ (this repo)
├─ src/
│   ├─ ingest/               loads a receipts report + resolves its snapshots
│   ├─ store/                 the append-only note/label log + SQLite fold
│   ├─ web/                   local HTTP server + UI (127.0.0.1, same pattern as claim-record)
│   └─ types.ts               Bench's own types, importing Report/Ledger/SourceTarget etc. from receipts
└─ data/
    ├─ notes.<labeler>.jsonl      one file per analyst, append-only, git-committed
    └─ bench.sqlite                  gitignored, rebuilt from the .jsonl logs by folding
```

Bench depends on Receipts' published types the way `claim-record` vendors
`assay` — but as a read dependency on shapes, not a subtree of engine code.
Bench never gains the ability to produce a `Ledger`; it only ever consumes
one.

## Data model

Two record kinds in Phase 1, both append-only, both shaped like GIN
Curator's `LabelRecord`: an id, a timestamp, a labeler identity, an optional
`supersedes` pointing at the record it overrides, and otherwise immutable.

**`RowNote`** — a note or judgment attached to one ledger row, or a specific
cited span within it:

```ts
{
  id: string
  reportId: string          // which reports/*.json this refers to
  rowRef: { chunkId: string, anchor?: [number, number] }
  labeler: string
  ts: string                 // UTC ISO-8601
  body: string                // free-text note
  agreement?: "agree" | "disagree" | "uncertain"
  supersedes?: string
}
```

**`ProjectNote`** — a freestanding note or write-up, not tied to one row:

```ts
{
  id: string
  projectId: string
  labeler: string
  ts: string
  body: string
  cites: { reportId: string, rowRef?: RowRef }[]   // links to evidence, never inlines it
  supersedes?: string
}
```

Folding matches Curator's `Store.fold_current`: read every labeler's
`.jsonl`, group by a stable key, latest `(ts, idx)` wins for "current" — but
nothing is discarded. The full history, including every labeler's take on
the same row, stays queryable. That's what makes multi-seat real rather than
nominal: two analysts' disagreeing `RowNote`s on the same row are both
first-class, not resolved to one, and the disagreement rate itself becomes a
property you can ask the store about later.

A malformed line in a `.jsonl` log fails loudly, with file and line number,
the same discipline as Curator's `Store.read_log` — never silently skipped.

## Storage and sync

Local-first: no server, no accounts, in Phase 1. Each analyst's Bench
instance writes to its own `notes.<labeler>.jsonl`, committed to git the
same way Receipts already commits `snapshots/` and `reports/*.json` — this
project's existing convention, not a new one. Multi-seat sync is `git pull`;
merging is safe by construction because concatenation-then-fold is exactly
what `fold_current` already does, regardless of which file a line came from
(the same property `curator_merge_check.py` relies on today).

A local SQLite database, gitignored and fully rebuildable from the `.jsonl`
logs, is the query engine behind the UI — fast row lookup, filters, and
(Phase 3) the graph view. SQLite is a cache. The `.jsonl` logs are the
source of truth.

This shape is deliberately transport-agnostic: Phase 2+ can add a shared
server that multiple analysts write to directly, without touching the
record schema — the write path changes from "append to a local file" to
"POST to an endpoint that appends," same record either way.

## Workflow (Phase 1)

1. `npm start` opens Bench on `127.0.0.1`, listing reports found under a
   configured Receipts checkout, grouped by project/subject.
2. Opening a report renders the ledger as Receipts produced it — rows
   grouped by status (`divergent` / `corroborated` / `context_unverified` /
   `unverified`), claimant and independent excerpts shown with quotes
   resolved against `snapshots/` bytes, so a span is verifiable in the UI,
   not just trusted. The coverage/audit numbers Receipts already computes
   (`claimantCovered`, `independentDocsAdmitted`, denial-code tallies)
   surface directly — Bench renders them, never recomputes them.
3. Clicking a row opens a note panel: attach a `RowNote`, see every
   labeler's notes on that row side by side — never collapsed — with
   history via the `supersedes` chain.
4. A separate notes view lets the analyst write a `ProjectNote` — freeform
   synthesis — with a citation picker that adds `cites` entries pointing at
   rows/spans rather than copying text in.

## Error handling

- A missing or malformed report is a typed refusal in the UI, not a silent
  empty state.
- A row whose snapshot bytes are missing (a broken pin) still renders from
  the ledger's stored quote text, flagged "unverifiable against snapshot"
  rather than failing to render or silently trusting it.
- A malformed `.jsonl` line raises loudly with file and line number.
- Two labelers disagreeing is not an error. It's the product working.

## Testing

- **Ingest**: pure function tests against fixture reports and snapshots. No
  network, no server, no live Receipts checkout required.
- **Store/fold**: a near-direct port of Curator's `fold_current` test
  shapes — latest-wins-for-current-but-nothing-discarded, `supersedes`
  chains, stable ordering independent of which file a record came from.
- **UI**: Phase 1 is deliberately thin (render + capture), so it gets
  smoke/integration coverage rather than exhaustive testing. Correctness
  lives in ingest and store, which are pure and fully unit-testable.

## Phased roadmap

### Phase 1 — Ledger + notes

Everything above. The thin end-to-end loop: open a report, read the
evidence, write down what you think, keep every labeler's take.

### Phase 2 — Source-trust curation

Adds a `SourceTrust` record:

```ts
{
  id: string
  domain: string
  labeler: string
  ts: string
  weight: number | "block" | "trust"
  rationale: string
  supersedes?: string
}
```

...and a gate function shaped like GIN's `trust_gate.is_trusted(peer_domains,
peer_weights, threshold)` — same signature, same "absence of information
never gates, only a known-bad weight does" invariant. This does **not**
plug into Receipts' fetch path directly; that would make Bench gate a live
pipeline, which is writing canonical state by another name. Instead, Bench's
fold of trust records produces a *proposed* `SourcePlan` override that a
human (or the existing `receipts` CLI) chooses to actually run. Bench still
only proposes.

### Phase 3 — Graph view

A read-only visualization over the same SQLite fold: nodes are sources,
ledger rows, and notes; edges are citations (`ProjectNote.cites`) and
row-to-source anchors. No new canonical data — presentation only, over data
Phase 1 and 2 already collected.

### Phase 4 — Narrator as research assistant

Receipts already has an optional Narrator (off by default) that restates
admitted rows with citations and never invents quotes. Bench extends its
role — summarizing across a project's notes and ledgers, surfacing
patterns, helping navigate — but keeps that discipline as the load-bearing
constraint: every Narrator output lands as a *suggested* `ProjectNote` a
human accepts, edits, or discards. A human's save action is what appends
the record. Narrator never writes one on its own. This is what keeps Phase
4 from quietly becoming epistemic logic inside Bench.

### Phase 5 — GIN / federation-shaped extension point (not built now)

`SourceTrust.labeler` is deliberately just a string, so a trust weight's
origin can later be a federation peer's node id instead of a human name —
the same fold absorbs it, no schema change. When GIN federation exists, a
peer's domain-trust summary becomes just another labeler in the same store,
not a special case requiring new plumbing. Likewise, the Phase 3 graph
model can add GIN corpus nodes/edges as another node type without changing
its shape.

If GIN-level labeling (Cartographer's typed edges, `issue_frame` divergence)
becomes a second surface Bench serves, it gets its own design pass — a
ledger row and a corpus-chunk-pair are different units of judgment from
different source systems, and collapsing them into one data model
prematurely would be the same mistake this document explicitly avoids
elsewhere.

## What this is not

- Not a fetcher. Bench never calls Solari, never runs a live fetch.
- Not an admission gate. It never decides `divergent` vs `corroborated` —
  only renders what Assay already decided.
- Not a database of record for Receipts. `receipts/` stays read-only from
  Bench's side.
- Not multi-tenant infrastructure in Phase 1. The commercial "seats" story
  is real but is a Phase 2+ transport decision, not a Phase 1 requirement.
- Not GIN-integrated yet. See Phase 5.

## Related documents

- [README.md](README.md) — status, quickstart, what a note means.
- [../receipts/architecture.md](../receipts/architecture.md) — the ledger
  Bench renders, the `src/fetch/` fan (Solari), pins and replay.
- [../gin/GIN/gin/curator/](../gin/GIN/gin/curator/) — the `LabelRecord` /
  `Store.fold_current` pattern Bench's note store is modeled on.
- [../gin/GIN/gin/federation/trust_gate.py](../gin/GIN/gin/federation/trust_gate.py)
  — the trust-gate shape Phase 2's `SourceTrust` gate is modeled on.
