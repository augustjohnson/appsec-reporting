# Data Import Tab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Data Import tab to `security-metrics-report.html` with drag-and-drop CSV buckets per tool, inline metric extraction, Apply to Report, Reset, and Get Zip.

**Architecture:** All changes live in the single HTML file. A persistent top bar wraps the page and hosts the tab switcher plus action buttons. The existing two-panel layout becomes the Report tab content. A new full-width Import tab hosts a bucket grid. Bucket state persists in `localStorage` under `smr_buckets_v1`. JSZip 3.10.1 is inlined for zip generation.

**Tech Stack:** Vanilla JS, HTML/CSS, JSZip 3.10.1 (inlined)

---

## File Structure

Only one file is modified throughout this entire plan:

- **Modify:** `metrics-report/security-metrics-report.html`

All tasks below add to or edit this file.

---

## Note on Testing

This is a zero-dependency browser-only tool with no test runner. For pure functions (CSV parser, apply logic), verification is done via browser console snippets. For UI features, manual step-by-step verification is described. Follow all verification steps before committing.

---

## Task 1: Top Bar + Tab Structure

**Files:**
- Modify: `metrics-report/security-metrics-report.html`

Restructure the page layout to support a persistent top bar and two swappable tab content areas. The existing two-panel `.app` div is wrapped inside `#tab-report`; a new `#tab-import` div is added for the import tab content.

- [ ] **Step 1: Open the file and read the current structure**

Confirm the current body opens with `<div class="app">` on line ~719, and the `.config-header` contains the Print/PDF button on line ~725.

- [ ] **Step 2: Update `body` CSS to flex column**

Find and replace the `body` rule in the CSS (around line 29):

```css
/* BEFORE */
body {
  font-family: var(--font-body);
  background: var(--bg);
  color: var(--text);
  line-height: 1.5;
  font-size: 15px;
}

/* AFTER */
body {
  font-family: var(--font-body);
  background: var(--bg);
  color: var(--text);
  line-height: 1.5;
  font-size: 15px;
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}
```

- [ ] **Step 3: Update `.app` CSS to use flex instead of fixed height**

Find and replace the `.app` rule (around line 38):

```css
/* BEFORE */
.app {
  display: flex;
  height: 100vh;
  overflow: hidden;
}

/* AFTER */
.app {
  display: flex;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}
```

- [ ] **Step 4: Add top bar and tab CSS**

Add the following CSS block immediately before the closing `</style>` tag (before line 715):

```css
  /* Top bar */
  .topbar {
    height: 52px;
    background: var(--panel-bg);
    border-bottom: 1px solid var(--border);
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 20px;
    flex-shrink: 0;
    gap: 16px;
    z-index: 10;
  }

  .topbar-title {
    font-size: 15px;
    font-weight: 700;
    letter-spacing: -0.01em;
    white-space: nowrap;
    color: var(--text);
  }

  .tab-switcher {
    display: flex;
    gap: 2px;
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: 7px;
    padding: 3px;
  }

  .tab-btn {
    padding: 5px 16px;
    border: none;
    border-radius: 5px;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-body);
    cursor: pointer;
    background: transparent;
    color: var(--text-muted);
    transition: all 0.15s;
  }

  .tab-btn:hover { color: var(--text); }

  .tab-btn.tab-active {
    background: var(--panel-bg);
    color: var(--text);
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }

  .topbar-actions {
    display: flex;
    gap: 8px;
    align-items: center;
  }

  .tab-content {
    flex: 1;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
```

- [ ] **Step 5: Update print media CSS**

Find the `@media print` block (around line 679) and update it:

```css
/* BEFORE */
@media print {
  body { background: white; }
  .config-panel { display: none !important; }
  .app { display: block; height: auto; }
  .preview-panel {
    padding: 0;
    overflow: visible;
  }
  /* ... rest of print rules ... */
}

/* AFTER — replace the entire @media print block with: */
@media print {
  body { background: white; display: block; height: auto; overflow: visible; }
  .topbar { display: none !important; }
  .tab-content { display: block !important; overflow: visible; }
  #tab-import { display: none !important; }
  #tab-report { display: block !important; }
  .config-panel { display: none !important; }
  .app { display: block; height: auto; }
  .preview-panel {
    padding: 0;
    overflow: visible;
  }
  .report {
    box-shadow: none;
    border-radius: 0;
    max-width: 100%;
  }
  .report-header { padding: 20px 30px 16px; }
  .report-body { padding: 20px 30px 30px; }
  .report-metric { break-inside: avoid; }
  .report-section { break-inside: avoid; }
}
```

