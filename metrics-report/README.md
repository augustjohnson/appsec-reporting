# Security Metrics Report

A self-contained HTML tool for building, configuring, and exporting application security metrics reports. No server, build step, or dependencies required — open the file in any browser and start editing.

---

## Getting Started

1. Open `security-metrics-report.html` in a browser.
2. Update the **Report Title**, **Reporting Period**, and **Date** in the left config panel.
3. Edit the pre-loaded sections and metrics, or clear them and build your own.
4. Print or export to PDF using the **Print / PDF** button.

---

## Interface Overview

The tool is split into two panels:

| Panel | Purpose |
|-------|---------|
| **Left — Config** | Edit report metadata, sections, metrics, and import/export JSON |
| **Right — Preview** | Live preview of the rendered report |

---

## Sections

Sections group related metrics under a named heading with optional commentary. Each section appears in the report with its title, commentary paragraph, and metric grid.

### Adding a section

Click **+ Add Section** at the bottom of the config panel. Each section has:

- **Name** — displayed as the section heading in the report
- **Commentary** — a short narrative paragraph rendered below the heading, visible in both the preview and print output

### Adding metrics to a section

Each section has its own **+ Add Metric** button. Metrics belong to the section they were created in.

### Removing a section

Click the **✕** button on the section card. This also removes all metrics within that section.

---

## Metrics

Each metric card has the following fields:

| Field | Description |
|-------|-------------|
| **Name** | Label shown in the report |
| **Value** | The metric's current value |
| **Type** | `Number`, `Percentage`, or `Text` |
| **Direction** | `Higher = Worse` or `Higher = Better` — controls how thresholds are evaluated |
| **Red threshold** | Value at which the metric turns red |
| **Yellow threshold** | Value at which the metric turns yellow |
| **N/A toggle** | Marks the metric as not applicable for this period |
| **Hide toggle** | Excludes the metric from the report without deleting it |

### Status logic

For **Higher = Worse** metrics (e.g. open findings, MTTR):

```
value >= red threshold   → Red
value >= yellow threshold → Yellow
otherwise               → Green
```

For **Higher = Better** metrics (e.g. SLA compliance, scanner coverage):

```
value <= red threshold   → Red
value <= yellow threshold → Yellow
otherwise               → Green
```

**Text** metrics have no thresholds and always render with a neutral style spanning the full row width.

---

## JSON Import / Export

The config panel includes a collapsible **JSON Import / Export** section. This lets you save and restore complete report configurations, share them with teammates, or version-control them alongside your other security documentation.

### Exporting

Click **Export** to generate a JSON snapshot of the current state. The JSON includes the report title, period, date, all sections, and all metrics.

### Importing

Paste a JSON config into the text area and click **Import** to load it. The current state is fully replaced.

### Format

```json
{
  "title": "Product Security Metrics — Q1 2026",
  "period": "Q1 2026",
  "date": "2026-03-31",
  "sections": [
    {
      "id": 0,
      "name": "Vulnerability Management",
      "commentary": "Tracks open findings by severity and remediation performance against defined SLAs."
    },
    {
      "id": 1,
      "name": "Program Health",
      "commentary": "Measures the breadth and maturity of the security program."
    }
  ],
  "metrics": [
    {
      "sectionId": 0,
      "label": "Open Critical Findings",
      "value": 3,
      "type": "number",
      "direction": "higher-is-worse",
      "redThreshold": 1,
      "yellowThreshold": 1,
      "hidden": false,
      "na": false
    },
    {
      "sectionId": 0,
      "label": "SLA Compliance",
      "value": 87,
      "type": "percentage",
      "direction": "higher-is-better",
      "redThreshold": 70,
      "yellowThreshold": 85,
      "hidden": false,
      "na": false
    },
    {
      "sectionId": 1,
      "label": "Scanner Coverage",
      "value": 94,
      "type": "percentage",
      "direction": "higher-is-better",
      "redThreshold": 75,
      "yellowThreshold": 90,
      "hidden": false,
      "na": false
    },
    {
      "sectionId": 1,
      "label": "Notes",
      "value": "Champion training scheduled for April. New SAST tool onboarding in progress.",
      "type": "text",
      "direction": "higher-is-worse",
      "redThreshold": "",
      "yellowThreshold": "",
      "hidden": false,
      "na": false
    }
  ]
}
```

> **Tip:** `sectionId` in each metric must match the `id` of an existing section. Metrics referencing a missing section ID will be silently ignored when rendered.

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
      "id": 0,
      "name": "Overview",
      "commentary": "Snapshot of key security indicators for the month."
    }
  ],
  "metrics": [
    {
      "sectionId": 0,
      "label": "Open Criticals",
      "value": 0,
      "type": "number",
      "direction": "higher-is-worse",
      "redThreshold": 1,
      "yellowThreshold": 1,
      "hidden": false,
      "na": false
    },
    {
      "sectionId": 0,
      "label": "Open Highs",
      "value": 5,
      "type": "number",
      "direction": "higher-is-worse",
      "redThreshold": 15,
      "yellowThreshold": 8,
      "hidden": false,
      "na": false
    },
    {
      "sectionId": 0,
      "label": "Pipeline Coverage",
      "value": 91,
      "type": "percentage",
      "direction": "higher-is-better",
      "redThreshold": 75,
      "yellowThreshold": 85,
      "hidden": false,
      "na": false
    }
  ]
}
```

### Report with a metric marked N/A

Set `"na": true` on any metric to display it as **N/A** in the report without removing it from the config. Useful when a metric is not yet instrumented for the current period.

```json
{
  "sectionId": 0,
  "label": "Pentest Findings",
  "value": "",
  "type": "number",
  "direction": "higher-is-worse",
  "redThreshold": 5,
  "yellowThreshold": 2,
  "hidden": false,
  "na": true
}
```

### Hiding a metric without deleting it

Set `"hidden": true` to exclude a metric from the report preview and print output while keeping it in the config. Useful for tracking metrics you are not yet ready to report on.

```json
{
  "sectionId": 1,
  "label": "Bug Bounty Submissions",
  "value": 12,
  "type": "number",
  "direction": "higher-is-worse",
  "redThreshold": 20,
  "yellowThreshold": 10,
  "hidden": true,
  "na": false
}
```

---

## Printing

Click **Print / PDF** in either panel to open the browser print dialog. The config panel is automatically hidden in the print output. For best results, use **Save as PDF** in the print dialog with margins set to **Default** or **None**.

---

## Files

| File | Description |
|------|-------------|
| `security-metrics-report.html` | Self-contained report builder — no other files needed |
| `README.md` | This guide |
