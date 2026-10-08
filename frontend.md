# FuelTrack Frontend — Context

Companion to `../PROJECT_CONTEXT.md` (backend). Read both when continuing work.

## 1. Purpose & Status

Responsive single-page UI for the Fuel Mileage Calculator REST API. Works on desktop and phone
(bottom navigation under 720 px). Light/dark theme. No UI library, no router, no state library.

Status (08-10-2026): **runs in dev mode** (`npm run dev` on :5173, proxied to backend :8081).
Production build (`npm run build` → served by Spring Boot) set up but not yet exercised.

Decision: lives in the SAME repository as the backend (monorepo) under `frontend/`.
Rationale: one clone, API+UI changes in one commit, one deployable jar, no CORS.

## 2. Tech Stack

| Item | Choice |
|------|--------|
| Framework | React 18 (JavaScript, no TypeScript) |
| Build tool | Vite 6 with `@vitejs/plugin-react` |
| Styling | Plain CSS in `src/styles.css` (CSS variables, `color-mix`, media queries) |
| Data fetching | native `fetch` via `src/api.js` |
| State | React `useState` / `useEffect` / `useCallback`; `localStorage` for preferences |
| Charts | Pure CSS bar charts (`BarChart.jsx`), no chart library |
| Node | v24 (any LTS ≥ 20 works) |

## 3. Folder Structure

```
frontend/
├── package.json          scripts: dev, build, preview
├── vite.config.js        dev server :5173, host=true, proxy /api → [http://localhost:8081,](http://localhost:8081,)
│                         build.outDir = ../src/main/resources/static (emptyOutDir)
├── index.html            mounts #root, loads /src/main.jsx
├── FRONTEND_CONTEXT.md   this file
└── src/
    ├── main.jsx          ReactDOM.createRoot, imports styles.css
    ├── App.jsx           top-level state: vehicles, vehicleId, tab, modal, toast, refreshKey, currency
    ├── api.js            api(path, options) wrapper + helpers: today, monthName, fmtNum, fmtMoney, formToObject
    ├── styles.css        full design system (tokens, layout, cards, forms, buttons, charts, modal, toast, mobile nav)
    └── components/
        ├── Header.jsx        brand, vehicle <select>, "+ Vehicle" button, theme toggle (persists fuel.theme)
        ├── Tabs.jsx          4 tabs: dashboard | log | history | calculator (bottom bar on mobile)
        ├── Dashboard.jsx     SUMMARY + MONTHLY(last 12 months) + last 10 entries → stat cards + 2 bar charts
        ├── BarChart.jsx      items [{label, value, text}] → animated CSS bars with hover tooltip
        ├── LogForm.jsx       fill-up form; prefills odometerStart from last entry and price from fuel.lastPrice;
        │                     live preview (distance, mileage, cost); POST entry; calls onSaved()
        ├── History.jsx       list of entries (size=100) with Delete; calls onChanged()
        ├── Calculator.jsx    POST /calculator/mileage and /calculator/trip-cost (stateless)
        ├── VehicleModal.jsx  POST /vehicles; calls onCreated(vehicle)
        └── Toast.jsx         transient message (2.8 s), red variant for errors
```

## 4. Data Flow

- `App` loads vehicles on mount (`GET /vehicles?size=100&sort=name`), keeps `vehicleId`
  in state + `localStorage('fuel.vehicleId')`.
- `refreshKey` (integer) is bumped by `refresh()` after any write; `Dashboard` and `History`
  re-fetch when `vehicleId` or `refreshKey` changes.
- `currency` comes from the SUMMARY report response and is passed to money formatters.
- `notify(message, isError)` shows a toast; all API errors surface `ApiError.message`
  (+ validation `details`) from the backend.
- `api()` returns `null` for 204, throws `Error(message)` for non-2xx.

localStorage keys: `fuel.vehicleId`, `fuel.theme` (`light`|`dark`), `fuel.lastPrice`.

## 5. Backend Endpoints Used (all under /api/v1)

