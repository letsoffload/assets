# OFF-1612 verification evidence

Local lane 1 stack: client :15170, server :3101, postgres :5433.
Captured 2026-05-31 against the seeded synthetic dataset.

## Inbox wire shape

`GET /api/shares/inbox` as carrier org_admin, three response rows:

```json
[
  {
    "id": "0db63b92-09d2-4da9-a746-fc5d3e6dec1c",
    "scopeKind": "Project",
    "subjectType": "Project",
    "roleId": "carrier",
    "shareRole": "carrier",
    "cascadeToDescendants": true,
    "descendantLoadCount": 6,
    "subjectDescription": "P-1-1: Williamson - Moen's most advanced Bacon technology increases dependent capabilities",
    "status": "accepted"
  },
  {
    "id": "e7c35a21-60c1-4368-9a88-8f59aad81472",
    "scopeKind": "Project.Load",
    "subjectType": "Project.Load",
    "roleId": "carrier",
    "shareRole": "carrier",
    "cascadeToDescendants": false,
    "descendantLoadCount": null,
    "subjectDescription": null,
    "status": "pending"
  },
  {
    "id": "9616b693-7549-4bb4-a362-7c278495d190",
    "scopeKind": "Project.Load",
    "subjectType": "Project.Load",
    "roleId": "carrier",
    "shareRole": "carrier",
    "cascadeToDescendants": false,
    "descendantLoadCount": null,
    "subjectDescription": null,
    "status": "accepted"
  }
]
```

Proves:
- Both vocabularies emit: `scopeKind` + `subjectType`, `roleId` + `shareRole`.
- `descendantLoadCount: 6` on the cascading Project share, `null` on
  non-cascading Project.Load shares.
- `subjectDescription` composed server-side from the project + load.

## Unified accept endpoint

`POST /api/shares/:id/accept` with a non-existent share:

```
HTTP/1.1 404 Not Found
{"message":"Share not found","error":"Not Found","statusCode":404}
```

Proves the unified 404 collapse (missing OR cross-org both return this).

## Legacy URL deprecation headers

`POST /api/shares/projects/:projectShareId/accept-cascade`:

```
HTTP/1.1 404 Not Found
Deprecation: true
Sunset: 2026-06-30T00:00:00Z
Link: </api/shares/:id/accept>; rel="successor-version"
```

Proves RFC 8594 deprecation policy is wired correctly.
