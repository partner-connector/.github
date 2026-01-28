# Getting Started with Partner Connector

**New to Partner Connector?** This guide will get you productive in **under 3 hours**.

---

## What is Partner Connector?

Partner Connector is a **lead marketplace platform** where agencies buy and sell non-fit SQLs. It's built as a **4-repository system**:

1. **api** - NestJS backend (REST API, MongoDB, Redis)
2. **client** - Nuxt 4 frontend (Vue 3, Tailwind)
3. **partner-connector-hubspot-app** - HubSpot OAuth configuration
4. **.github** - Organization documentation

---

## Prerequisites (30 minutes)

Install these tools before starting:

### Required Tools

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **Node.js** | 22+ (API), 20.18.1+ (Client) | Runtime for both apps | [Download](https://nodejs.org/) |
| **pnpm** | 9+ | Package manager | `npm install -g pnpm` |
| **Docker** | Latest | Redis and local services | [Download](https://docker.com/) |
| **Git** | Latest | Version control | [Download](https://git-scm.com/) |

### Optional Tools

- **MongoDB Compass** - Database GUI ([Download](https://www.mongodb.com/products/compass))
- **Postman** - API testing ([Download](https://www.postman.com/))
- **VS Code** - Recommended editor ([Download](https://code.visualstudio.com/))

---

## Step 1: Clone All Repositories (10 minutes)

Create a workspace directory and clone all 4 repos:

```bash
# Create workspace
mkdir partner-connector-workspace
cd partner-connector-workspace

# Clone all repositories
git clone https://github.com/partner-connector/api.git
git clone https://github.com/partner-connector/client.git
git clone https://github.com/partner-connector/partner-connector-hubspot-app.git
git clone https://github.com/partner-connector/.github.git

# Verify all repos are present
ls -la
# Expected output:
# api/
# client/
# partner-connector-hubspot-app/
# .github/
```

**Result:** All 4 repositories in a single workspace folder.

---

## Step 2: Set Up API Backend (30 minutes)

The API is the core of the platform. Set it up first.

### 2.1 Install Dependencies

```bash
cd api
pnpm install
```

### 2.2 Configure Environment Variables

```bash
# Copy template
cp .env.example .env

# Edit .env with your values
nano .env
```

**Required Variables:**

```env
# Database (get from MongoDB Atlas)
MONGODB_URL=mongodb+srv://username:password@cluster.mongodb.net/partner-connector?retryWrites=true&w=majority

# Authentication
JWT_SECRET=your-super-secret-jwt-key-change-this
INITIAL_OWNER_EMAIL=admin@example.com
INITIAL_OWNER_PASSWORD=SecurePassword123!
INITIAL_OWNER_NAME=Admin User

# Redis (Docker will provide these)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis-password

# CORS (Client URL)
CORS_ORIGIN=http://localhost:8000

# HubSpot (get from HubSpot app settings)
HUBSPOT_CLIENT_ID=your-hubspot-client-id
HUBSPOT_CLIENT_SECRET=your-hubspot-client-secret
HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback
HUBSPOT_WEBHOOK_SECRET=your-webhook-secret

# Mailgun (optional for development)
MAILGUN_API_KEY=key-...
MAILGUN_DOMAIN=mg.example.com

# Stripe (optional for development)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

### 2.3 Start the API

```bash
# Option 1: With Docker (includes Redis)
pnpm start:dev

# Option 2: Local (requires separate Redis)
pnpm start:watch
```

### 2.4 Verify API is Running

```bash
# Health check
curl http://localhost:3000
# Expected: {"status":"ok"}

# Swagger docs
open http://localhost:3000/api/docs
```

**Result:** API running on port 3000, Swagger docs accessible.

---

## Step 3: Set Up Client Frontend (20 minutes)

The client is the user-facing web application.

### 3.1 Install Dependencies

```bash
cd ../client
pnpm install
```

### 3.2 Configure Environment Variables

```bash
# Copy template
cp .env.example .env

# Edit .env
nano .env
```

**Required Variables:**

```env
# API URL
NUXT_PUBLIC_API_URL=http://localhost:3000

# HubSpot OAuth (must match API .env)
NUXT_PUBLIC_HUBSPOT_CLIENT_ID=your-hubspot-client-id
NUXT_PUBLIC_HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback
```

### 3.3 Start the Client

```bash
pnpm dev
```

### 3.4 Verify Client is Running

```bash
# Open in browser
open http://localhost:8000
```

**Result:** Client running on port 8000, connected to API.

---

## Step 4: Set Up HubSpot App (Optional - 20 minutes)

Only needed if you're working on HubSpot integration.

### 4.1 Install HubSpot CLI

```bash
npm install -g @hubspot/cli
```

### 4.2 Authenticate with HubSpot

```bash
cd ../partner-connector-hubspot-app
hs auth
# Follow prompts to authenticate
```

### 4.3 Review App Configuration

```bash
# View app settings
cat src/app/app.json
```

**Key Settings to Verify:**
- `redirectUri` matches API `HUBSPOT_REDIRECT_URI`
- `scopes` includes `crm.objects.contacts.read` and `crm.objects.deals.read`

**Result:** HubSpot CLI authenticated and app configuration reviewed.

---

## Step 5: Run Your First Build (10 minutes)

Verify everything works together.

### 5.1 Sign In to Client

1. Open http://localhost:8000
2. Sign in with initial owner credentials:
   - Email: `admin@example.com` (from API .env)
   - Password: `SecurePassword123!` (from API .env)

### 5.2 Test API Connection

```bash
# Test authenticated endpoint
curl http://localhost:3000/v1/partners \
  -H "Cookie: access_token=YOUR_TOKEN"
```

### 5.3 Verify Database

```bash
# Check MongoDB for initial owner user
mongosh "YOUR_MONGODB_URL"
> use partner-connector
> db.users.findOne({ email: "admin@example.com" })
```

**Result:** Successfully signed in, API responding, database populated.

---

## Step 6: Make Your First Change (30 minutes)

Practice the development workflow with a simple task.

### Practice Task: Add a New API Endpoint

**Goal:** Add a `GET /health-detailed` endpoint to the API.

#### 6.1 Create the Endpoint

```typescript
// File: api/src/app.controller.ts
@Get('health-detailed')
getHealthDetailed() {
  return {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    environment: process.env.NODE_ENV || 'development',
  };
}
```

#### 6.2 Test the Endpoint

```bash
curl http://localhost:3000/health-detailed
```

**Expected Response:**
```json
{
  "status": "ok",
  "timestamp": "2026-01-28T12:00:00.000Z",
  "uptime": 123.456,
  "environment": "development"
}
```

#### 6.3 Write a Test

```typescript
// File: api/src/app.controller.spec.ts
it('should return detailed health status', () => {
  const result = appController.getHealthDetailed();
  expect(result).toHaveProperty('status', 'ok');
  expect(result).toHaveProperty('timestamp');
  expect(result).toHaveProperty('uptime');
  expect(result).toHaveProperty('environment');
});
```

#### 6.4 Run the Test

```bash
cd api
pnpm test app.controller.spec.ts
```

**Result:** New endpoint working, test passing, development workflow understood.

---

## Step 7: Understand the Codebase (30 minutes)

Read these key files to understand architecture:

### API (Backend)

| File | Purpose |
|------|---------|
| `api/CLAUDE.md` | Complete API development guide (read this first!) |
| `api/src/main.ts` | Application entry point |
| `api/src/app.module.ts` | Module structure and imports |
| `api/src/modules/leads/` | Example module (lead management) |

### Client (Frontend)

| File | Purpose |
|------|---------|
| `client/CLAUDE.md` | Complete client development guide |
| `client/app.vue` | Root component |
| `client/pages/` | Page components (file-based routing) |
| `client/composables/` | Reusable logic (API calls, auth) |

### Organization Docs

| File | Purpose |
|------|---------|
| `.github/IMPLEMENTATION_PLAN.md` | 6-sprint development roadmap |
| `.github/ARCHITECTURE.md` | System architecture |

**Reading Order:**
1. **Start with:** `api/CLAUDE.md` (comprehensive backend guide)
2. **Then read:** `client/CLAUDE.md` (comprehensive frontend guide)
3. **Finally read:** `.github/IMPLEMENTATION_PLAN.md` (overall project plan)

---

## Troubleshooting

### API Won't Start

**Problem:** "Cannot connect to MongoDB"

**Solution:**
1. Verify `MONGODB_URL` in `api/.env` is correct
2. Check MongoDB Atlas network access (allow your IP)
3. Test connection with MongoDB Compass

---

**Problem:** "Cannot connect to Redis"

**Solution:**
1. Verify Docker is running: `docker ps`
2. Check Redis container: `docker ps | grep redis`
3. Restart Redis: `pnpm start:dev` (in API directory)

---

### Client Won't Start

**Problem:** "Module not found: @/utils/api"

**Solution:**
1. Clear Nuxt cache: `rm -rf .nuxt node_modules/.cache`
2. Reinstall: `pnpm install`
3. Restart dev server: `pnpm dev`

---

**Problem:** "API request failed: CORS error"

**Solution:**
1. Verify `CORS_ORIGIN` in `api/.env` is `http://localhost:8000`
2. Restart API server
3. Clear browser cache

---

### HubSpot Integration

**Problem:** "OAuth redirect_uri_mismatch"

**Solution:**
1. Verify `HUBSPOT_REDIRECT_URI` matches in all 3 places:
   - HubSpot app `src/app/app.json`
   - API `.env`
   - Client `.env`
2. Must be exact: `http://localhost:3000/v1/integrations/hubspot/callback`

---

## Next Steps

Now that you're set up, choose your learning path:

### For Backend Developers

- Read [`api/CLAUDE.md`](https://github.com/partner-connector/api/blob/main/CLAUDE.md) thoroughly
- Explore the `api/src/modules/` directory
- Look at Sprint 1-13 implementation summaries in `api/docs/`
- Try adding a new API endpoint or service method

### For Frontend Developers

- Read [`client/CLAUDE.md`](https://github.com/partner-connector/client/blob/main/CLAUDE.md) thoroughly
- Explore the `client/pages/` and `client/components/` directories
- Review the design system in `.claude/frontend-system/`
- Try adding a new page or component

### For Full-Stack Developers

- Review the HubSpot OAuth integration flow
- Trace a webhook from HubSpot → API → Client
- Implement a new feature end-to-end (API endpoint + client page)

### For DevOps/Infrastructure

- Review `api/docs/DEPLOYMENT_RUNBOOK.md`
- Explore Kubernetes manifests in `api/k8s/`
- Study the CI/CD pipeline in `api/.github/workflows/`

---

## Resources

| Resource | Link |
|----------|------|
| **API Documentation** | [api/CLAUDE.md](https://github.com/partner-connector/api/blob/main/CLAUDE.md) |
| **Client Documentation** | [client/CLAUDE.md](https://github.com/partner-connector/client/blob/main/CLAUDE.md) |
| **Implementation Plan** | [.github/IMPLEMENTATION_PLAN.md](https://github.com/partner-connector/.github/blob/main/IMPLEMENTATION_PLAN.md) |
| **Architecture Guide** | [.github/ARCHITECTURE.md](https://github.com/partner-connector/.github/blob/main/ARCHITECTURE.md) |
| **HubSpot App Setup** | [partner-connector-hubspot-app/README.md](https://github.com/partner-connector/partner-connector-hubspot-app/blob/main/README.md) |

---

## Get Help

**Stuck? Have questions?**

1. **Check documentation first** - Most answers are in CLAUDE.md files
2. **Search existing issues** - Someone may have solved your problem
3. **Ask the team** - Use internal communication channels
4. **Create an issue** - Document the problem for others

---

**Congratulations!** You're now set up and ready to contribute to Partner Connector. 🎉
