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

Bucket contents (file name, raw CSV text, extracted values) are stored in `localStorage`. A page refresh restores all loaded buckets. **Reset** clears `localStorage` and resets all buckets to empty state.

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
- A number, percentage value, or string → used as the metric value
- `null` → metric is marked N/A

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
2. For each metric, check if the bucket's `extractors` map contains a key matching the metric's `label`
3. **Match + non-null value:** set `metric.value` to the extracted value, set `metric.na = false`
4. **Match + null value (stub):** set `metric.na = true`
5. **No match:** leave metric unchanged
6. Show a toast notification: e.g. `"Applied 3 values, 2 N/A, 4 unchanged"`
7. Switch to the Report tab so the user can review

Re-applying a bucket (e.g. after dropping a new file) overwrites previously applied values. Metrics not covered by the bucket remain unchanged.

---

## Out of Scope

- Auto-detection of tool type from CSV columns (user assigns by dropping into the correct bucket)
- UI for editing metric-to-extractor mappings (hardcoded in JS source)
- Multiple CSV files per bucket
- Cross-bucket metric deduplication
