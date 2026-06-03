# PromotionOS — Software Development Lifecycle (SDL)

> This document defines the complete SDL for building PromotionOS.
> Every phase, every sprint, every team's work is sequenced here.
> This is the single source of truth for how we build.

---

## What We Are Building

**PromotionOS** — an AI-assisted Trade Promotions Management platform that challenges Oracle Trade Management. Built by software developers using Spec Driven Development (SDD), Domain Driven Design (DDD), and Claude as a force multiplier.

**Tech Stack:**
- Campaign, Eligibility, Redemption → Java / Spring Boot
- Catalog & Customer, Analytics, Event Store → Go / Gin
- Frontend → React.js
- Event Bus → Redis Pub/Sub
- Databases → PostgreSQL (one per bounded context)
- Deploy → Railway (backend), Vercel (frontend)

---

## The SDL at a Glance

```
Phase 0 — Data & Design        ✅ Complete
Phase 1 — Setup                ⬜ Pseudo org + skeletons
Phase 2 — Transition           ⬜ Build Campaign + Analytics, plant bugs
Phase 3 — Sessions             ⬜ 4 sprints × 2hrs
```

---

## Service Dependency Graph

Understanding dependencies is critical — no team should be blocked by another team's sprint.

```
Campaign Service (pre-built in Phase 2)
    │
    ├──→ Eligibility Service ←── Catalog & Customer Service
    │         │                  (both Sprint 1 — parallel, no dependency)
    │         │
    │         └──→ Redemption Service
    │                   │        (Sprint 2 — depends on Eligibility API)
    │                   │
    │                   └──→ Analytics Service
    │                             (Sprint 3 — bug fix, depends on Redemption events)
    │
    └──→ Analytics Service (pre-built in Phase 2)

Frontend (Sprint 4 — depends on all services live)
```

**Key insight:** Campaign Service and Analytics Service are pre-built in Phase 2 with real business logic. This means Sprint 1 teams (Eligibility + Catalog) always have something real to integrate against. No team waits on another team's sprint output.

---

## Phase 0 — Data & Design ✅

**Owner:** Session facilitator
**Duration:** Completed

**Deliverables:**
- System architecture (bounded contexts, DDD, event sourcing)
- Ubiquitous language
- Dense real-world test data (48 UPCs, 12 customers, 12 campaigns, 30 validation scenarios)
- 5 inter-service contracts
- Requirements per team per sprint (4 sprints × 6 teams = 24 requirement docs)
- All skills (16 skills — generic, codebase, contract)
- Session guide + demo script

---

## Phase 1 — Setup

**Owner:** Facilitator + team via MCP
**Duration:** ~1 day
**Depends on:** Phase 0 complete

### What Gets Built

**GitHub Org: `promotionos`**

| Repo | Purpose |
|------|---------|
| `campaign-service` | Java / Spring Boot — Campaign bounded context |
| `eligibility-service` | Java / Spring Boot — Eligibility bounded context |
| `redemption-service` | Java / Spring Boot — Redemption bounded context |
| `catalog-customer-service` | Go / Gin — Catalog & Customer bounded context |
| `analytics-service` | Go / Gin — Analytics bounded context |
| `frontend-dashboard` | React.js — MX + CX + Analytics views |
| `shared-contracts` | JSON Schema — all event + API contracts |
| `skills` | Markdown — all 16 skills |

**Each repo contains:**
- Folder structure only
- Domain entities + value objects (fields, no methods)
- Repository + domain service interfaces
- DB schema + Flyway/Goose migrations
- GitHub Actions — test + deploy pipelines wired to Railway/Vercel
- README pointing to relevant gists

**Railway:**
- 6 PostgreSQL databases provisioned
- Redis provisioned
- 6 backend services wired — auto-deploy on merge to main
- Test data seeded into all DBs

**Vercel:**
- Frontend project wired — auto-deploy on merge to main

**GitHub Gists:**
- All 51 documents published (foundation, contracts, skills, session materials, sprint requirements)
- Sprint requirement gists are secret — handed to teams at sprint start

### Exit Criteria

- Every service boots and connects to its DB and Redis
- Every DB has test data seeded
- Every service returns `501 Not Implemented` on all routes
- All gists published
- All skills readable by Claude

