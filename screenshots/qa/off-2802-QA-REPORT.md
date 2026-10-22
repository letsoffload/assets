# QA Report: [OFF-2802] Permit review form flips Washington permits to District of Columbia

**Branch:** `nate/off-2802-permit-review-form-flips-washington-permits-to-district-of` at `e31900ad0` (PR #1162)  **Date:** 2026-10-21  **Tested at:** `http://localhost:15173` (lane pr1162-p2, server :15004, isolated Postgres :15436)
**Shots:** `/tmp/qa-off-2802/`  **Base comparison:** same lane and data with the six changed source files checked out at `6145e6510` (origin/main), captured in the proof run.

## Summary

Walked the permit coordinator (broker org) through the legacy permit edit and smart-scan review modals on web desktop, and the `/review-permit/:jobId` mobile web view (coordinator, then the carrier driver the view is built for). Read-path enrichment, open-and-save, the D.C./city/Kansas cases and the TN empty-type scan all behave at head. The manual picker refusal cannot be reached from the UI: clearing the Jurisdiction box clears the state first, so clear, type "Washington", take the top result still saves a Washington permit as District of Columbia, same as main.

**Verdict: PASS WITH NOTES** (F-1 is a real gap in fix item 1/2's intent, see Findings)

## Acceptance criteria

| # | Criterion (from OFF-2802 "Fix") | Result | Evidence | Notes |
|---|-----------|--------|----------|-------|
| 1 | Opening and saving a Washington permit keeps `state=Washington` | PASS | #01, #02 | DB after save: `Washington`, city/county empty, fc `Washington, United States` (region) |
| 2 | Server enrich rejects a feature whose region does not match | PASS | proof DB pair | Base read cached `Washington, District of Columbia` on the WA permit and `Kansas City, Missouri` on KS; head caches `Washington, United States` and `Kansas, United States` |
| 3 | A real D.C. permit stays D.C. | PASS | #03 | DB `District of Columbia`, fc region D.C. |
| 4 | City permit (Springfield, IL) and Kansas permit | PASS | #04, #05 | Springfield fc `Springfield, Illinois`; KS fc is the Kansas region, no Kansas City MO |
| 5 | Smart-scan TN with empty area type is saveable as Tennessee | PASS | #06, #06b, #08 | Web: Confirm & Save created permit 5005 `state=Tennessee`; base shows a blank Jurisdiction and "Please specify a location" and does not save. Mobile web view prefills Tennessee |
| 6 | Manual picker refuses a pick outside the permit's state | FAIL | #10, #11 | Unreachable in the running app; see F-1 |
| 7 | Clearing then picking works | PASS | #10, #11 | Clear then pick D.C. sets District of Columbia (also how F-1 happens) |

**Coverage: 7/7 tested, 6 pass, 1 fail, 0 skip.**

## Findings

| # | Surface | Persona | Expected | Actual | Severity | Evidence |
|---|---------|---------|----------|--------|----------|----------|
| F-1 | Edit Permit modal, Jurisdiction picker | permit_coordinator | Picking "Washington D.C." on a Washington permit is refused | The only way into search mode is the × (`AddressLookup.tsx` handleClear calls `onChange(null)`), and `GenericForm.tsx:105-111` then sets `state=''`, so `verifyJurisdictionPick` at `:116-119` sees no current state and accepts D.C. Typing "Washington" lists Washington D.C. first (Google). Saved row: `state='District of Columbia', city='Washington'` | high | #10, #11 |
| F-2 | `/review-permit/:jobId` | permit_coordinator | A refusal or redirect | `GET /api/drivers/smart-scan-jobs/911620` 404 (driver-only), page shows a spinner forever. Pre-existing, not this PR | low | #07 |
| F-3 | `/review-permit/:jobId` Save | driver | Save creates the permit or says why not | `MobileReviewPermit.tsx:375-382` returns silently when the job has no permit routes; the fixture job had none, so Save did nothing (form validated with `state: "Tennessee"` in the console). Pre-existing, not this PR | low | #09 |

## Screenshot manifest

| # | File | Description |
|---|------|-------------|
| 01 | `01-act1-coordinator-wa-permit-reopened.png` | WA permit reopened after save, Jurisdiction "Washington" |
| 02 | `02-act1-coordinator-wa-card-after-save.png` | Sidebar card "WA - #PR1162-WA" under Washington |
| 03 | `03-act2-coordinator-dc-permit.png` | D.C. permit, Jurisdiction "District of Columbia" |
| 04 | `04-act2-coordinator-springfield-permit.png` | City permit, "Springfield, Illinois" |
| 05 | `05-act2-coordinator-kansas-permit.png` | Kansas permit, "Kansas" |
| 06 | `06-act2-coordinator-tn-scan-review.png` | Smart-scan review, Jurisdiction prefilled "Tennessee" |
| 06b | `06b-act2-coordinator-tn-card-after-save.png` | Saved card "TN - #PR1162-TN" |
| 07 | `07-act3-coordinator-mobile-review-spinner.png` | Mobile web view as coordinator: spinner (F-2) |
| 08 | `08-act3-driver-mobile-review-tennessee.png` | Mobile web view as carrier driver, Details tab: "Tennessee" |
| 09 | `09-act3-driver-mobile-save-noop.png` | After Save, unchanged (F-3) |
| 10 | `10-act4-coordinator-clear-then-pick-dc.png` | After ×, "Washington" search, top result picked: accepted, no message |
| 11 | `11-act4-coordinator-wa-card-flipped-to-dc.png` | Card reads "DC - Washington (City) - #PR1162-WA" after save (F-1) |

## Callouts

- **The Good** - Jurisdiction label now reads from the permit's own fields, so it is never blank and never a geocoder string ("Washington" vs main's "Washington, USA").
- **The Bad** - F-1: the realistic manual path (clear, type the state name, take the top hit) still flips the state.
- **The Clever** - The read-path enrich no longer trusts `features[0]`; the Kansas case shows it caught a second wrong-place class, not just D.C.

## Test environment

- Browser: agent-browser (Chromium)  Viewport: 1920x1080 desktop, 390x844 mobile web view
- Auth: direct Auth0 login as the seeded permit_coordinator (broker) and driver (carrier)
- Stack: lane pr1162-p2 at `e31900ad0`, served code confirmed (`pickMatchingGeoFeature` present in `dist/apps/server/main.js`; tracked tree clean)
- Data: `yarn seed:synthetic` (tag pr1162-p2) plus raw fixtures: permits PR1162-WA/DC/SPI/KS on L-1-3 with off-route jurisdictions, document_process_job 911620 (L-1-3) and 911621 (L-1-21) carrying `{geographic_area_name:'TN', geographic_area_type:''}`. No extraction run, no Bedrock. Geocoding hit live Google Places (client) and Mapbox (server).
