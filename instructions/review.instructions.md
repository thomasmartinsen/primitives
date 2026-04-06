---
applyTo: '**/*'
description: 'Code review checklist and governance rules'
---

# Review guidelines

Ensure correctness, consistency, and governance alignment across specs, contracts, documentation, and code.

---

## Scope

- Spec template reviews.
- API/MCP contract reviews.
- Security documentation review.
- Test plan review.
- Structured feedback.

---

## Non-goals

- No implementation decisions.
- No architecture ownership.
- No CI/CD changes.
- No package approvals.

---

## Review rules

- Classify issues as **Blocker**, **Major**, or **Minor**.
- Flag scope creep.
- Verify REST vs MCP separation.
- Enforce auth assumptions (Microsoft Entra ID, JWT Bearer tokens, `fetchAuthorized()` usage).
- Require testability for all new service methods, endpoints, and components.
- Hand off cross-domain issues explicitly.

---

## Output format expectations

Markdown review report with the following sections:

- **Summary** — high-level assessment.
- **Blockers** — issues that must be resolved before merge.
- **Majors** — significant issues that should be addressed.
- **Minors** — suggestions and nits.
- **Next actions** — concrete follow-up steps.

---

## Code review checklist

Before committing code, verify:

- [ ] Follows SOLID principles and Clean Architecture layering rules.
- [ ] No AutoMapper or MediatR usage.
- [ ] Uses `System.Text.Json` (not Newtonsoft.Json).
- [ ] Async/await used correctly with `CancellationToken` support.
- [ ] No blocking calls (`.Result`, `.Wait()`).
- [ ] All I/O is async.
- [ ] Tests written and passing.
- [ ] Error handling implemented via global exception handling middleware.
- [ ] No secrets in code — uses environment variables, Azure Key Vault, or managed identity.
- [ ] Domain entities are not exposed through API responses — DTOs are used.
- [ ] Endpoints are thin: validate input, call application services, return DTOs.
- [ ] Endpoints do not access repositories directly.
- [ ] All inputs validated at system boundaries.
- [ ] No SQL or NoSQL queries built via string concatenation.
- [ ] File-scoped namespaces used.
- [ ] Blank line between block and subsequent statement.
- [ ] `docs/openapi/studio-api-v1.json` has been regenerated when API contracts changed.
- [ ] Configuration documented.