---

## Phase 2 — Transition

**Owner:** Facilitator
**Duration:** ~1 day
**Depends on:** Phase 1 complete

### What Gets Built

**Campaign Service — fully implemented:**
- `Campaign.draft()` — create campaign with offer, funding, budget, date range
- `Campaign.publish()` — validate funding, check UPC overlap, emit CampaignPublished
- `Campaign.pause()` — emit CampaignPaused
- `FundingValidator` — vendorShare + krogerShare must equal 100%
- `ConflictResolver` — UPC overlap check across active campaigns
- `BudgetTracker` — real-time burn, threshold check, emit BudgetExhausted

**Analytics Service — fully implemented:**
- `LiftCalculator` — actual vs baseline, ROI calculation
- `BudgetBurnTracker` — real-time burn update, 95% threshold alert
- Event consumers for CampaignPublished, OfferRedeemed, BudgetExhausted

**Event Store — wired:**
- Consumes ALL domain events
- Appends to append-only PostgreSQL table
- No business logic — pure infrastructure

**Bugs planted:**

| Service | Bug | Location | Symptom |
|---------|-----|----------|---------|
| Campaign | Off-by-one in budget threshold | `BudgetTracker.checkThreshold()` | Campaigns pause at 94.9% not 95.0% |
| Analytics | Divide-by-zero on zero-cost campaigns | `LiftCalculator.calculateROI()` | 500 panic on camp-003, camp-004 |

### Exit Criteria

- Campaign Service fully functional except for the planted bug
- Analytics Service fully functional except for the planted bug
- Validation scenarios 1-6, 11-17, 21-25 pass
- Validation scenarios 8, 19, 20, 26 fail (bugs surface)
- Scenarios 7, 18, 27-30 pending (Redemption not built yet)
- Event Store receiving all Campaign + Analytics events

---

## Phase 3 — Sessions (4 Sprints × 2hrs)

**Owner:** Participant teams
**Duration:** 4 × 2 hours
**Depends on:** Phase 2 complete

### Sprint Structure (Every Sprint, Every Team)

```
0:00 - 0:15  Read sprint requirements from secret gist
0:15 - 0:35  Write spec using sdd-spec-writer skill
0:35 - 0:45  Cross-team contract review using contract skill
0:45 - 0:50  Record ADR using adr-recorder skill
0:50 - 1:40  Implement using codebase skill + Claude
1:40 - 1:50  Validate against test data scenarios
1:50 - 2:00  PR → GitHub Actions → deploy → confirm live
```

### SDD Process (Non-Negotiable)

Every team follows this for every feature, every sprint:

```
Requirements → Spec → Contract Review → ADR → Code → Validate → PR → Deploy
```

No code without a spec. No spec without requirements. No merge without validation passing.

### Sprint 1 — Foundations (2hrs)

**All 6 teams work in parallel on their service foundations.**

**Dependency note:** Team 2 calls Team 4's `GET /customers/:id` this sprint. Team 4 must deploy their endpoint before Team 2 runs final validation. Coordinate deployment order: Team 4 deploys first.

| Team | Service | Scenarios Targeted |
|------|---------|-------------------|
| Team 1 | Campaign Service | 9, 26 (bug fix — isolated, no dependencies) |
| Team 2 | Eligibility Service | 2, 3, 11, 12 (segment + status filter only) |
| Team 3 | Redemption Service | 7, 17 (idempotency + confirm) |
| Team 4 | Catalog & Customer | 21, 22, 23 (UPC lookup + tier boundary) |
| Team 5 | Analytics Service | 18, 19, 20 (bug fix — isolated, no dependencies) |
| Team 6 | Frontend | campaign list against mock data |

**What Team 2 builds Sprint 1:**
- `SegmentMatcher.match()` — segment restriction check only
- Campaign status filter — ACTIVE only
- `SegmentMatcher.match()` — loyalty tier + multi-segment
- `ACL.translate()` — translates CampaignPublished into eligibility rules
- Event consumers for CampaignPublished, CampaignPaused, CatalogItemExcluded

**What Team 4 builds:**
- `HierarchyResolver.isExcluded()` — full upward traversal
- `HierarchyResolver.resolveInheritance()` — downward exclusion propagation
- `SegmentCalculator.calculate()` — 90-day rolling window, tier calculation
- Event publishers for CatalogItemExcluded, SegmentUpdated

