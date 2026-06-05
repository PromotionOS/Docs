# Session 1 Script — Foundations

> Facilitator guide. Read this before the session. Bold text = say out loud.
> Duration: 2 hours
> Participants: 8 TLs — one per service
> Format: Each TL works solo with Claude. No sub-teams.

---

## Pre-Session (15 min before)

- Confirm all 8 repos are live and returning 200 on `/health`
- Confirm test data is seeded in Railway Postgres (including camp-013 at $75K, vendor-pepsi-001)
- Confirm Redis is running
- Have the secret gist links for Sprint 1 ready — do not share yet
- Have the ubiquitous language gist URL ready to paste in chat
- Note: Campaign Service and Analytics Service are pre-built. Event Store is pre-built infrastructure with no TL.

---

## Opening (0:00 — 0:10)

**"Welcome. This is not a Claude tutorial. This is not a demo. This is a real engineering session. By the end of today you will have shipped something that runs in production."**

Pause.

**"Let me tell you what we are building. PromotionOS is a trade promotions management platform. Oracle charges retail companies like Kroger $500,000 a year for this. We are going to build the core of it. Today. Together. In 2 hours."**

**"But here is the thing — we are not just building software. We are building it in a specific way. Spec Driven Development. Every feature starts with a spec. No code without a spec. No spec without requirements. Claude helps you implement. You own the decisions."**

**"There are 8 of you. Each of you owns one bounded context. One service. You are not building the whole thing — you are building your piece so well that when all pieces connect, the system works."**

**"You are working alone. Claude is your team. You write the spec. Claude helps you implement. You own every decision."**

**"Two of the teams — Campaign and Analytics — have pre-built services. Those services have bugs. Your job is to find the bug, understand why it was introduced, write a root cause analysis, fix it, and document it. That is SDD applied to debugging."**

**"There is also an Event Store. It is pre-built infrastructure — no TL owns it. It is the backbone that carries domain events between services. You will use it. You do not maintain it."**

**"The other six teams are building from scratch. You have a skeleton — the structure, the domain model, the interfaces. Your job is to implement the business logic that makes it work."**

---

## The Three Tools (0:10 — 0:20)

**"Before you open your repo — three things you need to understand."**

**"First: the Ubiquitous Language."**

Share gist URL in chat.

**"Every word in this document is a domain term. Campaign. Offer. Funding. Budget. Redemption. Vendor. Notification. These are not just variable names. They are the shared language of this system. If you write 'promotion' instead of 'campaign' in your code — wrong. If you write 'transaction' instead of 'redemption' — wrong. The language is the contract."**

**"Second: Skills."**

**"You have access to skills in Claude Code. They are tools that encode knowledge — about the SDD process, about DDD, about how to write an RCA, about your specific codebase. Before you write a single line of code, you use the sdd-spec-writer skill to write your spec. That is non-negotiable."**

**"Third: The SDD process."**

Write on whiteboard or share screen:

```
Requirements → Spec → ADR → Code → Validate → PR
```

**"This is the loop. Every feature. Every fix. No shortcuts. The spec comes before the code. Always."**

---

## Team Assignments + Sprint 1 Requirements (0:20 — 0:25)

**"Here are your team assignments."**

Read out:
- Team 1 → Campaign Service — bug fix sprint (Java/Spring Boot, pre-built)
- Team 2 → Eligibility Service — build sprint
- Team 3 → Redemption Service — build sprint
- Team 4 → Catalog & Customer Service — build sprint
- Team 5 → Analytics Service — bug fix sprint (pre-built)
- Team 6 → Frontend Dashboard — build sprint
- Team 7 → Vendor Service — build sprint (Java/Spring Boot)
- Team 8 → Notification Service — build sprint (Go/Gin)

**"I am now sharing your Sprint 1 requirements. These are secret gists — one per TL. Read yours fully before touching any code. You have 15 minutes to read and understand. Then write your spec."**

Share secret gist links to each TL privately.

**"Frontend TL — important. Campaign Service is live right now. Your Sprint 1 work wires to the real Campaign Service API. No mock data. Real campaigns, real data, from minute one."**

**"Deployment order matters for this session. Campaign TL and Analytics TL deploy first — they are fixing pre-built services and everything downstream depends on them. Then Catalog TL deploys — the store hierarchy they expose is needed by Notification TL in later sessions. Notification TL deploys after Campaign is live — they need Campaign events to flow. After those four are up, the remaining teams can deploy in parallel."**

**"Vendor TL — Sprint 1 scope is vendor profile CRUD and the funding proposal submit endpoint. You are not implementing approval logic yet. That comes in Session 2."**

