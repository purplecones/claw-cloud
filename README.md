# Claw Cloud

**One-click deploy your own AI assistant. Bring your own cloud.**

A platform that lets users spin up OpenClaw instances on their own cloud infrastructure (AWS, Azure, GCP, DigitalOcean, etc.) and manage them through a unified UI.

## Status

🚧 **Planning Phase** — This repo tracks the product plan and architecture.

---

## Vision

Make self-hosted AI assistants accessible to everyone — not just people who know how to SSH into a VPS. Users get:

- **True ownership** — runs in their cloud account, their data stays theirs
- **One-click setup** — no terminal required
- **Unified management** — chat UI, config, monitoring all in one place

## Core Features (MVP)

1. **Cloud Provider Integration**
   - Connect AWS, Azure, GCP, DigitalOcean, Vultr, Hetzner
   - BYOC model — users bring their own cloud credentials
   - Scoped permissions via IAM roles / service principals

2. **One-Click Provisioning**
   - Select cloud provider + region + instance size
   - Automated VPS creation + OpenClaw installation
   - ~2 minute deploy time target

3. **Chat Interface**
   - Web-based chat UI that proxies to user's OpenClaw instance
   - Real-time streaming responses
   - Mobile-responsive

4. **Instance Management**
   - Start/stop/restart instances
   - View logs and status
   - Update OpenClaw version
   - Basic cost monitoring

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Claw Cloud App                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Web UI    │  │   API       │  │  Provisioning       │  │
│  │  (Next.js)  │  │  (Node.js)  │  │  (Terraform/Pulumi) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │         User's Cloud Account            │
        │  ┌─────────────────────────────────┐    │
        │  │     VPS with OpenClaw           │    │
        │  │  ┌─────────────────────────┐    │    │
        │  │  │   OpenClaw Gateway      │    │    │
        │  │  │   (exposed via HTTPS)   │    │    │
        │  │  └─────────────────────────┘    │    │
        │  └─────────────────────────────────┘    │
        └─────────────────────────────────────────┘
```

## Tech Stack (Proposed)

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Frontend | Next.js 14 + Tailwind | Fast, SSR, good DX |
| API | Node.js + tRPC or Hono | Type-safe, lightweight |
| Auth | Clerk or NextAuth | Quick to implement |
| Database | PostgreSQL (Supabase or Neon) | User accounts, instance metadata |
| Provisioning | Pulumi (TypeScript) | Multi-cloud, programmatic |
| Secrets | User-provided, encrypted at rest | We never see their cloud costs |

## Open Questions

- [ ] Pricing model: flat subscription, usage-based, or freemium?
- [ ] Do we offer a managed option (we run the VPS) alongside BYOC?
- [ ] How do we handle OpenClaw updates across user instances?
- [ ] Minimum viable cloud providers for launch?
- [ ] Domain/SSL strategy for each instance?

## Competitive Landscape

| Product | Model | Difference |
|---------|-------|------------|
| ChatGPT/Claude | Centralized SaaS | No self-hosting, no customization |
| OpenClaw (direct) | Self-hosted DIY | Requires technical setup |
| **Claw Cloud** | Self-hosted + managed | Easy setup, user owns infra |

## Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Auth + user accounts
- [ ] DigitalOcean integration (simplest API)
- [ ] Basic provisioning flow
- [ ] Minimal chat UI

### Phase 2: Core Features (Weeks 3-4)
- [ ] AWS integration
- [ ] Instance management (start/stop/logs)
- [ ] OpenClaw config editor
- [ ] Billing/usage tracking

### Phase 3: Polish (Weeks 5-6)
- [ ] Azure + GCP integration
- [ ] Mobile-responsive UI
- [ ] Onboarding flow
- [ ] Documentation

### Phase 4: Launch
- [ ] Beta program
- [ ] Pricing page
- [ ] Marketing site

---

## Development

```bash
# Clone
git clone https://github.com/purplecones/claw-cloud.git
cd claw-cloud

# Install (TBD)
npm install

# Run (TBD)
npm run dev
```

## Planning Log

This section tracks iteration notes as the plan evolves.

### 2026-02-08 — Initial Concept
- Mirza proposed: app that spins up VPS with OpenClaw, BYOC model
- Core insight: "OpenClaw as a Service" for non-technical users
- Key differentiator: users own their infrastructure, we're just the orchestration layer

---

## License

TBD

