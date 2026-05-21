# USAR Status Dashboard — Project Context

## What this is
A single-file HTML dashboard for PIKA International's USAR (Urban Search and
Rescue) Industrial Hygiene contracts. It reads an Excel workbook at runtime
(via file picker, using the xlsx CDN library) and renders it as an
interactive multi-tab app. No build step, no framework — it runs by opening
the HTML file in a browser.

The maintainer is a project manager at PIKA working on USAR IH contracts
(currently 4 of 5 in the final year, with rebid activity underway involving
teaming partners Tetra Tech and Battelle).

## The file
Everything lives in one self-contained HTML file:
- All CSS in a single `<style>` block
- All JS in a single `<script>` block
- No external dependencies except the xlsx CDN library for parsing

**Do not** split into modules, add a build step, or introduce a framework.
The single-file constraint is intentional — it makes the dashboard portable
and openable anywhere.

## The data
The workbook ("Master IHA Projected Schedule and POC Lists") has:
- An **AllSites** tab (catch-all, includes a concatenated "Full Address")
- Four **per-RD/MSC tabs**: 63rd, 81st, 99th, 9th MSC (these break address
  into separate columns: Street #, Street Name, City, State, Zip)

Sites are tracked across contract years: **Base Year (BY)** and **Option
Years 1-4 (OY1-OY4)**.

### Tabs in the app
Dashboard, Work in Progress (WIP), Invoice, Report Status, Site Variations
(Scope Changes + NEW Sites sections), Site Variations Summary, Sites List.

## The three teaming companies
| Company | Short | Color | Lead IH for |
|---------|-------|-------|-------------|
| PIKA | PIKA | blue `#1d4ed8` | (always gets its own share) |
| Tetra Tech | TT | dark red `#b91c1c` | 63rd, 81st |
| Battelle | Battelle | purple `#6b21a8` | 99th, 9th MSC |

## Key domain logic (already implemented — preserve unless asked to change)

### Per-RD precedence
Per-RD tab data takes precedence over AllSites when populated. Conflicts are
surfaced via `console.warn`, not silently overwritten.

### Incurred cost per company (`computeIncurredPerCompany`)
Splits the site's incurred cost by priority amount:
- **$632 (PR23):** Lead IH $462, PIKA $118, Team $52
- **$1,268 (PR1):** Lead IH $462, PIKA $403, Team $403
- **$846 (Lead-IH-only, no Team share):** PIKA $118, Lead IH co $462,
  other non-PIKA co $266
- **Any other non-zero amount:** entire amount goes to the Team company
- **$0:** all companies $0

All standard amounts use a ±$1 tolerance. The Lead IH share goes to TT for
63rd/81st, Battelle for 99th/9th MSC. The Team share goes to whichever
company is tagged Team on the site.

### Descope per company (`computeDescopePerCompany`)
Partitions a site's Total Cost across the three companies (positive
magnitudes). PIKA + TT + Battelle should equal Total Cost — verified by the
Descope QA column.

### Scope Changes filtering (Site Variations view)
Excludes:
- Confirmed-scheduled sites (Scope Variation = Y + Dates Confirmed = "x" +
  Start/End populated) — see `siteIsConfirmedScheduled`
- Sites with blank Dates Confirmed
EXCEPTION: a site with positive Incurred Cost re-enters as an
"incurred-only clone" (Total Cost / CIH / Lead IH / Additional Cost / Team
Cost nulled out, `_scopeChangesIncurredOnly: true` marker set) — see
`cloneSiteForIncurredOnlyScopeChange`.

### BY→OY1 rollforward
Completely-descoped BY sites' incurred costs roll forward to OY1 at the
subtotal level (per-row data unchanged). See `computeBaseYearRollforward`.

### Credit-used vs descope (BY)
A BY site is "credit-used" (not a descope) when it has BOTH a PSC Accept
date AND a Final IHA Report Approved date AND Incurred Cost = $0 — see
`siteFundedWithCredit`. Used by the Site Variations Summary's BY de-scope
figure to exclude these sites.

### Site Variations Summary tab
Per-RD contract rollup: editable "Funded Field" input + fixed "On Contract"
(CLINS task-order total minus $25K, non-editable), computed BY/OY1-4
adjustments (descope + new sites per year), per-company columns (PIKA/TT/
Battelle) with a QA column verifying PIKA + TT + Battelle = row total (±$1).
Contract input defaults are in `VAR_SUMMARY_DEFAULTS`.

## CLINS reference (contract source of truth)
Per-RD task order totals from the CLINS document (master contract
W912DY-20-D-0019):
| RD/MSC | CLINS Total | Dashboard "On Contract" (CLINS − $25K) |
|--------|-------------|----------------------------------------|
| 63rd | $4,999,786 | $4,974,786 |
| 81st | $6,231,786 | $6,206,786 |
| 99th | $6,024,786 | $5,999,786 |
| 9th MSC | $1,700,786 | $1,675,786 |

## Working conventions
- **Validate before saving:** run `node --check` on the extracted JS before
  writing any change. (Extract the `<script>` block contents to a temp .js
  file and check it.)
- **Single-file rule:** keep all CSS/JS inline in the one HTML file.
- **Commit descriptions:** after finishing a change, provide a short
  commit-style summary — what changed, why, edge cases handled, and what
  was deliberately left untouched.
- **Surgical edits:** prefer minimal targeted changes over rewrites. Match
  the existing code style.
- **Fidelity to the workbook:** the output must match the team's exact
  spreadsheet structure. When in doubt, the source workbook (and the CLINS
  document) win.
- **Inline-style quoting gotcha:** when building HTML via template literals,
  use single quotes for font-family names inside `style="..."` attributes
  (e.g. `font-family:'IBM Plex Sans'`). Double quotes break the attribute
  and silently drop every CSS property after them.

## Formatting / display conventions
- Money formatted to 2 decimals; negatives shown as `$ (1,234.00)`.
- Per-company columns always use the brand colors above.
- QA columns: ✓ green / ✗ red, with ±$1 tolerance on sum checks.
- Multi-select filter popups: checkboxes must override the broad
  `.filter-group input` rule (min-width / padding / border) or they break.
