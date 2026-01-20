<div align="center">

# Partner Connector

![Backend](https://img.shields.io/badge/Backend-v1.7.0-blue)
![Frontend](https://img.shields.io/badge/Frontend-v1.14.0-blue)
![Build](https://img.shields.io/badge/Build-Passing-success)
![Tests](https://img.shields.io/badge/Tests-17%2F17-success)
![NestJS](https://img.shields.io/badge/NestJS-10-E0234E?logo=nestjs&logoColor=white)
![Nuxt](https://img.shields.io/badge/Nuxt-4-00DC82?logo=nuxt.js&logoColor=white)
![Vue](https://img.shields.io/badge/Vue-3-4FC08D?logo=vue.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-8-47A248?logo=mongodb&logoColor=white)
![Tailwind](https://img.shields.io/badge/Tailwind-4-06B6D4?logo=tailwindcss&logoColor=white)
![pnpm](https://img.shields.io/badge/pnpm-9-F69220?logo=pnpm&logoColor=white)

**Enterprise B2B SaaS Platform for Partner Management & Lead Tracking**

[Features](#key-features) · [Quick Start](#quick-start) · [Architecture](#architecture) · [Roadmap](#development-roadmap) · [Documentation](#documentation)

Built by [Belkins Group](https://belkins.io/)

</div>

---

## Overview

Partner Connector is a comprehensive B2B SaaS platform designed for managing business partner relationships and tracking leads through the sales pipeline. It provides a complete solution for partner onboarding, lead management, CRM integration, and meeting scheduling with enterprise-grade security and scalability.

### What Makes Partner Connector Special?

- **Full-Stack TypeScript** - End-to-end type safety from database to UI
- **Modern Tech Stack** - Built with Nuxt 4, NestJS 10, and the latest web technologies
- **CRM Integration** - Seamless HubSpot synchronization for contacts and activities
- **Real-Time Updates** - Background job processing with BullMQ for async operations
- **Role-Based Access** - Hierarchical permissions (Owner → Admin → User)
- **Enterprise Ready** - Production-tested with comprehensive error handling and monitoring

---

## Project Structure

```
Partner Connector/
├── api-main/           # NestJS Backend (v1.7.0)
│   ├── src/
│   │   ├── modules/    # Feature modules (auth, partners, leads, meetings)
│   │   ├── common/     # Guards, decorators, integrations
│   │   └── static/     # Constants, enums, collections
│   ├── README.md       # Backend quick start & overview
│   └── CLAUDE.md       # Backend development guide
│
├── client-main/        # Nuxt 4 Frontend (v1.14.0)
│   ├── app/
│   │   ├── pages/      # File-based routing
│   │   ├── components/ # UI components (shadcn-ui)
│   │   ├── stores/     # State management
│   │   └── gateway/    # API client layer
│   ├── README.md       # Frontend quick start & overview
│   └── CLAUDE.md       # Frontend development guide
│
└── .github/
    ├── README.md       # This file - project overview
    └── IMPLEMENTATION_PLAN.md  # 6-sprint development roadmap
```

**Each repository contains:**
- **README.md** - Quick start guide, installation, and API/component reference
- **CLAUDE.md** - Detailed development patterns, conventions, and best practices

---

## Architecture

### Full-Stack Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Partner Connector Platform                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐           HTTP/JWT           ┌──────────────────┐
│  │   Client App     │◄────────────────────────────►│   API Server     │
│  │   (Nuxt 4)       │    Cookies (HttpOnly)        │   (NestJS 10)    │
│  │   Vue 3 + TS     │                              │   TypeScript     │
│  │   localhost:8000 │                              │   localhost:3000 │
│  └──────────────────┘                              └────────┬─────────┘
│                                                              │
│                          ┌───────────────────────────────────┼─────────────┐
│                          ▼                   ▼               ▼             ▼
│                    ┌──────────┐       ┌───────────┐   ┌──────────┐  ┌─────────┐
│                    │ MongoDB  │       │   Redis   │   │ HubSpot  │  │Mailgun  │
│                    │ (Atlas)  │       │ (BullMQ)  │   │   CRM    │  │  Email  │
│                    │ 8 Colls. │       │ 5 Queues  │   │   API    │  │  API    │
│                    └──────────┘       └───────────┘   └──────────┘  └─────────┘
│                                                              │
│                                      ┌───────────────────────┤
│                                      ▼                       ▼
│                                ┌──────────┐           ┌──────────┐
│                                │ChiliPiper│           │  Stripe  │
│                                │ Meetings │           │ Billing  │
│                                └──────────┘           └──────────┘
│                                                       (Sprint 4 - In Progress)
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

<table>
<tr>
<th>Layer</th>
<th>Backend (api-main)</th>
<th>Frontend (client-main)</th>
</tr>
<tr>
<td><strong>Framework</strong></td>
<td>NestJS 10, Express</td>
<td>Nuxt 4, Vue 3</td>
</tr>
<tr>
<td><strong>Language</strong></td>
<td>TypeScript 4.7</td>
<td>TypeScript 5.8.3</td>
</tr>
<tr>
<td><strong>Database</strong></td>
<td>MongoDB 8 (Mongoose ODM)</td>
<td>N/A (API client)</td>
</tr>
<tr>
<td><strong>Caching/Jobs</strong></td>
<td>Redis 7, BullMQ</td>
<td>N/A</td>
</tr>
<tr>
<td><strong>Styling</strong></td>
<td>N/A</td>
<td>Tailwind CSS 4</td>
</tr>
<tr>
<td><strong>UI Components</strong></td>
<td>N/A</td>
<td>shadcn-ui (reka-ui primitives)</td>
</tr>
<tr>
<td><strong>Forms</strong></td>
<td>class-validator</td>
<td>VeeValidate + Zod</td>
</tr>
<tr>
<td><strong>Tables</strong></td>
<td>N/A</td>
<td>TanStack Table</td>
</tr>
<tr>
<td><strong>Auth</strong></td>
<td>JWT (access + refresh tokens)</td>
<td>HTTP-only cookies</td>
</tr>
<tr>
<td><strong>API Docs</strong></td>
<td>Swagger/OpenAPI</td>
<td>N/A</td>
</tr>
<tr>
<td><strong>Package Manager</strong></td>
<td colspan="2">pnpm 9</td>
</tr>
</table>

### Integrations

| Service | Purpose | Repository |
|---------|---------|------------|
| **HubSpot CRM** | Contact synchronization, property mapping | Backend |
| **ChiliPiper** | Meeting scheduling webhooks | Backend |
| **Mailgun** | Transactional emails (password reset, notifications) | Backend |
| **Stripe** | Invoice generation, payment webhooks (Sprint 4) | Backend |

---

## Quick Start

### Prerequisites

- **Node.js** 22+ (Backend) / 20.18.1+ (Frontend)
- **pnpm** 9+
- **Docker** (for Redis, recommended)
- **MongoDB Atlas** account (or local MongoDB)

### Installation & Running

**Terminal 1: Start Backend API (with Docker for Redis)**

```bash
cd api-main
pnpm install

# Copy environment template
cp .env.example .env

# Configure environment variables (see api-main/README.md)
# MONGODB_URL, JWT_SECRET, REDIS_HOST, etc.

# Start API with Docker (includes Redis)
pnpm start:dev
```

The API server will be available at **http://localhost:3000**

- **Swagger Docs**: http://localhost:3000/api/docs
- **Health Check**: http://localhost:3000

**Terminal 2: Start Frontend Client**

```bash
cd client-main
pnpm install

# Copy environment template
cp .env.example .env.dev

# .env.dev should contain:
# NUXT_PUBLIC_API_URL=http://localhost:3000

# Start dev server
pnpm dev
```

The client app will be available at **http://localhost:8000**

### Default Login Credentials (Development)

```
Email: hs.dev@belkins.io
Password: test2020
```

---

## Key Features

### Partner Management
- **Partner Onboarding** - Create partners with custom configurations
- **Billing Configuration** - Set billing terms per partner
- **ICP Tracking** - Ideal Customer Profile display with badge + popover
- **Last Appointment** - Track most recent meeting across all partner leads
- **Partner Stats** - Admin-only endpoint for analytics (`GET /partners/:id/stats`)

### Lead Pipeline Tracking
- **Advanced Data Table** - Sorting, filtering, pagination with TanStack Table
- **Bulk Import** - CSV/Excel upload with validation
- **Lead Notes** - Timestamped notes with user tracking
- **Deal Status Flow** - Track leads from NEW → SCHEDULED → IN_TOUCH → WON/LOST
- **HubSpot Sync** - Automatic contact synchronization from CRM

### Meeting Management
- **ChiliPiper Integration** - Webhook-based meeting creation
- **Real-Time Sync** - Meeting → Lead sync with `scheduledDate` field
- **Meeting History** - Track all appointments for partners and leads

### User Management & Access Control
- **Role-Based Access** - Hierarchical permissions (Owner > Admin > User)
- **Active/Inactive Status** - Toggle user access with auth blocking
- **Partner Assignment** - Scope users to specific partners
- **Invite System** - Email-based user invitations

### Email & Notifications
- **Mailgun Integration** - Transactional email service with Handlebars templates
- **Password Reset** - Complete forgot/reset password flow
- **Daily Reminders** - Cron job for scheduled lead reminders with magic links
- **Lost Lead Alerts** - Async email alerts for lost leads (non-blocking)

### Background Processing
- **BullMQ Queues** - 5 queues for async job processing
  - `hubspot-webhook` - HubSpot event processing
  - `chilipiper-webhook` - Meeting event processing
  - `events` - Audit event logging
  - `sync-leads` - Bulk contact synchronization
  - `partner-api-lead` - Partner API status updates

### Public Partner API
- **Token Authentication** - Bearer token-based API for partners
- **Lead Status Updates** - External partners can update lead statuses
- **Webhook Support** - Event notifications for partners

---

## Development Roadmap

### 📋 Implementation Plan (6 Sprints, 8-12 Weeks)

**[View Full Implementation Plan →](./IMPLEMENTATION_PLAN.md)**

Our development follows a structured 6-sprint plan with clear deliverables for each phase. The plan includes detailed task breakdowns, code examples, and acceptance criteria.

### Sprint Progress Overview

<table>
<tr>
<th>Sprint</th>
<th>Focus Area</th>
<th>Backend Status</th>
<th>Frontend Status</th>
</tr>
<tr>
<td><strong>Sprint 1</strong></td>
<td>Data Foundation & Core Sync</td>
<td>✅ <strong>COMPLETE</strong><br/>scheduledDate field, meeting sync, database migration</td>
<td>N/A</td>
</tr>
<tr>
<td><strong>Sprint 2</strong></td>
<td>Email & Notifications</td>
<td>✅ <strong>COMPLETE</strong><br/>Mailgun service, password reset backend, cron jobs</td>
<td>✅ <strong>COMPLETE</strong><br/>Password reset pages</td>
</tr>
<tr>
<td><strong>Sprint 3</strong></td>
<td>User Management & Admin</td>
<td>✅ <strong>COMPLETE</strong><br/>Partner stats endpoint, user active/inactive status</td>
<td>✅ <strong>COMPLETE</strong><br/>ICP & Last Appointment columns, Switch component, user toggles</td>
</tr>
<tr>
<td><strong>Sprint 4</strong></td>
<td>Stripe Integration</td>
<td>🔄 <strong>IN PROGRESS</strong><br/>Stripe service, invoice generation, webhooks</td>
<td>📋 <strong>PLANNED</strong><br/>Admin finance UI, invoice table, manual controls</td>
</tr>
<tr>
<td><strong>Sprint 5</strong></td>
<td>Frontend Pages & UX</td>
<td>N/A</td>
<td>📋 <strong>PLANNED</strong><br/>Appointments page, advanced filtering, UI polish</td>
</tr>
<tr>
<td><strong>Sprint 6</strong></td>
<td>Bug Fixes & Technical Debt</td>
<td>✅ <strong>PARTIALLY COMPLETE</strong><br/>17 TypeScript errors fixed, 17 tests added</td>
<td>✅ <strong>PARTIALLY COMPLETE</strong><br/>Build protocol established, package management rules</td>
</tr>
</table>

**Key Milestones:**
- ✅ Sprint 1-3: **Core platform features complete** (January 2026)
- 🔄 Sprint 4: **Financial integrations in progress** (Stripe)
- 📋 Sprint 5-6: **UX polish and stability improvements planned**

---

## Recent Updates (January 2026)

### 🎯 Sprint 1-3 Completion

**Major Achievements Across Both Repositories:**

#### Backend Improvements (v1.7.0)

**Critical Bug Fixes:**
- ✅ **Meeting Sync Bug** - Fixed incorrect query field (`meeting._id` → `meeting.lead`)
  - Created new `getManyByLeadIds()` repository method
  - Added 6 comprehensive unit tests to prevent regression
  - Now correctly syncs `scheduledDate` from meetings to leads

**TypeScript Quality:**
- ✅ Fixed all **17 TypeScript build errors** - zero errors now
- ✅ Eliminated `any` types from critical code paths
- ✅ Added proper null checks and type guards
- ✅ Build status: **PASSING** ✅

**Test Coverage:**
- ✅ Added **17 comprehensive unit tests** (100% pass rate)
  - 6 tests for Meetings Service (critical bug fix verification)
  - 11 tests for Mailgun Service (email functionality coverage)
- ✅ Jest configuration with path alias support
- ✅ Test runtime: ~2 seconds

**New Features:**
- ✅ Partner stats endpoint (`GET /partners/:id/stats`)
  - Returns `lastAppointmentDate` and `totalLeads`
  - Admin-only access with RouteGuard
  - Used by frontend for Last Appointment column
- ✅ User active/inactive status management
- ✅ Mailgun email service with 3 Handlebars templates
- ✅ HTTP-based cron jobs with CronAuthGuard
- ✅ Daily lead reminder cron job with magic links
- ✅ Lost lead email alerts (async, non-blocking)

**Environment Configuration:**
- ✅ All production credentials configured
- ✅ MongoDB Atlas connection string set
- ✅ Mailgun API key configured
- ✅ Secure secrets generated (MAGIC_LINK_SECRET, CRON_SECRET)

#### Frontend Improvements (v1.14.0)

**New Components:**
- ✅ `ICPCell.vue` - Badge + popover display for Ideal Customer Profile
- ✅ `LastAppointmentCell.vue` - Relative time display for appointments
- ✅ Switch component added to shadcn-ui library (using reka-ui)

**Partners Table Enhancements:**
- ✅ ICP column with badge display
- ✅ Last Appointment column with relative time formatting
- ✅ Parallel stats fetching for performance
- ✅ Admin-only column visibility

**User Management:**
- ✅ Active/inactive status toggle in Users table
- ✅ Switch component integration
- ✅ Auth blocking for inactive users

**Development Rules Established:**
- ✅ **Build-before-deploy protocol** - Always test `pnpm build` before committing
- ✅ **Package management rules** - Never downgrade, always use latest versions
- ✅ **Configuration stability** - Don't modify working configs without approval
- ✅ **Reference commit** - Commit `8420119` contains verified working package configuration

**Quality Metrics:**
- ✅ Build status: **PASSING** ✅
- ✅ Zero build errors
- ✅ All packages at latest stable versions
- ✅ Node.js version documented (v20.18.1+ required, v20.9.0 has local issues)

### 📊 Combined Quality Metrics

<table>
<tr>
<th>Metric</th>
<th>Backend</th>
<th>Frontend</th>
</tr>
<tr>
<td><strong>Build Status</strong></td>
<td>✅ PASSING (0 TypeScript errors)</td>
<td>✅ PASSING (0 build errors)</td>
</tr>
<tr>
<td><strong>Test Coverage</strong></td>
<td>17/17 tests passing (100%)</td>
<td>Build validation established</td>
</tr>
<tr>
<td><strong>Test Runtime</strong></td>
<td>~2 seconds</td>
<td>N/A</td>
</tr>
<tr>
<td><strong>Type Safety</strong></td>
<td>All critical paths typed</td>
<td>Full TypeScript coverage</td>
</tr>
<tr>
<td><strong>Documentation</strong></td>
<td>Comprehensive README + CLAUDE.md</td>
<td>Comprehensive README + CLAUDE.md</td>
</tr>
</table>

### 🎓 Lessons Learned

**Critical Development Rules (January 20, 2026):**

1. **Always Test Build Before Deploy**
   - Run `pnpm build` before every commit
   - Vercel deployments fail if build fails
   - Local testing catches issues early

2. **Never Downgrade Packages**
   - Always use latest versions
   - Downgrading breaks compatibility
   - Reference commit `8420119` for working state

3. **Never Modify Working Configs**
   - If `nuxt.config.ts`, `tailwind.config.js`, `tsconfig.json` work, don't touch them
   - Configuration changes require thorough testing
   - Always get explicit approval for config changes

4. **Verify Database Queries**
   - Wrong field selection = silent bugs (empty results)
   - Write unit tests immediately for critical queries
   - Test with real data to catch edge cases

5. **Component Library Awareness**
   - Frontend uses **reka-ui** (NOT radix-vue) for shadcn components
   - Follow existing component patterns
   - Don't mix primitive libraries

---

## Documentation

### Repository-Specific Documentation

Each repository contains detailed documentation for its specific domain:

#### Backend (api-main/)
- **[README.md](./api-main/README.md)** - API quick start, installation, endpoints, testing
  - Tech stack and architecture
  - Environment variables
  - Deployment guides (Docker, production)
  - Testing coverage breakdown
  - Sprint status for backend tasks
- **[CLAUDE.md](./api-main/CLAUDE.md)** - Backend development guide
  - Module structure and patterns
  - Authentication and authorization
  - Database schemas and collections
  - Queue system (BullMQ)
  - External integrations (HubSpot, ChiliPiper, Mailgun)
  - Debugging and troubleshooting
  - Code quality standards

**Interactive API Documentation:**
- [Swagger UI](http://localhost:3000/api/docs) - Full API reference with try-it-now functionality

#### Frontend (client-main/)
- **[README.md](./client-main/README.md)** - Client quick start, installation, architecture
  - Tech stack and features
  - Build and deployment (Vercel)
  - Environment configuration
  - Package management rules
  - Sprint status for frontend tasks
  - Node.js version requirements
- **[CLAUDE.md](./client-main/CLAUDE.md)** - Frontend development guide
  - Component library (shadcn-ui with reka-ui)
  - State management patterns
  - API gateway architecture
  - Routing and middleware
  - Forms and validation (VeeValidate + Zod)
  - Styling with Tailwind CSS 4
  - Critical development rules
  - Package audit and verification

#### Project-Wide Documentation
- **[IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md)** - 6-sprint development roadmap
  - Detailed task breakdowns for each sprint
  - Code examples and implementation patterns
  - Acceptance criteria and quality standards
  - Cross-repository feature coordination
  - Testing and deployment strategies

---

<div align="center">

**[⬆ Back to Top](#partner-connector)**

**Current Versions:** Backend v1.7.0 · Frontend v1.14.0

**Status:** ✅ Sprints 1-3 Complete · 🔄 Sprint 4 In Progress

Built with [NestJS](https://nestjs.com/) + [Nuxt](https://nuxt.com/) · Powered by [MongoDB](https://www.mongodb.com/) + [Redis](https://redis.io/) · Styled with [Tailwind CSS](https://tailwindcss.com/)

</div>
