# Bench

The human port. Every other piece of this portfolio is an interface between
automated things — GIN is machine-to-corpus, Assay is machine-to-span,
Receipts is machine-to-web, federation is machine-to-machine. Bench is where
a person touches it: a research workstation for reviewing what Receipts
produced, writing down judgment, and keeping that judgment as data instead
of throwing it away.

**Status: design stage.** This repo currently holds the architecture, not
the implementation. See [architecture.md](architecture.md) for the detailed
overview-level design, including the phased build plan. Nothing below the
"Run locally" heading works yet — it documents the intended interface.

## What it's for

Receipts turns a subject into a `SourcePlan`, fans out fetches across the
web, and hands the result to Assay, which produces a report — a ledger of
`divergent` / `disputed` / `corroborated` / `context_unverified` /
`unverified` rows, or a typed refusal. That ledger is Assay's output, not
the end of the work: a person still has to read it, decide what to trust,
and write down what they concluded. Today that happens in a terminal or a
text file, and it disappears. Bench is the instrument for doing that work
instead of improvising it, and for keeping the record it produces.

## Design principle

Bench contains no epistemic logic. It renders state Receipts and Assay
already produced, captures human judgment as append-only notes and labels,
and — later — submits proposals back toward the pipeline (a candidate
`SourcePlan` override, eventually a federation-shaped trust signal). It
never re-runs admission, never decides what's true, and never edits
anything under a `receipts/` checkout. Cartographer proposes and doesn't
write; the Bookkeeper is the sole admission gate. Bench inherits that
discipline rather than being the place it leaks. Full reasoning in
[architecture.md](architecture.md#design-principles).

## Run locally (intended, once built)

```bash
npm install
cp .env.example .env    # BENCH_RECEIPTS_PATH=/path/to/receipts checkout
                         # BENCH_LABELER=<your-name> - whose .jsonl log you write to
npm start
```

Open `http://127.0.0.1:8787`. Bench reads `reports/*.json` and `snapshots/`
from the configured Receipts checkout — read-only — maps them into
Bench-owned DTOs (no compile-time import from receipts), and keeps its own
notes under `data/`, one append-only `.jsonl` log per labeler, git-committed
the same way Receipts commits its own snapshots. Multi-seat from day one:
every labeler's current notes are visible; sync is `git pull`.

## What a note means

- **`RowNote`** — a note (and optional agree / disagree / uncertain call)
  attached to one ledger row (`rowId` assigned at ingest) or a specific
  cited span (`spanRef`: `docId` + offsets) within it.
- **`ProjectNote`** — a freestanding note or write-up, citing rows and spans
  rather than copying their text in, so the note stays grounded without
  duplicating evidence.

Two labelers disagreeing on the same row is not an error and is never
collapsed to one answer. The fold keeps each labeler's current note; both
are visible, and that divergence is itself queryable — the Curator move,
one level up, with fold keys that include `labeler`.

## What this is not

- Not a fetcher. Bench never calls Solari, never runs a live fetch.
- Not an admission gate. Bench never decides `divergent` vs `disputed` vs
  `corroborated` — it renders what Assay already decided.
- Not a database of record for Receipts. `receipts/` stays read-only from
  Bench's side; Bench's own SQLite index is a rebuildable cache over its
  `.jsonl` logs, not the source of truth.
- Not a compile-time consumer of receipts/assay packages.
- Not GIN-integrated yet. Federation and corpus-level labeling are a later
  phase — see [architecture.md](architecture.md#phase-5--gin--federation-shaped-extension-point-not-built-now).

## Tests

None yet — no code. The architecture doc specifies what gets tested and how
once implementation starts.
