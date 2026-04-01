# Design: Data Import Tab for Security Metrics Report

**Date:** 2026-03-31
**Status:** Approved
**File affected:** `metrics-report/security-metrics-report.html`

---

## Overview

Add a **Data Import** tab to the existing `security-metrics-report.html` one-pager. Users drag CSV exports from known security tools into named drop zones (buckets), review extracted metric values, and apply them to the report config. No new files, no server, no build step — everything stays in the single HTML file.

---

## Section 1: Overall Structure

A persistent top bar replaces (or wraps) the existing header. It contains:

- Report title (left)
- Tab switcher: **Data Import** | **Report** (center)
- Action buttons: **Print / PDF**, **Reset**, **Get Zip** (right)

Switching tabs swaps the full content area below the bar. The existing left config panel + right preview becomes the **Report** tab content, unchanged. The new **Data Import** tab occupies the same content area when active.

---

## Section 2: Data Import Tab — Bucket Grid

The import tab renders a grid of named drop zones, one per configured tool. Initial tools:

- BitSight Findings
- Qualys Findings
- ArmorCode
- SDElements

### Bucket States

| State | Appearance |
|-------|-----------|
| **Empty** | Dashed border, tool name heading, "Drop CSV here" label |
| **Loaded** | Filename shown, expandable list of extracted metric values. Each value displays as `"Label: value"` or `"Label: N/A (not implemented)"` for stubs. A ✕ button clears the bucket. An **Apply to Report** button is visible. |
| **Applied** | Subtle visual indicator (e.g. green checkmark or muted border) showing values were pushed to the report. |

### Persistence

Bucket contents are stored in `localStorage` under the key `smr_buckets_v1` as a JSON object keyed by bucket `id`. Each entry stores: `{ fileName, csvText, extracted, applied }` where `applied` is a boolean indicating whether the bucket's values have been pushed to the report. On page refresh, all loaded buckets are restored — buckets with `applied: true` show the applied indicator. **Reset** clears only the `smr_buckets_v1` key; the report state (sections, metrics, title, etc.) is not affected.

---

## Section 3: Mapping & Extractor Architecture

A `TOOL_BUCKETS` array at the top of the script block defines all tools and their metric extractors. Each entry:

```js
{
  id: "bitsight",           // unique identifier
  label: "BitSight Findings", // displayed as bucket heading
  extractors: {
    "Open Critical Findings": (rows, headers) => null, // stub
    "Overall BitSight Score": (rows, headers) => null, // stub
  }
}
```

Each extractor is a plain function receiving:
- `rows` — array of objects keyed by header name (parsed CSV)
- `headers` — array of column name strings

Return values:
- A number or string → used as the metric's `value` field. The extractor must return the raw numeric value (e.g. `87`, not `"87%"`); `metric.type` is never changed by Apply to Report — the existing type set in the report config is preserved.
- `null` → metric is marked N/A (`metric.na = true`)

**Stubs return `null`** and display as `"N/A (not implemented)"` in the bucket. This makes missing implementations visible without breaking the tool.

### Extending

- **New tool:** add one object to `TOOL_BUCKETS`
- **New metric for existing tool:** add one key to the tool's `extractors` map
- **Implement a stub:** replace `() => null` with real logic

---

## Section 4: Action Buttons

### Reset

- Shows a confirmation prompt before clearing
- Clears all `localStorage` bucket state
- Resets all buckets to empty in the UI

### Get Zip

- Works from either tab — no tab switch required
- Bundles all currently loaded CSV files into a zip
- Uses an inlined, minified copy of **JSZip** (~100KB) embedded directly in the HTML to preserve the self-contained, zero-dependency model
- Files named as originally dropped (e.g. `bitsight_export.csv`)
- Button is disabled / hidden when no buckets are loaded

### Print / PDF

- Unchanged from existing behavior
- If clicked while on the Data Import tab, switches to Report tab first, then triggers `window.print()`

---

## Section 5: Apply to Report Flow

When **Apply to Report** is clicked on a loaded bucket:

1. Walk every metric in the current report state (all sections)
2. For each metric, check if the bucket's `extractors` map contains a key matching the metric's `label`. **Matching is exact case-sensitive string equality after trimming leading/trailing whitespace from both sides.**
3. **Match + non-null value:** set `metric.value` to the extracted value, set `metric.na = false`
4. **Match + null value (stub):** set `metric.na = true`
5. **No match:** leave metric unchanged
6. Mark the bucket as `applied: true` in `localStorage`
7. Show a toast notification: e.g. `"Applied 3 values, 2 N/A, 4 unchanged"`
8. Switch to the Report tab so the user can review

Dropping a new CSV into a bucket that is already in the Applied state resets `applied` to `false` (both in memory and in `localStorage`) immediately — before the user clicks Apply again. The applied indicator clears as soon as the new file lands.

Re-applying a bucket (e.g. after dropping a new CSV into it) overwrites previously applied values using the same rules — if the new file's extractor is still a stub, `metric.na` remains `true`. Metrics not covered by the bucket remain unchanged.

If the report contains multiple metrics with the same label in different sections, Apply updates **all** of them. The toast notification counts by unique extractor key, not by number of metrics updated (e.g. if "Open Critical Findings" appears in two sections, it counts as 1 applied, not 2).

---

## CSV Parsing

Use a minimal RFC 4180-compliant parser built inline (no library). Requirements:
- Trim UTF-8 BOM (`\uFEFF`) from the start of the file before parsing — common in Excel exports
- Handle CRLF and LF line endings
- Handle quoted fields containing commas and embedded newlines
- First row is always treated as headers; subsequent rows become objects keyed by header name

---

## Out of Scope

- Auto-detection of tool type from CSV columns (user assigns by dropping into the correct bucket)
- UI for editing metric-to-extractor mappings (hardcoded in JS source)
- Multiple CSV files per bucket
- Cross-bucket metric deduplication