- [ ] **Step 6: Restructure the HTML body**

Replace the entire body HTML (from `<div class="app">` to its closing `</div>` — around lines 719–769) with:

```html
<!-- Top Bar -->
<div class="topbar">
  <span class="topbar-title">Security Metrics Report</span>
  <div class="tab-switcher">
    <button class="tab-btn tab-active" data-tab="import" onclick="switchTab('import')">Data Import</button>
    <button class="tab-btn" data-tab="report" onclick="switchTab('report')">Report</button>
  </div>
  <div class="topbar-actions">
    <button class="btn btn-sm" onclick="printReport()">Print / PDF</button>
    <button class="btn btn-sm" onclick="resetAll()">Reset</button>
    <button class="btn btn-sm" id="btnZip" disabled onclick="getZip()">Get Zip</button>
  </div>
</div>

<!-- Import Tab -->
<div id="tab-import" class="tab-content">
  <div id="bucketGrid" class="bucket-grid"></div>
</div>

<!-- Report Tab (hidden by default) -->
<div id="tab-report" class="tab-content" style="display:none">
  <div class="app">
    <!-- Config Panel -->
    <div class="config-panel">
      <div class="config-header">
        <h2>Metrics Config</h2>
        <span id="versionBadge" style="font-size:11px;font-weight:600;color:var(--text-muted);letter-spacing:0.03em;"></span>
      </div>
      <div class="config-body">
        <!-- JSON Import/Export -->
        <div class="json-section">
          <button class="json-toggle" onclick="toggleJson()">
            <span class="arrow" id="jsonArrow">&#9654;</span>
            JSON Import / Export
          </button>
          <div class="json-body" id="jsonBody">
            <textarea id="jsonArea" placeholder='Paste JSON config here to load, or click "Export" to generate...'></textarea>
            <div class="json-actions">
              <button class="btn btn-sm btn-primary" onclick="importJson()">Import</button>
              <button class="btn btn-sm" onclick="exportJson()">Export</button>
            </div>
          </div>
        </div>

        <!-- Report Title -->
        <div class="report-title-config">
          <label>Report Title</label>
          <input type="text" id="reportTitle" value="Product Security Metrics Report" oninput="render()">
          <div class="report-date-row">
            <div>
              <label>Reporting Period</label>
              <input type="text" id="reportPeriod" placeholder="e.g. Q1 2026" oninput="render()">
            </div>
            <div>
              <label>Date</label>
              <input type="date" id="reportDate" oninput="render()">
            </div>
          </div>
        </div>

        <!-- Sections -->
        <div id="metricsList"></div>
        <button class="add-section-btn" onclick="addSection()">+ Add Section</button>
      </div>
    </div>

    <!-- Preview Panel -->
    <div class="preview-panel">
      <div class="report" id="reportPreview"></div>
    </div>
  </div>
</div>
```

- [ ] **Step 7: Add `switchTab` and update `printReport` in the script block**

Find `function printReport() { window.print(); }` and replace it with:

```js
  function switchTab(name) {
    document.getElementById('tab-import').style.display = name === 'import' ? 'flex' : 'none';
    document.getElementById('tab-report').style.display  = name === 'report'  ? 'flex' : 'none';
    document.querySelectorAll('.tab-btn').forEach(b => {
      b.classList.toggle('tab-active', b.dataset.tab === name);
    });
  }

  function printReport() {
    switchTab('report');
    setTimeout(() => window.print(), 100);
  }
```

- [ ] **Step 8: Verify manually**

Open `security-metrics-report.html` in a browser. Confirm:
- Top bar visible with "Data Import | Report" tabs and three buttons
- Data Import tab is active by default (Import tab content area visible, empty bucket grid placeholder visible)
- Clicking "Report" tab shows the existing config/preview panels
- Clicking "Print / PDF" from the Import tab switches to Report then opens print dialog
- Get Zip button is visually disabled

