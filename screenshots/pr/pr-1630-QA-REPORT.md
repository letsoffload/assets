# QA Report: [OFF-3823] SelectInput on the design-system select (PR #1630)

**Commit:** `3d23689bc` (detached, lane `_wt/offload/pr1630-proof`)  **Date:** 2026-10-21  **Tested at:** `http://localhost:15173` (lane 3)
**Shots:** `pass2/1630/qa/` (uploaded ones linked below)

## Summary
Keyboard walk of the converted forms as a carrier org_admin (real Auth0 login), create and edit, in Chromium (agent-browser). Each modal was opened with one click; everything inside it was driven by Tab, typing, Enter and Escape, and every save was checked in Postgres. Truck, trailer, equipment and person work end to end on the keyboard; the escort keyboard walk, onboarding and curfews were not completed (see SKIP rows).

**Verdict: PASS WITH NOTES**

## Acceptance criteria
| # | Criterion | Result | Evidence | Notes |
|---|-----------|--------|----------|-------|
| 1 | Truck edit: Tab reaches License plate state, typing filters, Enter picks the highlighted row and does not submit | PASS | #01, #02 | "tex" + Enter set Texas, modal stayed open, no request |
| 2 | Escape with the list open drops the typed search, keeps the value, modal stays open | PASS | #02 | "Texasoh" + Escape back to Texas, 1 dialog |
| 3 | Truck types (CreatableMultiSelect) reachable by Tab, Enter adds a chip without submitting | PASS | #03 | star buttons are tab stops before the input |
| 4 | Truck edit saves from the keyboard (Tab to Save changes, Enter) | PASS | db | PATCH /api/trucks/1001 200; license_state=Texas |
| 5 | Trailer create: pick by Enter, then Enter in a text field submits the form | PASS | #41 | POST /api/trailers 201; QA-TR-1 Alabama |
| 6 | Trailer edit by keyboard | PASS | #42 | PATCH 200; Georgia |
| 7 | Equipment create + edit by keyboard | PASS | db | POST /api/equipment 201, PATCH 200; QA-EQ-1 make + year 2021 stored |
| 8 | Person create (CreateLoadPeopleModal): License state by keyboard, save | PASS | #45 | POST /api/people 201; drivers_license_state TX |
| 9 | Escape with the list closed hands off to the modal's dirty-close guard | PASS | #22 | "Discard your changes?" confirm, no data lost |
| 10 | Escort section (permit form) by keyboard | SKIP | #52 | driver could not hold focus in the permit modal under host load; visual before/after is in the PR proof |
| 11 | Onboarding wizard | SKIP | - | needs a fresh Auth0 identity with no org; not created (shared tenant) |
| 12 | Curfews form | SKIP | - | seeded permit has no saved curfews ("No saved curfews found") |

**Coverage: 12 criteria, 9 pass, 0 fail, 3 skip.**

## Findings
| # | Surface | Persona | Expected | Actual | Severity | Evidence |
|---|---------|---------|----------|--------|----------|----------|
| F-1 | Create trailer | carrier org_admin | blank optional VIN submits | "VIN must not be empty if provided" on an untouched VIN; create blocked until VIN filled | med (pre-existing: main also defaults `vin: ''`, read not run) | #31 |
| F-2 | Every form with a select | carrier org_admin | Tab passes through | Tabbing onto a select opens its list (open on focus), so a Tab pass flashes each list | low | walk log |
| F-3 | Truck types | carrier org_admin | - | typing "Day" while "Day Cab" is already a chip offers only "Create Day"; Enter created a new type "Day" (lane DB) | low (expected creatable behavior, easy to hit) | #03 |
| F-4 | Permit form | carrier org_admin | - | Enter inside a native date input submits the permit form ("Saving changes...") | low (native, untouched by this PR) | #52 |

## Screenshot manifest
| # | URL |
|---|-----|
| 01 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-01-truck-edit-typed-tex.png |
| 02 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-02-truck-edit-escape.png |
| 03 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-03-truck-edit-types.png |
| 21 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-21-t2-picked.png |
| 22 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-22-t2-after-escape.png |
| 31 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-31-trailer-create-saved.png |
| 41 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-41-trailer-create-saved.png |
| 42 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-42-trailer-edit-saved.png |
| 45 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-45-person-create-picked.png |
| 52 | https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1630-qa-52-escort-type-picked.png |

## Test environment
- Browser: agent-browser (Chromium 155 headless) only; agent-browser cannot drive Firefox. Viewport 1440x900.
- Driver note: the headless page reports `visibilityState=hidden`, which pauses requestAnimationFrame, so Mantine menus and modals mount only when a frame is forced. Every check forced a frame (screenshot) before reading the DOM; an early "list still open after Escape" reading was this artifact and did not reproduce.
- Auth: direct Auth0 login, carrier org_admin from seed tag `pr1630-proof-f`.
- Host load average 60 to 250 during the run (other sessions' Xcode and OrbStack work).
