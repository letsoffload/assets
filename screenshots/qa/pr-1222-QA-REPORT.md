# QA Report: [OFF-2986] Guard people soft-delete against stranding open load assignments

**PR:** letsoffload/offload#1222  **Head:** `2bb60a073`  **Base:** `origin/main` `e47047f49`  **Date:** 2026-10-21
**Tested at:** lane `pr1222-p2` (client `localhost:15174`, server `localhost:15005`, seeded carrier org "Witting - Collins Transport")
**Served build:** server restarted from the lane at `2bb60a073` with `git diff HEAD -- apps/server` empty; the base run checked out `origin/main`'s two people files and restarted.

## Summary

Walked People delete as the org admin on base and head with a Kyle Butz style duplicate (driver on an open load, driver on a delivered load, crew card), then a load People-tab contact, then a user-linked person. Dispatcher has no delete path (menu has no Delete item, API 403), so the guard is not reachable for that role. Head refuses every stranding delete, names the loads, and the delete goes through once each assignment is reassigned or removed.

**Verdict: PASS WITH NOTES** (notes are UX findings below, none block the guard)

## Acceptance criteria

| # | Criterion | Result | Evidence | Notes |
|---|-----------|--------|----------|-------|
| 1 | Base: deleting a person still on loads succeeds and leaves the assignments on a hidden person | PASS (bug reproduced) | 00b, 00c, `raw/base-db-after-delete.txt` | Load still renders "Kyle Butz +15552010187" for a person gone from People; DB shows 3 live rows on deleted person 2001 |
| 2 | Head: delete refused (409) while driver on an open load, a delivered load and a crew card; message names all three | PASS | 01 | "still assigned to L-2-3, L-2-7, crew P-2-1 on 2026-11-03" |
| 3 | Head: a `load_people` contact row also blocks | PASS | 03 | names L-2-5 |
| 4 | Reassign to the kept record (Change driver) and remove (Unassign driver, Remove from load), then delete succeeds | PASS | 02a, 02b, 05, 02c, 06 | crew card cleared via the board's PATCH endpoint, not the board UI (see Not shown) |
| 5 | Linked internal user gets its own 409 first, even with live loads | PASS | 07, 07b | UI disables Delete; API returns the linked-user 409 for Vaughn Towne (driver on 4 loads) |
| 6 | Dispatcher persona | N/A | 08 | no Delete item; `DELETE /api/people/2002` = 403 `delete on Organization.Resources` |
| 7 | "and N more" past 10 | SKIP | - | unit test `caps the named loads at 10 and counts the rest` covers it |

**Coverage: 6/7 tested, 5 pass, 1 n/a, 1 skip.**

## Findings

| # | Surface | Persona | Expected | Actual | Severity | Evidence |
|---|---------|---------|----------|--------|----------|----------|
| F-1 | Delete person modal | org admin | One error, in the DS pattern | Same 409 shown twice: inline modal Alert (`libs/client/src/features/entity-menus/resource/useResourceMenu.tsx:273-277`) AND a red toast (`libs/client/src/features/assets/services/people/mutations.ts:378-388`). Drop the toast when the modal shows the error | med | 01 |
| F-2 | 409 message | org admin | Load names are links | Plain text. Server returns a string (`apps/server/src/modules/people/people.service.ts:65-76`). A structured 409 body (`assignments: [{kind, label, uuid}]`) would let the modal render each as a link to the load / board day | med | 01 |
| F-3 | Delete modal copy | org admin | Copy matches the new rule | Body still says delete "can't be undone and will impact historical assignment data" (`libs/client/src/features/resources/model.ts:312-315`); on head a delete with assignments is refused, and the row is a soft delete | low | 01 |
| F-4 | Delete modal | org admin | Confirm disabled (or relabelled) after a 409 | "Delete person" stays live and re-fires the same 409 | low | 01 |
| F-5 | Crew label | org admin | Office user can find the card | "crew P-2-1 on 2026-11-03": job number, not the phase (P-2-1-SITE) or slot name, ISO date (`people.service.ts:74-75`); the board opens on today, so finding Nov 3 is a manual hunt | low | 01 |
| F-6 | Replace driver confirm | org admin | Duplicates distinguishable | "Kyle Butz -> Kyle Butz" with no phone or id; the exact duplicate case this PR is about (`libs/client/src/features/dispatch/components/confirmation/AssignmentConfirmDialog.tsx`) | low | 02a |
| F-7 | Load People tab remove | org admin | DS confirm dialog | Native `window.confirm('Remove this person from the load?')` (`libs/client/src/features/loads/components/projection/sections/ProjectionPeopleContacts.tsx:121`); pre-existing, but it is the remediation path the 409 points to, and it hung the automation browser | low | 05 |
| F-8 | Linked-user copy | org admin | One instruction | Menu tooltip says "Remove their access in Settings instead" (`libs/client/src/features/entity-menus/reasons.ts:71`); the server 409 says "Contact a system admin to deactivate their access" | low | 07, 07b |

## Not shown

- Crew card cleared through the Dispatch board UI: the board needed `digitalDispatcher` turned on for the lane org and opens on today; the clear went through `PATCH /api/dispatcher/schedule/:instance/dispatch/:crew` (the endpoint the board calls). UNVERIFIED in the board UI.
- Partner-org loads (known gap, pinned by the `KNOWN GAP` integration case) and load copy carrying soft-deleted people: not walked. UNVERIFIED here.

## Screenshot manifest

| # | File | Description |
|---|------|-------------|
| 00 | `00-base-two-kyles.png` | Base: two Kyle Butz records |
| 00b | `00b-base-delete-succeeds.png` | Base: delete of 201-0187 succeeds |
| 00c | `00c-base-load-still-shows-deleted.png` | Base: L-2-7 still shows the deleted Kyle as driver |
| 01 | `01-admin-refused-driver-crew.png` | Head: 409 names L-2-3, L-2-7, crew card |
| 02a | `02a-admin-replace-driver-confirm.png` | Head: Change driver to the kept record |
| 02b | `02b-admin-l27-reassigned.png` | Head: L-2-7 driver now 201-0609 |
| 02c | `02c-admin-delete-ok.png` | Head: delete succeeds after reassign/unassign/crew clear |
| 03 | `03-admin-loadpeople-refused.png` | Head: contact on L-2-5 blocks |
| 04 | `04-admin-load-people-tab.png` | L-2-5 People tab |
| 05 | `05-admin-load-people-removed.png` | After Remove from load |
| 06 | `06-admin-delete-ok.png` | Head: delete succeeds |
| 07 | `07-admin-linked-user-menu.png` | Linked user: Delete disabled, tooltip |
| 07b | `07b-linked-user-409.txt` | Linked user: API 409 body |
| 08 | `08-dispatcher-menu.png` | Dispatcher: no Delete item |

## Test environment

- agent-browser (Chromium), viewport 1600x900, direct Auth0 login per persona (carrier org_admin, carrier dispatcher)
- Lane Postgres (`pr1222-p2-pg`), synthetic seed + two Kyle Butz records, one phase, one crew card; `digitalDispatcher` row added for org 2 in the lane DB only