- [ ] **Step 9: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add top bar and tab structure to metrics report"
```

---

## Task 2: TOOL_BUCKETS Config + parseCSV

**Files:**
- Modify: `metrics-report/security-metrics-report.html` (script block)

Add the tool bucket configuration array and the RFC 4180 CSV parser. These are pure data/logic — no DOM interaction yet.

- [ ] **Step 1: Verify parseCSV doesn't exist yet**

Open browser console and type `parseCSV`. Confirm `ReferenceError`.

- [ ] **Step 2: Add TOOL_BUCKETS and parseCSV**

Add the following at the very top of the `<script>` block, before `const VERSION`:

```js
  // ─── Tool Bucket Configuration ───────────────────────────────────────────
  // Add tools here. Each extractor receives (rows, headers) and returns a
  // raw value (number or string) or null for N/A / not implemented.
  const TOOL_BUCKETS = [
    {
      id: 'bitsight',
      label: 'BitSight Findings',
      extractors: {
        'Overall BitSight Score': (rows, headers) => null, // stub
        'Open Critical Findings': (rows, headers) => null, // stub
      }
    },
    {
      id: 'qualys',
      label: 'Qualys Findings',
      extractors: {
        'Open Critical Findings': (rows, headers) => null, // stub
        'Open High Findings':     (rows, headers) => null, // stub
        'SLA Compliance':         (rows, headers) => null, // stub
      }
    },
    {
      id: 'armorcode',
      label: 'ArmorCode',
      extractors: {
        'Open Critical Findings': (rows, headers) => null, // stub
        'Open High Findings':     (rows, headers) => null, // stub
      }
    },
    {
      id: 'sdelements',
      label: 'SDElements',
      extractors: {
        'Task Completion Rate': (rows, headers) => null, // stub
        'Overdue Tasks':        (rows, headers) => null, // stub
      }
    }
  ];

  // ─── CSV Parser (RFC 4180) ────────────────────────────────────────────────
  function parseCSV(text) {
    text = text.replace(/^\uFEFF/, '');                      // strip BOM
    text = text.replace(/\r\n/g, '\n').replace(/\r/g, '\n'); // normalize CRLF

    const rows = [];
    let i = 0;

    function parseField() {
      if (text[i] === '"') {
        i++;
        let field = '';
        while (i < text.length) {
          if (text[i] === '"' && text[i + 1] === '"') { field += '"'; i += 2; }
          else if (text[i] === '"') { i++; break; }
          else field += text[i++];
        }
        return field;
      }
      let field = '';
      while (i < text.length && text[i] !== ',' && text[i] !== '\n') field += text[i++];
      return field;
    }

    while (i < text.length) {
      const fields = [];
      while (i < text.length && text[i] !== '\n') {
        fields.push(parseField());
        if (text[i] === ',') i++;
      }
      if (text[i] === '\n') i++;
      if (fields.length > 1 || fields[0] !== '') rows.push(fields);
    }

    if (rows.length === 0) return { headers: [], rows: [] };
    const headers = rows[0];
    const dataRows = rows.slice(1).map(r => {
      const obj = {};
      headers.forEach((h, idx) => { obj[h] = r[idx] ?? ''; });
      return obj;
    });
    return { headers, rows: dataRows };
  }
```

- [ ] **Step 3: Verify parseCSV in browser console**

Open the page, open DevTools console, and run:

```js
// BOM stripping
let r = parseCSV('\uFEFFName,Count\nFoo,1');
console.assert(r.headers[0] === 'Name', 'FAIL: BOM not stripped');
console.assert(r.rows[0]['Count'] === '1', 'FAIL: basic value');

// Quoted field with comma
r = parseCSV('"Foo, Bar",Baz\n"Hello, World",42');
console.assert(r.rows[0]['Foo, Bar'] === 'Hello, World', 'FAIL: quoted field with comma');

// CRLF
r = parseCSV('A,B\r\n1,2\r\n3,4');
console.assert(r.rows.length === 2, 'FAIL: CRLF row count');

// Double-quoted quote
r = parseCSV('A\n"He said ""hi"""');
console.assert(r.rows[0]['A'] === 'He said "hi"', 'FAIL: escaped quote');

console.log('All parseCSV tests passed');
```

All four asserts must pass with no errors.

- [ ] **Step 4: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add TOOL_BUCKETS config and RFC 4180 CSV parser"
```

---

## Task 3: Bucket State Management

**Files:**
- Modify: `metrics-report/security-metrics-report.html` (script block)