**"Notification TL — Sprint 1 scope is email notifications for CampaignPaused and BudgetExhausted events. Subscribe to the Event Store. When Campaign Service publishes CampaignPaused, you deliver an email. That is your first contract."**

---

## Build Phase (0:25 — 1:40)

Teams work. You circulate.

**Interventions — say these when you see these things:**

If a team starts coding without a spec:
**"Stop. Open the sdd-spec-writer skill. Write the spec first. I know it feels slower. It is not."**

If a team uses wrong domain terms:
**"Check the ubiquitous language gist. What is the correct term? Change it in your spec before you implement."**

If Team 2 is blocked waiting for Team 4:
**"Use the contract skill. The catalog-eligibility-contract skill tells you exactly what Team 4's API returns. Build against the contract, not a running service. They will deploy — you keep moving."**

If a team changes a contract without telling anyone:
**"Stop. Update the contract skill first. Bump the schema version. Tell the consuming team. Then continue."**

If Notification TL is unsure how to subscribe to events:
**"The Event Store is pre-built. Your contract skill tells you exactly which topics exist and what the payload shapes are. CampaignPaused is already published by Campaign Service. Subscribe to it and fire the email."**

**At 1:10 — the key teaching moment for Session 1:**

Watch for Campaign Service publishing CampaignPaused for camp-001 (which hits 95.2% burn in test data). When Notification TL's service picks it up and delivers the email:

**"Everyone pause. Look at Team 8's logs."**

Show Notification Service receiving the CampaignPaused event and emitting the email delivery log.

**"That is the first cross-service event flow in this cohort. Team 1 published an event. Team 8 consumed it. Two services. Two TLs. Zero coordination beyond a contract. That is how this system is going to work across all 8 services."**

**At 1:10 — check in:**
**"15 minutes to validation. If you are still implementing — stop new features. Validate what you have against the test data scenarios in your requirements doc. A working partial implementation is better than a broken full one."**

---

## Validation Phase (1:40 — 1:50)

**"Validation time. Run your scenarios against the live service. Check them off. PR open and GitHub Actions green before we wrap."**

Walk around. Check each team's scenario list.

**Teams 1 and 5 (bug fix teams):**
- Scenario 26: camp-012 at 95.0% → CampaignPaused emitted ✅
- Scenario 9: camp-010 publish → 400 NO_FUNDING_SOURCE ✅

**Teams 2, 3, 4 (build teams) — their specific scenarios from sprint docs.**

**Team 7 (Vendor TL):**
- Scenario 32: camp-013 at $75K budget → publish attempt blocked, requires funding approval ✅

**Team 8 (Notification TL):**
- Scenario 35: CampaignPaused for camp-001 → email delivered within 30 seconds ✅

---

## Sprint Close (1:50 — 2:00)

**"30 seconds each. What did you build? Which scenarios pass? One thing Claude helped you do faster than you could have done alone."**

Go around all 8 teams. Keep it tight.

After the round:

**"Look at what just happened. In 2 hours, 8 teams built 8 pieces of a real system. Not a tutorial project. Not a hello world. A real trade promotions platform running on Railway, backed by a pre-built Event Store that is already carrying domain events between services."**

**"Here is what you should take from today: The spec was not overhead. The spec was why it worked. When Claude had a clear spec to work from — domain rules explicit, acceptance criteria mapped to real test data, edge cases documented — it produced correct code. When you skipped the spec, you debugged for 45 minutes."**

**"Next session: your services go deeper. The approval workflow fires for the first time. Two TLs will create an event contract live, in session, without a meeting. See you then."**

---

## Session 1 Checklist

- [ ] All 8 TLs wrote specs before coding
- [ ] All 8 TLs have at least one ADR recorded
- [ ] Campaign TL: scenarios 9, 26 passing
- [ ] Analytics TL: scenarios 18, 19, 20 passing
- [ ] Eligibility TL: scenarios 2, 3, 11, 12 passing
- [ ] Redemption TL: scenarios 7, 17 passing
- [ ] Catalog TL: scenarios 21, 22, 23 passing
- [ ] Frontend TL: scenario 30 passing (campaign list live from Campaign Service)
- [ ] Vendor TL: scenario 32 passing (camp-013 publish blocked — budget >= $50K requires approval)
- [ ] Notification TL: scenario 35 passing (CampaignPaused email delivered for camp-001)
- [ ] All PRs merged and services deployed
- [ ] Deployment order respected: Campaign + Analytics → Catalog → Notification → others
