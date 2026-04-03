# Cashual - Real-Time Social Communication Platform

## Overview

Cashual is a real-time peer-to-peer communication platform that combines spontaneous voice calls, text chat, friend connections, and a gamified reward system. Users are randomly matched 1-on-1 based on shared interests, can rate each other after conversations, and earn points toward leaderboard rankings and rewards.

## Quick Links

| Document | Description |
|----------|-------------|
| [Architecture](./ARCHITECTURE.md) | System architecture, infrastructure, and deployment |
| [API Reference](./API.md) | Complete REST API and WebSocket endpoint documentation |
| [Database Schema](./DATABASE.md) | DBML schema, Mermaid ERD, and model relationships |
| [Backend](./BACKEND.md) | Backend services, cron jobs, middleware, and business logic |
| [Frontend](./FRONTEND.md) | Frontend pages, hooks, stores, and providers |
| [Real-Time Communication](./REALTIME.md) | WebSocket events, matching algorithm, and call/chat flows |
| [Business Logic](./BUSINESS-LOGIC.md) | Core features, matching, points, subscriptions, and notifications |

## Tech Stack

### Backend
- **Runtime**: Bun (TypeScript)
- **Framework**: Express.js v5 + Socket.IO v4.8
- **Database**: PostgreSQL 16 (via Prisma ORM)
- **Cache/Pub-Sub**: Redis 7 (IORedis/Iovalkey)
- **Auth**: Better-Auth (magic link, OAuth, anonymous)
- **Payments**: Polar + Helio (Solana)
- **Email**: Resend (React Email templates)
- **Queue**: BullMQ
- **Storage**: AWS S3
- **Locking**: Redlock (distributed cron coordination)

### Frontend
- **Framework**: Next.js 16 (App Router, React 19)
- **State**: Zustand (encrypted persistence)
- **Data Fetching**: TanStack Query
- **UI**: Tailwind CSS v4 + shadcn/ui + Radix UI
- **Real-Time**: Socket.IO client + SSE
- **Wallet**: Solana Wallet Adapter
- **Forms**: React Hook Form + Zod

### Infrastructure
- **Containers**: Docker Compose (PostgreSQL 16 + Redis 7)
- **Frontend Hosting**: Vercel
- **Monitoring**: PostHog analytics, Pino/CloudWatch logging
- **Admin**: Socket.IO Admin UI, BullMQ Board

## User Tiers

| Tier | Features |
|------|----------|
| **Anonymous** | Auto-generated username, basic chat/call, no login required |
| **Registered** | Full functionality, friend system, leaderboard eligibility, OAuth/magic link |
| **Pro** | Smart matching filters, ad-free, priority access, pro badge, skip queue, chat/call history |

## Pricing

| Plan | Price |
|------|-------|
| Pro Weekly | $0.99/week |
| Pro Monthly | $2.99/month |
| Pro Annually | $29.99/year |

## Project Structure

```
cashual/
├── cashual-backend/          # Express + Socket.IO backend (Bun)
│   ├── prisma/               # Prisma schema & migrations
│   └── src/
│       ├── config/           # Logger, Redis, WebSocket config
│       ├── constants/        # Pricing, enums
│       ├── controller/       # HTTP + WebSocket request handlers
│       ├── cron/             # Scheduled jobs (match, heartbeat, subscription)
│       ├── lib/              # Auth, Prisma, Redis, Polar, Email
│       ├── middleware/       # Auth, validation, socket middleware
│       ├── routes/           # Express route definitions
│       ├── service/          # Business logic services
│       ├── websocket/        # Socket.IO namespaces (chat, call)
│       └── index.ts          # Server entry point
├── cashual-frontend/         # Next.js 16 frontend
│   └── src/
│       ├── app/              # App Router pages & layouts
│       ├── components/       # UI components by feature
│       ├── constants/        # Config, pricing, interests
│       ├── hooks/            # Custom React hooks
│       ├── lib/              # API clients, auth, crypto, utils
│       ├── provider/         # Context providers
│       ├── store/            # Zustand state stores
│       └── validation/       # Zod schemas
├── strapi/                   # Headless CMS for blog content
├── docs/                     # This documentation
└── docker-compose.yaml       # PostgreSQL + Redis containers
```

## Getting Started

### Prerequisites
- Bun (v1.0+)
- Node.js (v18+)
- Docker & Docker Compose
- AWS S3 access (for uploads)

### 1. Start Infrastructure

```bash
docker-compose up -d
```

This starts PostgreSQL 16 (port 5432) and Redis 7 (port 6379).

### 2. Backend Setup

```bash
cd cashual-backend
cp .env.example .env   # Configure environment variables
bun install
bunx prisma migrate dev
bun run dev
```

### 3. Frontend Setup

```bash
cd cashual-frontend
cp .env.example .env.local   # Configure environment variables
bun install
bun run dev
```

### Environment Variables

**Backend (`.env`)**:
- `PORT` - Server port (default: 8080)
- `DATABASE_URL` - PostgreSQL connection string
- `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` - Redis connection
- `JWT_SECRET`, `JWT_EXPIRES_IN` - JWT configuration
- `FRONTEND_URL` - Allowed CORS origin
- `BETTER_AUTH_SECRET` - Auth encryption secret
- `GOOGLE_CLIENT_ID/SECRET`, `TWITTER_CLIENT_ID/SECRET`, `DISCORD_CLIENT_ID/SECRET` - OAuth
- `RESEND_API_KEY` - Email service
- `POLAR_ACCESS_TOKEN` - Payment processor
- `AWS_S3_*` - S3 configuration for uploads

**Frontend (`.env.local`)**:
- `NEXT_PUBLIC_API_URL` - Backend API URL
- `NEXT_PUBLIC_FRIEND_CHAT_API_URL` - Friend chat API URL
- `NEXT_PUBLIC_POSTHOG_KEY` - PostHog analytics key
