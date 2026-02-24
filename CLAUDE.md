# CLAUDE.md — AiWskazniki

## Project Overview

**AiWskazniki** is a Polish-language, offline-capable, single-file web tool for verifying the data quality of EFS+ (European Social Fund Plus) project participants and the correctness of declared indicator levels in payment requests (wnioski o płatność).

It is deployed as a static site on **GitHub Pages**: https://r4z3l.github.io/AiWskazniki/

The application reads CSV exports from the **SL2021/SM+** system (the Polish national IT system for EFS+ grant management), runs a battery of validation modules, and produces a detailed printable/PDF report.

---

## Repository Structure

```
AiWskazniki/
├── index.html                    # Redirect to the latest version (GitHub Pages entry point)
├── EFS_Wskazniki_v38.7.html      # CURRENT production version (the main application)
├── EFS_Wskazniki_v38.5.html      # Previous version snapshot
├── EFS_Wskazniki_v38.4.html      # Previous version snapshot
├── EFS_Wskazniki_v38.2.html      # Previous version snapshot
├── EFS_Wskazniki_v38.1.html      # Previous version snapshot
├── EFS_Wskazniki_v37.8.html      # Previous version snapshot
├── Instrukcja AiWskazniki.pdf    # End-user manual (Polish)
└── README.md                     # One-line project description + live link
```

**The entire application lives in a single self-contained HTML file.** There is no build pipeline, no `package.json`, and no external assets—all JavaScript (bundled and minified), CSS (Tailwind via PostCSS/Autoprefixer), icons (Lucide), and images (base64-inlined) are embedded in the HTML file. Each file is approximately 4 MB.

---

## Technology Stack

| Layer | Technology |
|---|---|
| UI framework | **React 18** (embedded, `ReactDOM.createRoot`) |
| Styling | **TailwindCSS** (bundled, JIT via PostCSS/Autoprefixer) |
| Icons | **Lucide React** (bundled) |
| State management | React hooks (`useState`, `useEffect`, `useMemo`, `useCallback`) — no external state library |
| Data parsing | Custom CSV parser (handles `,` and `;` separators, quoted fields) |
| Persistence | `localStorage` (theme and zoom settings only) |
| Build tooling | None visible in repo — files appear to be pre-built bundles |
| Deployment | GitHub Pages (static) |
| Language | **Polish** throughout the UI, variable names, and comments |

---

## Application Architecture

### Entry Point

The React app mounts at line ~1273:
```js
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```

### Main Component: `App`

The single `App` functional component holds all state and logic. Key state variables:

| Variable | Purpose |
|---|---|
| `data` | Processed/calculated analysis results object |
| `rawRows` | Parsed rows from the uploaded participants CSV |
| `allWnpRows` | Parsed rows from the uploaded WNP (payment request) CSV |
| `startDate` / `endDate` | Reporting period date range |
| `selectedClaimNumber` | Selected payment request number for WNP reconciliation |
| `theme` | UI theme: `'light'`, `'dark'`, or `'sepia'` (persisted in `localStorage`) |
| `zoom` | UI zoom level as integer (e.g. `100`), persisted in `localStorage` |
| `collapsed` | Object controlling which report sections are collapsed/expanded |

### Key Functions

| Function | Description |
|---|---|
| `parseCSV(text)` | Parses CSV text with auto-detection of `,` or `;` separator |
| `parseCSVWithSep(text, forceSep)` | Parses CSV with explicitly forced separator |
| `processFile(e)` | Handles main participants CSV file input; extracts date from filename (pattern: `YYYY-MM-DD.NN_Lista...`) |
| `processBeneficiaryFile(e)` | Handles WNP beneficiary CSV file input |
| `calculate(rows, startLimit)` | **Core analysis function** — processes all participant rows and computes all validation module results |
| `parseAgeFromColumn(val)` | Extracts numeric age from a column value |
| `parseCSVDate(dateStr)` | Parses date strings in formats `YYYY-MM-DD`, `DD.MM.YYYY`, `DD-MM-YYYY` |
| `toNaturalCase(str)` | Converts string to sentence case |

### Computed Values (useMemo)

| Memo | Description |
|---|---|
| `availableClaims` | Unique payment request numbers from WNP file matching the project number |
| `filteredBeneficiaryData` | WNP rows filtered by `selectedClaimNumber`, mapped to indicator objects |
| `beneficiaryName` | Extracted beneficiary name from WNP data |
| `reconciliation` | Compares calculated indicator values against declared WNP values |

---

## Validation Modules

The `calculate()` function produces a `data` object containing results for these modules, displayed as collapsible report sections:

| Module ID | Polish Name | Description |
|---|---|---|
| M1 | Walidacja Grupy Docelowej | Detects participant-type conflicts (Przedszkole + Uczeń) and age errors (age > 6 in preschool group) |
| M2 | Alerty Jakości Danych | Substantive inconsistencies and logical errors requiring explanation (SM+ specific rules) |
| M3 | Weryfikacja Kwalifikowalności Wskaźnikowej | Checks indicator definition conditions for PLFCO13 (vocational guidance), PLFCO06 (staff), PLEFCO05 (student internships), EECO13/14 (third-country nationals) |
| — | Weryfikacja Spójności Danych | Data consistency checks across fields |
| — | Statusy (statusErrors) | Participant status validation |
| — | Kompletność (kompleErrors) | Completeness/missing-field checks |
| — | Daty (dateErrors) | Date range and chronology validation |
| — | Pochodzenie (origErrors) | Origin/nationality field validation |
| — | Zakończenie (endErrors) | Project completion status validation |
| — | Statystyki Instytucji | Per-institution participation statistics |
| — | Uzgodnienie z WNP | Reconciliation of calculated indicator values vs. declared values in the payment request |

