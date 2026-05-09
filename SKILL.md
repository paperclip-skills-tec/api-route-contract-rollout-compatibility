---
name: api-route-contract-rollout-compatibility
description: Use when introducing or modifying Express routes, replacing legacy route shapes, publishing a canonical endpoint, or rolling out any breaking API route changes. Trigger for tasks involving route renaming, path migrations, legacy-to-canonical transitions, adding/removing path parameters, or any change that alters an endpoint's address. Also trigger when a QA engineer or monitor consumer needs to verify an API route change didn't break existing callers. This skill is critical before merging any PR that touches Express router definitions — it prevents the 404s, shape drift, QA dead-ends, and monitor breakage that repeatedly follow undocumented route migrations. Even if the task just says "add a route" or "move this endpoint", use this skill — it defines the canonical order of operations that avoids rework.
---

# API Route Contract Rollout & Compatibility

This skill ensures that when Express routes are introduced, renamed, or replaced, all consumers get advance notice, all paths get tests, and no one hits an unexpected 404.

## Why This Matters

Route migrations without backward compatibility cause downstream failures that are hard to trace. A QA engineer tests `/api/issues/` and it works. A monitor hits `/api/companies/:id/issues` because that's what the docs say. The docs are wrong. The monitor pages. This checklist prevents that chain of events.

Past incidents that prompted this skill: multiple issues required rework because documented endpoints were unavailable (404) or route variants drifted from implementation, causing QA dead-ends and monitor breakage.

## When to Apply

Apply this skill whenever you are:

- Adding a new canonical route that replaces or supersedes a legacy path
- Renaming or moving an existing API route
- Adding or removing route parameters from a path
- Changing a route's HTTP method (e.g. GET → POST)
- Retiring or deprecating an endpoint
- Creating any route that will be consumed by monitors, QA scripts, agents, or external callers

## The Compatibility Checklist

Work through these five steps in order. Do not merge a route-change PR without completing all of them.

---

### Step 1: Write a Route Change Record

Before implementing the change, document your decision. Add it as an issue comment or a `route-change-record.md` in the PR:

```
Route Change Record
===================
Legacy path:     GET /api/companies/:companyId/issues/:issueId
Canonical path:  GET /api/issues/:issueId
Reason:          Legacy path included company prefix unnecessarily; single-issue
                 lookups don't need company context
Coexistence:     Both paths supported for 30 days post-merge, then legacy deprecated
Consumers:       QA test suite, monitor scripts, agent heartbeat loop
Breaking:        No (legacy kept alive during coexistence window)
```

This record is what prevents future agents and humans from being surprised by a route's history. It also serves as the acceptance criteria for QA.

---

### Step 2: Build a Backward-Compatibility Matrix

List every legacy path affected and its disposition. Be explicit — implicit is how 404s happen:

| Legacy Path | Status | Canonical Path | Works Until |
|---|---|---|---|
| `GET /api/companies/:id/issues/:issueId` | KEPT (forwards to canonical) | `GET /api/issues/:issueId` | 30 days post-merge |
| `POST /api/companies/:id/issues/:issueId/comments` | KEPT | `POST /api/issues/:issueId/comments` | 30 days post-merge |
| `GET /api/companies/:id/issues/:issueId/status` | REMOVED | *(absorbed into response body)* | Removed at merge |

Any path in a "REMOVED" row must either have no known callers or have a documented migration path in the PR before merge. If callers exist, switch them to the canonical path first or extend the coexistence window.

---

### Step 3: Update Swagger / OpenAPI

Update the OpenAPI spec in the same PR as the route change. The spec is what QA engineers and monitors read when they don't have the Express router source. If the spec drifts, downstream breakage is guaranteed.

- Add the canonical path as the primary documented route
- Mark deprecated routes with `deprecated: true`
- Add a `description` note on deprecated entries: `"Deprecated — use GET /api/issues/{issueId} instead. Will be removed 30 days post-merge."`
- Remove entries for routes that are fully gone

---

### Step 4: Write Route-Level Tests for Both Paths

Write tests that cover both the legacy path and the canonical path during the coexistence window. This is the single most important defensive measure against forward-compat regressions.

```javascript
describe('Issue lookup — backward compatibility', () => {
  it('canonical path responds with issue', async () => {
    const res = await request(app).get('/api/issues/TEC-123');
    expect(res.status).toBe(200);
    expect(res.body.id).toBe('TEC-123');
  });

  it('legacy company-prefixed path still resolves during coexistence', async () => {
    const res = await request(app)
      .get('/api/companies/abc123/issues/TEC-123');
    expect(res.status).toBe(200);
    expect(res.body.id).toBe('TEC-123');
  });
});
```

If the legacy path is **removed at merge** (not kept), write a test asserting it returns `404` or `410 Gone` — not that it silently returns something unexpected:

```javascript
it('removed legacy path returns 410 Gone', async () => {
  const res = await request(app)
    .get('/api/companies/abc123/issues/TEC-123/status');
  expect(res.status).toBe(410);
});
```

---

### Step 5: Provide Verification Examples for QA and Monitors

Before closing the task, provide at least one concrete example a QA engineer or monitor can run to verify the change. These go in the issue's closing comment or PR description so QA engineers can copy-paste them without reading router source:

```bash
# Verify canonical path
curl -s -H "Authorization: Bearer $TOKEN" \
  "$API_URL/api/issues/TEC-123" | jq '.id'
# Expected: "TEC-123"

# Verify legacy path still works during coexistence window
curl -s -H "Authorization: Bearer $TOKEN" \
  "$API_URL/api/companies/$COMPANY_ID/issues/TEC-123" | jq '.id'
# Expected: "TEC-123"

# Verify removed path returns 410 (if applicable)
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN" \
  "$API_URL/api/companies/$COMPANY_ID/issues/TEC-123/status"
# Expected: 410
```

---

## Pre-Merge Completion Gate

Before marking the task done or merging the PR, verify each item:

- [ ] Route Change Record written (issue comment or PR description)
- [ ] Backward-Compatibility Matrix explicit about every affected path
- [ ] Swagger/OpenAPI spec updated in the same PR
- [ ] Tests exist for canonical path AND legacy path (or 410 test if legacy removed)
- [ ] Verification curl examples provided for QA and monitors
- [ ] Any monitors or scheduled jobs hitting legacy paths have been notified or updated

---

## Scope Boundaries

This skill governs **route lifecycle**: introducing routes, managing coexistence between legacy and canonical paths, and ensuring all consumers have a verified migration path.

It deliberately does **not** cover:

- **Per-route request/response hardening** (input validation, error shapes, response body schema enforcement) — see `express-api-hardening-playbook`
- **Consumer-side response shape verification** (asserting the API client handles the shape correctly) — see `api-contract-validation`

These three skills are complementary, not redundant: this one handles the route *address*, the hardening playbook handles the route *body*, and api-contract-validation handles the *client side*. When a route change also changes the response shape, apply all three.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