**Contract moment:**
Team 4 publishes `CatalogItemExcluded`. Team 2 consumes it via ACL. If Team 4 changes the event schema mid-sprint — they update the `catalog-eligibility-contract` skill. Team 2 uses that skill to refactor their ACL. No Slack. No meeting.

**Sprint 1 exit criteria — what actually works end of Sprint 1:**
- Scenarios 2, 3, 11, 12 pass (Team 2 — segment + status)
- Scenarios 7, 17 pass (Team 3 — idempotency)
- Scenarios 18, 19, 20 pass (Team 5 — analytics bug fix)
- Scenarios 21, 22, 23 pass (Team 4 — catalog + tier)
- Scenario 9 passes (Team 1 — no funding validation)
- Scenario 26 passes (Team 1 — budget threshold fix isolated)
- All 6 services deployed and live
- **Not yet:** 1, 4, 5, 6, 8, 10, 13-16, 24, 25, 27-30 (require Sprint 2 dependencies)

---

### Sprint 2 — Depth (2hrs)

**All teams deepen their Sprint 1 implementations. Full event chain becomes live.**

**Dependency note:** Team 2 now needs Team 4's `CatalogItemExcluded` event and Team 3's `OfferRedeemed` event feeds Team 5's burn tracker this sprint. All teams deploy in sequence: Team 4 → Team 2 → Team 3 → Team 5 → Team 1.

**Team 1 — Campaign Service:**
- Adds conflict resolution (UPC overlap check with date range)
- Adds `GET /campaigns/:id/summary` with vendorShare — **needed by Team 3 Sprint 2 for claim deduction**

**Team 2 — Eligibility Service:**
- Adds threshold evaluation, exclusion check, geo validation, stack validation
- Adds full ACL for CampaignPublished, CatalogItemExcluded, SegmentUpdated
- Adds `GET /offers` endpoint

**Team 3 — Redemption Service:**
- Adds claim generation using `GET /campaigns/:id/summary` from Team 1
- Adds ClaimSubmitted event publishing
- Adds `GET /redemptions/:id`

**Team 4 — Catalog & Customer:**
- Adds `POST /catalog/exclusions` + `CatalogItemExcluded` event
- Adds segment recalculation + `SegmentUpdated` event

**Team 5 — Analytics:**
- Adds real-time burn tracking via OfferRedeemed events
- Adds BudgetExhausted event publishing (exactly once)

**Team 6 — Frontend:**
- Wires campaign list to live Campaign Service API
- Adds analytics dashboard
- Adds real-time burn polling + alert banner

**Contract moment:**
Team 3 publishes `OfferRedeemed`. Team 5 consumes it and emits `BudgetExhausted`. Team 1 consumes that and emits `CampaignPaused`. This is the first full cross-service event chain. Validate it end-to-end at end of Sprint 2.

**Sprint 2 exit criteria — what actually works end of Sprint 2:**
- Scenarios 1, 4, 5, 6 pass (threshold + exclusions)
- Scenarios 8, 27 pass (full BudgetExhausted chain live)
- Scenario 10 passes (UPC overlap conflict)
- Scenarios 13-16 pass (geo + stacking)
- Scenarios 24, 25 pass (tobacco geo)
- Scenario 28 passes (claim at T+24 — needs live session timestamp)
- Scenario 29 passes (claim deduction = 60% of discount)
- Scenario 30 passes (GET /offers multi-segment)
- **Not yet:** full Frontend demo flows (Sprint 3)

---

### Sprint 3 — Bug Fixes (2hrs)

**Teams 1 and 5 fix planted bugs via SDD**
**Independent — no inter-dependency**

| Team | Service | Bug | Scenarios |
|------|---------|-----|-----------|
| Team 1 | Campaign | Off-by-one budget threshold | 8, 26, 27 |
| Team 5 | Analytics | Divide-by-zero ROI | 18, 19, 20 |

