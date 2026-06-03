# PromotionOS — Pseudo Org Setup

> Minimum setup required before the session.
> Goal: every team opens their repo and gists and starts writing specs immediately.
> Documentation: GitHub Gists only. No Notion.

---

## GitHub Org

**Org name:** `promotionos`

### Repositories

| Repo | Service | Language |
|------|---------|----------|
| `campaign-service` | Campaign Bounded Context | Java / Spring Boot |
| `eligibility-service` | Eligibility Bounded Context | Java / Spring Boot |
| `redemption-service` | Redemption Bounded Context | Java / Spring Boot |
| `catalog-customer-service` | Catalog & Customer Bounded Context | Go / Gin |
| `analytics-service` | Analytics Bounded Context | Go / Gin |
| `frontend-dashboard` | Frontend | React.js |
| `shared-contracts` | Event schemas + API contracts | JSON Schema |
| `test-data` | All seed datasets | JSON |
| `skills` | All 16 skills | Markdown |

### Branch Protection
- `main` is protected on all repos
- PRs required — no direct push to main
- GitHub Actions must pass before merge

### Teams

| GitHub Team | Repos |
|-------------|-------|
| `team-campaign` | `campaign-service`, `shared-contracts` |
| `team-eligibility` | `eligibility-service`, `shared-contracts` |
| `team-redemption` | `redemption-service`, `shared-contracts` |
| `team-catalog` | `catalog-customer-service`, `shared-contracts` |
| `team-analytics` | `analytics-service`, `shared-contracts` |
| `team-frontend` | `frontend-dashboard` |

---

## GitHub Gists

All documentation lives as GitHub Gists. No Notion. No Confluence. No wiki.

### Gist Publishing Order

Publish in this exact order — later gists reference earlier ones.

**Group 1 — Foundation (public gists)**

| File | Gist title |
|------|-----------|
| `gists/foundation/sdl.md` | PromotionOS — SDL |
| `gists/foundation/ubiquitous-language.md` | PromotionOS — Ubiquitous Language |
| `architecture.md` | PromotionOS — Architecture |
| `test-data.md` | PromotionOS — Test Data |

**Group 2 — Contracts (public gists)**

| File | Gist title |
|------|-----------|
| `gists/contracts/contract-campaign-eligibility.md` | PromotionOS — Contract: Campaign → Eligibility |
| `gists/contracts/contract-catalog-eligibility.md` | PromotionOS — Contract: Catalog → Eligibility |
| `gists/contracts/contract-eligibility-redemption.md` | PromotionOS — Contract: Eligibility → Redemption |
| `gists/contracts/contract-redemption-analytics.md` | PromotionOS — Contract: Redemption → Analytics |
| `gists/contracts/contract-analytics-campaign.md` | PromotionOS — Contract: Analytics → Campaign |

**Group 3 — Skills (public gists)**

| File | Gist title |
|------|-----------|
| `gists/skills/generic/skill-sdd-spec-writer.md` | PromotionOS — Skill: SDD Spec Writer |
| `gists/skills/generic/skill-adr-recorder.md` | PromotionOS — Skill: ADR Recorder |
| `gists/skills/generic/skill-rca-writer.md` | PromotionOS — Skill: RCA Writer |
| `gists/skills/generic/skill-ddd-guide.md` | PromotionOS — Skill: DDD Guide |
| `gists/skills/generic/skill-pr-reviewer.md` | PromotionOS — Skill: PR Reviewer |
| `gists/skills/codebase/skill-campaign-service-guide.md` | PromotionOS — Skill: Campaign Service Guide |
| `gists/skills/codebase/skill-eligibility-service-guide.md` | PromotionOS — Skill: Eligibility Service Guide |
| `gists/skills/codebase/skill-redemption-service-guide.md` | PromotionOS — Skill: Redemption Service Guide |
| `gists/skills/codebase/skill-catalog-customer-guide.md` | PromotionOS — Skill: Catalog & Customer Guide |
| `gists/skills/codebase/skill-analytics-service-guide.md` | PromotionOS — Skill: Analytics Service Guide |
| `gists/skills/codebase/skill-frontend-guide.md` | PromotionOS — Skill: Frontend Guide |
| `gists/skills/contracts/skill-contract-campaign-eligibility.md` | PromotionOS — Contract Skill: Campaign → Eligibility |
| `gists/skills/contracts/skill-contract-catalog-eligibility.md` | PromotionOS — Contract Skill: Catalog → Eligibility |
| `gists/skills/contracts/skill-contract-eligibility-redemption.md` | PromotionOS — Contract Skill: Eligibility → Redemption |
| `gists/skills/contracts/skill-contract-redemption-analytics.md` | PromotionOS — Contract Skill: Redemption → Analytics |
| `gists/skills/contracts/skill-contract-analytics-campaign.md` | PromotionOS — Contract Skill: Analytics → Campaign |

