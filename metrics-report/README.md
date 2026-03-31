# Security Metrics Report `v0.2.0`

A self-contained HTML tool for building, configuring, and exporting application security metrics reports. No server, build step, or dependencies required — open the file in any browser and start editing.

---

## Getting Started

1. Open `security-metrics-report.html` in a browser.
2. Update the **Report Title**, **Reporting Period**, and **Date** in the left config panel.
3. Edit the pre-loaded sections and metrics, or clear them and build your own.
4. Print or export to PDF using the **Print / PDF** button in the config panel header.

---

## Interface Overview

| Panel | Purpose |
|-------|---------|
| **Left — Config** | Edit report metadata, sections, metrics, and import/export JSON |
| **Right — Preview** | Live preview of the rendered report |

---

## Sections

Sections group related metrics under a named heading with optional commentary. Each section appears in the report with its title, a commentary paragraph, and a metric grid below.

### Adding a section

Click **+ Add Section** at the bottom of the config panel. Each section has:

- **Name** — displayed as the section heading in the report
- **Commentary** — a short narrative paragraph rendered below the heading in both preview and print

### Removing a section

Click the **✕** button on the section card. This also removes all metrics within that section.

---

## Metrics

Each metric card has the following fields:

| Field | Description |
|-------|-------------|
| **Name** | Label shown in the report |
| **Value** | The metric's current value |
| **Type** | `Number`, `Percentage`, `Text`, or `Table` |
| **Higher is better** | Checkbox — controls how the red threshold is evaluated. Unchecked = higher is worse (e.g. open findings). Checked = higher is better (e.g. coverage %). |
| **Red threshold** | Value at which the metric turns red. Leave blank to disable. |
| **N/A toggle** | Available on numeric metrics only. Marks the metric as not applicable for this period. |
| **Hide toggle** | Excludes the metric from the report without deleting it. |

### Status logic

Status is either **Red** or **Green**. There is no yellow state.

For metrics where **Higher is worse** (e.g. open findings, MTTR):

```
value >= red threshold → Red
otherwise             → Green
```

For metrics where **Higher is better** (e.g. SLA compliance, scanner coverage):

```
value <= red threshold → Red
otherwise             → Green
```

`Text` and `Table` metrics have no threshold and always render without a status colour.

### Table metrics

Select type **Table** to display structured data instead of a single value. The metric renders as a table spanning the full report width.

Paste directly from Excel or Google Sheets — copied cells use tab-separated format which is handled automatically. The first row becomes the column headers. CSV (comma-separated) is also accepted as a fallback.

---

## JSON Import / Export

The config panel includes a collapsible **JSON Import / Export** section. Use it to save and restore report configurations, share them with teammates, or version-control them.

### Exporting

Click **Export** to generate a JSON snapshot of the current state.

### Importing

Paste a JSON config into the text area and click **Import**. The current state is fully replaced.

### Format

Metrics are nested inside their section — there are no cross-reference IDs.

```json
{
  "title": "Product Security Metrics — Q1 2026",
  "period": "Q1 2026",
  "date": "2026-03-31",
  "sections": [
    {
      "name": "Vulnerability Management",
      "commentary": "Tracks open findings by severity and remediation performance against defined SLAs.",
      "metrics": [
        {
          "label": "Open Critical Findings",
          "value": 3,
          "type": "number",
          "higherIsBetter": false,
          "redThreshold": 1,
          "hidden": false,
          "na": false
        },
        {
          "label": "SLA Compliance",
          "value": 87,
          "type": "percentage",
          "higherIsBetter": true,
          "redThreshold": 70,
          "hidden": false,
          "na": false
        }
      ]
    },
    {
      "name": "Program Health",
      "commentary": "Measures the breadth and maturity of the security program.",
      "metrics": [
        {
          "label": "Scanner Coverage",
          "value": 94,
          "type": "percentage",
          "higherIsBetter": true,
          "redThreshold": 75,
          "hidden": false,
          "na": false
        },
        {
          "label": "Notes",
          "value": "Champion training scheduled for April.",
          "type": "text",
          "higherIsBetter": false,
          "redThreshold": "",
          "hidden": false,
          "na": false
        }
      ]
    }
  ]
}
```

---

## Example Configurations

### Minimal single-section report

```json
{
  "title": "AppSec Metrics — March 2026",
  "period": "March 2026",
  "date": "2026-03-31",
  "sections": [
    {
      "name": "Overview",
      "commentary": "Snapshot of key security indicators for the month.",
      "metrics": [
        {
          "label": "Open Criticals",
          "value": 0,
          "type": "number",
          "higherIsBetter": false,
          "redThreshold": 1,
          "hidden": false,
          "na": false
        },
        {
          "label": "Open Highs",
          "value": 5,
          "type": "number",
          "higherIsBetter": false,
          "redThreshold": 15,
          "hidden": false,
          "na": false
        },
        {
          "label": "Pipeline Coverage",
          "value": 91,
          "type": "percentage",
          "higherIsBetter": true,
          "redThreshold": 75,
          "hidden": false,
          "na": false
        }
      ]
    }
  ]
}
```

### Metric marked N/A

Set `"na": true` to display the metric as **N/A** without removing it from the config. Only applies to `number` and `percentage` types.

```json
{
  "label": "Pentest Findings",
  "value": "",
  "type": "number",
  "higherIsBetter": false,
  "redThreshold": 5,
  "hidden": false,
  "na": true
}
```

### Hiding a metric without deleting it

Set `"hidden": true` to exclude a metric from the preview and print output while keeping it in the config.

```json
{
  "label": "Bug Bounty Submissions",
  "value": 12,
  "type": "number",
  "higherIsBetter": false,
  "redThreshold": 20,
  "hidden": true,
  "na": false
}
```

### Table metric

Paste tab-separated data (e.g. copied from Excel) into the value field. First row is headers.

```json
{
  "label": "Finding Breakdown",
  "value": "Severity\tCount\nCritical\t3\nHigh\t12\nMedium\t45",
  "type": "table",
  "higherIsBetter": false,
  "redThreshold": "",
  "hidden": false,
  "na": false
}
```

---

## Printing

Click **Print / PDF** in the config panel header to open the browser print dialog. The config panel is automatically hidden in the output. For best results, use **Save as PDF** with margins set to **Default** or **None**.

---

## Files

| File | Description |
|------|-------------|
| `security-metrics-report.html` | Self-contained report builder — no other files needed |
| `README.md` | This guide |