Add the in-memory `bucketState` object and localStorage persistence helpers.

- [ ] **Step 1: Add state and helpers**

Add the following block immediately after the `parseCSV` function:

```js
  // ─── Bucket State ─────────────────────────────────────────────────────────
  // Shape: { [bucketId]: { fileName, csvText, extracted, applied } }
  let bucketState = {};

  function saveBucketState() {
    localStorage.setItem('smr_buckets_v1', JSON.stringify(bucketState));
  }

  function loadBucketState() {
    try {
      const saved = localStorage.getItem('smr_buckets_v1');
      if (saved) bucketState = JSON.parse(saved);
    } catch (e) {
      bucketState = {};
    }
  }
```

- [ ] **Step 2: Verify persistence in browser console**

```js
bucketState['bitsight'] = { fileName: 'test.csv', csvText: 'A,B\n1,2', extracted: { 'Overall BitSight Score': 720 }, applied: false };
saveBucketState();
// Check storage written:
console.assert(localStorage.getItem('smr_buckets_v1') !== null, 'FAIL: nothing saved');
// Simulate reload:
bucketState = {};
loadBucketState();
console.assert(bucketState['bitsight'].fileName === 'test.csv', 'FAIL: state not restored');
console.log('Bucket state persistence OK');
// Clean up:
delete bucketState['bitsight'];
saveBucketState();
```