| Component | Calls |
|-----------|-------|
| App | GET /vehicles?size=100&sort=name |
| VehicleModal | POST /vehicles |
| Dashboard | GET /vehicles/{id}/reports/SUMMARY · GET /vehicles/{id}/reports/MONTHLY?from&to · GET /vehicles/{id}/entries?size=10 |
| LogForm | GET /vehicles/{id}/entries?size=1 · POST /vehicles/{id}/entries |
| History | GET /vehicles/{id}/entries?size=100 · DELETE /vehicles/{id}/entries/{entryId} |
| Calculator | POST /calculator/mileage · POST /calculator/trip-cost |

Request/response shapes: see PROJECT_CONTEXT.md section 7.

## 6. Run / Build

Dev (two terminals):
```
# terminal 1 — project root
.\mvnw.cmd spring-boot:run              # backend on :8081
# terminal 2 — frontend/
npm install                             # first time only
npm run dev                             # [http://localhost:5173](http://localhost:5173)
```
Prod build (served by Spring Boot, phone-ready):
```
cd frontend && npm run build            # writes to ../src/main/resources/static
# restart backend → [http://localhost:8081/%20%20or%20%20http://<pc-ip>:8081/%20on%20phone%20(same%20Wi-Fi)](http://localhost:8081/%20%20or%20%20http://<pc-ip>:8081/%20on%20phone%20(same%20Wi-Fi))
```
Decision: commit the built `src/main/resources/static/` so other devices run without Node.
Rebuild after any frontend change. (Later: automate with frontend-maven-plugin.)

Phone access: `ipconfig` → IPv4 address; allow inbound TCP 8081 in Windows Firewall if blocked:
`New-NetFirewallRule -DisplayName "FuelTrack 8081" -Direction Inbound -LocalPort 8081 -Protocol TCP -Action Allow`

## 7. Conventions

- Function components only; one component per file; default export.
- Class names come from `styles.css`; no inline styles except dynamic bar heights.
- Forms: controlled (`LogForm`) when live preview is needed, otherwise uncontrolled + `formToObject(e.target)`.
- Keep `api.js` the single place that knows the base path `/api/v1`.
- `// eslint-disable-line react-hooks/exhaustive-deps` comments are intentional on data-loading effects.
- No external UI/chart libraries unless a clear need arises (keeps bundle tiny, no CDN/URL paste risks).

## 8. Gotchas Encountered (don't repeat)

1. Pasting from chat mangles URLs into Markdown links → `vite.config.js` builds the backend URL as
   `'http' + '://localhost:8081'` on purpose. Keep it that way.
2. Windows PowerShell `Set-Content -Encoding UTF8` writes a BOM → Vite fails with
   `Unexpected token '﻿'` on package.json. Create files in VS Code or write with
   `[System.IO.File]::WriteAllText(path, text, UTF8Encoding($false))`.
3. `npm` blocked by PowerShell execution policy → fixed with
   `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` (or use `npm.cmd`).
4. VS Code must be restarted after installing Node so the terminal sees the new PATH.
5. File placement matters: `main.jsx`, `App.jsx`, `api.js`, `styles.css` directly in `src/`;
   the 9 components in `src/components/`. Wrong placement → "Failed to resolve import".
6. Terminal location: `npm` commands run inside `frontend/`; `mvnw` runs at project root.
7. `color-mix()` requires a modern browser (Chrome/Edge 111+, Safari 16.2+).

## 9. Ideas / Next Steps

- `npm run build` + verify Spring Boot serves the SPA at `/`; test on phone.
- PWA (vite-plugin-pwa): installable icon, offline calculator.
- React Router for shareable URLs per tab; TanStack Query for caching/retries.
- Recharts for line/area charts; date-range picker for reports.
- Edit fill-up (needs PUT endpoint on backend); vehicle edit/delete UI (endpoints exist).
- Export CSV of history; service-reminder banner when backend feature lands.
- TypeScript migration if the app grows.