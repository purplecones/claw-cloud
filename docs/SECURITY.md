# Claw Cloud — Security Architecture

This document details how Claw Cloud protects user data across all system components.

---

## Table of Contents

1. [Chat Content (Proxied, Not Stored)](#1-chat-content-proxied-not-stored)
2. [OpenClaw Memory/Files](#2-openclaw-memoryfiles)
3. [Cloud Credentials (Encrypted)](#3-cloud-credentials-encrypted)
4. [User's Other Cloud Resources](#4-users-other-cloud-resources)
5. [Verification & Trust](#5-verification--trust)

---

## 1. Chat Content (Proxied, Not Stored)

### We're a Pipe, Not a Bucket

Claw Cloud **never stores chat content**. Messages flow through our infrastructure like water through a pipe — they pass through, but they don't stay.

```
┌─────────────┐     WebSocket/SSE      ┌──────────────┐     HTTP/WS     ┌─────────────┐
│   Browser   │◄─────────────────────► │  Claw Cloud  │◄──────────────► │  OpenClaw   │
│   (User)    │                        │   (Proxy)    │                 │   Gateway   │
└─────────────┘                        └──────────────┘                 └─────────────┘
                                              │
                                              │ ✗ No database writes
                                              │ ✗ No message logging
                                              │ ✗ No content storage
                                              ▼
                                        [Pass-through only]
```

### Technical Implementation

#### WebSocket Passthrough
```typescript
// Simplified proxy logic
ws.on('message', async (data) => {
  // Forward immediately — no storage, no logging
  await upstreamSocket.send(data);
});

// What we DO NOT do:
// ❌ db.insert('messages', { content: data })
// ❌ logger.info('Message received', { body: data })
// ❌ queue.push({ message: data })
```

#### SSE (Server-Sent Events) Passthrough
For streaming responses, we use the same pass-through pattern:
```typescript
upstream.on('data', (chunk) => {
  // Pipe directly to client — zero retention
  res.write(chunk);
});
```

### What Gets Logged (Metadata Only)

We log **operational metadata** for debugging and monitoring:

| Logged ✓ | NOT Logged ✗ |
|----------|--------------|
| Timestamp | Message content |
| User ID | Chat history |
| Connection duration | Prompts or responses |
| Bytes transferred | File contents |
| Error codes | Tool outputs |

Example log entry:
```json
{
  "timestamp": "2026-02-08T12:34:56Z",
  "user_id": "usr_abc123",
  "instance_id": "inst_xyz789",
  "event": "ws_message",
  "direction": "upstream",
  "bytes": 1247,
  "latency_ms": 23
}
```

### Database Schema

Our database has **no tables for message content**:

```sql
-- Tables we HAVE:
CREATE TABLE users (...);           -- Account info
CREATE TABLE instances (...);       -- VPS metadata
CREATE TABLE credentials (...);     -- Encrypted cloud tokens
CREATE TABLE audit_log (...);       -- Connection events (no content)

-- Tables we DON'T HAVE:
-- ❌ messages
-- ❌ chat_history
-- ❌ conversations
-- ❌ prompts
-- ❌ responses
```

---

## 2. OpenClaw Memory/Files

### Your Files Live on Your Server

All OpenClaw memory, files, and configuration reside **exclusively on the user's VPS filesystem**. Claw Cloud has no access to this storage.

```
User's VPS (e.g., DigitalOcean Droplet)
├── /root/.openclaw/
│   ├── workspace/          # User's files
│   │   ├── MEMORY.md       # Long-term memory
│   │   ├── memory/         # Daily logs
│   │   └── projects/       # User data
│   ├── config/             # OpenClaw configuration
│   └── secrets/            # API keys, tokens
```

### What Claw Cloud Interacts With

We communicate **only** with the OpenClaw Gateway API:

```
┌─────────────────────────────────────────────────────────────────┐
│                      User's VPS                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │   ┌──────────────┐                ┌─────────────────────┐ │  │
│  │   │   Gateway    │◄───────────────│   Filesystem        │ │  │
│  │   │   (HTTP)     │   Internal     │   (Memory, Files)   │ │  │
│  │   └──────────────┘                └─────────────────────┘ │  │
│  │          ▲                                                 │  │
│  │          │ Port 3000 (HTTPS)                              │  │
│  └──────────┼────────────────────────────────────────────────┘  │
│             │                                                    │
└─────────────┼────────────────────────────────────────────────────┘
              │
     Claw Cloud connects here ONLY
              │
┌─────────────┴─────────────────────────────────────────────────────┐
│                       Claw Cloud                                   │
│   • Send messages to Gateway API                                  │
│   • Receive responses from Gateway API                            │
│   • ✗ No SSH access                                               │
│   • ✗ No storage mounting                                         │
│   • ✗ No file system access                                       │
│   • ✗ No database replication                                     │
└───────────────────────────────────────────────────────────────────┘
```

### Access Boundaries

| Claw Cloud CAN | Claw Cloud CANNOT |
|----------------|-------------------|
| Send HTTP requests to Gateway | SSH into VPS |
| Receive API responses | Mount filesystems |
| Check instance health | Read memory files |
| Proxy chat messages | Access ~/.openclaw/ |
| Trigger provisioning | View stored secrets |

### Zero Access by Default

We have **no access** to user instances:
- No SSH keys installed
- No backdoors or management ports
- No remote access capability

**If users need help:**
1. Their OpenClaw can help debug (it has full access to its own system)
2. We provide comprehensive docs and troubleshooting guides
3. User can optionally grant temporary support access:

**Temporary Support Access (opt-in only)**
```
Dashboard:
┌─────────────────────────────────────────┐
│ Support Access: OFF                     │
│ [Grant access - expires in 24h]         │
└─────────────────────────────────────────┘
```

Flow:
1. User clicks "Grant access"
2. Our public SSH key is injected into their instance
3. Access auto-expires after 24 hours (cron removes key)
4. Full audit log visible to user showing all commands run
5. User can revoke immediately anytime

**This means:**
- By default, we literally cannot access your instance
- Your data is truly yours — not just a promise, but technically enforced
- Support access is temporary, audited, and user-controlled

### Why This Matters

1. **Data residency:** Your data stays in the region you chose
2. **Compliance:** GDPR, HIPAA concerns are simplified — we don't have the data
3. **Exit strategy:** Delete your Claw Cloud account; your data remains on your VPS
4. **Recovery:** Backup your VPS; you have everything

---

## 3. Cloud Credentials (Encrypted)

### Encryption Before Storage

When you provide cloud credentials (e.g., DigitalOcean API token), we:

1. **Encrypt immediately** using AES-256-GCM
2. **Derive encryption key** from your unique identifier
3. **Store ciphertext only** — plaintext never hits the database

```typescript
// Credential storage flow
async function storeCredential(userId: string, credential: string) {
  // 1. Derive per-user encryption key
  const userKey = await deriveKey(userId, masterSecret);
  
  // 2. Encrypt with authenticated encryption
  const encrypted = await encrypt(credential, userKey, {
    algorithm: 'aes-256-gcm',
    iv: randomBytes(16)
  });
  
  // 3. Store only ciphertext
  await db.credentials.insert({
    user_id: userId,
    provider: 'digitalocean',
    ciphertext: encrypted.ciphertext,
    iv: encrypted.iv,
    tag: encrypted.authTag
    // ❌ plaintext field does NOT exist
  });
}
```

### Key Derivation Options

#### Option A: Platform-Managed Keys
```typescript
// Key derived from user ID + platform secret
const userKey = await pbkdf2(
  masterSecret,
  userId + salt,
  100000,  // iterations
  32,      // key length
  'sha256'
);
```

#### Option B: User Password-Derived (More Secure)
```typescript
// Key derived from user's password (we never see the key)
const userKey = await argon2id(
  userPassword,
  userSalt,
  { memory: 65536, iterations: 3, parallelism: 4 }
);
```

#### Option C: Zero-Knowledge (Best)
For maximum security, we support:
- **AWS:** IAM assume-role with external ID — we never see credentials
- **GCP:** Workload Identity Federation — no service account keys
- **OAuth flows:** Token refresh without storing secrets

```
┌─────────────────────────────────────────────────────────────────┐
│               Assume-Role Flow (AWS Example)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User's AWS Account                    Claw Cloud              │
│   ┌─────────────────┐                  ┌─────────────────┐      │
│   │   IAM Role      │                  │   Our AWS       │      │
│   │   trust policy  │◄─────────────────│   Account       │      │
│   │   allows us to  │   AssumeRole     │   (trusted)     │      │
│   │   assume        │                  │                 │      │
│   └─────────────────┘                  └─────────────────┘      │
│          │                                                       │
│          ▼                                                       │
│   Temporary credentials (1-hour expiry)                         │
│   We NEVER see their long-term access keys                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Credential Lifecycle

| Stage | What Happens |
|-------|--------------|
| Input | Encrypted client-side before transmission (optional) |
| Transit | HTTPS/TLS 1.3 encryption |
| Storage | AES-256-GCM encrypted at rest |
| Usage | Decrypted in memory, used, discarded |
| Deletion | Cryptographic erasure (delete key = data gone) |

---

## 4. User's Other Cloud Resources

### Scoped IAM Roles with Minimal Permissions

We require only the **minimum permissions necessary** to provision and manage OpenClaw instances:

#### DigitalOcean Scope
```
Required permissions:
✓ droplet:create
✓ droplet:delete
✓ droplet:read

NOT required (we don't ask for):
✗ database access
✗ kubernetes access
✗ spaces/storage access
✗ domain management
✗ billing access
```

#### AWS IAM Policy (Example)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/ManagedBy": "claw-cloud"
        }
      }
    }
  ]
}
```

### Principle of Least Privilege

```
┌─────────────────────────────────────────────────────────────────┐
│                    User's Cloud Account                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Permission Boundary                    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │         Resources Claw Cloud CAN access            │  │   │
│  │  │                                                     │  │   │
│  │  │   • OpenClaw VPS instances                         │  │   │
│  │  │   • Associated security groups                     │  │   │
│  │  │   • Instance metadata                              │  │   │
│  │  │                                                     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │                                                           │   │
│  │       ┌────────────────────────────────────────────┐     │   │
│  │       │    Resources we CANNOT access              │     │   │
│  │       │    (outside permission boundary)           │     │   │
│  │       │                                            │     │   │
│  │       │    • Other VMs/EC2 instances               │     │   │
│  │       │    • Databases (RDS, etc.)                 │     │   │
│  │       │    • Storage buckets (S3, Spaces)          │     │   │
│  │       │    • Kubernetes clusters                   │     │   │
│  │       │    • DNS/Domains                           │     │   │
│  │       │    • Billing information                   │     │   │
│  │       │    • Other services                        │     │   │
│  │       │                                            │     │   │
│  │       └────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Physical Access Limitation

We **physically cannot** access resources outside the permission boundary:
- AWS/GCP/Azure enforce IAM at the API level
- Requests for unauthorized resources return 403 Forbidden
- No backdoors, no admin overrides
- User controls the IAM policy entirely

---

## 5. Verification & Trust

### Open Source Transparency

Our proxy and provisioning code is **open source**:

```
github.com/purplecones/claw-cloud
├── packages/
│   ├── proxy/           # WebSocket/SSE passthrough code
│   ├── provisioner/     # Pulumi infrastructure code
│   └── web/             # Dashboard (no sensitive logic)
```

**You can verify:**
- ✓ No database writes for message content
- ✓ No logging of chat bodies
- ✓ Credential encryption implementation
- ✓ IAM policies we request

### Network Inspection

Users can verify our claims by inspecting network traffic:

```bash
# Watch what we send to your VPS
tcpdump -i any port 3000 -A

# Verify we only make Gateway API calls
mitmproxy --mode transparent

# Check our requests contain no surprises
wireshark filter: host claw.cloud
```

### No Hidden Storage

Our database schema is documented. There are **no tables for message content**:

```sql
-- You can audit our schema
\dt  -- List all tables

-- Result:
-- users
-- instances  
-- credentials  (encrypted)
-- audit_log    (connection metadata only)
-- sessions
-- plans

-- NOT present:
-- messages ❌
-- chats ❌
-- history ❌
```

### Third-Party Audit

For enterprise customers, we support:
- **SOC 2 Type II** compliance roadmap
- **Penetration testing** reports available
- **Security questionnaire** responses
- **On-site audit** for enterprise contracts

---

## Summary

| Data Type | Storage Location | Claw Cloud Access |
|-----------|-----------------|-------------------|
| Chat messages | None (pass-through) | Transit only, not stored |
| OpenClaw memory/files | User's VPS | No access |
| Cloud credentials | Our DB (encrypted) | Ciphertext only |
| User's other cloud resources | User's cloud | Outside permission boundary |
| Account info (email, plan) | Our DB | Full access |
| Connection metadata | Our logs | Timestamps, bytes, no content |

**The bottom line:** We've architected Claw Cloud so that even if we were compromised, attackers wouldn't get your chat history, files, or usable credentials. Your data lives on your infrastructure, protected by encryption we can't reverse and permissions we can't escalate.

---

*Questions? Security concerns? Email security@claw.cloud or open a GitHub issue.*