- [ ] **Step 3: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add bucket state with localStorage persistence"
```

---

## Task 4: Import Tab Rendering + Bucket Grid CSS

**Files:**
- Modify: `metrics-report/security-metrics-report.html`

Add CSS for the bucket grid and states, and the `renderImportTab()` function.

- [ ] **Step 1: Add bucket grid CSS**

Add the following block inside `<style>`, just before the closing `</style>` tag:

```css
  /* Bucket Grid */
  .bucket-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 20px;
    padding: 32px;
    align-content: start;
    overflow-y: auto;
    flex: 1;
  }

  .bucket {
    border: 2px dashed var(--border);
    border-radius: 10px;
    padding: 24px;
    background: var(--panel-bg);
    transition: border-color 0.15s, background 0.15s;
    min-height: 160px;
    display: flex;
    flex-direction: column;
  }

  .bucket-header {
    font-size: 14px;
    font-weight: 700;
    margin-bottom: 12px;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .bucket-applied-badge {
    font-size: 11px;
    font-weight: 600;
    color: var(--green);
  }

  .bucket.is-applied {
    border-color: var(--green);
    border-style: solid;
  }

  .bucket-empty-label {
    color: var(--text-muted);
    font-size: 13px;
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .bucket.drag-over {
    border-color: var(--accent);
    border-style: solid;
    background: rgba(37, 99, 235, 0.04);
  }

  .bucket-file-row {
    font-size: 12px;
    font-weight: 600;
    color: var(--text-muted);
    margin-bottom: 10px;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .bucket-metrics-list {
    list-style: none;
    font-size: 13px;
    flex: 1;
    margin-bottom: 12px;
  }

  .bucket-metrics-list li {
    padding: 5px 0;
    border-bottom: 1px solid var(--border);
    display: flex;
    justify-content: space-between;
    gap: 8px;
  }

  .bucket-metrics-list li:last-child { border-bottom: none; }

  .bucket-metric-label {
    color: var(--text-muted);
    font-size: 12px;
  }

  .bucket-metric-value {
    font-weight: 600;
    font-family: var(--font-mono);
    font-size: 12px;
  }

  .bucket-metric-value.is-na {
    color: var(--text-muted);
    font-weight: 400;
    font-style: italic;
  }

  .bucket-actions {
    display: flex;
    gap: 8px;
    margin-top: auto;
    padding-top: 12px;
  }
```

- [ ] **Step 2: Add `renderImportTab` and `updateZipButton` functions**

Add the following functions in the script block, after the `loadBucketState` function:

```js
  function updateZipButton() {
    const hasFiles = Object.keys(bucketState).length > 0;
    document.getElementById('btnZip').disabled = !hasFiles;
  }

  function renderImportTab() {
    const grid = document.getElementById('bucketGrid');
    if (!grid) return;
    grid.innerHTML = TOOL_BUCKETS.map(bucket => {
      const state = bucketState[bucket.id];
      const isLoaded  = !!state;
      const isApplied = isLoaded && state.applied;

      const headerHtml = `
        <div class="bucket-header">
          <span>${escHtml(bucket.label)}</span>
          ${isApplied ? '<span class="bucket-applied-badge">✓ Applied</span>' : ''}
        </div>`;

      const dragAttrs = `
        ondragover="bucketDragOver(event,'${bucket.id}')"
        ondragleave="bucketDragLeave(event,'${bucket.id}')"
        ondrop="bucketDrop(event,'${bucket.id}')"`;

      if (!isLoaded) {
        return `
          <div class="bucket" id="bucket-${bucket.id}" ${dragAttrs}>
            ${headerHtml}
            <div class="bucket-empty-label">Drop CSV here</div>
          </div>`;
      }

      const metricsHtml = Object.entries(bucket.extractors).map(([label, _fn]) => {
        const val  = state.extracted[label];
        const isNa = val === null || val === undefined;
        return `
          <li>
            <span class="bucket-metric-label">${escHtml(label)}</span>
            <span class="bucket-metric-value ${isNa ? 'is-na' : ''}">
              ${isNa ? 'N/A (not implemented)' : escHtml(String(val))}
            </span>
          </li>`;
      }).join('');

      return `
        <div class="bucket ${isApplied ? 'is-applied' : ''}" id="bucket-${bucket.id}" ${dragAttrs}>
          ${headerHtml}
          <div class="bucket-file-row">
            <span>&#128196; ${escHtml(state.fileName)}</span>
            <button class="remove-btn" onclick="clearBucket('${bucket.id}')" title="Clear">&#10005;</button>
          </div>
          <ul class="bucket-metrics-list">${metricsHtml}</ul>
          <div class="bucket-actions">
            <button class="btn btn-sm btn-primary" onclick="applyBucket('${bucket.id}')">Apply to Report</button>
          </div>
        </div>`;
    }).join('');
    updateZipButton();
  }
```

- [ ] **Step 3: Verify bucket grid renders**

Open the page. The Data Import tab should now show 4 empty buckets (BitSight Findings, Qualys Findings, ArmorCode, SDElements) arranged in a grid with dashed borders and "Drop CSV here" labels.

Run in console to test a loaded state renders:
```js
bucketState['bitsight'] = {
  fileName: 'bitsight.csv',
  csvText: 'A,B\n1,2',
  extracted: { 'Overall BitSight Score': 720, 'Open Critical Findings': null },
  applied: false
};
renderImportTab();
// Expect: bitsight bucket shows filename, 720 for score, "N/A (not implemented)" for findings
// Clean up:
delete bucketState['bitsight'];
renderImportTab();
```

- [ ] **Step 4: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add bucket grid CSS and import tab rendering"
```

---

## Task 5: Drag-and-Drop File Handling

**Files:**
- Modify: `metrics-report/security-metrics-report.html` (script block)

Wire up drag events and the drop handler that reads the CSV file, runs extractors, and updates state.

- [ ] **Step 1: Add drag event handlers and file drop logic**

Add the following functions in the script block, after `renderImportTab`:

```js
  function bucketDragOver(e, bucketId) {
    e.preventDefault();
    document.getElementById('bucket-' + bucketId).classList.add('drag-over');
  }

  function bucketDragLeave(e, bucketId) {
    document.getElementById('bucket-' + bucketId).classList.remove('drag-over');
  }

  function bucketDrop(e, bucketId) {
    e.preventDefault();
    document.getElementById('bucket-' + bucketId).classList.remove('drag-over');
    const file = e.dataTransfer.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = ev => handleDrop(bucketId, file.name, ev.target.result);
    reader.readAsText(file);
  }

  function handleDrop(bucketId, fileName, csvText) {
    const bucket = TOOL_BUCKETS.find(b => b.id === bucketId);
    if (!bucket) return;
    const { rows, headers } = parseCSV(csvText);
    const extracted = {};
    Object.entries(bucket.extractors).forEach(([label, fn]) => {
      try { extracted[label] = fn(rows, headers); }
      catch (_e) { extracted[label] = null; }
    });
    bucketState[bucketId] = { fileName, csvText, extracted, applied: false };
    saveBucketState();
    renderImportTab();
  }

  function clearBucket(bucketId) {
    delete bucketState[bucketId];
    saveBucketState();
    renderImportTab();
  }
```

- [ ] **Step 2: Verify drag-and-drop manually**

Create a simple test CSV file (`test.csv`) with content:
```
Name,Value
Foo,1
Bar,2
```

1. Open the page. Go to Data Import tab.
2. Drag `test.csv` onto any bucket.
3. Confirm: bucket shows the filename and the extractor list (all N/A since stubs).
4. Click ✕ on the bucket. Confirm it returns to empty state.
5. Drag the file back in. Reload the page. Confirm the file is restored from localStorage.

- [ ] **Step 3: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add drag-and-drop CSV file handling for import buckets"
```

---

## Task 6: Apply to Report + Toast

**Files:**
- Modify: `metrics-report/security-metrics-report.html`

Add the toast notification UI element and CSS, then implement `applyBucket` which matches extracted values to report metrics by label and pushes them.

- [ ] **Step 1: Add toast CSS**

Add inside `<style>`, before `</style>`:

```css
  /* Toast */
  .toast {
    position: fixed;
    bottom: 24px;
    right: 24px;
    background: var(--text);
    color: white;
    padding: 10px 18px;
    border-radius: 8px;
    font-size: 13px;
    font-weight: 600;
    opacity: 0;
    transform: translateY(8px);
    transition: opacity 0.2s, transform 0.2s;
    pointer-events: none;
    z-index: 1000;
  }
  .toast.show { opacity: 1; transform: translateY(0); }
```

- [ ] **Step 2: Add toast HTML element**

Add immediately before the closing `</body>` tag:

```html
<div id="toast" class="toast"></div>
```

- [ ] **Step 3: Add `showToast` and `applyBucket` functions**

Add in the script block, after the `clearBucket` function:

```js
  let _toastTimer = null;
  function showToast(msg) {
    const el = document.getElementById('toast');
    el.textContent = msg;
    el.classList.add('show');
    clearTimeout(_toastTimer);
    _toastTimer = setTimeout(() => el.classList.remove('show'), 3000);
  }

  function applyBucket(bucketId) {
    const bucket = TOOL_BUCKETS.find(b => b.id === bucketId);
    const state  = bucketState[bucketId];
    if (!bucket || !state) return;

    let applied = 0, na = 0, unchanged = 0;

    Object.entries(bucket.extractors).forEach(([label, _fn]) => {
      const trimmed = label.trim();
      const matched = metrics.filter(m => m.label.trim() === trimmed);
      const val     = state.extracted[label];

      if (matched.length === 0) {
        unchanged++;
        return;
      }
      if (val !== null && val !== undefined) {
        matched.forEach(m => { m.value = val; m.na = false; });
        applied++;
      } else {
        matched.forEach(m => { m.na = true; });
        na++;
      }
    });

    state.applied = true;
    saveBucketState();
    renderImportTab();
    refresh();
    showToast(`Applied ${applied} value${applied !== 1 ? 's' : ''}, ${na} N/A, ${unchanged} unchanged`);
    switchTab('report');
  }
```

- [ ] **Step 4: Verify Apply to Report**

1. Load the page. Go to the Report tab. Note the default metric "Open Critical Findings" has value 3.
2. Switch to Data Import. Drop any CSV onto the **Qualys Findings** bucket.
3. Click "Apply to Report".
4. Confirm: switched to Report tab, toast appears bottom-right with counts.
5. Check the "Open Critical Findings" metric — since the Qualys extractor is a stub, it should now show N/A.
6. Verify in console: `metrics.find(m => m.label === 'Open Critical Findings').na` should be `true`.

- [ ] **Step 5: Verify drop on applied bucket resets applied state**

After the above, switch back to Data Import. Confirm the Qualys bucket shows "✓ Applied". Drop a new CSV onto it. Confirm the applied badge disappears immediately.

- [ ] **Step 6: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add Apply to Report with toast notification"
```

---

## Task 7: Reset + Get Zip (JSZip)

**Files:**
- Modify: `metrics-report/security-metrics-report.html`

Add `resetAll`, inline JSZip, and `getZip`.

- [ ] **Step 1: Obtain JSZip 3.10.1 minified**

Download the JSZip 3.10.1 package from npm:

```bash
cd /tmp
npm pack jszip@3.10.1
tar -xzf jszip-3.10.1.tgz
# Minified file is at: /tmp/package/dist/jszip.min.js
```

- [ ] **Step 2: Inline JSZip**

Add a `<script>` block containing the full contents of `jszip.min.js`, placed immediately before the main `<script>` block (just before `<script>` that contains `const TOOL_BUCKETS`):

```html
<script>
/* JSZip 3.10.1 - https://stuk.github.io/jszip/ - MIT License */
[PASTE FULL CONTENTS OF jszip.min.js HERE]
</script>
```

- [ ] **Step 3: Verify JSZip loaded**

Open the page, open console. Type `JSZip`. Should log the JSZip constructor function, not a ReferenceError.

- [ ] **Step 4: Add `resetAll` and `getZip` functions**

Add in the script block, after `applyBucket`:

```js
  function resetAll() {
    if (!confirm('Clear all loaded CSV files? Your report config will not be affected.')) return;
    bucketState = {};
    localStorage.removeItem('smr_buckets_v1');
    renderImportTab();
  }

  async function getZip() {
    const entries = Object.values(bucketState);
    if (entries.length === 0) return;
    const zip = new JSZip();
    entries.forEach(state => zip.file(state.fileName, state.csvText));
    const blob = await zip.generateAsync({ type: 'blob' });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href     = url;
    a.download = 'security-metrics-csvs.zip';
    a.click();
    URL.revokeObjectURL(url);
  }
```

- [ ] **Step 5: Verify Reset and Get Zip manually**

1. Drop a CSV file onto any bucket.
2. Click **Get Zip**. Confirm a zip file downloads containing the CSV.
3. Click **Reset**. Confirm the confirmation dialog appears.
4. Confirm the reset: all buckets return to empty state, localStorage `smr_buckets_v1` is gone.
5. Verify Get Zip button is disabled when no buckets are loaded.

- [ ] **Step 6: Commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: add Reset, Get Zip actions and inline JSZip 3.10.1"
```

---

## Task 8: Init Wiring + Version Bump

**Files:**
- Modify: `metrics-report/security-metrics-report.html` (script block)

Wire `loadBucketState` and `renderImportTab` into `init()`, and bump the version to 0.3.0.

- [ ] **Step 1: Update `init` to load bucket state**

Find the `init()` function. Add `loadBucketState()` and `renderImportTab()` as the first two lines:

```js
  function init() {
    loadBucketState();          // restore buckets from localStorage
    renderImportTab();          // render import tab (uses restored state)
    document.getElementById('versionBadge').textContent = 'v' + VERSION;
    // ... rest of existing init code unchanged ...
  }
```

- [ ] **Step 2: Bump the version**

Find `const VERSION = '0.2.0';` and change to:

```js
  const VERSION = '0.3.0';
```

- [ ] **Step 3: Full end-to-end verification**

Perform this complete flow:

1. Open the page fresh. Version badge on the Report tab header reads `v0.3.0`.
2. Go to Data Import. 4 empty buckets are visible.
3. Drop a CSV on two different buckets.
4. Reload the page. Both buckets are still loaded (restored from localStorage).
5. Click "Apply to Report" on one bucket. Toast appears, page switches to Report tab. Report metrics updated correctly (stubs → N/A, since all extractors are stubs).
6. Switch back to Data Import. The applied bucket shows "✓ Applied".
7. Drop a new file onto the applied bucket. The "✓ Applied" badge disappears immediately.
8. Click Get Zip. A zip downloads containing both CSV files.
9. Click Reset → confirm. All buckets empty, localStorage cleared, Get Zip disabled.
10. Print/PDF from Import tab → switches to Report tab, print dialog opens.

- [ ] **Step 4: Final commit**

```bash
git add metrics-report/security-metrics-report.html
git commit -m "feat: wire init, bump version to 0.3.0 — data import tab complete"
```

---

## Extending Extractors (Reference)

To implement a real extractor, edit the `TOOL_BUCKETS` array at the top of the script. Example for a Qualys extractor that counts Critical severity rows:

```js
// In the 'qualys' bucket's extractors:
'Open Critical Findings': (rows, headers) => rows.filter(r => r['Severity'] === '5').length,
```

Each extractor receives:
- `rows` — array of objects keyed by header name (from `parseCSV`)
- `headers` — array of column name strings

Returns a raw number/string, or `null` for N/A.
