# /rms-figma-sync — Figma DS ↔ Consumer Figma Sync

**What it does:** Audits whether a *consumer Figma file* (e.g. a product design file that subscribes to the DS library) is in sync with the DS. Compares DS snapshot tokens against the consumer's linked library copy and local overrides. Outputs an HTML report with collection tabs, per-status filter buttons, and a color-coded token table.

> **Sister skill:** `/rms-figma-code-parity` audits whether the *CSS codebase* implements the DS faithfully. Use that for code validation; use this one for design handoff and library-sync validation.

---

## What It Checks

**A. Library sync** — DS snapshot tokens vs the linked library copy inside the consumer file.
- `SYNCED` — token exists in both DS snapshot and consumer's linked library copy, same value
- `PENDING` — token exists in DS but the consumer's linked copy has an older value (pending Figma library update)
- `STALE` — token exists in the consumer's linked copy but was removed from the DS (library update will remove it)

**B. Brand coverage** — Consumer's linked library tokens vs consumer's local override collection.
- `LOCAL` — token has a local override in the consumer file (the consumer is customising this token)

---

## How to Run

```
/rms-figma-sync
```

This skill always runs `consumer-audit.mjs` from the DS project root (`design-tokens-library-master/` or wherever `ds-config.json` lives). Never run it from the consumer project root.

**Script directly:**
```bash
# from DS project root:
node scripts/consumer-audit.mjs --file <consumerFileKey> --report-html <output>.html
node scripts/consumer-audit.mjs --file GfHErcAjjw277iPunsZXCU --report-html bancobai-parity.html
node scripts/consumer-audit.mjs --file GfHErcAjjw277iPunsZXCU --fresh     # bypass cache, re-fetch from Figma
```

The `--file` flag is the Figma file key of the **consumer** file (the product design file). It is NOT the DS file key. Extract it from the consumer file's Figma URL: the path segment after `/design/` or `/file/`.

---

## Project Config

Read `ds-config.json` from the DS project root. Required fields:
- `figmaFileKey` — DS Figma file key
- `figma.colorCollection` — name of the DS color variable collection (e.g. `"Theme"`)
- `figma.primitivePrefix` — path prefix to exclude primitives (e.g. `"primitives/"`)
- `paths.snapshotVars` — path to `figma-vars.snapshot.json` (DS ground truth)
- `figma.modes` — array of `{ name, snapshotKey }` defining DS modes

Optional overrides (avoids Figma API name lookup):
- `dsFileName` — display name for the DS file in the HTML title/meta line
- `consumerFileName` — display name for the consumer file in the HTML title/meta line

The consumer's Figma variables are cached in `consumer-vars-cache.<fileKey>.json`. Pass `--fresh` to bypass cache and re-fetch.

---

## Prerequisites

1. **DS snapshot must be current** — `figma-vars.snapshot.json` must have been refreshed via `/rms-figma-code-parity` Phase 1 (or `node scripts/audit.mjs --snapshot-only`). If the snapshot is stale (>24h), warn the user and offer to run Phase 1 first.
2. **FIGMA_TOKEN** must be set in `.env` with at least viewer access to the **consumer** file. If the consumer file requires a different token, check for `FIGMA_TOKEN_CONSUMER` in `.env` first.
3. The consumer file must have published DS variables linked via Figma library (not just component instances).

---

## Workflow

### Step 1: Read config

Read `ds-config.json`. Extract `figmaFileKey`, collection names, modes, snapshot path, and any name overrides.

Check snapshot age. If stale (>24h), warn: `⚠️ DS snapshot is Xh old. Run /rms-figma-code-parity Phase 1 first to ensure tokens are current.` Then continue (snapshot may still be accurate enough for a quick check).

### Step 2: Get consumer file key

The consumer file key comes from:
1. `--file` CLI argument (e.g. `--file GfHErcAjjw277iPunsZXCU`)
2. `consumerFileKey` field in `ds-config.json`
3. If neither: ask the user for the consumer Figma file URL and parse the key from it

### Step 3: Run the audit

```bash
node scripts/consumer-audit.mjs --file <consumerFileKey> --report-html <output>.html
```

Run from the DS project root. The script:
1. Reads `figma-vars.snapshot.json` as the DS ground truth
2. Fetches consumer file variables from Figma REST API (`GET /v1/files/:key/variables/local`)
3. Compares linked library copy tokens against DS snapshot → SYNCED / PENDING / STALE
4. Compares linked tokens against local override collection → LOCAL
5. Writes `consumer-audit-report.json` (machine-readable)
6. Writes `<output>.html` (visual report)

If `--report-html` is not specified, default to `consumer-audit-report.html`.

### Step 4: Report results

Print a summary:
```
✅ Synced:  N tokens match DS
⏳ Pending: N tokens need a Figma library update in the consumer file
🗑  Stale:   N tokens removed from DS but still in consumer's linked copy
🎨 Local:   N tokens overridden in the consumer file
```

Then: `🌐 HTML report → <output>.html`

If there are PENDING or STALE tokens, explain the action:
- **PENDING**: The consumer designer must accept the library update in Figma (File menu → Libraries → Update)
- **STALE**: The tokens will be removed automatically when the designer accepts the next library update

---

## Cache Behaviour

Consumer variables are cached in `consumer-vars-cache.<fileKey>.json` in the DS project root. Cache is used by default to avoid hammering the Figma API. Pass `--fresh` to re-fetch.

The DS file name and consumer file name are also cached here (as `_dsFileName` and `_consumerFileName`) to avoid repeated API calls. Both can be overridden in `ds-config.json` via `dsFileName` and `consumerFileName`.

---

## Key Technical Notes

**Library detection via `remote` flag** — The ONLY reliable way to detect DS library linkage is the Figma Variables REST API. Each collection has a `remote` flag:
- `remote: false` → local to the consumer file (brand overrides)
- `remote: true` → from a linked library (the DS)

Never infer library linkage from component names or page structure — consumer files wrap DS instances in local components and wrapper names reveal nothing.

**Ghost collections** — Multiple Figma collections can share the same name (e.g. `"Breakpoint"`). A ghost collection (1–5 vars) with stale modes is excluded from mode columns automatically. Collections with ≤5 variables are ignored when building mode columns.

**Mode column filtering** — If a collection has modes that don't appear in ≥1% of its tokens, those modes are hidden from the HTML table to reduce noise.
