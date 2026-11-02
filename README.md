# Studio Audit Dashboard

A single HTML page for exploring the JSON reports that `npm run audit` / `npm run scan` write to
[`../reports/`](../reports/). It shows hardcoded secrets, direct API calls in browser code
(Client-mode functions and forms), and configuration made available for client, broken down by
app, kind of risk, item type and location.

The page is self-contained (one file, no external scripts, fonts or network calls), so it works
offline and report data never leaves your machine.

## Usage

1. Run an audit so `reports/` holds at least one report (see the [main README](../README.md)).
2. Open [`index.html`](index.html) in a browser (double-click it; no server needed).
3. Drop one or more report `.json` files on the page, or click **Load report JSON**.

Load a new set of files at any time to replace the current view.

## Which reports it reads

| Report | File name |
|---|---|
| Org (all apps) | `<org>_ALL-APPS_all_ALL.json` |
| One app, one or all types | `<org>_<app>_<type>_ALL.json` |
| Single item | `<org>_<app>_<type>_<name>.json` |

`<type>` is `function`, `service`, `form` or `configuration`.

**Combining files.** You can load several files at once, for example an org function/service run
plus separate `audit form` and `audit config` runs. For each app, the **newest scan of each item
type** wins (by `scannedAt`). So two runs of the same type don't double-count, and runs of
different types add up.

## What the page shows

- **Summary**: a one-line takeaway with what to do about each kind of HIGH finding, and tiles
  for HIGH / MEDIUM / LOW counts, items with findings, items scanned and items not scanned.
- **Findings by app**: a stacked bar per app with findings, sorted by HIGH count. Click a bar to
  filter the findings table to that app.
- **By kind of risk / item type / location / most common keys**: severity-split bars.
  *Location* separates live code from commented-out code, which still ships in the source.
- **Scan coverage**: clean vs. with findings vs. empty code vs. errors. Items that weren't read
  were **not** checked.
- **Findings table**: filter by severity, app, kind and location, run a free-text search, sort
  any column, and use **Export findings CSV** to export the rows that are currently filtered.
  Line numbers point into the item's code in the studio editor (for forms, into the handler shown
  under the form name).
- **All apps**: per-app totals, branch and scan time. Click a row with findings to filter to it.
- **Not scanned**: lists that failed to open and items whose code stayed empty after reopening.
  These should be rescanned or checked by hand.

## Report checks

A yellow **Report checks** banner appears when a file looks inconsistent. The dashboard still
loads it, using counts recalculated from the file's details:

- the totals written in the file don't match its items and findings;
- items in an older merged org report have no `app`. The app is inferred from matching findings,
  or the items are shown under `(unknown app)`;
- apps that have items but no row in the file's `apps` list.

A red banner lists files that aren't valid JSON or aren't studio-audit reports.

## Handling the data

- Values are **already masked by the scanner** (`abcd****xy`). The dashboard never sees or shows
  real secrets.
- Masked values and code snippets still reveal where credentials live. Treat the reports and any
  exported CSV as internal, the same as `reports/` (git-ignored).
- The only thing stored in the browser is the light/dark theme choice (`localStorage`,
  key `ss-theme`).

## Changing the page

Everything (styles, markup, script) is in `index.html`. The report shape it expects:

- `items[]`: `{ app?, type, name, status: "scanned" | "empty" | "error", counts: { HIGH, MEDIUM, LOW }, error? }`
- `findings[]`: `{ app, type, functionName, line, item, location, severity, reason, maskedValue, codeSnippet?, handler? }`
- optional `org`, `scannedAt`, `branch`, `totals`, `apps[]` (org reports), `target` (single-item reports).

If you change the report format in [`lib/report.ts`](../lib/report.ts), update `normalize()` in
`index.html` to match. To test, load a few old and new reports together.