The `data` object also contains arrays of **product indicators** (`productIndicators`) and **result indicators** (`resultIndicators`) with counts broken down by gender (`k` = kobiety/women, `m` = mężczyźni/men, `o` = ogółem/total) for both cumulative totals and within-period totals (`kPeriod`, `mPeriod`, `oPeriod`).

---

## Versioning Convention

Files use the naming pattern: `EFS_Wskazniki_vXX.Y.html`

- **XX** = major version number (currently `38`)
- **Y** = minor version number (currently `7`)
- Old version files are **kept in the repository** as historical snapshots
- `index.html` is always updated to redirect to the **latest** version file
- The `<title>` tag inside each HTML file reflects the version: `[OFFLINE] AiWskazniki vXX.Y | Arkusz Pomiaru Wskaźników EFS+`

**When releasing a new version:**
1. Save the new version as `EFS_Wskazniki_vXX.Y.html`
2. Update `index.html` to point to the new file name
3. Update the `README.md` link if the live URL changes
4. Commit and push to `master`

---

## Development Workflow

There is no automated build step or test suite in this repository. The HTML file is a pre-built, self-contained bundle.

### Editing the Application

When modifying the app, you are editing the JSX/JavaScript code embedded directly inside the `<script>` block of the HTML file (starting around line 700+). Key areas:

- **Lines 1–400 approx.**: Bundled third-party libraries (React, ReactDOM, Lucide, PostCSS/Tailwind, etc.) — **do not edit manually**
- **Lines 400–530 approx.**: Print CSS styles (`@media print`) and theme CSS variables
- **Lines 530–700 approx.**: Tailwind configuration and helper functions
- **Lines 700–1270 approx.**: The entire application source (`App` component, `calculate()`, all logic, JSX render tree)
- **Lines 1273–1274**: React root mount

### Deploying to GitHub Pages

```bash
git add EFS_Wskazniki_vXX.Y.html index.html
git commit -m "Deploy vXX.Y: <description of changes>"
git push origin master
```

GitHub Pages serves the `master` branch root. The live URL is:
`https://r4z3l.github.io/AiWskazniki/EFS_Wskazniki_vXX.Y.html`

---

## UI / UX Conventions

- **Themes**: `light` (default), `dark`, `sepia` — toggled via UI; persisted to `localStorage` as key `aiw_theme`
- **Zoom**: Integer percentage (e.g. 80–150%); applied as CSS `transform: scale()` on `.main-content-wrapper`; persisted to `localStorage` as key `aiw_zoom`
- **Print/PDF**: Tailwind `print:` variants and `@media print` styles control print layout; each major section is wrapped in a `page-break` class to separate PDF pages
- **Collapsible sections**: Each validation module header is clickable and toggles visibility via the `collapsed` state object; all collapsed sections become visible (`print:block`) when printing
- **Status badges**: Green (emerald) = no errors, amber = few errors (< 5), red = many errors (≥ 5)
- **Language**: All UI text is in Polish

---

## Input File Formats

### Participants CSV (main file)
- Source: SL2021/SM+ system export ("Lista uczestników projektu")
- Separator: `;` or `,` (auto-detected)
- Filename convention: `YYYY-MM-DD.NN_Lista...` — the app extracts the end date from the filename and auto-sets `startDate` to 3 months prior (first day of that month)
- Row 0: Header row (skipped)
- Key columns (0-indexed):
  - `[0]` — Project number
  - `[3]` — Institution name

### WNP Beneficiary CSV (optional, for reconciliation)
- Source: WNP (Wniosek o Płatność) export
- Separator: `;` or `,` (auto-detected)
- Row 0: Header row (skipped)
- Key columns:
  - `[0]` — Payment request number (claim number)
  - `[1]` — Beneficiary name
  - `[4]` — Indicator name
  - `[8]` — Value O (ogółem / total)
  - `[9]` — Value K (kobiety / women)
  - `[10]` — Value M (mężczyźni / men)

---

## Key Constraints

- **Offline-first**: The app is titled `[OFFLINE]` and has no runtime network requests. All libraries are bundled inline.
- **No backend**: Pure client-side processing. No data ever leaves the user's browser.
- **No build system in repo**: The bundled HTML files are the final artifact. If a source/unbundled version exists, it is not tracked in this repository.
- **Single component**: All logic is in the `App` component — there is no component decomposition into separate files or modules.
- **Polish domain terminology**: EFS+ = European Social Fund Plus; WNP = Wniosek o Płatność (payment request); SM+ = System Monitorowania (monitoring system); wskaźnik = indicator; uczestnik = participant; beneficjent = beneficiary.
