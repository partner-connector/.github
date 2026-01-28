# Environment Variables Reference

**Last Updated:** January 28, 2026
**Version:** 1.0
**Status:** Production

---

## Table of Contents

1. [Overview](#overview)
2. [API Repository Variables](#api-repository-variables)
3. [Client Repository Variables](#client-repository-variables)
4. [HubSpot App Configuration](#hubspot-app-configuration)
5. [Shared Variables](#shared-variables)
6. [Environment-Specific Values](#environment-specific-values)
7. [Setup Instructions](#setup-instructions)

---

## Overview

Partner Connector uses environment variables for configuration across 3 repositories:

| Repository | Config Method | File Location |
|------------|---------------|---------------|
| **API** | `.env` file | `api/.env` |
| **Client** | `.env` file | `client/.env` |
| **HubSpot App** | JSON config | `partner-connector-hubspot-app/src/app/app.json` |

**Note:** `.env` files should NEVER be committed to git. Use `.env.example` as templates.

---

## API Repository Variables

**Location:** `api/.env`

### Database

```env
# MongoDB connection string (required)
MONGODB_URL=mongodb+srv://username:password@cluster.mongodb.net/partner-connector?retryWrites=true&w=majority
```

**Source:** MongoDB Atlas dashboard → Connect → Drivers

---

### Redis (Queue & Cache)

```env
# Redis connection (required for BullMQ)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis-password
```

**Local Development:** Docker provides these (see `docker-compose.yml`)
**Production:** DigitalOcean Managed Redis or StatefulSet

---

### Authentication

```env
# JWT secret for token signing (required)
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production

# Initial owner account (created on first startup)
INITIAL_OWNER_EMAIL=admin@example.com
INITIAL_OWNER_PASSWORD=SecurePassword123!
INITIAL_OWNER_NAME=Admin User
```

**Security:**
- Generate `JWT_SECRET` with: `openssl rand -hex 32`
- Use strong password for initial owner (16+ chars, mixed case, numbers, symbols)

---

### CORS

```env
# Allowed origins for CORS (comma-separated)
CORS_ORIGIN=http://localhost:8000,https://app.partners.belkins.io
```

**Local:** `http://localhost:8000` (Client dev server)
**Staging:** `https://staging-app.partners.belkins.io`
**Production:** `https://app.partners.belkins.io`

---

### HubSpot Integration (Marketplace OAuth - Integration 2)

> **Note**: These variables are for **Integration 2 (Marketplace OAuth)** only.
> Integration 1 (ChiliPiper routing) does not require OAuth credentials.

```env
# OAuth credentials (required for marketplace partners to connect their HubSpot)
HUBSPOT_CLIENT_ID=12345678-1234-1234-1234-123456789012
HUBSPOT_CLIENT_SECRET=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# OAuth redirect URI (must match HubSpot app configuration)
HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback

# Webhook signature secret (required for webhook validation)
HUBSPOT_WEBHOOK_SECRET=a1b2c3d4e5f6...
```

**Source:** HubSpot Developer Dashboard → Applications → Your App
**See also:** [OAUTH_INTEGRATION_GUIDE.md](./OAUTH_INTEGRATION_GUIDE.md)

---

### Stripe Payment Processing

```env
# Stripe API keys (required for billing)
STRIPE_SECRET_KEY=sk_test_... # Test key
STRIPE_SECRET_KEY=sk_live_... # Production key

# Webhook endpoint secret (for signature validation)
STRIPE_WEBHOOK_SECRET=whsec_...
```

**Source:** Stripe Dashboard → Developers → API Keys
**Webhook Secret:** Stripe Dashboard → Developers → Webhooks → Add endpoint

---

### Email Service (Mailgun)

```env
# Mailgun credentials (required for transactional emails)
MAILGUN_API_KEY=key-...
MAILGUN_DOMAIN=mg.partners.belkins.io
MAILGUN_FROM_EMAIL=noreply@partners.belkins.io
MAILGUN_FROM_NAME=Partner Connector
```

**Source:** Mailgun Dashboard → Sending → Domain Settings

---

### Error Tracking (BetterStack/Sentry)

```env
# Sentry DSN (optional - production only)
SENTRY_DSN=https://{token}@logs.betterstack.com/1

# Application version (for release tracking)
APP_VERSION=6.4.0
```

**Source:** BetterStack → Sources → Add Source
**See also:** `docs/BETTERSTACK_SETUP.md` in API repo

---

### Server Configuration

```env
# Server port (default: 3000)
PORT=3000

# Node environment
NODE_ENV=development # development | staging | production

# Client URL (for redirects after OAuth)
CLIENT_URL=http://localhost:8000
```

---

### Cron Security

```env
# Secret for authenticating cron jobs
CRON_SECRET=your-random-cron-secret
```

**Generate with:** `openssl rand -hex 32`
**Used by:** Kubernetes CronJobs for `/cron/*` endpoints

---

## Client Repository Variables

**Location:** `client/.env`

### API Connection

```env
# API base URL (required)
NUXT_PUBLIC_API_URL=http://localhost:3000
```

**Local:** `http://localhost:3000`
**Staging:** `https://staging-api.partners.belkins.io`
**Production:** `https://api.partners.belkins.io`

---

### HubSpot OAuth

```env
# HubSpot OAuth client ID (must match API)
NUXT_PUBLIC_HUBSPOT_CLIENT_ID=12345678-1234-1234-1234-123456789012

# OAuth redirect URI (must match API and HubSpot app)
NUXT_PUBLIC_HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback
```

**Note:** Must use `NUXT_PUBLIC_` prefix to expose to browser

---

### Application Metadata

```env
# Application version (optional)
NUXT_PUBLIC_APP_VERSION=1.2.0

# Environment name (for display)
NUXT_PUBLIC_ENVIRONMENT=development
```

---

## HubSpot App Configuration

**Location:** `partner-connector-hubspot-app/src/app/app.json`

**Note:** HubSpot app uses JSON configuration, NOT environment variables

### OAuth Configuration

```json
{
  "scopes": {
    "crm": [
      "crm.objects.contacts.read",
      "crm.objects.contacts.write",
      "crm.objects.deals.read",
      "crm.objects.deals.write",
      "crm.schemas.contacts.read",
      "crm.schemas.deals.read"
    ]
  },
  "redirectUrls": [
    "http://localhost:3000/v1/integrations/hubspot/callback",
    "https://staging-api.partners.belkins.io/v1/integrations/hubspot/callback",
    "https://api.partners.belkins.io/v1/integrations/hubspot/callback"
  ]
}
```

---

### Webhook Configuration

```json
{
  "webhooks": {
    "targetUrl": "https://api.partners.belkins.io/v1/leads/webhooks/hubspot",
    "subscriptions": [
      {
        "subscriptionType": "contact.propertyChange",
        "propertyName": "*"
      },
      {
        "subscriptionType": "deal.propertyChange",
        "propertyName": "*"
      }
    ]
  }
}
```

**See also:** [WEBHOOK_FLOWS.md](./WEBHOOK_FLOWS.md)

---

## Shared Variables

These variables must have **identical values** across multiple repositories:

### HubSpot OAuth Credentials

| Variable | API Location | Client Location | HubSpot App Location |
|----------|--------------|-----------------|---------------------|
| **Client ID** | `HUBSPOT_CLIENT_ID` | `NUXT_PUBLIC_HUBSPOT_CLIENT_ID` | Generated by HubSpot |
| **Client Secret** | `HUBSPOT_CLIENT_SECRET` | ❌ NOT exposed to client | N/A |
| **Redirect URI** | `HUBSPOT_REDIRECT_URI` | `NUXT_PUBLIC_HUBSPOT_REDIRECT_URI` | `redirectUrls[]` in app.json |

**Critical:** All 3 values of Redirect URI must match EXACTLY:
```
✅ http://localhost:3000/v1/integrations/hubspot/callback
❌ http://localhost:3000/v1/integrations/hubspot/callback/  (trailing slash)
❌ https://localhost:3000/v1/integrations/hubspot/callback  (https vs http)
```

---

### HubSpot Webhook Secret

| Variable | API Location | Source |
|----------|--------------|--------|
| **Webhook Secret** | `HUBSPOT_WEBHOOK_SECRET` | HubSpot Dashboard → Webhooks |

**Used for:** Validating webhook signatures (see `HubSpotWebhookGuard`)

---

## Environment-Specific Values

### Local Development

**API `.env`:**
```env
MONGODB_URL=mongodb://localhost:27017/partner-connector
REDIS_HOST=localhost
REDIS_PORT=6379
CORS_ORIGIN=http://localhost:8000
HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback
STRIPE_SECRET_KEY=sk_test_...
CLIENT_URL=http://localhost:8000
NODE_ENV=development
```

**Client `.env`:**
```env
NUXT_PUBLIC_API_URL=http://localhost:3000
NUXT_PUBLIC_HUBSPOT_REDIRECT_URI=http://localhost:3000/v1/integrations/hubspot/callback
NUXT_PUBLIC_ENVIRONMENT=development
```

---

### Staging Environment

**API `.env` (Kubernetes secret):**
```env
MONGODB_URL=mongodb+srv://...@cluster-staging.mongodb.net/partner-connector-staging
REDIS_HOST=redis-staging.partner-connector-staging.svc.cluster.local
CORS_ORIGIN=https://staging-app.partners.belkins.io
HUBSPOT_REDIRECT_URI=https://staging-api.partners.belkins.io/v1/integrations/hubspot/callback
STRIPE_SECRET_KEY=sk_test_...
CLIENT_URL=https://staging-app.partners.belkins.io
NODE_ENV=staging
```

**Client `.env`:**
```env
NUXT_PUBLIC_API_URL=https://staging-api.partners.belkins.io
NUXT_PUBLIC_HUBSPOT_REDIRECT_URI=https://staging-api.partners.belkins.io/v1/integrations/hubspot/callback
NUXT_PUBLIC_ENVIRONMENT=staging
```

**HubSpot App `app.json`:**
```json
{
  "redirectUrls": [
    "https://staging-api.partners.belkins.io/v1/integrations/hubspot/callback"
  ],
  "webhooks": {
    "targetUrl": "https://staging-api.partners.belkins.io/v1/leads/webhooks/hubspot"
  }
}
```

---

### Production Environment

**API `.env` (Kubernetes secret):**
```env
MONGODB_URL=mongodb+srv://...@cluster-production.mongodb.net/partner-connector
REDIS_HOST=redis.partner-connector.svc.cluster.local
CORS_ORIGIN=https://app.partners.belkins.io
HUBSPOT_REDIRECT_URI=https://api.partners.belkins.io/v1/integrations/hubspot/callback
STRIPE_SECRET_KEY=sk_live_...
CLIENT_URL=https://app.partners.belkins.io
NODE_ENV=production
SENTRY_DSN=https://...@logs.betterstack.com/1
```

**Client `.env`:**
```env
NUXT_PUBLIC_API_URL=https://api.partners.belkins.io
NUXT_PUBLIC_HUBSPOT_REDIRECT_URI=https://api.partners.belkins.io/v1/integrations/hubspot/callback
NUXT_PUBLIC_ENVIRONMENT=production
```

**HubSpot App `app.json`:**
```json
{
  "redirectUrls": [
    "https://api.partners.belkins.io/v1/integrations/hubspot/callback"
  ],
  "webhooks": {
    "targetUrl": "https://api.partners.belkins.io/v1/leads/webhooks/hubspot"
  }
}
```

---

## Setup Instructions

### Step 1: Copy Templates

```bash
# API
cd api
cp .env.example .env

# Client
cd ../client
cp .env.example .env
```

---

### Step 2: Fill in Required Values

**Minimum required for local development:**

**API `.env`:**
```env
MONGODB_URL=mongodb://localhost:27017/partner-connector
REDIS_HOST=localhost
REDIS_PORT=6379
JWT_SECRET=development-secret-change-in-production
INITIAL_OWNER_EMAIL=admin@example.com
INITIAL_OWNER_PASSWORD=DevPassword123!
INITIAL_OWNER_NAME=Admin User
CORS_ORIGIN=http://localhost:8000
```

**Client `.env`:**
```env
NUXT_PUBLIC_API_URL=http://localhost:3000
```

---

### Step 3: Add Integration Credentials (Optional)

If testing HubSpot integration:

1. Create HubSpot developer account
2. Create test app in HubSpot Developer Dashboard
3. Copy `client_id` and `client_secret` to API `.env`
4. Copy `client_id` to Client `.env`
5. Configure redirect URIs in HubSpot app settings

If testing Stripe payments:

1. Create Stripe account
2. Get test API keys from Dashboard
3. Copy `sk_test_...` to API `.env`

---

### Step 4: Verify Configuration

```bash
# Start API
cd api
pnpm start:dev

# Check for errors in logs
# Look for: "Successfully connected to MongoDB"
#           "Redis connection established"

# Start Client
cd ../client
pnpm dev

# Check browser console for API connection errors
```

---

## Security Best Practices

### 1. Never Commit `.env` Files

```bash
# Verify .env is in .gitignore
cat .gitignore | grep .env

# Expected output:
# .env
# .env.local
# .env.*.local
```

---

### 2. Use Strong Secrets

```bash
# Generate secure random secrets
openssl rand -hex 32  # For JWT_SECRET, CRON_SECRET, etc.
openssl rand -base64 48  # Alternative format
```

---

### 3. Rotate Secrets Regularly

**Recommended rotation schedule:**
- **JWT_SECRET:** Every 90 days (requires re-login for all users)
- **CRON_SECRET:** Every 90 days
- **STRIPE_WEBHOOK_SECRET:** When compromised
- **HUBSPOT_WEBHOOK_SECRET:** When compromised

---

### 4. Use Different Secrets Per Environment

❌ **Don't:**
```env
# Same secret in all environments
JWT_SECRET=my-secret-key  # Used everywhere
```

✅ **Do:**
```env
# Local
JWT_SECRET=local-dev-secret

# Staging
JWT_SECRET=staging-d8f7a6b5c4e3f2g1

# Production
JWT_SECRET=prod-9a8b7c6d5e4f3a2b1c0d
```

---

### 5. Limit Access to Production Secrets

**Kubernetes Secrets:**
```bash
# Only DevOps team should have access
kubectl get secrets -n partner-connector

# View secret (requires cluster admin)
kubectl get secret partner-connector-secrets -o yaml
```

**Recommended:** Use secret management tools like:
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Doppler

---

## Troubleshooting

### Error: "MONGODB_URL is not defined"

**Cause:** Missing or incorrect `.env` file

**Solution:**
```bash
cd api
ls -la .env  # Verify file exists
cat .env | grep MONGODB_URL  # Verify variable set

# If missing, copy from template
cp .env.example .env
nano .env  # Edit and add MONGODB_URL
```

---

### Error: "redirect_uri_mismatch"

**Cause:** Mismatch between API, Client, and HubSpot app redirect URIs

**Solution:**
```bash
# API
cat api/.env | grep HUBSPOT_REDIRECT_URI

# Client
cat client/.env | grep HUBSPOT_REDIRECT_URI

# HubSpot App
cat partner-connector-hubspot-app/src/app/app.json | grep redirectUrls

# All 3 must match EXACTLY (including protocol, port, trailing slashes)
```

---

### Error: "Invalid webhook signature"

**Cause:** Wrong `HUBSPOT_WEBHOOK_SECRET` in API `.env`

**Solution:**
```bash
# Get correct secret from HubSpot Dashboard
# 1. Go to: https://app.hubspot.com/developer/{accountId}/applications
# 2. Select your app
# 3. Navigate to: Webhooks → Webhook Secret
# 4. Copy secret

# Update API .env
nano api/.env
# Set: HUBSPOT_WEBHOOK_SECRET=<copied-secret>

# Restart API
cd api && pnpm start:dev
```

---

## Next Steps

### For New Developers
1. Follow [GETTING_STARTED.md](./GETTING_STARTED.md) - Complete setup guide
2. Review [OAUTH_INTEGRATION_GUIDE.md](./OAUTH_INTEGRATION_GUIDE.md) - HubSpot OAuth setup
3. Review [ARCHITECTURE.md](./ARCHITECTURE.md) - System overview

### For DevOps
1. Review [Deployment Runbook](https://github.com/partner-connector/api/blob/main/docs/DEPLOYMENT_RUNBOOK.md)
2. Set up Kubernetes secrets for staging/production
3. Configure secret rotation policies

---

**Last Updated:** January 28, 2026 | **Next Review:** February 28, 2026
