# Daily Fruit — Procurement app

Notes for Claude sessions working on this repo. Read before making changes.

## Standing rules (from the owner — always follow)

1. **Say so when the owner is doing something wrong** or asking for something that will cause problems. Don't just go along with it.
2. **Send only the files that changed.** When handing work back, list the changed files. Don't re-send unchanged ones.
3. **Every tile is clickable.** Any stat/summary tile (`.stat`) must do something when clicked, usually filtering the table below. Clicking the active tile again clears the filter. See the Shopping List Archive tiles (`data-arch-stat`) for the pattern.
4. **Every click shows a loading indicator.** Any action that waits on the server must show progress. Use `setSyncBusy(true/false)` for the sync pill, or a busy state on the button/card.
5. **Never put credentials in chat.** No PINs, passwords, API keys or tokens in messages, commits or code. Secrets live in Apps Script **Script Properties**.
6. **Never open the Users sheet** (the `Users` tab in the Google Sheet). It holds staff names and PINs.
7. **Remind the owner to redeploy `Code.gs`** whenever it changes: Apps Script → Deploy → Manage deployments → edit → **New version**. Saving the file alone does not update the live web app. Also say in the version-history line when a front-end change "Needs the new Code.gs".

## What's in the repo

- `index.html` is the whole front end in one file (~650 KB): HTML, CSS and JS, with no build step. It's served by GitHub Pages at `procure.dailyfruit.co.za` (see `CNAME`) from the `main` branch.
- `manifest.json` and `icons/` provide the home-screen app icon and standalone launch. Icon URLs carry a `?v=` cache-buster.
- The Apps Script back end is **not in this repo**, because this repo is public. The master copies are in the private repo `branscombeshane/DFMasterPrivate`, folder `procurement-apps-script/`; ask the owner to add that repo to the session when server code needs changing. That repo's root holds a different app (Financials), so only touch the `procurement-apps-script/` folder. Never commit `.gs` files here.

## How the app fits together

- **Back end:** a Google Apps Script web app bound to the Google Sheet. `index.html` talks to it through `API_URL` (`script.google.com/macros/s/.../exec`). That URL is public, so it grants nothing by itself.
  - **Every call goes through `apiPost_()`**: a `POST` with `Content-Type: text/plain` and a JSON body `{action, token, ...}`. Never call `fetch(API_URL)` directly.
  - `login` checks name + PIN **on the server** and returns a session token, which is kept in `currentUser.token` and sessionStorage. PINs are never sent to the browser: `DATA.users` has no `pin`, and the Users page sends a PIN only when one is typed (blank = keep).
  - `getData` loads everything, `version` is the cheap 20-second change check (it also keeps the session alive), `getShoppingListSnapshot` returns archive rows, and saves and purchase actions follow the same pattern.
  - A reply with `authError: true` means the session has ended. `apiPost_` sends the user back to the login screen (`sessionEnded_`).
- **Apps Script files, Procurement project** (deployed by hand by the owner):
  - `Auth.gs` handles login, sessions (Script Properties `AUTH_SESS_<token>`, 2 hours idle), the 5-wrong-PIN lockout, and `doGet`/`doPost` with the permission checks: Users changes need an owner (or the Users page ticked), a full restore needs an owner, and the market import needs `MARKET_IMPORT_KEY`.
  - `Code.gs` holds the sheet schema and read/write helpers, saves, purchase actions (`handlePurchaseAction_(data, user)`) and snapshots (e.g. `snapshotShoppingListToday`).
  - `SageSalesPull.gs` runs the daily pull of Item Sales reports from Sage, through the Packing app. A day counts as "waiting" until Sage is due to deliver it. It also handles the live Sage read and the price push (the role is checked from the session).
  - `MarketStatementImport.gs` is a **separate standalone project**. It imports market statements for Market Statements and Market vs Receiving, and must send `MARKET_IMPORT_KEY`, which is the same Script Property value in both projects.
  - The Packing project (a different app) holds the only Sage login. Its files are not Procurement code.
- **Sage:** there are two companies, *Daily Fruit – Veg* and *Pyroca 79*. Selling prices are pushed through the separate Packing app in small batches. Average cost goes out via the cost CSV.
- **Login:** each user has a name, PIN and role (`owner`, `procurement`, `stocktaker`, `receiving`, `market`). Login is checked on the server, and disabling a user or changing their role ends their session. Page access in the app is controlled by `ROLE_TABS` and `userCanAccessTab()`. The server does not yet check page access for each sheet (that is still to do), only the owner-only actions listed above. A receiving-only user gets the kiosk layout.
- **Time:** business dates are South African (`todaySA()`, `Africa/Johannesburg`), not UTC. A stock take started before 06:00 belongs to the previous evening.

## Code conventions in `index.html`

- The UI is rebuilt by `render()`, which replaces `#main` innerHTML on each call. Event listeners are attached after render using `data-*` attributes. Page state lives in top-level `let` variables declared near `APP_VERSION`.
- Use `esc()` on every value interpolated into HTML.
- Status keys are `Critical` / `Buy` / `OK` / `Overstocked`. `shoppingStatusLabel()` maps them to display text ("Buy" shows as "Consider Buying").
- Search boxes and tile filters are combined with AND.
- **Versioning (every change):** bump `APP_VERSION` to `vYYYY.MM.DD` plus a letter (a, b, c… for multiple builds the same day). Add a line at the top of the `VERSION HISTORY` comment block above it, written in plain language for the owner.
- Comments explain *why*, in plain business terms, often citing the owner's request.

## Workflow

- Edit `index.html` in the repo. The owner no longer uploads files by hand, so the repo copy is the live source of truth.
- Work on the assigned branch, then open a PR into `main` when asked. Merging to `main` deploys via GitHub Pages.
- Changes to `.gs` files still have to be pasted into Apps Script and redeployed as a new version (rule 7). A front-end change that depends on new server code must not be merged to `main` until the owner has deployed that server code, because the live site would break.
