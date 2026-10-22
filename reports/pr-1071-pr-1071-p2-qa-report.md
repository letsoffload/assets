# QA: PR #1071 (OFF-2615) share inbox role gate

Head 37de0fb5d, base 6145e6510, one local stack (client 15174, server 15005), synthetic seed. Recipient org Goodwin - Dare Transport; sender Spinka and Sons Freight Brokers; five pending Project.Load shares L-1-10..L-1-14 inserted as `resource_role_grants` rows (status pending, recipient_org_id set), the shape the inbox query reads.

**Verdict: PASS-WITH-NOTES.** Every AC that a persona walk can reach holds at head. Notes are UX, not access.

## Persona walk at head

| Persona | inbox / pending / sent | accept | decline | Web dialog |
|---|---|---|---|---|
| org_admin | 200, 14 / 3 / 5 rows | accepted L-1-10 in the UI | not exercised (404 on a random id) | live buttons |
| dispatcher | 200, 14 / 3 / 5 | accepted L-1-11 in the UI | not exercised | live buttons |
| viewer | 200, 14 / 2 / 5 | 403 | 403 | reason line, both disabled |
| permit_coordinator | 200, 14 / 2 / 5 | 403 | 403 | reason line, both disabled |
| driver | 200, 0 / 0 / 0 | 403 | 403 | no Partners entry, /partners redirects to /loads |

Base, same driver: inbox 200 with 14 rows (5 pending), `POST accept` 201, row flipped to accepted with `accepted_by_user_id` = the driver. Base viewer dialog: Decline live, Accept live once a project is picked.

L-1-12 stayed `pending` after the viewer, permit coordinator and driver decline/accept attempts (DB read back).

## Acceptance criteria

| AC | Verdict | Evidence |
|---|---|---|
| inbox/pending/sent return 200 [] for field tiers | PASS for driver (live); escort/rigger/crane by spec | driver probe; matrix-validation "field tiers resolve to []" |
| same endpoints return rows for office tiers | PASS (admin, dispatcher, viewer, PC live; ops_manager by spec) | probes |
| accept/decline/cascade 403 for field tiers, viewer, PC | PASS for accept + decline (driver, viewer, PC live); cascade by spec | probes, cascade-accept.http spec in CI |
| accept/decline still work for admin, ops_manager, dispatcher | PASS accept (admin, dispatcher live) | UI accepts, DB rows |
| scenario spec per role through the real guard | PASS | share-inbox-role-matrix.scenario.spec (CI server-integration green) |
| decorator-contract snapshot reviewed | PASS | 7 newly visible entries listed in the PR body |
| CI enumerates ungated mutating routes | PASS | route-authorization-inventory.spec 22/22 at head |

## Not run

- Maestro `apps/offload-mobile/.maestro/share-accept-role-gate.yaml`: NOT RUN. A lab phone (Samsung 2) was free, but no debug build of `com.letsoffload.mobile.local` exists for this branch and a from-scratch Gradle build did not fit the lean budget on this box.
- `ShareAcceptScreen.spec.tsx` locally: NOT RUN (lane has no mobile node_modules). CI `checks` is green at 37de0fb5d.
- ops_manager, project_planner, escort, rigger, crane: not logged in; covered by the matrix and scenario specs only.

## Findings (UX lens: a field or read-only user who can no longer act)

1. `libs/client/src/features/loads/components/share-load/AcceptShareDisclaimer.tsx:217` and `apps/offload-mobile/src/screens/ShareAcceptScreen.tsx:311`: the reason line says the account lacks "a permission" but not who can act. Suggest naming the seats: "Your org admin, ops manager or a dispatcher can accept or decline it." Kinder, and the user knows who to ask.
2. `AcceptShareDisclaimer.tsx:216-218`: the reason is plain `fg-mute` text above Permissions, the same weight as captions. An info callout from `@offload/ui` would read as a status, not fine print.
3. `AcceptShareDisclaimer.tsx:249-257`: the project picker stays live for a user who cannot accept (picked P-2-1 as the viewer at head). Its options include "Create new project" with no capability check (`onCreateProject` at :254 is ungated; the option list was read on the base viewer, and this code is unchanged at head). Disable it with the buttons so nothing invites a choice that goes nowhere.
4. `libs/client/src/features/loads/components/share-load/SharedLoadsInbox.tsx:458`: the row tooltip says "Review and accept" to a viewer who cannot accept. Say "Review" when `UPDATE Organization.Shares` is missing.
5. `libs/client/src/features/access-management/services/partners-attention.ts:205,214`: the Partners nav badge and the "Shared with me" pill count pending shares for READ-only seats (viewer and PC saw "2 pending"). That is a call to action they cannot take. Count it only for `UPDATE`.
6. Driver on mobile: per the ticket design the inbox is simply empty ("all caught up"). A driver a broker told to "accept the share" gets no clue to ask the office. Low priority; a one-line empty-state hint for field roles would close it.

## Shots

- Viewer dialog before/after, context: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-viewer-drawer-ctx.png
- Viewer dialog before/after, zoom: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-viewer-drawer-zoom.png
- Driver API before/after: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-driver-api.png
- Permit coordinator dialog: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-qa-pc-drawer.png
- Dispatcher dialog: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-qa-dispatcher-drawer.png
- Dispatcher after accept: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-qa-dispatcher-accepted.png
- Org admin dialog / after accept: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-head-admin-drawer.png , https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-head-admin-accepted.png
- Driver web landing: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-qa-driver-web.png
- Viewer dialog at phone width: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-head-viewer-drawer-phone.png
- Role matrix rows: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-role-matrix.png
- Route-inventory ratchet: https://raw.githubusercontent.com/letsoffload/assets/media/screenshots/pr/pr-1071-p2-route-inventory.png
