# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Communication Style
- Be professional, concise, and direct
- Do NOT use emojis in code reviews, changelogs, or any generated content. You may use professional visual indicators or favor markdown formatting over emojis.
- Focus on substance over style
- Use clear technical language

## Project Overview

# ShipCraft AI

**Domain**: `shipcraft.ai`
**Customer Apps**: `*.goeslive.dev` (e.g., `myapp.goeslive.dev`)

AI-powered full-stack application generation platform built on Cloudflare infrastructure. Transform natural language into production-ready React applications with live preview and one-click deployment.

**Tagline**: "Ship faster. Craft better."

**Target**: Solo developers and small teams/startups
**Goal**: Build a Lovable-like SaaS with deep Cloudflare integration

### Brand Identity
- **Name**: ShipCraft AI
- **Main Domain**: shipcraft.ai
- **Customer Apps Domain**: goeslive.dev
- **Value Proposition**: Speed ("Ship") + Quality ("Craft") + AI

**Tech Stack:**
- Frontend: React 19, TypeScript, Vite, TailwindCSS, React Router v7
- Backend: Cloudflare Workers, Durable Objects, D1 (SQLite)
- AI/LLM: OpenAI, Anthropic, Google AI Studio (Gemini)
- WebSocket: PartySocket for real-time communication
- Sandbox: Custom container service with CLI tools
- Git: isomorphic-git with SQLite filesystem

**Project Structure**

**Frontend (`/src`):**
- React application with 80+ components
- Single source of truth for types: `src/api-types.ts`
- All API calls in `src/lib/api-client.ts`
- Custom hooks in `src/hooks/`
- Route components in `src/routes/`

**Backend (`/worker`):**
- Entry point: `worker/index.ts` (7860 lines)
- Agent system: `worker/agents/` (88 files)
  - Core: SimpleCodeGeneratorAgent (Durable Object, 2800+ lines)
  - Operations: PhaseGeneration, PhaseImplementation, UserConversationProcessor
  - Tools: tools for LLM (read-files, run-analysis, regenerate-file, etc.)
  - Git: isomorphic-git with SQLite filesystem
- Database: `worker/database/` (Drizzle ORM, D1)
- Services: `worker/services/` (sandbox, code-fixer, oauth, rate-limit, secrets)
- API: `worker/api/` (routes, controllers, handlers)

**Other:**
- `/shared` - Shared types between frontend/backend (not worker specific types that are also imported in frontend)
- `/migrations` - D1 database migrations
- `/container` - Sandbox container tooling
- `/templates` - Project scaffolding templates

**Core Architecture:**
- Each chat session is a Durable Object instance (SimpleCodeGeneratorAgent)
- State machine drives code generation (IDLE → PHASE_GENERATING → PHASE_IMPLEMENTING → REVIEWING)
- Git history stored in SQLite, full clone protocol support
- WebSocket for real-time streaming and state synchronization

## Key Architectural Patterns

**Durable Objects Pattern:**
- Each chat session = Durable Object instance
- Persistent state in SQLite (blueprint, files, history)
- Ephemeral state in memory (abort controllers, active promises)
- Single-threaded per instance

**State Machine:**
IDLE → PHASE_GENERATING → PHASE_IMPLEMENTING → REVIEWING → IDLE

**CodeGenState (Agent State):**
- Project Identity: blueprint, projectName, templateName
- File Management: generatedFilesMap (tracks all files)
- Phase Tracking: generatedPhases, currentPhase
- State Machine: currentDevState, shouldBeGenerating
- Sandbox: sandboxInstanceId, commandsHistory
- Conversation: conversationMessages, pendingUserInputs

**WebSocket Communication:**
- Real-time streaming via PartySocket
- State restoration on reconnect (agent_connected message)
- Message deduplication (tool execution causes duplicates)

**Git System:**
- isomorphic-git with SQLite filesystem adapter
- Full commit history in Durable Object storage
- Git clone protocol support (rebase on template)
- FileManager auto-syncs from git via callbacks

## Common Development Tasks

**Change LLM Model for Operation:**
Edit `/worker/agents/inferutils/config.ts` → `AGENT_CONFIG` object

**Modify Conversation Agent Behavior:**
Edit `/worker/agents/operations/UserConversationProcessor.ts` (system prompt line 50)