**Group 4 — Session Materials (public gists)**

| File | Gist title |
|------|-----------|
| `gists/session/session-guide.md` | PromotionOS — Session Guide |
| `gists/session/demo-script.md` | PromotionOS — Demo Script |

**Group 5 — Sprint Requirements (secret gists — hand out at sprint start)**

| File | Gist title | Visibility |
|------|-----------|-----------|
| `gists/sprints/team1-sprint1.md` | Team 1 — Campaign — Sprint 1 | Secret |
| `gists/sprints/team1-sprint2.md` | Team 1 — Campaign — Sprint 2 | Secret |
| `gists/sprints/team1-sprint3.md` | Team 1 — Campaign — Sprint 3 | Secret |
| `gists/sprints/team1-sprint4.md` | Team 1 — Campaign — Sprint 4 | Secret |
| `gists/sprints/team2-sprint1.md` | Team 2 — Eligibility — Sprint 1 | Secret |
| `gists/sprints/team2-sprint2.md` | Team 2 — Eligibility — Sprint 2 | Secret |
| `gists/sprints/team2-sprint3.md` | Team 2 — Eligibility — Sprint 3 | Secret |
| `gists/sprints/team2-sprint4.md` | Team 2 — Eligibility — Sprint 4 | Secret |
| `gists/sprints/team3-sprint1.md` | Team 3 — Redemption — Sprint 1 | Secret |
| `gists/sprints/team3-sprint2.md` | Team 3 — Redemption — Sprint 2 | Secret |
| `gists/sprints/team3-sprint3.md` | Team 3 — Redemption — Sprint 3 | Secret |
| `gists/sprints/team3-sprint4.md` | Team 3 — Redemption — Sprint 4 | Secret |
| `gists/sprints/team4-sprint1.md` | Team 4 — Catalog — Sprint 1 | Secret |
| `gists/sprints/team4-sprint2.md` | Team 4 — Catalog — Sprint 2 | Secret |
| `gists/sprints/team4-sprint3.md` | Team 4 — Catalog — Sprint 3 | Secret |
| `gists/sprints/team4-sprint4.md` | Team 4 — Catalog — Sprint 4 | Secret |
| `gists/sprints/team5-sprint1.md` | Team 5 — Analytics — Sprint 1 | Secret |
| `gists/sprints/team5-sprint2.md` | Team 5 — Analytics — Sprint 2 | Secret |
| `gists/sprints/team5-sprint3.md` | Team 5 — Analytics — Sprint 3 | Secret |
| `gists/sprints/team5-sprint4.md` | Team 5 — Analytics — Sprint 4 | Secret |
| `gists/sprints/team6-sprint1.md` | Team 6 — Frontend — Sprint 1 | Secret |
| `gists/sprints/team6-sprint2.md` | Team 6 — Frontend — Sprint 2 | Secret |
| `gists/sprints/team6-sprint3.md` | Team 6 — Frontend — Sprint 3 | Secret |
| `gists/sprints/team6-sprint4.md` | Team 6 — Frontend — Sprint 4 | Secret |

### Each Repo README

Every service repo README must link to the relevant gists:

```markdown
# [Service Name]

## Documentation
- [SDL](gist-url) — full project lifecycle
- [Ubiquitous Language](gist-url) — domain terms
- [Architecture](gist-url) — system design
- [Your Contract](gist-url) — what this service publishes/consumes
- [Your Codebase Skill](gist-url) — how to navigate this skeleton

## Sprint Requirements
Hand out by facilitator at sprint start — secret gist links.

## Test Data
[test-data gist url]
```

---

