# rms-figma-sync

A Claude Code skill that audits whether a **consumer Figma file** is in sync with your Design System library.

Compares the DS token snapshot against the consumer file's linked library copy and local brand overrides. Outputs an interactive HTML report with collection tabs, per-status filter buttons, and a color-coded token table.

---

## What it checks

| Status | Meaning |
|--------|---------|
| ✅ **Synced** | Token is in both DS and consumer's linked library, same value |
| ⏳ **Pending** | Token added to DS but not yet in consumer's linked copy — needs a Figma library update |
| 🗑 **Stale** | Token removed from DS but still in consumer's old linked copy — will disappear on next update |
| 🎨 **Local** | Token has a local brand override in the consumer file |

---

## Setup

**1. Add the skill to Claude Code**

Copy `rms-figma-sync.md` to your `~/.claude/commands/` directory:

```bash
cp rms-figma-sync.md ~/.claude/commands/
```

**2. Add the scripts to your DS project**

Copy `consumer-audit.mjs` and `report-html.mjs` into your DS project's `scripts/` directory (alongside your other rms-parity scripts).

**3. Configure `ds-config.json`**

The script reads from your DS project root. Required fields:

```json
{
  "figmaFileKey": "YOUR_DS_FILE_KEY",
  "figma": {
    "colorCollection": "Theme",
    "primitivePrefix": "primitives/",
    "modes": [
      { "name": "Light", "snapshotKey": "light" },
      { "name": "Dark",  "snapshotKey": "dark"  }
    ]
  },
  "paths": {
    "snapshotVars": "src/styles/figma-vars.snapshot.json"
  }
}
```

Optional name overrides (bypasses Figma API lookup):
```json
{
  "dsFileName": "My Design System",
  "consumerFileName": "My Product — Mobile"
}
```

**4. Set your Figma token**

Add to `.env` in your DS project root:

```
FIGMA_TOKEN=your_figma_personal_access_token
```

The token needs at least viewer access to the consumer Figma file.

---

## Usage

### Via Claude Code skill

```
/rms-figma-sync
```

Claude will ask for the consumer file URL if not already configured, then run the audit and open the HTML report.

### Directly from the command line

Run from your **DS project root**:

```bash
node scripts/consumer-audit.mjs --file <consumerFileKey> --report-html output.html
node scripts/consumer-audit.mjs --file GfHErcAjjw277iPunsZXCU --report-html review.html
node scripts/consumer-audit.mjs --file GfHErcAjjw277iPunsZXCU --fresh   # bypass cache
```

The consumer file key is the path segment after `/design/` or `/file/` in the Figma URL.

---

## Output

- **`<output>.html`** — Interactive report: collection tabs, Synced/Pending/Stale/Local filter buttons, token search, sticky column headers
- **`consumer-audit-report.json`** — Machine-readable summary
- **`consumer-vars-cache.<fileKey>.json`** — Cached Figma variables (bypassed with `--fresh`)

---

## Sister skill

**[rms-figma-code-parity](https://github.com/rafaelmatosds/rms-figma-code-parity)** — audits whether your CSS codebase faithfully implements the DS (14 gates: token values, alias chains, structure, unused vars, hardcoded values, and more).