**Add New WebSocket Message:**
1. Add type to `worker/api/websocketTypes.ts`
2. Handle in `worker/agents/core/websocket.ts`
3. Handle in `src/routes/chat/utils/handle-websocket-message.ts`

**Add New LLM Tool:**
1. Create `/worker/agents/tools/toolkit/my-tool.ts`
2. Export `createMyTool(agent, logger)` function
3. Import in `/worker/agents/tools/customTools.ts`
4. Add to `buildTools()` (conversation) or `buildDebugTools()` (debugger)

**Add API Endpoint:**
1. Define types in `src/api-types.ts`
2. Add to `src/lib/api-client.ts`
3. Create service in `worker/database/services/`
4. Create controller in `worker/api/controllers/`
5. Add route in `worker/api/routes/`
6. Register in `worker/api/routes/index.ts`

## Important Context

**Deep Debugger:**
- Location: `/worker/agents/assistants/codeDebugger.ts`
- Model: Gemini 2.5 Pro (reasoning_effort: high, 32k tokens)
- Diagnostic priority: run_analysis → get_runtime_errors → get_logs
- Can fix multiple files in parallel (regenerate_file)
- Cannot run during code generation (checked via isCodeGenerating())

**User Secrets Store (Durable Object):**
- Location: `/worker/services/secrets/`
- Purpose: Encrypted storage for user API keys with key rotation
- Architecture: One DO per user, XChaCha20-Poly1305 encryption, SQLite backend
- Key derivation: MEK → UMK → DEK (hierarchical PBKDF2)
- Features: Key rotation, soft deletion, access tracking, expiration support
- RPC Methods: Return `null`/`boolean` on error, never throw exceptions
- Testing: 90 comprehensive tests in `/test/worker/services/secrets/`

**Git System:**
- GitVersionControl class wraps isomorphic-git
- Key methods: commit(), reset(), log(), show()
- FileManager auto-syncs via callback registration
- Access control: user conversations get safe commands, debugger gets full access
- SQLite filesystem adapter (`/worker/agents/git/fs-adapter.ts`)

**Abort Controller Pattern:**
- `getOrCreateAbortController()` reuses controller for nested operations
- Cleared after top-level operations complete
- Shared by parent and nested tool calls
- User abort cancels entire operation tree

**Message Deduplication:**
- Tool execution causes duplicate AI messages
- Backend skips redundant LLM calls (empty tool results)
- Frontend utilities deduplicate live and restored messages
- System prompt teaches LLM not to repeat

## Core Rules (Non-Negotiable)

**1. Strict Type Safety**
- NEVER use `any` type
- Frontend imports types from `@/api-types` (single source of truth)
- Search codebase for existing types before creating new ones

**2. DRY Principle**
- Search for similar functionality before implementing
- Extract reusable utilities, hooks, and components
- Never copy-paste code - refactor into shared functions

**3. Follow Existing Patterns**
- Frontend APIs: All in `/src/lib/api-client.ts`
- Backend Routes: Controllers in `worker/api/controllers/`, routes in `worker/api/routes/`
- Database Services: In `worker/database/services/`
- Types: Shared in `shared/types/`, API in `src/api-types.ts`

**4. Code Quality**
- Production-ready code only - no TODOs or placeholders
- No hacky workarounds
- Comments explain purpose, not narration
- No overly verbose AI-like comments

**5. File Naming**
- React Components: PascalCase.tsx
- Utilities/Hooks: kebab-case.ts
- Backend Services: PascalCase.ts

## Common Pitfalls

**Don't:**
- Use `any` type (find or create proper types)
- Copy-paste code (extract to utilities)
- Use Vite env variables in Worker code
- Forget to update types when changing APIs
- Create new implementations without searching for existing ones
- Use emojis in code or comments
- Write verbose AI-like comments

**Do:**
- Search codebase thoroughly before creating new code
- Follow existing patterns consistently
- Keep comments concise and purposeful
- Write production-ready code
- Test thoroughly before submitting

## SaaS Features (In Development)

### Billing & Subscriptions
**Provider**: Stripe
**Location**: `worker/services/billing/`

**Subscription Tiers:**
| Tier | Price | Apps | LLM Calls/Day | Storage | Custom Domains | Teams |
|------|-------|------|---------------|---------|----------------|-------|
| Free | $0 | 3 | 50 | 100MB | 0 | No |
| Starter | $19/mo | 10 | 500 | 1GB | 1 | No |
| Pro | $49/mo | 50 | 2,000 | 10GB | 5 | Yes |
| Enterprise | Custom | Unlimited | Unlimited | Unlimited | Unlimited | Yes |

