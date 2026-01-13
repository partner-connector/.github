<div align="center">

# Partner Connector

![NestJS](https://img.shields.io/badge/NestJS-E0234E?logo=nestjs&logoColor=white)
![Vue](https://img.shields.io/badge/Vue_3-4FC08D?logo=vue.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?logo=mongodb&logoColor=white)
![Tailwind](https://img.shields.io/badge/Tailwind-06B6D4?logo=tailwindcss&logoColor=white)

**B2B Partner Management & Lead Tracking Platform**

</div>

---

## What is Partner Connector?

Partner Connector is an enterprise SaaS platform designed for managing business partner relationships and tracking leads through the sales pipeline. It integrates seamlessly with HubSpot CRM and ChiliPiper for meeting scheduling.

### Key Features

- **Partner Management** - Onboard and manage business partners with custom configurations
- **Lead Pipeline** - Track leads from initial contact to deal closure
- **CRM Integration** - Sync contacts and activities with HubSpot
- **Meeting Scheduling** - ChiliPiper webhook integration
- **Role-Based Access** - Granular permissions for teams
- **Bulk Operations** - Import/export leads via CSV

---

## Repositories

| Repository | Description | Tech Stack |
|------------|-------------|------------|
| [**api**](https://github.com/partner-connector/api) | Backend REST API | NestJS, MongoDB, Redis |
| [**client**](https://github.com/partner-connector/client) | Frontend SPA | Nuxt 4, Vue 3, Tailwind |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Partner Connector                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐                      ┌─────────────┐       │
│  │   Client    │◄────── HTTPS ──────►│     API     │       │
│  │  (Nuxt 4)   │                      │  (NestJS)   │       │
│  └─────────────┘                      └──────┬──────┘       │
│                                              │               │
│                          ┌───────────────────┼───────────┐  │
│                          ▼                   ▼           ▼  │
│                    ┌──────────┐       ┌───────────┐ ┌──────┐│
│                    │ MongoDB  │       │   Redis   │ │HubSpot│
│                    │ (Atlas)  │       │ (BullMQ)  │ │  API ││
│                    └──────────┘       └───────────┘ └──────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### Prerequisites
- Node.js 22+
- pnpm 9+
- Docker (for local Redis)

### Clone Both Repos

```bash
# Clone API
git clone https://github.com/partner-connector/api.git

# Clone Client
git clone https://github.com/partner-connector/client.git
```

### Run Locally

```bash
# Terminal 1: Start API (includes Redis via Docker)
cd api
pnpm install
pnpm start:dev

# Terminal 2: Start Client
cd client
pnpm install
pnpm dev
```

**Access the app at** http://localhost:8000

---

## Tech Stack

### Backend ([api](https://github.com/partner-connector/api))
- **Framework**: NestJS 10
- **Database**: MongoDB 8 + Mongoose
- **Queue**: Redis + BullMQ
- **Auth**: JWT with HTTP-only cookies
- **Docs**: Swagger/OpenAPI

### Frontend ([client](https://github.com/partner-connector/client))
- **Framework**: Nuxt 4 + Vue 3
- **Styling**: Tailwind CSS 4
- **Components**: shadcn-vue
- **Forms**: VeeValidate + Zod
- **Tables**: TanStack Table

---

## Documentation

Each repository contains:
- **README.md** - Quick start and overview
- **CLAUDE.md** - Detailed development guide for AI-assisted coding

---

<div align="center">

**Built by Belkins Group**

</div>
