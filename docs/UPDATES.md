# Update & Upgrade Strategy

This document covers how updates work for both the Claw Cloud platform and user-managed OpenClaw instances.

---

## Table of Contents

1. [Claw Cloud Platform Updates](#claw-cloud-platform-updates)
2. [OpenClaw Instance Updates](#openclaw-instance-updates)
3. [Update Safety](#update-safety)
4. [API Reference](#api-reference)

---

## Claw Cloud Platform Updates

### How It Works

Claw Cloud follows a standard CI/CD workflow via Vercel:

1. **Development** — Changes pushed to feature branches
2. **Review** — Preview deployments for PR review
3. **Merge** — Approved changes merged to `main`
4. **Deploy** — Vercel auto-deploys to production

### Key Properties

| Property | Value |
|----------|-------|
| Deploy trigger | Push to `main` branch |
| Downtime | Zero (Vercel handles blue-green) |
| User action | None required |
| Rollback | One-click in Vercel dashboard |

### What Gets Updated

- Dashboard UI and chat interface
- API endpoints and proxying logic
- Provisioner improvements
- Documentation and onboarding flows

### What Stays Stable

- Database schema (migrations only, backward-compatible)
- Instance connection protocols
- Cloud provider integrations (versioned)

---

## OpenClaw Instance Updates

User-managed OpenClaw instances run on their own infrastructure. Updates require coordination between Claw Cloud and the instance.

### Strategy Comparison

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **User-triggered** | Dashboard notification → user clicks → API call | Full user control, predictable | Users forget, instances drift |
| **Auto-update** | Nightly cron checks + updates | Always current, less drift | Might break things unexpectedly |
| **Managed rollout** | We push in waves, monitor health | Safe, controlled | Complex, requires more infrastructure |

### Recommended: User-Triggered + Nudge (MVP)

For MVP, we implement user-triggered updates with gentle nudging:

- Show banner when update available
- User decides when to update
- No forced updates (their infrastructure, their choice)
- Periodic reminders for critical security updates

---

### Dashboard UX

#### Update Banner

When an update is available, show a prominent but non-intrusive banner:

```
┌────────────────────────────────────────────────────────────────┐
│ 🆕 Update available: v2.3.0 → v2.4.0                           │
│                                                                 │
│    [View changes]  [Update now]  [Remind me later]             │
└────────────────────────────────────────────────────────────────┘
```

#### Version Display

Always show current version in instance details:

```
Instance: my-assistant
Status: ● Running
Version: v2.3.0 (update available)
Region: nyc1
```

#### Changelog Modal

"View changes" opens a modal with:
- Version number and release date
- Summary of changes (from GitHub releases)
- Breaking changes highlighted with ⚠️
- Estimated update time
- Rollback availability

---

### Technical Flow

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   Claw Cloud    │         │   OpenClaw      │         │    GitHub       │
│   Dashboard     │         │   Instance      │         │    Releases     │
└────────┬────────┘         └────────┬────────┘         └────────┬────────┘
         │                           │                           │
         │     1. Report version     │                           │
         │◄──────────────────────────│                           │
         │     (heartbeat/on-demand) │                           │
         │                           │                           │
         │ 2. Fetch latest release   │                           │
         │──────────────────────────────────────────────────────►│
         │                           │                           │
         │◄──────────────────────────────────────────────────────│
         │     v2.4.0 released       │                           │
         │                           │                           │
         │ 3. Compare versions       │                           │
         │ (v2.3.0 < v2.4.0 → show)  │                           │
         │                           │                           │
┌────────┴────────┐                  │                           │
│ Show update     │                  │                           │
│ banner to user  │                  │                           │
└────────┬────────┘                  │                           │
         │                           │                           │
         │ 4. User clicks [Update]   │                           │
         │                           │                           │
         │ 5. Call update endpoint   │                           │
         │──────────────────────────►│                           │
         │                           │                           │
         │                    ┌──────┴──────┐                    │
         │                    │ Run update  │                    │
         │                    │ + restart   │                    │
         │                    └──────┬──────┘                    │
         │                           │                           │
         │◄──────────────────────────│                           │
         │ 6. Report new version     │                           │
         │                           │                           │
┌────────┴────────┐                  │                           │
│ Update complete │                  │                           │
│ Hide banner     │                  │                           │
└─────────────────┘                  │                           │
```

### Step-by-Step

1. **Version reporting** — Instance reports its version via:
   - Regular heartbeat (every 5 minutes)
   - On-demand when dashboard loads
   
2. **Version comparison** — Claw Cloud:
   - Caches latest OpenClaw release from GitHub
   - Compares instance version against latest
   - Considers version pinning preferences

3. **Banner display** — If outdated:
   - Show update banner with version diff
   - Include changelog summary
   - Highlight security updates

4. **User initiates update** — Click "Update now":
   - Confirm dialog with changelog
   - Show estimated downtime (~30 seconds)

5. **Execute update** — API call to instance:
   - Instance runs `openclaw update`
   - Restarts via systemd
   - Brief downtime during restart

6. **Confirm success** — Instance reports back:
   - New version number
   - Health check passes
   - Dashboard updates status

---

### Rollback

Every update preserves the ability to roll back:

#### Automatic Backup

Before updating, OpenClaw:
1. Saves current binary/tarball to `/opt/openclaw/backup/`
2. Records the version in `/opt/openclaw/backup/version`
3. Keeps last 3 versions

#### One-Click Rollback

Dashboard shows rollback option if previous version exists:

```
Current: v2.4.0 (installed 2h ago)
Previous: v2.3.0 [Revert]
```

#### Rollback Flow

```
POST /api/system/rollback
{
  "targetVersion": "2.3.0"
}
```

1. Stop OpenClaw service
2. Restore backed-up version
3. Start service
4. Report version to Claw Cloud
5. Run health check

---

### Version Pinning

Power users can control update behavior:

#### Settings

```
┌─────────────────────────────────────────────────┐
│ Update Preferences                              │
├─────────────────────────────────────────────────┤
│                                                 │
│ ○ Auto-update (always run latest)              │
│ ● Notify only (I'll update manually)    ← MVP  │
│ ○ Pin to version: [v2.3.0 ▾]                   │
│                                                 │
│ □ Notify for security updates only             │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### Behavior by Setting

| Setting | Banner | Auto-update | Notes |
|---------|--------|-------------|-------|
| Auto-update | No | Yes (nightly) | For hands-off users |
| Notify only | Yes | No | MVP default |
| Pin to version | No | No | For specific compatibility needs |
| Security only | Security only | No | Minimize noise, stay safe |

---

## Update Safety

Updates should be safe by default. Multiple layers of protection:

### 1. Staged Rollouts

New versions roll out in waves:

| Stage | % of Instances | Duration | Criteria to Proceed |
|-------|----------------|----------|---------------------|
| Canary | 1% | 24 hours | Zero errors reported |
| Early | 10% | 24 hours | <0.1% error rate |
| General | 50% | 24 hours | <0.01% error rate |
| Complete | 100% | — | — |

*Note: Staged rollouts for Phase 2. MVP does simultaneous release.*

### 2. Health Checks

After every update, the instance runs health checks:

```bash
# Health check sequence
1. Verify service is running (systemctl is-active openclaw)
2. Verify API responds (curl -f http://localhost:8080/health)
3. Verify agent can process (simple test prompt)
4. Report status to Claw Cloud
```

#### Health Check Failures

If health check fails after update:

1. **Automatic rollback** — Restore previous version
2. **Alert user** — "Update failed, reverted to v2.3.0"
3. **Report to us** — Anonymous failure telemetry (opt-in)

### 3. Changelog Transparency

Before updating, users always see:

- **What's new** — Features and improvements
- **What's fixed** — Bug fixes
- **What's breaking** — ⚠️ Breaking changes highlighted
- **Migration steps** — If manual action needed

### 4. Update Window (Future)

For auto-update users, respect preferred update windows:

```
Preferred update window: [2:00 AM ▾] - [4:00 AM ▾] [UTC ▾]
```

---

## API Reference

### Report Version

Instances report their version to Claw Cloud:

```http
POST /api/instances/{instanceId}/version
Authorization: Bearer <instance-token>
Content-Type: application/json

{
  "version": "2.3.0",
  "gitCommit": "abc1234",
  "buildDate": "2026-02-08T12:00:00Z"
}
```

**Response:**

```json
{
  "current": "2.3.0",
  "latest": "2.4.0",
  "updateAvailable": true,
  "releaseNotes": "https://github.com/openclaw/openclaw/releases/tag/v2.4.0",
  "isSecurityUpdate": false
}
```

### Trigger Update

Dashboard calls instance's update endpoint:

```http
POST /api/system/update
Authorization: Bearer <instance-token>
Content-Type: application/json

{
  "targetVersion": "2.4.0"  // optional, defaults to latest
}
```

**What it triggers on the instance:**

```bash
#!/bin/bash
# Executed by OpenClaw's update handler

# 1. Backup current version
cp /opt/openclaw/openclaw /opt/openclaw/backup/openclaw-$(cat /opt/openclaw/VERSION)

# 2. Run update
openclaw update

# 3. Restart service
systemctl restart openclaw

# 4. Health check
sleep 5
curl -f http://localhost:8080/health || (
  # Rollback on failure
  cp /opt/openclaw/backup/openclaw-* /opt/openclaw/openclaw
  systemctl restart openclaw
  exit 1
)

# 5. Report new version
curl -X POST https://cloud.openclaw.io/api/instances/$INSTANCE_ID/version \
  -H "Authorization: Bearer $INSTANCE_TOKEN" \
  -d '{"version": "'$(openclaw --version)'"}'
```

**Response:**

```json
{
  "status": "updating",
  "previousVersion": "2.3.0",
  "targetVersion": "2.4.0",
  "estimatedSeconds": 30
}
```

### Trigger Rollback

```http
POST /api/system/rollback
Authorization: Bearer <instance-token>
Content-Type: application/json

{
  "targetVersion": "2.3.0"
}
```

**Response:**

```json
{
  "status": "rolling_back",
  "currentVersion": "2.4.0",
  "targetVersion": "2.3.0"
}
```

### Get Update Status

```http
GET /api/system/update/status
Authorization: Bearer <instance-token>
```

**Response:**

```json
{
  "status": "idle",  // idle | updating | rolling_back | failed
  "currentVersion": "2.4.0",
  "lastUpdateAt": "2026-02-08T13:45:00Z",
  "availableRollback": "2.3.0",
  "autoUpdate": false,
  "pinnedVersion": null
}
```

---

## Implementation Checklist

### MVP (Phase 1)

- [ ] Version reporting endpoint in Claw Cloud API
- [ ] GitHub release version fetching + caching
- [ ] Update banner component in dashboard
- [ ] Update trigger endpoint (calls instance)
- [ ] `/api/system/update` endpoint in OpenClaw
- [ ] Basic rollback support
- [ ] Changelog display modal

### Phase 2

- [ ] Version pinning UI and persistence
- [ ] Auto-update option with nightly cron
- [ ] Staged rollout infrastructure
- [ ] Update window preferences
- [ ] Security update flagging

### Phase 3

- [ ] Anonymous telemetry for update success/failure
- [ ] Update analytics dashboard (for us)
- [ ] Bulk update for team accounts
- [ ] Update scheduling ("update tomorrow at 2am")

---

*Last updated: 2026-02-08*
