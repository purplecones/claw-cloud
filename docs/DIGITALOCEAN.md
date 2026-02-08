# DigitalOcean Integration — Implementation Guide

This document details the technical implementation for provisioning OpenClaw instances on DigitalOcean.

---

## Table of Contents

1. [Authentication](#authentication)
2. [API Endpoints](#api-endpoints)
3. [Droplet Creation Flow](#droplet-creation-flow)
4. [Bootstrap Script](#bootstrap-script)
5. [Firewall Configuration](#firewall-configuration)
6. [SSH Key Management](#ssh-key-management)
7. [Cost Estimates](#cost-estimates)
8. [Error Handling](#error-handling)
9. [Pulumi Implementation](#pulumi-implementation)

---

## Authentication

### Token Requirements

DigitalOcean uses Personal Access Tokens (PAT) for API access.

**Required Scopes:**
- `read` — List droplets, regions, sizes, images
- `write` — Create/delete droplets, firewalls, SSH keys

**Token Creation URL:** `https://cloud.digitalocean.com/account/api/tokens`

### Storing Credentials

```typescript
interface DOCredentials {
  token: string;        // Personal Access Token
  createdAt: Date;
  lastUsed: Date;
  label?: string;       // User-friendly name
}
```

**Security:**
- Encrypt with AES-256-GCM before storing in database
- Use a separate encryption key stored in environment variables
- Never log the full token (only last 4 chars for debugging)

---

## API Endpoints

### Base URL
```
https://api.digitalocean.com/v2
```

### Endpoints We Use

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/account` | GET | Validate token, check account status |
| `/regions` | GET | List available regions |
| `/sizes` | GET | List available droplet sizes |
| `/images?type=distribution` | GET | List base images |
| `/droplets` | POST | Create droplet |
| `/droplets/{id}` | GET | Get droplet status |
| `/droplets/{id}` | DELETE | Destroy droplet |
| `/droplets/{id}/actions` | POST | Power on/off, reboot |
| `/firewalls` | POST | Create firewall |
| `/firewalls/{id}/droplets` | POST | Attach firewall to droplet |
| `/account/keys` | GET/POST | Manage SSH keys |

### Rate Limits

- **5,000 requests per hour** per token
- Implement exponential backoff on 429 responses
- Cache region/size data (changes rarely)

---

## Droplet Creation Flow

### Sequence Diagram

```
User                    Claw Cloud API             DigitalOcean API
  │                           │                           │
  │  Click "Deploy"           │                           │
  │ ─────────────────────────>│                           │
  │                           │                           │
  │                           │  POST /account            │
  │                           │ ─────────────────────────>│
  │                           │<───────────── 200 OK ─────│
  │                           │                           │
  │                           │  POST /account/keys       │
  │                           │  (add mgmt SSH key)       │
  │                           │ ─────────────────────────>│
  │                           │<───────────── 201 ────────│
  │                           │                           │
  │                           │  POST /droplets           │
  │                           │  (with user_data script)  │
  │                           │ ─────────────────────────>│
  │                           │<───────────── 202 ────────│
  │                           │                           │
  │  "Provisioning..."        │                           │
  │ <─────────────────────────│                           │
  │                           │                           │
  │                           │  GET /droplets/{id}       │
  │                           │  (poll every 5s)          │
  │                           │ ─────────────────────────>│
  │                           │<───────── status: active ─│
  │                           │                           │
  │                           │  POST /firewalls          │
  │                           │ ─────────────────────────>│
  │                           │<───────────── 202 ────────│
  │                           │                           │
  │                           │  (Wait for callback       │
  │                           │   from bootstrap script)  │
  │                           │                           │
  │  "Ready! Click to chat"   │                           │
  │ <─────────────────────────│                           │
```

### Create Droplet Request

```typescript
interface CreateDropletRequest {
  name: string;              // "openclaw-{userId}-{timestamp}"
  region: string;            // "nyc1", "sfo3", "lon1", etc.
  size: string;              // "s-1vcpu-2gb", "s-2vcpu-4gb"
  image: string;             // "ubuntu-24-04-x64"
  ssh_keys: number[];        // SSH key IDs
  backups: boolean;          // false for MVP
  ipv6: boolean;             // true
  monitoring: boolean;       // true
  user_data: string;         // Bootstrap script (cloud-init)
  tags: string[];            // ["claw-cloud", "user-{userId}"]
}
```

### Response Handling

```typescript
interface DropletResponse {
  droplet: {
    id: number;
    name: string;
    status: 'new' | 'active' | 'off' | 'archive';
    networks: {
      v4: Array<{
        ip_address: string;
        type: 'public' | 'private';
      }>;
      v6: Array<{
        ip_address: string;
        type: 'public';
      }>;
    };
    region: { slug: string };
    size: { slug: string };
    created_at: string;
  };
}
```

### Polling Strategy

```typescript
async function waitForDropletActive(dropletId: number, token: string): Promise<Droplet> {
  const maxAttempts = 60;  // 5 minutes total
  const pollInterval = 5000;  // 5 seconds
  
  for (let i = 0; i < maxAttempts; i++) {
    const droplet = await getDroplet(dropletId, token);
    
    if (droplet.status === 'active' && droplet.networks.v4.length > 0) {
      return droplet;
    }
    
    await sleep(pollInterval);
  }
  
  throw new Error('Droplet failed to become active within timeout');
}
```

---

## Bootstrap Script

The bootstrap script runs via cloud-init when the droplet first boots.

### Script Outline

```bash
#!/bin/bash
set -euo pipefail

# ============================================
# Claw Cloud Bootstrap Script
# ============================================

# Configuration (injected at provision time)
OPENCLAW_VERSION="${OPENCLAW_VERSION:-latest}"
CALLBACK_URL="${CALLBACK_URL}"
INSTANCE_TOKEN="${INSTANCE_TOKEN}"
API_KEYS_ENCRYPTED="${API_KEYS_ENCRYPTED}"

# Logging
exec > >(tee -a /var/log/claw-cloud-bootstrap.log) 2>&1
echo "[$(date -Iseconds)] Starting Claw Cloud bootstrap..."

# ============================================
# 1. System Updates
# ============================================
echo "[$(date -Iseconds)] Updating system packages..."
export DEBIAN_FRONTEND=noninteractive
apt-get update -qq
apt-get upgrade -yqq

# ============================================
# 2. Install Dependencies
# ============================================
echo "[$(date -Iseconds)] Installing dependencies..."
apt-get install -yqq \
    curl \
    git \
    jq \
    unzip \
    fail2ban \
    ufw \
    certbot

# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -yqq nodejs

# ============================================
# 3. Create OpenClaw User
# ============================================
echo "[$(date -Iseconds)] Creating openclaw user..."
useradd -m -s /bin/bash openclaw
usermod -aG sudo openclaw

# ============================================
# 4. Install OpenClaw
# ============================================
echo "[$(date -Iseconds)] Installing OpenClaw..."
su - openclaw -c "
    curl -fsSL https://get.openclaw.ai | bash
    # Or: npm install -g openclaw@${OPENCLAW_VERSION}
"

# ============================================
# 5. Configure OpenClaw
# ============================================
echo "[$(date -Iseconds)] Configuring OpenClaw..."
mkdir -p /home/openclaw/.openclaw
chown openclaw:openclaw /home/openclaw/.openclaw

# Write gateway config
cat > /home/openclaw/.openclaw/gateway.json << 'EOF'
{
  "host": "0.0.0.0",
  "port": 8080,
  "auth": {
    "type": "token",
    "token": "${GATEWAY_TOKEN}"
  },
  "tls": {
    "enabled": false
  }
}
EOF

# Decrypt and write API keys (provided encrypted)
echo "${API_KEYS_ENCRYPTED}" | base64 -d > /home/openclaw/.openclaw/secrets.env.enc
# Decryption handled by openclaw init

chown -R openclaw:openclaw /home/openclaw/.openclaw

# ============================================
# 6. Create Systemd Service
# ============================================
echo "[$(date -Iseconds)] Creating systemd service..."
cat > /etc/systemd/system/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=openclaw
WorkingDirectory=/home/openclaw/.openclaw
ExecStart=/usr/bin/openclaw gateway start
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable openclaw
systemctl start openclaw

# ============================================
# 7. Configure Firewall
# ============================================
echo "[$(date -Iseconds)] Configuring firewall..."
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 8080/tcp  # OpenClaw gateway
ufw allow 443/tcp   # HTTPS (future)
ufw --force enable

# ============================================
# 8. Configure Fail2Ban
# ============================================
echo "[$(date -Iseconds)] Configuring fail2ban..."
systemctl enable fail2ban
systemctl start fail2ban

# ============================================
# 9. Callback to Claw Cloud
# ============================================
echo "[$(date -Iseconds)] Registering with Claw Cloud..."
PUBLIC_IP=$(curl -s http://169.254.169.254/metadata/v1/interfaces/public/0/ipv4/address)

# Wait for OpenClaw to be ready
for i in {1..30}; do
    if curl -s http://localhost:8080/health > /dev/null 2>&1; then
        break
    fi
    sleep 2
done

# Send callback
curl -X POST "${CALLBACK_URL}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${INSTANCE_TOKEN}" \
    -d "{
        \"status\": \"ready\",
        \"ip\": \"${PUBLIC_IP}\",
        \"port\": 8080,
        \"version\": \"$(openclaw --version)\"
    }"

echo "[$(date -Iseconds)] Bootstrap complete!"
```

### Script Injection

The script is passed via the `user_data` field in base64:

```typescript
function buildUserData(config: ProvisionConfig): string {
  const script = bootstrapTemplate
    .replace('${CALLBACK_URL}', config.callbackUrl)
    .replace('${INSTANCE_TOKEN}', config.instanceToken)
    .replace('${OPENCLAW_VERSION}', config.openclawVersion)
    .replace('${GATEWAY_TOKEN}', config.gatewayToken)
    .replace('${API_KEYS_ENCRYPTED}', config.apiKeysEncrypted);
  
  return Buffer.from(script).toString('base64');
}
```

---

## Firewall Configuration

### DigitalOcean Cloud Firewall

In addition to UFW on the droplet, we create a DO cloud firewall for defense in depth.

```typescript
interface CreateFirewallRequest {
  name: string;  // "claw-cloud-{userId}"
  inbound_rules: InboundRule[];
  outbound_rules: OutboundRule[];
  droplet_ids: number[];
  tags: string[];
}

const defaultFirewallRules: CreateFirewallRequest = {
  name: "claw-cloud-firewall",
  inbound_rules: [
    {
      protocol: "tcp",
      ports: "22",
      sources: {
        addresses: ["0.0.0.0/0", "::/0"]  // Or limit to user's IP
      }
    },
    {
      protocol: "tcp",
      ports: "8080",
      sources: {
        addresses: ["0.0.0.0/0", "::/0"]  // OpenClaw gateway
      }
    },
    {
      protocol: "tcp",
      ports: "443",
      sources: {
        addresses: ["0.0.0.0/0", "::/0"]  // HTTPS (future)
      }
    },
    {
      protocol: "icmp",
      sources: {
        addresses: ["0.0.0.0/0", "::/0"]  // Ping for health checks
      }
    }
  ],
  outbound_rules: [
    {
      protocol: "tcp",
      ports: "all",
      destinations: {
        addresses: ["0.0.0.0/0", "::/0"]
      }
    },
    {
      protocol: "udp",
      ports: "all",
      destinations: {
        addresses: ["0.0.0.0/0", "::/0"]
      }
    },
    {
      protocol: "icmp",
      destinations: {
        addresses: ["0.0.0.0/0", "::/0"]
      }
    }
  ],
  droplet_ids: [],
  tags: ["claw-cloud"]
};
```

### Security Hardening Checklist

- [x] Cloud firewall (defense in depth)
- [x] UFW on droplet (local firewall)
- [x] Fail2Ban (brute force protection)
- [x] Non-root user for OpenClaw
- [x] Token-based gateway auth
- [ ] Optional: Restrict SSH to user's IP
- [ ] Optional: Automatic security updates
- [ ] Future: Let's Encrypt SSL via certbot

---

## SSH Key Management

### Our Management Key

We generate an SSH key pair for emergency access and debugging:

```typescript
// Generate once per Claw Cloud instance, store in secrets manager
const CLAW_CLOUD_PUBLIC_KEY = "ssh-ed25519 AAAAC3...";

// Add to DigitalOcean account on first use
async function ensureSSHKey(token: string): Promise<number> {
  const existing = await listSSHKeys(token);
  const found = existing.find(k => k.name === 'claw-cloud-management');
  
  if (found) return found.id;
  
  const created = await createSSHKey(token, {
    name: 'claw-cloud-management',
    public_key: CLAW_CLOUD_PUBLIC_KEY
  });
  
  return created.id;
}
```

### User SSH Keys (Optional)

Allow users to add their own SSH key for direct access:

```typescript
interface UserSSHKey {
  userId: string;
  name: string;
  publicKey: string;  // Validated before storing
  fingerprint: string;
}
```

---

## Cost Estimates

### Droplet Pricing (as of Feb 2026)

| Size Slug | vCPUs | Memory | Storage | Transfer | Price/Month | Price/Hour |
|-----------|-------|--------|---------|----------|-------------|------------|
| `s-1vcpu-1gb` | 1 | 1 GB | 25 GB | 1 TB | $6 | $0.00893 |
| `s-1vcpu-2gb` | 1 | 2 GB | 50 GB | 2 TB | $12 | $0.01786 |
| `s-2vcpu-2gb` | 2 | 2 GB | 60 GB | 3 TB | $18 | $0.02679 |
| `s-2vcpu-4gb` | 2 | 4 GB | 80 GB | 4 TB | $24 | $0.03571 |
| `s-4vcpu-8gb` | 4 | 8 GB | 160 GB | 5 TB | $48 | $0.07143 |

### Recommended Sizes for OpenClaw

| Use Case | Recommended Size | Monthly Cost | Notes |
|----------|------------------|--------------|-------|
| Light (1 user) | `s-1vcpu-2gb` | $12 | Minimum viable |
| Standard (1-3 users) | `s-2vcpu-4gb` | $24 | Good balance |
| Power (3-10 users) | `s-4vcpu-8gb` | $48 | Room to grow |

### Additional Costs

| Resource | Cost | Notes |
|----------|------|-------|
| Backups | +20% of droplet | Optional, recommended |
| Snapshots | $0.06/GB/month | For migration/backup |
| Bandwidth overage | $0.01/GB | After included transfer |
| Reserved IP | $5/month | If droplet is deleted |

### Cost Display in UI

```typescript
interface PricingDisplay {
  dropletMonthly: number;
  dropletHourly: number;
  backupsMonthly?: number;
  estimatedTotal: number;
  notes: string[];
}

function calculatePricing(size: string, options: ProvisionOptions): PricingDisplay {
  const base = DROPLET_PRICES[size];
  const backups = options.backups ? base.monthly * 0.2 : 0;
  
  return {
    dropletMonthly: base.monthly,
    dropletHourly: base.hourly,
    backupsMonthly: options.backups ? backups : undefined,
    estimatedTotal: base.monthly + backups,
    notes: [
      "Billed hourly by DigitalOcean",
      "Stop the droplet to pause billing",
      "Transfer included, overage billed separately"
    ]
  };
}
```

---

## Error Handling

### Common Errors

| Error | Cause | User Message | Recovery |
|-------|-------|--------------|----------|
| `401 Unauthorized` | Invalid token | "Your DigitalOcean token is invalid or expired" | Re-enter token |
| `403 Forbidden` | Insufficient permissions | "Token needs 'write' scope" | Generate new token |
| `422 Region unavailable` | Region at capacity | "Region is currently full. Try another region." | Suggest alternatives |
| `422 Size unavailable` | Size not in region | "This instance size isn't available in the selected region" | Filter available sizes |
| `429 Rate limit` | Too many requests | "Please wait a moment and try again" | Retry with backoff |
| `500+ Server error` | DO infrastructure | "DigitalOcean is having issues. Try again in a few minutes." | Retry later |

### Graceful Degradation

```typescript
async function provisionWithRetry(config: ProvisionConfig): Promise<Droplet> {
  const maxRetries = 3;
  const backoff = [1000, 5000, 15000];
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await createDroplet(config);
    } catch (err) {
      if (!isRetryable(err) || i === maxRetries - 1) {
        throw err;
      }
      await sleep(backoff[i]);
    }
  }
}

function isRetryable(err: Error): boolean {
  const code = (err as any).statusCode;
  return code === 429 || code >= 500;
}
```

---

## Pulumi Implementation

### Stack Structure

```
packages/provisioner/
├── src/
│   ├── index.ts
│   ├── providers/
│   │   ├── digitalocean.ts     # This file
│   │   ├── aws.ts              # Future
│   │   └── types.ts
│   ├── scripts/
│   │   └── bootstrap.sh
│   └── utils/
│       └── crypto.ts
├── Pulumi.yaml
└── package.json
```

### Pulumi Program

```typescript
// packages/provisioner/src/providers/digitalocean.ts

import * as digitalocean from "@pulumi/digitalocean";
import * as pulumi from "@pulumi/pulumi";
import { bootstrapScript } from "../scripts/bootstrap";

export interface DOProvisionConfig {
  userId: string;
  region: string;
  size: string;
  openclawVersion: string;
  callbackUrl: string;
  gatewayToken: string;
  apiKeysEncrypted: string;
}

export async function provisionOpenClaw(config: DOProvisionConfig) {
  const name = `openclaw-${config.userId}-${Date.now()}`;
  
  // SSH Key (or reference existing)
  const sshKey = new digitalocean.SshKey("claw-cloud-key", {
    name: "claw-cloud-management",
    publicKey: process.env.CLAW_CLOUD_SSH_PUBLIC_KEY!,
  });

  // Droplet
  const droplet = new digitalocean.Droplet(name, {
    name,
    region: config.region,
    size: config.size,
    image: "ubuntu-24-04-x64",
    sshKeys: [sshKey.fingerprint],
    monitoring: true,
    ipv6: true,
    userData: bootstrapScript(config),
    tags: ["claw-cloud", `user-${config.userId}`],
  });

  // Firewall
  const firewall = new digitalocean.Firewall(`${name}-fw`, {
    name: `${name}-firewall`,
    dropletIds: [droplet.id.apply(id => parseInt(id))],
    inboundRules: [
      { protocol: "tcp", portRange: "22", sourceAddresses: ["0.0.0.0/0", "::/0"] },
      { protocol: "tcp", portRange: "8080", sourceAddresses: ["0.0.0.0/0", "::/0"] },
      { protocol: "tcp", portRange: "443", sourceAddresses: ["0.0.0.0/0", "::/0"] },
      { protocol: "icmp", sourceAddresses: ["0.0.0.0/0", "::/0"] },
    ],
    outboundRules: [
      { protocol: "tcp", portRange: "1-65535", destinationAddresses: ["0.0.0.0/0", "::/0"] },
      { protocol: "udp", portRange: "1-65535", destinationAddresses: ["0.0.0.0/0", "::/0"] },
      { protocol: "icmp", destinationAddresses: ["0.0.0.0/0", "::/0"] },
    ],
  });

  return {
    dropletId: droplet.id,
    ipv4: droplet.ipv4Address,
    ipv6: droplet.ipv6Address,
    region: droplet.region,
    status: droplet.status,
  };
}

export async function destroyOpenClaw(dropletId: number) {
  // Pulumi destroy or direct API call for immediate destruction
}
```

### Running Pulumi Programmatically

```typescript
import { LocalWorkspace } from "@pulumi/pulumi/automation";

async function runProvision(config: DOProvisionConfig) {
  const stack = await LocalWorkspace.createOrSelectStack({
    stackName: `openclaw-${config.userId}`,
    projectName: "claw-cloud-instances",
    program: async () => provisionOpenClaw(config),
  });

  // Set DigitalOcean token
  await stack.setConfig("digitalocean:token", { 
    value: config.token, 
    secret: true 
  });

  const result = await stack.up({ onOutput: console.log });
  
  return {
    outputs: result.outputs,
    summary: result.summary,
  };
}
```

---

## Next Steps

1. **Implement API client** — Type-safe wrapper for DO API
2. **Build bootstrap script** — Test on manual droplet first
3. **Set up Pulumi workspace** — Shared state backend (S3 or Pulumi Cloud)
4. **Create job queue** — BullMQ for async provisioning
5. **Build callback endpoint** — Receive registration from new instances
6. **Add region/size selector UI** — Fetch and cache available options

---

*Last updated: 2026-02-08*