**Database Tables (billing):**
- `subscription_plans` - Plan definitions (cached from Stripe)
- `subscriptions` - User subscriptions
- `usage_records` - Usage tracking per billing period
- `stripe_events` - Webhook event log

### Multi-Tenancy
**Location**: `worker/services/rbac/`, `worker/database/services/OrganizationService.ts`

**RBAC Model:**
```
Organization
├── owner     - Full control, billing, delete org
├── admin     - Member management, settings, all apps
├── member    - Create/edit apps in team
└── viewer    - Read-only access
```

**Database Tables (multi-tenancy):**
- `organizations` - Organization entities
- `organization_members` - Membership with roles
- `teams` - Teams within organizations
- `organization_invites` - Pending invitations

### Custom Domains
**Location**: `worker/services/domains/`
**Integration**: Cloudflare Custom Hostnames API

**Database Tables (domains):**
- `custom_domains` - User custom domains with SSL status
- `domain_registrations` - Registered domains (via Cloudflare Registrar)
- `domain_availability_cache` - Domain search cache

### Admin Panel
**Location**: `src/routes/admin/`, `worker/api/controllers/admin/`
**Access**: Users with `is_admin=1` and `admin_role` set

### New Environment Variables (SaaS)
```env
# Domains
MAIN_DOMAIN=shipcraft.ai
CUSTOMER_APPS_DOMAIN=goeslive.dev
CUSTOM_DOMAIN=goeslive.dev

# Stripe
STRIPE_SECRET_KEY=sk_xxx
STRIPE_PUBLISHABLE_KEY=pk_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Cloudflare Domains (for Custom Hostnames)
CLOUDFLARE_ZONE_ID=xxx
```

### URLs Structure
- **Main App**: https://shipcraft.ai
- **API**: https://api.shipcraft.ai (or https://shipcraft.ai/api)
- **Customer Apps**: https://{app-name}.goeslive.dev
- **Example**: https://my-awesome-app.goeslive.dev

## Development Phases

### Phase 1: MVP - Monetization (Current)
- [ ] Billing database schema
- [ ] Stripe service integration
- [ ] Subscription management
- [ ] Usage tracking & quota enforcement
- [ ] Pricing page
- [ ] Billing dashboard

### Phase 2: Core SaaS Features
- [ ] Organizations & teams
- [ ] RBAC permissions
- [ ] Custom domain management
- [ ] Domain registration (Cloudflare Registrar)
- [ ] Admin panel

### Phase 3: Growth Features
- [ ] Webhook system
- [ ] Email service (Resend/MailChannels)
- [ ] Analytics dashboard
- [ ] Version history & rollback

### Phase 4: Enterprise Features
- [ ] SSO/SAML
- [ ] Feature flags
- [ ] Template marketplace
- [ ] White-labeling

## Quick Reference

**Add Billing Endpoint:**
1. Add types to `src/api-types.ts`
2. Create handler in `worker/api/controllers/billing/controller.ts`
3. Add route in `worker/api/routes/billingRoutes.ts`
4. Add to API client in `src/lib/api-client.ts`

**Add Organization Feature:**
1. Update schema in `worker/database/schema.ts`
2. Create service in `worker/database/services/OrganizationService.ts`
3. Add permission check via `worker/services/rbac/PermissionService.ts`

**Quota Enforcement:**
- Check quotas in middleware before protected operations
- Record usage in `usage_records` table
- Return 402 Payment Required if exceeded

## Git Workflow & Upstream Sync

### Repository Structure
```
cloudflare/vibesdk (upstream) ─── Original Cloudflare repo
         │
         ▼ Fork
your-github/shipcraft (origin) ─── Your fork with custom features
         │
         ▼ Deploy
Cloudflare Workers (shipcraft.ai)
```

### Initial Setup
```bash
# Clone your fork
git clone https://github.com/YOUR-USERNAME/shipcraft.git
cd shipcraft

# Add upstream remote
git remote add upstream https://github.com/cloudflare/vibesdk.git

# Verify remotes
git remote -v
# origin    github.com/YOUR-USERNAME/shipcraft.git (your fork)
# upstream  github.com/cloudflare/vibesdk.git (Cloudflare's repo)
```

