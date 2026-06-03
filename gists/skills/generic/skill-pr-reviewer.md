# Skill — PR Reviewer

> Type: Generic Utility
> Used by: Every team, every sprint
> Purpose: Review a PR against its Spec, contracts, and DDD principles before merging

---

## What This Skill Does

A PR review in PromotionOS is not a style check. It is a verification that the implementation matches the Spec, respects the contracts, follows DDD principles, and passes all validation scenarios.

**The rule:** Every PR must be reviewed against its Spec before merging. No exceptions.

---

## When to Use

Use this skill when:
- Your implementation is complete and you are about to open a PR
- You are reviewing another team member's PR
- You want Claude to review your implementation before you push

---

## The Review Checklist

Run through every item. A PR fails review if any item is NO.

### 1. Spec Compliance

- [ ] Every domain rule listed in the Spec is implemented
- [ ] Every acceptance criterion in the Spec has a corresponding test or validation
- [ ] Nothing is implemented that is not in the Spec (no gold plating)
- [ ] Out of scope items from the Spec are not in this PR

### 2. Ubiquitous Language

- [ ] All domain terms in code match the Ubiquitous Language doc exactly
- [ ] No synonyms — not "promotion" instead of "campaign", not "transaction" instead of "redemption"
- [ ] Variable names, method names, field names use the correct domain terms

### 3. DDD Principles

- [ ] Business logic is in the domain layer — not in controllers, not in repositories
- [ ] Aggregate root is the entry point — no direct manipulation of child entities from outside
- [ ] Value objects are immutable — no setters
- [ ] Repository interface is in domain layer — implementation is in infrastructure
- [ ] Domain events are raised by the aggregate — not by the application service or controller
- [ ] ACL is used to translate incoming events — no raw event fields used directly in domain logic
- [ ] No cross-context database calls — only API calls or event consumption

### 4. Contract Compliance

- [ ] All published events match the exact schema in the relevant contract doc
- [ ] All consumed events are translated through ACL before entering domain logic
- [ ] `schemaVersion` is set correctly on all published events
- [ ] `tenantId` is present on all events and API responses
- [ ] No contract fields have been renamed, removed, or type-changed without schema version bump

### 5. Validation

- [ ] All acceptance criteria scenarios from the Spec pass
- [ ] All regression scenarios (previously passing) still pass
- [ ] Edge cases from the Spec are tested
- [ ] Boundary conditions are tested (e.g. exactly 95.0% not 94.9%)

### 6. Tenant Isolation

- [ ] Every DB query filters by `tenantId`
- [ ] Every API response is scoped to the request's `tenantId`
- [ ] No data from one tenant is accessible to another

### 7. Error Handling

- [ ] All error codes match the contract exactly (e.g. `NO_FUNDING_SOURCE` not `FUNDING_MISSING`)
- [ ] No raw exceptions returned to API callers — mapped to domain error codes
- [ ] No 500 errors for domain validation failures — these must be 4xx

### 8. ADR

- [ ] An ADR exists for every significant decision in this PR
- [ ] ADR status is ACCEPTED
- [ ] ADR is linked in the PR description

---

## How to Use This Skill with Claude

**Self-review before pushing:**
"Use the pr-reviewer skill. Here is my implementation: [paste key methods]. Here is the Spec it should match: [paste Spec]. Review it."

**Review another team member's PR:**
"Use the pr-reviewer skill. Here is the PR diff: [paste diff]. Here is the Spec: [paste Spec]. What is missing or wrong?"

Claude will run through the checklist and flag every item that fails. It will not approve a PR that violates a contract or domain rule.

---

## PR Description Template

Every PR must include this description:

```
## Spec Reference
[Link to or paste the Spec this PR implements]

## ADR Reference
[Link to or paste the ADR for significant decisions]

## Validation Scenarios Passing
- [ ] Scenario N: [description]
- [ ] Scenario N: [description]

## Contract Impact
- [ ] No contract changes
- OR
- [ ] Contract [name] updated — schemaVersion bumped from X to Y
- [ ] Contract skill [name] updated
- [ ] Consuming team [name] notified

## Regression Check
- [ ] All previously passing scenarios still pass

## DDD Checklist
- [ ] Business logic in domain layer
- [ ] No cross-context DB calls
- [ ] Tenant isolation enforced
```

---

## Common PR Failures

| Failure | What to do |
|---------|-----------|
| Business logic in controller | Move to aggregate or domain service |
| Raw event fields used in domain | Add ACL translation layer |
| Missing tenantId filter in query | Add to every DB query |
| Error code doesn't match contract | Fix to match contract exactly |
| Acceptance criterion not tested | Add test or manual validation |
| ADR missing | Write ADR before merging |
| Contract changed without version bump | Bump schemaVersion, update skill |
| Ubiquitous Language violated | Rename to match exactly |