**What Team 1 does:**
- Reproduce bug using scenario 8 and 26
- Write RCA using `rca-writer` skill
- Write fix spec using `sdd-spec-writer` skill
- Fix `BudgetTracker.checkThreshold()` — `>= 95.0` not `> 95.0`
- Validate scenario 26 (exactly 95.0% boundary)
- Validate scenario 27 (BudgetExhausted fires exactly once)

**What Team 5 does:**
- Reproduce bug using scenario 20
- Write RCA using `rca-writer` skill
- Write fix spec using `sdd-spec-writer` skill
- Fix `LiftCalculator.calculateROI()` — null guard on zero vendorShare
- Validate scenario 18 (zero lift)
- Validate scenario 19 (negative lift)
- Validate scenario 20 (null ROI, no 500)

**Sprint 3 exit criteria:**
- Scenarios 8, 26, 27 pass
- Scenarios 18, 19, 20 pass
- Both fixes deployed and live

---

### Sprint 4 — Frontend (2hrs)

**Team 6 builds Frontend Dashboard**
**Depends on:** Sprints 1, 2, 3 complete (all backend services live)

**What Team 6 builds:**
- MX Dashboard — campaign list, campaign form, publish flow, alert banner
- CX View — customer selector, offer fetch, eligibility result display
- Analytics Dashboard — burn chart, lift chart, campaign performance table
- Real-time campaign status polling (10s interval)

**The 3 demo flows Team 6 makes visible:**

**Flow 1 — Campaign Creation:**
Create campaign → add funding → publish → appears ACTIVE in list

**Flow 2 — Customer Offer Fetch:**
Select cust-001 (PLATINUM) → check offers → sees wine promo → select cust-002 (GOLD) → no wine promo, SEGMENT_MISMATCH shown

**Flow 3 — Budget Exhaustion:**
camp-001 shows 95.2% burn → status PAUSED → alert banner fires → offers stop serving

**Sprint 4 exit criteria:**
- All 3 demo flows work end-to-end
- All 30 validation scenarios pass across the full system
- Frontend deployed on Vercel
- System is demo-ready

---

## Validation Scenario Coverage Across Sprints

| Scenarios | Sprint | Team | Dependency |
|-----------|--------|------|-----------|
| 2, 3, 11, 12 | Sprint 1 | Team 2 | Team 4 deploys first |
| 7, 17 | Sprint 1 | Team 3 | None |
| 9, 26 | Sprint 1 | Team 1 | None (isolated — pre-built service) |
| 18, 19, 20 | Sprint 1 | Team 5 | None (isolated — pre-built service) |
| 21, 22, 23 | Sprint 1 | Team 4 | None |
| 1, 4, 5, 6 | Sprint 2 | Team 2 | Team 4 CatalogItemExcluded event |
| 8, 27 | Sprint 2 | Teams 1+5 | Full chain: T3 OfferRedeemed → T5 BudgetExhausted → T1 CampaignPaused |
| 10 | Sprint 2 | Team 1 | None |
| 13, 14 | Sprint 2 | Team 2 | Geo scope from CampaignPublished ACL |
| 15, 16 | Sprint 2 | Team 2 | Stack rules from CampaignPublished ACL |
| 24, 25 | Sprint 2 | Team 2 + Team 4 | Tobacco geo + Catalog exclusion override |
| 28, 29 | Sprint 2 | Team 3 | Team 1 Sprint 2 `GET /campaigns/:id/summary` |
| 30 | Sprint 2 | Team 2 | GET /offers endpoint + all active campaigns live |
| All above | Sprint 3 | All | Integration + hardening — regression check |
| All 30 | Sprint 4 | Team 6 | All services live — 3 demo flows visible |

---

## Cross-Team Communication Rules

1. **No Slack for contract changes** — update the contract skill, the consuming team uses it
2. **No direct DB access** — teams only call each other's APIs or consume events
3. **No breaking contract changes without skill update** — schema version must bump
4. **Every spec must reference validation scenarios** — no spec without acceptance criteria
5. **Every ADR must be recorded before merge** — no exceptions

---

## What Success Looks Like

End of Sprint 4:
- 6 services deployed and live
- All 30 validation scenarios passing
- 3 demo flows running against real test data
- Every team has written specs, ADRs, and validated against real data
- Claude acted as the multiplier — not the author

> "5 teams. 4 sprints. 8 hours total. A system that Oracle charges $500K/year for."
