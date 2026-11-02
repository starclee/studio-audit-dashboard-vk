# Studio Audit Dashboard

A single HTML page for exploring the JSON reports that `npm run audit` / `npm run scan` write to
[`../reports/`](../reports/). It shows hardcoded secrets, hardcoded endpoint URLs, direct API calls in
client-side functions and form handlers, and configuration made available for client, broken down by
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
- **Reviewed config exceptions**: every entry of the loaded exceptions file with its status (see
  below). Reviewed entries are removed from the HIGH counts, and a **Reviewed exceptions** tile
  shows how many were applied.
- **All apps**: per-app totals, branch and scan time. Click a row with findings to filter to it.
- **Not scanned**: lists that failed to open and items whose code stayed empty after reopening.
  These should be rescanned or checked by hand.

## Config exceptions file

A business setting the client side needs may stay **available for client** after a manual review.
Record each reviewed entry in one JSON file for the whole org, then load it with **Load config
exceptions** (or drop it together with the reports). The file stays in your browser, like the reports.

```json
{
  "kind": "studio-audit-config-exceptions",
  "updatedOn": "2026-11-02",
  "orgs": {
    "Samunnati_Finance": {
      "Renewal Process": [
        { "entry": "maxLoanAmount", "reason": "Loan limit shown on the form", "reviewedBy": "QA - name", "reviewedOn": "2026-11-02" }
      ]
    }
  }
}
```

To start one, load the reports and click **Download exceptions template**. It lists every
client-available entry that isn't API-related, and keeps whatever the loaded file already says.
Delete the rows that must be switched off, then fill in the rest.

| Status | Meaning |
|---|---|
| **Applied** | Reviewed business setting. Not counted as a finding. |
| **Applied, review overdue** | `reviewedOn` is more than 7 days old. Still applied; review it this week and update the date. |
| **Not applied: API-related entry** | The name looks like a URL, path, host, key, token, secret, password, scope or auth entry. These can never be exceptions; switch them off. |
| **Not applied: needs reason, reviewedBy and reviewedOn** | Every exception needs all three, with the date as `YYYY-MM-DD`. |
| **Not in report** | The entry is no longer available for client, or was renamed. Remove it from the file. |
| **Not checked** | That app isn't in the loaded reports. |

Only configuration entries can be exceptions. Secrets, endpoint URLs and API calls in code are
always reported.

## Exports

- **Export findings CSV**: the findings currently filtered (`status` = `open`), followed by the
  reviewed exceptions (`status` = `reviewed exception`, with reason, reviewer and date).
- **Export consolidated report**: one JSON file, `<org>_consolidated_<date>.json`, combining all
  loaded reports with the exceptions applied. Its `findings` are the open findings only; the
  reviewed entries are listed under `exceptions`, and the file used is under `configExceptions`.
  Use it for validation and sign-off. Loading it back into the dashboard shows the same numbers.

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