## GitHub Actions — Per Repo

Every repo gets two workflows:

### `test.yml` — runs on every PR
```yaml
name: Test
on:
  pull_request:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: ./scripts/test.sh
```

### `deploy.yml` — runs on merge to main
```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Railway
        run: ./scripts/deploy.sh
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

---

## Railway Setup

One Railway project: `promotionos`

### Services

| Service | Port | Env Vars |
|---------|------|----------|
| campaign-service | 8081 | DB_URL, REDIS_URL, TENANT_ID |
| eligibility-service | 8082 | DB_URL, REDIS_URL, TENANT_ID |
| redemption-service | 8083 | DB_URL, REDIS_URL, TENANT_ID |
| catalog-customer-service | 8084 | DB_URL, REDIS_URL, TENANT_ID |
| analytics-service | 8085 | DB_URL, REDIS_URL, TENANT_ID |
| event-store | 8086 | DB_URL, REDIS_URL |
| redis | 6379 | — |

### Databases (PostgreSQL — one per service)

| Database | Used by |
|----------|---------|
| campaign-db | campaign-service |
| eligibility-db | eligibility-service |
| redemption-db | redemption-service |
| catalog-db | catalog-customer-service |
| analytics-db | analytics-service |
| eventstore-db | event-store |

### Setup Steps
1. Create Railway project `promotionos`
2. Add Redis service
3. Add 6 PostgreSQL databases
4. Add 6 backend services — connect each to its DB and Redis
5. Set env vars per service
6. Connect each Railway service to its GitHub repo (auto-deploy on merge to main)

---

## Vercel Setup

**Project name:** `promotionos-frontend`

### Setup Steps
1. Connect `frontend-dashboard` GitHub repo to Vercel
2. Set env vars:
   - `REACT_APP_CAMPAIGN_API_URL` → Railway campaign-service URL
   - `REACT_APP_ELIGIBILITY_API_URL` → Railway eligibility-service URL
   - `REACT_APP_ANALYTICS_API_URL` → Railway analytics-service URL
3. Auto-deploy on merge to main

---

## Shared Contracts Repo Structure

```
shared-contracts/
├── events/
│   ├── CampaignPublished.json
│   ├── CampaignPaused.json
│   ├── BudgetExhausted.json
│   ├── CatalogItemExcluded.json
│   ├── SegmentUpdated.json
│   ├── OfferRedeemed.json
│   └── ClaimSubmitted.json
├── api/
│   ├── campaign-service-api.json
│   ├── eligibility-service-api.json
│   ├── redemption-service-api.json
│   ├── catalog-customer-api.json
│   └── analytics-service-api.json
└── README.md
```

---

## Pre-Session Checklist

### GitHub
- [ ] Org `promotionos` created
- [ ] All 9 repos created
- [ ] Branch protection on all repos
- [ ] GitHub teams created and assigned
- [ ] GitHub Actions workflows added to all repos
- [ ] All repo READMEs link to correct gists

### Gists
- [ ] All 27 public gists published (Groups 1-4)
- [ ] All 24 secret sprint gists published (Group 5)
- [ ] All gist URLs collected and added to repo READMEs
- [ ] Skills gists tested with Claude before session

### Railway
- [ ] Project created
- [ ] Redis provisioned
- [ ] 6 PostgreSQL databases provisioned
- [ ] 6 backend services connected to repos
- [ ] Env vars set on all services
- [ ] All services deploy successfully (returning 501s)

### Vercel
- [ ] Project connected to frontend repo
- [ ] Env vars set
- [ ] Frontend deploys successfully

### Shared Contracts
- [ ] All event schemas written and in shared-contracts repo
- [ ] All API contracts written and in shared-contracts repo

### Skills
- [ ] All 16 skills in skills repo
- [ ] All 16 skills also published as public gists
- [ ] Skills tested with Claude — Claude can read and apply them

### Data
- [ ] All DB schemas migrated
- [ ] Test data seeded in all DBs
- [ ] redeem-008 seeded with SESSION_START_TIME set to session date/time

### Sprint Requirements
- [ ] All 24 secret sprint gists created
- [ ] Sprint 1 gist URLs ready to share at session start
- [ ] Sprint 2-4 gist URLs held back until their sprint begins