### Syncing Upstream Updates
```bash
# Fetch latest from Cloudflare
git fetch upstream

# Merge into your main branch
git checkout main
git merge upstream/main

# Resolve conflicts if any, then push
git push origin main
```

### Avoiding Merge Conflicts

**CRITICAL: Follow these rules to minimize conflicts when Cloudflare updates vibesdk**

#### 1. Custom Code Goes in NEW Folders
```
worker/services/
├── billing/           ← OUR CODE (new folder)
├── domains/           ← OUR CODE (new folder)
├── rbac/              ← OUR CODE (new folder)
├── sandbox/           ← UPSTREAM (don't modify)
├── oauth/             ← UPSTREAM (don't modify)
└── rate-limit/        ← UPSTREAM (don't modify)

src/routes/
├── billing/           ← OUR CODE (new folder)
├── admin/             ← OUR CODE (new folder)
├── pricing/           ← OUR CODE (new folder)
├── chat/              ← UPSTREAM (don't modify)
├── apps/              ← UPSTREAM (don't modify)
└── home/              ← UPSTREAM (don't modify)

src/components/
├── billing/           ← OUR CODE (new folder)
├── admin/             ← OUR CODE (new folder)
├── domains/           ← OUR CODE (new folder)
└── ui/                ← UPSTREAM (don't modify)
```

#### 2. Database Schema - Add at END
```typescript
// worker/database/schema.ts

// ========== UPSTREAM TABLES (don't modify) ==========
export const users = sqliteTable('users', { ... });
export const apps = sqliteTable('apps', { ... });
// ... other upstream tables ...

// ========== SHIPCRAFT CUSTOM TABLES (add below) ==========
// Add all custom tables AFTER upstream tables to avoid conflicts

export const subscriptionPlans = sqliteTable('subscription_plans', { ... });
export const subscriptions = sqliteTable('subscriptions', { ... });
export const usageRecords = sqliteTable('usage_records', { ... });
export const organizations = sqliteTable('organizations', { ... });
// ... our custom tables ...
```

#### 3. Route Registration - Separate Section
```typescript
// worker/api/routes/index.ts

// ========== UPSTREAM ROUTES ==========
setupAuthRoutes(app);
setupAppRoutes(app);
// ... upstream routes ...

// ========== SHIPCRAFT CUSTOM ROUTES ==========
setupBillingRoutes(app);      // Our billing routes
setupAdminRoutes(app);        // Our admin routes
setupOrganizationRoutes(app); // Our org routes
```

#### 4. API Client - Add Methods at End
```typescript
// src/lib/api-client.ts

// Add our methods at the END of the apiClient object
export const apiClient = {
  // ... upstream methods ...

  // ========== SHIPCRAFT CUSTOM METHODS ==========
  billing: {
    getPlans: () => fetchApi('/billing/plans'),
    getSubscription: () => fetchApi('/billing/subscription'),
    // ...
  },
  admin: { ... },
  organizations: { ... },
};
```

#### 5. Files We CAN Modify (with care)
| File | How to Modify |
|------|---------------|
| `worker/database/schema.ts` | Add tables at END only |
| `worker/api/routes/index.ts` | Add imports & setup calls at END |
| `src/lib/api-client.ts` | Add methods at END |
| `src/App.tsx` | Add context providers (wrap existing) |
| `package.json` | Add dependencies (auto-merges usually) |
| `wrangler.jsonc` | Add bindings at END |

#### 6. Files We Should NOT Modify
- `worker/agents/*` - Core AI agent logic
- `worker/services/sandbox/*` - Sandbox service
- `src/routes/chat/*` - Chat UI
- `src/components/ui/*` - Base UI components
- Any existing upstream component/service

### Conflict Resolution Strategy
```bash
# If conflicts occur during merge:

# 1. Check which files have conflicts
git status

# 2. For each conflicted file:
#    - Keep UPSTREAM changes for their code sections
#    - Keep OUR changes for our custom sections
#    - If both modified same line, prefer upstream + re-add our code

# 3. After resolving
git add .
git commit -m "Merge upstream/main - resolved conflicts"
git push origin main
```

### Recommended Branch Strategy
```
main ─────────────────────────────────► Production
  │
  ├── feature/billing ──────► Merge to main when ready
  ├── feature/domains ──────► Merge to main when ready
  └── upstream-sync ────────► Temporary branch for merging upstream
```