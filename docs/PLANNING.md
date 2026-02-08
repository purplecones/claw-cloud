# Claw Cloud — Detailed Planning

This document expands on the README with deeper technical and business planning.

---

## Table of Contents

1. [Product Definition](#product-definition)
2. [User Personas](#user-personas)
3. [Technical Architecture](#technical-architecture)
4. [Cloud Provider Integration](#cloud-provider-integration)
5. [Security Model](#security-model)
6. [Business Model](#business-model)
7. [Go-to-Market](#go-to-market)
8. [Risk Analysis](#risk-analysis)
9. [Iteration Log](#iteration-log)

---

## Product Definition

### One-Liner
"Deploy your own AI assistant in 2 minutes. Your cloud, your data, your control."

### Problem Statement
- Self-hosting OpenClaw requires VPS setup, SSH, command line skills
- Most people want AI assistants but don't want vendor lock-in or data concerns
- No middle ground between "use ChatGPT" and "run your own server"

### Solution
A web app that:
1. Connects to user's cloud account (BYOC)
2. Provisions a VPS with OpenClaw pre-installed
3. Provides a chat UI to interact with their instance
4. Handles updates, monitoring, and management

### Success Metrics
- Time to first message: < 5 minutes from signup
- Provision success rate: > 95%
- User retention (30-day): > 40%

---

## User Personas

### 1. Privacy-Conscious Professional
- **Who:** Lawyer, doctor, consultant handling sensitive data
- **Need:** AI assistant that doesn't send data to third parties
- **Blocker:** Doesn't know how to self-host
- **Value prop:** "Your data never leaves your server"

### 2. Developer Who Wants Customization
- **Who:** Software engineer, tinkerer
- **Need:** AI they can extend, automate, integrate
- **Blocker:** Time to set up and maintain
- **Value prop:** "One-click deploy, full SSH access"

### 3. Small Business Owner
- **Who:** Runs a local business, not technical
- **Need:** AI to help with operations, scheduling, email
- **Blocker:** Overwhelmed by options, scared of complexity
- **Value prop:** "Works like ChatGPT, but it's yours"

### 4. Enterprise IT Admin
- **Who:** Manages infra for a company
- **Need:** Deploy AI assistants for teams, control data residency
- **Blocker:** Compliance, security review, internal approvals
- **Value prop:** "BYOC + audit logs + your cloud = compliance-friendly"

---

## Technical Architecture

### System Components

```
┌────────────────────────────────────────────────────────────────┐
│                        Claw Cloud Platform                      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   Frontend   │  │     API      │  │   Provisioner         │  │
│  │   (Next.js)  │◄─┤   (Hono)     │◄─┤   (Pulumi Engine)     │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
│         │                 │                      │               │
│         │                 ▼                      ▼               │
│         │          ┌──────────────┐     ┌───────────────────┐   │
│         │          │   Database   │     │   Job Queue       │   │
│         │          │  (Postgres)  │     │   (BullMQ/Redis)  │   │
│         │          └──────────────┘     └───────────────────┘   │
│         │                                                        │
└─────────┼────────────────────────────────────────────────────────┘
          │
          │ WebSocket / SSE
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    User's Cloud Account                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                         VPS                                 │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │                    OpenClaw                          │   │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │ │
│  │  │  │   Gateway   │  │    Agent    │  │   Memory    │  │   │ │
│  │  │  │   (HTTP)    │  │   (Claude)  │  │   (Files)   │  │   │ │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **User signs up** → creates account in our Postgres
2. **User adds cloud credentials** → encrypted, stored securely
3. **User clicks "Deploy"** → job queued for provisioner
4. **Provisioner runs Pulumi** → creates VPS in user's cloud
5. **Bootstrap script runs** → installs OpenClaw, configures API access
6. **Instance registers** → calls back to our API with connection details
7. **User chats** → frontend proxies to their OpenClaw gateway

### Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| IaC tool | Pulumi (TypeScript) | Type-safe, multi-cloud, programmable |
| Frontend | Next.js 14 (App Router) | SSR, great DX, Vercel deploy |
| API | Hono | Lightweight, fast, edge-ready |
| Database | Supabase (Postgres) | Quick setup, good free tier |
| Auth | Clerk | Best-in-class UX, quick integration |
| Queue | BullMQ + Redis | Reliable job processing |
| Hosting | Vercel + Railway | Serverless frontend, managed backend |

---

## Cloud Provider Integration

### Priority Order

1. **DigitalOcean** — Simplest API, good for MVP
2. **AWS** — Largest market share, enterprise requirement
3. **Hetzner** — Cheapest, popular in EU
4. **Vultr** — Good alternative to DO
5. **Azure** — Enterprise, Microsoft shops
6. **GCP** — Enterprise, Google shops

### Credential Handling

#### DigitalOcean
- Personal access token with `create` and `delete` droplet scopes
- Simple, single token

#### AWS
- Option A: IAM user with scoped policy (access key + secret)
- Option B: IAM role with external ID for assume-role (more secure)
- Required permissions: EC2 (create/terminate), VPC (if needed), security groups

#### Azure
- Service principal with Contributor role on a resource group
- Client ID + Client Secret + Tenant ID + Subscription ID

#### GCP
- Service account JSON key
- Required roles: Compute Instance Admin, Service Account User

### Credential Security

1. **Encryption at rest** — AES-256 encryption before storing
2. **Never log credentials** — Scrub from all logs
3. **Minimal retention** — Option to delete after provisioning
4. **Scoped permissions** — Document minimum required permissions
5. **User education** — Guide to creating limited credentials

---

## Security Model

### Threat Model

| Threat | Mitigation |
|--------|------------|
| Credential theft | Encryption, minimal retention, assume-role where possible |
| Man-in-the-middle | HTTPS everywhere, certificate pinning |
| Unauthorized access | Auth (Clerk), rate limiting, audit logs |
| Instance compromise | User's responsibility (their cloud), we provide hardening guide |
| Data exfiltration | We never access user's OpenClaw — only proxy API calls |

### What We See vs. Don't See

**We see:**
- User account info (email, plan)
- Cloud provider type and region
- Instance metadata (IP, status, size)
- API call logs (not content)

**We don't see:**
- Chat content (proxied, not stored)
- OpenClaw memory/files
- Raw cloud credentials (encrypted)
- User's other cloud resources

---

## Business Model

### Pricing Options

#### Option A: Subscription Tiers
| Tier | Price | Instances | Features |
|------|-------|-----------|----------|
| Free | $0 | 1 | Basic chat UI, community support |
| Pro | $19/mo | 5 | Priority support, advanced features |
| Team | $49/mo | 20 | Team management, SSO, audit logs |
| Enterprise | Custom | Unlimited | SLA, dedicated support, custom integrations |

#### Option B: Usage-Based
- $0.10 per provisioning
- $0.01 per API call proxied
- Free tier: 100 provisions, 10k API calls/month

#### Option C: Freemium + Managed Option
- BYOC: Free forever (they pay their cloud)
- Managed: $9/mo (we run the VPS, simpler)

### Revenue Projections (Conservative)

| Metric | Month 1 | Month 6 | Month 12 |
|--------|---------|---------|----------|
| Users | 100 | 1,000 | 5,000 |
| Paid % | 5% | 10% | 15% |
| ARPU | $15 | $18 | $20 |
| MRR | $75 | $1,800 | $15,000 |

### Costs

- **Vercel**: Free tier → $20/mo at scale
- **Supabase**: Free tier → $25/mo at scale
- **Railway**: $5/mo (Redis + workers)
- **Domain/SSL**: $20/year
- **Support tooling**: $0 → $50/mo

**Break-even:** ~$100/mo fixed costs at MVP scale

---

## Go-to-Market

### Launch Strategy

1. **Soft launch** — Post on OpenClaw Discord, get 50 beta users
2. **Iterate** — Fix bugs, improve onboarding based on feedback
3. **Content** — Blog posts: "Why self-hosted AI matters", "BYOC explained"
4. **Hacker News** — Show HN post when polished
5. **Product Hunt** — Coordinated launch with assets ready

### Positioning

- **Not competing with OpenAI/Anthropic** — we're the deployment layer
- **Not competing with OpenClaw** — we're the easy mode wrapper
- **Unique angle:** "The Vercel of AI assistants"

### Early Channels

1. OpenClaw community (natural fit)
2. r/selfhosted (love this stuff)
3. Hacker News (privacy + self-hosting crowd)
4. Twitter/X (indie hackers, AI enthusiasts)
5. Dev.to / Hashnode (technical content)

---

## Risk Analysis

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| OpenClaw changes break us | Medium | High | Pin versions, contribute upstream, maintain compatibility |
| Cloud provider API changes | Low | Medium | Pulumi abstracts some, monitor changelogs |
| Security breach | Low | Critical | Security audit, bug bounty, minimal data retention |
| Low adoption | Medium | High | Validate demand before building, iterate on positioning |
| Pricing wrong | Medium | Medium | Start free, learn willingness to pay |

---

## Product Roadmap

### Phase 1: Foundation (MVP)
- [x] Planning and architecture docs
- [ ] DigitalOcean provisioner
- [ ] Web dashboard with auth
- [ ] Chat proxy to user's OpenClaw
- [ ] Instance management (create/delete/status)

### Phase 2: Multi-Cloud
- [ ] AWS integration
- [ ] Hetzner integration
- [ ] Vultr integration
- [ ] Cloud credential management UI

### Phase 3: Mini Apps Platform
- [ ] App creation via chat
- [ ] iFrame sandbox runtime
- [ ] Dev mode (use/edit toggle)
- [ ] App registry and dashboard tiles
- [ ] Permission system
- [ ] JSON/SQLite data storage

### Phase 4: Marketplace & Teams
- [ ] App marketplace
- [ ] Publish and review flow
- [ ] Revenue sharing (70/30)
- [ ] Team workspaces
- [ ] SSO for enterprise

### Phase 5: Scale & Polish
- [ ] Edge function runtime option
- [ ] Advanced monitoring/alerts
- [ ] Custom domains
- [ ] White-label option

---

## Iteration Log

### 2026-02-08 13:57 — Zero Access Security Policy
- **Major security policy update:** Removed all management SSH key references
- Updated `docs/SECURITY.md`:
  - Added comprehensive "Zero Access by Default" section
  - Documents: no SSH keys, no backdoors, no remote access by default
  - Added temporary support access flow (opt-in only with 24h expiry)
  - Emphasizes user control and technical enforcement
- Updated `docs/DIGITALOCEAN.md`:
  - Removed management SSH key generation code
  - Removed mgmt key injection from provisioning sequence diagram
  - Updated Pulumi implementation to not install any SSH keys by default
  - Added opt-in temporary support access documentation
  - Updated security hardening checklist
- **Philosophy:** "Your data is truly yours — not just a promise, but technically enforced"
- Support flow: User explicitly grants access → 24h expiry → full audit log → immediate revoke

### 2026-02-08 13:48 — Update & Upgrade Strategy
- Created `docs/UPDATES.md` with comprehensive update strategy
- **Claw Cloud updates:** Standard Vercel CI/CD, zero downtime
- **OpenClaw instance updates:** User-triggered + nudge (MVP)
  - Dashboard banner: "Update available: v2.3.0 → v2.4.0"
  - Buttons: View changes, Update now, Remind me later
- **Technical flow documented:**
  1. Instance reports version (heartbeat/on-demand)
  2. Claw Cloud compares against latest GitHub release
  3. Shows banner if outdated
  4. User clicks → API call to instance update endpoint
  5. Instance runs `openclaw update && systemctl restart openclaw`
  6. Reports new version back
- **Rollback:** Keep previous version, one-click revert in dashboard
- **Version pinning:** Power users can lock to specific version
- **Update safety:**
  - Staged rollouts (1% → 10% → 100%) for Phase 2
  - Health check after every update
  - Auto-rollback if health check fails
  - Changelog shown before update
- API specs for: version reporting, update trigger, rollback, status
- Implementation checklist for MVP through Phase 3

### 2026-02-08 13:33 — Security & Mini Apps Architecture
- Created `docs/SECURITY.md` with comprehensive security model:
  - Chat content proxying (no storage)
  - OpenClaw memory/files isolation
  - Cloud credential encryption
  - Scoped IAM permissions
  - Verification and trust mechanisms
- Created `docs/MINI_APPS.md` with full architecture:
  - App structure and manifest format
  - Creation flow via natural language
  - Runtime options (iFrame sandbox recommended)
  - Dev mode with split view and hot reload
  - TypeScript data model
  - Marketplace vision with 70/30 revenue split
  - Security layers and permission system
- Added Product Roadmap with 5 phases
- Mini Apps planned for Phase 3

### 2026-02-08 12:29 — DigitalOcean Implementation Deep Dive
- Created `docs/DIGITALOCEAN.md` with comprehensive implementation details
- Documented all required API endpoints and their usage
- Designed complete droplet creation flow with sequence diagram
- Wrote detailed bootstrap script outline (cloud-init based):
  - System hardening (UFW, fail2ban)
  - OpenClaw installation and systemd service
  - Callback mechanism to Claw Cloud API
- Defined DigitalOcean Cloud Firewall rules (defense in depth)
- Added cost estimates for all relevant droplet sizes
- Included Pulumi implementation example with automation API
- Error handling patterns and retry strategies
- **Next iteration focus:** API client implementation or AWS integration

### 2026-02-08 06:00 — Initial Planning Session
- Mirza proposed the concept during late-night brainstorm
- Key insight: BYOC model means zero infrastructure cost for us
- Created initial README and this planning doc
- Set up 2-hour iteration cron job
- **Next iteration focus:** Deep dive on DigitalOcean provisioning flow

---

*This document is updated every 2 hours by Jerry's planning job.*
