# Claw Cloud — Mini Apps Architecture

User-created mini apps: bespoke applications built through conversation with OpenClaw.

---

## Table of Contents

1. [Vision](#1-vision)
2. [App Structure](#2-app-structure)
3. [Creation Flow](#3-creation-flow)
4. [Runtime Options](#4-runtime-options)
5. [Dev Mode](#5-dev-mode)
6. [Data Model](#6-data-model)
7. [Marketplace (v2)](#7-marketplace-v2)
8. [Security](#8-security)

---

## 1. Vision

### The Dream

Users describe what they want in natural language. OpenClaw builds it. The app appears as an icon in their dashboard.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Dashboard                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  💬     │  │  📊     │  │  🏋️     │  │  📝     │           │
│   │  Chat   │  │ Expense │  │ Workout │  │  Notes  │           │
│   │         │  │ Tracker │  │ Logger  │  │         │           │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  🍕     │  │  📅     │  │  🎯     │  │  ➕     │           │
│   │ Recipe  │  │ Habit   │  │  Goals  │  │  Create │           │
│   │  Box    │  │ Tracker │  │ Tracker │  │   New   │           │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Matters

- **No-code for real:** Not dragging boxes, but describing what you want
- **Infinite customization:** Your app, your rules, your data
- **Living apps:** Ask OpenClaw to modify them anytime
- **Personal software:** Apps tailored to your exact needs

### Example Conversations

```
User: "Build me an app to track my coffee consumption. 
       I want to log each cup with the type and time, 
       and see a weekly chart of my caffeine intake."

OpenClaw: "I'll create a Coffee Tracker app for you..."
         [Creates app with logging form, chart view, data storage]
         
User: "Add a feature to estimate when I should stop 
       drinking coffee based on my bedtime."

OpenClaw: "Good idea! I'll add a caffeine calculator..."
         [Updates app with new feature]
```

---

## 2. App Structure

### Directory Layout

Each mini app lives in a self-contained directory:

```
/apps/
└── coffee-tracker-a1b2c3/
    ├── app.json              # Metadata, icon, permissions
    ├── src/
    │   ├── App.tsx           # Main React component
    │   ├── components/
    │   │   ├── LogForm.tsx
    │   │   ├── WeeklyChart.tsx
    │   │   └── CaffeineCalc.tsx
    │   └── styles.css
    ├── api/
    │   ├── log.ts            # POST /api/log - Add coffee entry
    │   ├── entries.ts        # GET /api/entries - List entries
    │   └── stats.ts          # GET /api/stats - Get statistics
    └── data/
        ├── entries.json      # Coffee log entries
        └── settings.json     # User preferences
```

### app.json Manifest

```json
{
  "id": "coffee-tracker-a1b2c3",
  "name": "Coffee Tracker",
  "description": "Track your daily caffeine intake",
  "icon": "☕",
  "version": "1.0.0",
  "created": "2026-02-08T12:00:00Z",
  "updated": "2026-02-08T14:30:00Z",
  "author": "user",
  
  "permissions": {
    "network": false,
    "filesystem": ["data/*"],
    "notifications": true,
    "calendar": false
  },
  
  "entry": "src/App.tsx",
  "api": "api/",
  
  "settings": {
    "theme": "auto",
    "dataRetention": "forever"
  }
}
```

### File Types

| Directory | Purpose | Examples |
|-----------|---------|----------|
| `src/` | React components, UI code | `.tsx`, `.css`, `.ts` |
| `api/` | Serverless functions | `.ts` handlers |
| `data/` | Persistent storage | `.json`, `.sqlite` |

---

## 3. Creation Flow

### Step-by-Step Process

```
┌─────────────────────────────────────────────────────────────────┐
│                     App Creation Flow                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. USER DESCRIBES                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ "I want an app to track my workouts. Log exercises,     │   │
│   │  sets, reps, and weight. Show progress over time."      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   2. OPENCLAW PLANS                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Analyzes requirements                                  │   │
│   │ • Designs data model                                     │   │
│   │ • Plans component structure                              │   │
│   │ • Identifies needed APIs                                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   3. SCAFFOLD GENERATED                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ /apps/workout-logger-x7y8z9/                            │   │
│   │   ├── app.json                                          │   │
│   │   ├── src/App.tsx                                       │   │
│   │   ├── api/workouts.ts                                   │   │
│   │   └── data/workouts.json                                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   4. CODE WRITTEN                                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • React components with forms and charts                │   │
│   │ • API handlers for CRUD operations                       │   │
│   │ • Initial data schema                                    │   │
│   │ • Styling to match dashboard theme                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   5. REGISTERED                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • App added to /apps/index.json                         │   │
│   │ • Icon appears in dashboard                              │   │
│   │ • Ready to use immediately                               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Registry Index

```json
// /apps/index.json
{
  "apps": [
    {
      "id": "coffee-tracker-a1b2c3",
      "name": "Coffee Tracker",
      "icon": "☕",
      "path": "/apps/coffee-tracker-a1b2c3"
    },
    {
      "id": "workout-logger-x7y8z9",
      "name": "Workout Logger",
      "icon": "🏋️",
      "path": "/apps/workout-logger-x7y8z9"
    }
  ]
}
```

---

## 4. Runtime Options

### Comparison Matrix

| Approach | Isolation | Performance | Complexity | Security |
|----------|-----------|-------------|------------|----------|
| **iFrame Sandbox** | ✅ Excellent | ⚠️ Moderate | ✅ Low | ✅ Excellent |
| **Server-Rendered** | ⚠️ Moderate | ✅ Excellent | ⚠️ Moderate | ⚠️ Moderate |
| **Edge Functions** | ✅ Excellent | ✅ Excellent | ❌ High | ✅ Excellent |

### Option A: iFrame Sandbox (Recommended for MVP)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Dashboard (Parent)                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │   ┌────────────────────────────────────────────────────┐  │  │
│  │   │              iFrame (Sandboxed)                     │  │  │
│  │   │                                                      │  │  │
│  │   │   ┌────────────────────────────────────────────┐   │  │  │
│  │   │   │            Mini App                         │   │  │  │
│  │   │   │                                             │   │  │  │
│  │   │   │   • Runs in isolated context               │   │  │  │
│  │   │   │   • Cannot access parent DOM               │   │  │  │
│  │   │   │   • Communicates via postMessage           │   │  │  │
│  │   │   │   • API calls through proxy                │   │  │  │
│  │   │   │                                             │   │  │  │
│  │   │   └────────────────────────────────────────────┘   │  │  │
│  │   │                                                      │  │  │
│  │   │   sandbox="allow-scripts allow-forms"               │  │  │
│  │   └────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**iFrame Sandbox Attributes:**
```html
<iframe
  src="/apps/coffee-tracker/index.html"
  sandbox="allow-scripts allow-forms allow-same-origin"
  allow="clipboard-write"
></iframe>
```

### Option B: Server-Rendered (SSR)

```
Request → Next.js API Route → Render React → Return HTML
                    ↓
              Read app files
              Execute API logic
              Return complete page
```

**Pros:** Fast initial load, SEO-friendly
**Cons:** Less isolation, server resources per app

### Option C: Edge Functions (Future)

```
Request → Edge Runtime (Deno/Cloudflare Workers) → Response
                    ↓
              Isolated V8 context per request
              Millisecond cold starts
              Global distribution
```

**Pros:** Maximum isolation, scalable
**Cons:** Complex setup, cost at scale

### Recommended Architecture (MVP)

**iFrame + Local API:**

```typescript
// Dashboard loads app in sandboxed iFrame
<iframe src={`/apps/${appId}/`} sandbox="..." />

// App makes API calls to local endpoints
fetch(`/api/apps/${appId}/entries`)
  .then(res => res.json())
  .then(data => setEntries(data));

// API routes proxy to app's api/ directory
// /api/apps/[appId]/[...path].ts
export async function GET(req, { params }) {
  const { appId, path } = params;
  const handler = await import(`/apps/${appId}/api/${path}.ts`);
  return handler.default(req);
}
```

---

## 5. Dev Mode

### Toggle: "Use" ↔ "Edit"

Every mini app has two modes accessible via a toggle:

```
┌─────────────────────────────────────────────────────────────────┐
│  Coffee Tracker                              [Use] [Edit] ✏️    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    (app content here)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Edit Mode: Split View

```
┌─────────────────────────────────────────────────────────────────┐
│  Coffee Tracker - Edit Mode                        [Use] [Edit] │
├────────────────────────────┬────────────────────────────────────┤
│                            │                                     │
│   App Preview              │   Chat with OpenClaw               │
│   ┌──────────────────────┐ │   ┌──────────────────────────────┐ │
│   │                      │ │   │                              │ │
│   │   ☕ Log Coffee      │ │   │ User: Can you add a dark    │ │
│   │   ┌──────────────┐   │ │   │       mode toggle?          │ │
│   │   │ Espresso  ▼  │   │ │   │                              │ │
│   │   └──────────────┘   │ │   │ OpenClaw: Sure! I'll add    │ │
│   │                      │ │   │ a theme toggle to the       │ │
│   │   [Log Cup]          │ │   │ settings...                 │ │
│   │                      │ │   │                              │ │
│   │   Today: 3 cups      │ │   │ [Code being written...]     │ │
│   │   ████████░░ 240mg   │ │   │                              │ │
│   │                      │ │   │ ✓ Added ThemeToggle.tsx     │ │
│   │                      │ │   │ ✓ Updated App.tsx           │ │
│   │                      │ │   │ ✓ Added CSS variables       │ │
│   │                      │ │   │                              │ │
│   └──────────────────────┘ │   │ Done! Try the toggle in     │ │
│                            │   │ the top right.               │ │
│   🔄 Hot Reload Active    │   │                              │ │
│                            │   └──────────────────────────────┘ │
│                            │                                     │
└────────────────────────────┴────────────────────────────────────┘
```

### Dev Mode Features

#### 1. Chat Has Full Code Context
```typescript
// OpenClaw automatically sees:
const appContext = {
  appId: 'coffee-tracker-a1b2c3',
  files: [
    'src/App.tsx',
    'src/components/LogForm.tsx',
    'src/components/WeeklyChart.tsx',
    'api/entries.ts',
    'data/entries.json'
  ],
  // Full file contents loaded for editing
};
```

#### 2. Hot Reload Preview
```typescript
// File watcher triggers reload
chokidar.watch(`/apps/${appId}/src/**`).on('change', (path) => {
  // Rebuild app
  await buildApp(appId);
  // Notify preview iframe
  previewFrame.postMessage({ type: 'reload' }, '*');
});
```

#### 3. Deploy = Save to Production
```typescript
// "Deploy" button actions:
async function deployApp(appId: string) {
  // 1. Validate app structure
  await validateApp(appId);
  
  // 2. Run any build steps
  await buildApp(appId);
  
  // 3. Update app.json version
  await bumpVersion(appId);
  
  // 4. Clear preview cache
  await clearCache(appId);
  
  // 5. App is now live (it's already on filesystem)
  console.log('Deployed!');
}
```

### Code View (Optional)

For technical users, a code view tab:

```
┌─────────────────────────────────────────────────────────────────┐
│  Coffee Tracker - Edit Mode          [Preview] [Code] [Chat]   │
├─────────────────────────────────────────────────────────────────┤
│  src/App.tsx                                                     │
├─────────────────────────────────────────────────────────────────┤
│  1  │ import { useState, useEffect } from 'react';              │
│  2  │ import { LogForm } from './components/LogForm';           │
│  3  │ import { WeeklyChart } from './components/WeeklyChart';   │
│  4  │ import { ThemeToggle } from './components/ThemeToggle';   │
│  5  │                                                            │
│  6  │ export default function App() {                           │
│  7  │   const [entries, setEntries] = useState([]);             │
│  8  │   const [theme, setTheme] = useState('light');            │
│  9  │                                                            │
│ 10  │   useEffect(() => {                                       │
│ 11  │     fetch('/api/entries')                                 │
│ 12  │       .then(res => res.json())                            │
│ 13  │       .then(data => setEntries(data));                    │
│ 14  │   }, []);                                                  │
│ 15  │                                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Data Model

### TypeScript Interfaces

```typescript
// types/mini-app.ts

interface MiniApp {
  id: string;                    // Unique identifier (nanoid)
  name: string;                  // Display name
  description: string;           // Short description
  icon: string;                  // Emoji or icon URL
  version: string;               // Semver
  created: Date;
  updated: Date;
  author: 'user' | 'marketplace' | string;  // Origin
  
  permissions: AppPermissions;
  entry: string;                 // Entry point file
  apiPath: string;               // API directory
  
  settings: AppSettings;
  status: 'active' | 'draft' | 'archived';
}

interface AppPermissions {
  network: boolean | string[];   // true, false, or allowed domains
  filesystem: string[];          // Allowed paths (glob patterns)
  notifications: boolean;
  calendar: boolean;
  contacts: boolean;
  location: boolean;
  camera: boolean;
  microphone: boolean;
}

interface AppSettings {
  theme: 'light' | 'dark' | 'auto';
  dataRetention: 'forever' | '30d' | '90d' | '1y';
  autoBackup: boolean;
  [key: string]: unknown;        // App-specific settings
}

interface AppFile {
  path: string;                  // Relative to app root
  content: string;               // File contents
  type: 'component' | 'api' | 'data' | 'config' | 'style';
  lastModified: Date;
}

interface AppRegistry {
  apps: AppRegistryEntry[];
  lastUpdated: Date;
}

interface AppRegistryEntry {
  id: string;
  name: string;
  icon: string;
  path: string;
  status: MiniApp['status'];
}
```

### Data Storage Options

#### Option A: JSON Files (Simple, MVP)
```
/apps/coffee-tracker/data/
├── entries.json      # Array of log entries
├── settings.json     # User preferences
└── cache.json        # Computed data
```

#### Option B: SQLite (Structured, Scalable)
```
/apps/coffee-tracker/data/
└── app.db
    ├── entries (table)
    ├── settings (table)
    └── migrations (table)
```

#### Option C: Hybrid (Recommended)
```
/apps/coffee-tracker/data/
├── app.db            # Structured data (SQLite)
├── settings.json     # Quick-access config
└── uploads/          # Binary files
```

---

## 7. Marketplace (v2)

### Vision

A curated marketplace where users can share and discover mini apps:

```
┌─────────────────────────────────────────────────────────────────┐
│                       App Marketplace                            │
├─────────────────────────────────────────────────────────────────┤
│  [Featured]  [Productivity]  [Health]  [Finance]  [Fun]         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ 🏋️ Workout Pro  │  │ 💰 Budget Boss  │  │ 📚 Book Club    │  │
│  │ ★★★★★ (234)     │  │ ★★★★☆ (89)      │  │ ★★★★★ (156)     │  │
│  │ Track workouts  │  │ 50/30/20 budget │  │ Reading lists   │  │
│  │ with AI coach   │  │ made easy       │  │ & discussions   │  │
│  │                 │  │                 │  │                 │  │
│  │ Free            │  │ $4.99           │  │ $2.99           │  │
│  │ [Install]       │  │ [Buy]           │  │ [Buy]           │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Publish Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      Publish Your App                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. PREPARE                                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ ☑ Add description and screenshots                       │   │
│   │ ☑ Set pricing (free or paid)                            │   │
│   │ ☑ Choose category                                        │   │
│   │ ☑ Review permissions requested                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   2. SUBMIT FOR REVIEW                                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Automated security scan                                │   │
│   │ • Code quality check                                     │   │
│   │ • Manual review (if paid or permissions require)         │   │
│   │ • ~24-48 hour turnaround                                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           ▼                                      │
│   3. PUBLISHED                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Listed in marketplace                                  │   │
│   │ • Analytics dashboard available                          │   │
│   │ • Earnings tracked (if paid)                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Install = Fork

When a user installs a marketplace app:

```typescript
async function installApp(marketplaceAppId: string, userId: string) {
  // 1. Fetch app template from marketplace
  const template = await marketplace.getApp(marketplaceAppId);
  
  // 2. Generate new unique ID for user's instance
  const newAppId = `${template.slug}-${nanoid(6)}`;
  
  // 3. Copy files to user's /apps directory
  await copyAppFiles(template, `/apps/${newAppId}/`);
  
  // 4. Initialize with user's data directory (empty)
  await initDataDir(`/apps/${newAppId}/data/`);
  
  // 5. Register in user's app index
  await registerApp(userId, newAppId);
  
  // User now owns this copy - can modify freely
  return newAppId;
}
```

### Revenue Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    Revenue Split: 70/30                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   App Sale: $4.99                                                │
│                                                                  │
│   ┌───────────────────────────────────────────┐ ┌─────────────┐ │
│   │                                           │ │             │ │
│   │              Developer: 70%               │ │  Platform   │ │
│   │                 $3.49                     │ │    30%      │ │
│   │                                           │ │   $1.50     │ │
│   └───────────────────────────────────────────┘ └─────────────┘ │
│                                                                  │
│   Payment: Monthly payout via Stripe Connect                    │
│   Minimum: $10 threshold                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Categories

| Category | Examples |
|----------|----------|
| Productivity | Task managers, note-taking, time tracking |
| Health & Fitness | Workout logs, meal planners, habit trackers |
| Finance | Budget tools, expense trackers, invoice generators |
| Learning | Flashcards, reading logs, course trackers |
| Fun & Social | Games, polls, group activities |
| Utilities | Calculators, converters, generators |
| Work | CRM tools, project trackers, meeting notes |
| Lifestyle | Recipe boxes, travel planners, wardrobe managers |

---

## 8. Security

### App Sandboxing

Every mini app runs in an isolated environment:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Security Layers                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Layer 1: iFrame Sandbox                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Separate JavaScript context                            │   │
│   │ • No access to parent window                             │   │
│   │ • Restricted capabilities via sandbox attribute          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Layer 2: Content Security Policy                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Script sources restricted to app origin                │   │
│   │ • No inline scripts unless explicitly allowed            │   │
│   │ • Network requests limited to approved domains           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Layer 3: API Permissions                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • Each app declares required permissions                 │   │
│   │ • User approves permissions on install                   │   │
│   │ • Runtime permission checks on every API call            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Layer 4: Filesystem Isolation                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • App can only access its own /data directory            │   │
│   │ • No access to other apps' data                          │   │
│   │ • No access to system files                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Permission System

```typescript
// Permission request and validation
const PERMISSION_LEVELS = {
  low: ['filesystem:own', 'notifications'],
  medium: ['network:limited', 'calendar:read'],
  high: ['network:all', 'contacts', 'location'],
  dangerous: ['filesystem:all', 'camera', 'microphone']
};

function validatePermission(app: MiniApp, permission: string): boolean {
  // Check if app declared this permission
  if (!app.permissions[permission]) {
    throw new PermissionDeniedError(
      `App ${app.id} does not have permission: ${permission}`
    );
  }
  
  // Check if user approved this permission
  if (!userApprovedPermissions[app.id]?.includes(permission)) {
    throw new PermissionNotGrantedError(
      `User has not granted permission: ${permission}`
    );
  }
  
  return true;
}
```

### Marketplace Code Review

For marketplace apps, additional security measures:

```
┌─────────────────────────────────────────────────────────────────┐
│                  Marketplace Security Review                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Automated Checks (All Apps)                                    │
│   ☑ Static analysis for malicious patterns                      │
│   ☑ Dependency vulnerability scan                               │
│   ☑ Permission scope validation                                 │
│   ☑ Code complexity limits                                      │
│                                                                  │
│   Manual Review (Paid Apps + High Permissions)                   │
│   ☑ Code walkthrough by security team                           │
│   ☑ Network behavior analysis                                   │
│   ☑ Data handling practices                                     │
│   ☑ Privacy policy verification                                 │
│                                                                  │
│   Runtime Monitoring                                             │
│   ☑ Anomaly detection for installed apps                        │
│   ☑ User reports and automatic flagging                         │
│   ☑ Emergency takedown capability                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Security Best Practices for App Developers

```markdown
## Do ✅
- Request only permissions you need
- Validate all user input
- Use parameterized queries for SQLite
- Sanitize data before rendering
- Document what data you collect

## Don't ❌
- Request network access unless required
- Store sensitive data in plain text
- Use eval() or dynamic code execution
- Access APIs without permission checks
- Collect data not needed for functionality
```

---

## Summary

Mini Apps transform Claw Cloud from a chat interface into a **personal software platform**. Users describe what they want, OpenClaw builds it, and apps live alongside the AI assistant.

### MVP Scope (v1)

- [x] App structure and manifest
- [x] Creation flow via chat
- [x] iFrame sandbox runtime
- [x] Basic dev mode (use/edit toggle)
- [x] JSON file storage
- [x] Permission system

### Future (v2)

- [ ] Marketplace with revenue sharing
- [ ] SQLite storage option
- [ ] Advanced code view
- [ ] App versioning and rollback
- [ ] Team collaboration on apps
- [ ] Edge function runtime option

---

*This architecture enables a new paradigm: software that builds itself, tailored to each user's exact needs.*
