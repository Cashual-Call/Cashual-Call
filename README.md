# Cashual Call

## Overview

Cashual Call is a web-based social voice platform where users connect through spontaneous 1-on-1 audio chats and text messaging. Users rate each other after conversations and compete daily to earn rewards in $CASHUAL tokens.

The platform offers three user tiers:
- **Anonymous**: Basic call/text functionality with no login required
- **Registered**: Full functionality with rewards eligibility (email or wallet login)
- **Pro**: Premium features including filters and ad-free experience (paid subscription)

## Tech Stack

### Frontend
- **Next.js**: React framework for the web application
- **Solana Wallet Adapter**: For crypto wallet integration
- **WebRTC**: For real-time voice communication

### Backend
- **Express/NestJS**: Backend API framework
- **PostgreSQL**: Primary database
- **Redis**: For caching and real-time features
- **BullMQ**: Job queue for background processing
- **Socket.IO**: For real-time communication and matchmaking

### Infrastructure & Services
- **Vercel**: Frontend hosting
- **S3**: File storage
- **Resend**: Email notifications
- **Solana Web3**: Blockchain integration for tokens

## Database Schema

![Database Schema](./images/database.png)

The database structure is defined in `a.dbml` and includes the following key tables:
- User
- Call
- Text
- Friendship
- Report
- LeaderboardEntry
- Subscription

## Application Flow

The application flow is visualized in the following diagram:

![User Flow](./images/userflow.png)

## Core Features

### Voice Calls
- Random 1-on-1 matching
- Rating system (5-star)
- In-call messaging
- Call duration tracking

### Text Messaging
- Random matches or friend chats
- Image sharing
- Blocking/reporting

### Profile System
- Wallet or email login
- Rating display
- Friend management

### Leaderboard & Rewards
- Daily leaderboard competition
- Luck-based $7 daily reward
- Anti-fraud mechanisms

### Pro Subscription
- Gender/region filters
- Ad-free experience
- Pro badge
- Reconnect with previous matches

## Setup Instructions

### Prerequisites
- Node.js (v16+)
- PostgreSQL
- Redis
- Solana development tools
- AWS S3 access

### Environment Variables

Create a `.env` file with the following variables:

```
# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgres://user:password@localhost:5432/cashual

# Redis
REDIS_URL=redis://localhost:6379

# Auth
JWT_SECRET=your_jwt_secret
SUPABASE_URL=your_supabase_url
SUPABASE_KEY=your_supabase_key

# Services
S3_BUCKET=your_s3_bucket
S3_REGION=your_s3_region
S3_ACCESS_KEY=your_access_key
S3_SECRET_KEY=your_secret_key
RESEND_API_KEY=your_resend_key

# Solana
SOLANA_RPC_URL=your_solana_rpc
TREASURY_WALLET=your_treasury_wallet_address
```

### Installation

1. Clone the repository:
```bash
git clone https://github.com/your-org/cashual-call.git
cd cashual-call
```

2. Install dependencies:
```bash
npm install
```

3. Set up the database:
```bash
npx prisma migrate dev
```

4. Start development servers:
```bash
# Start Redis
redis-server

# Start backend
npm run dev:server

# Start frontend
npm run dev:client
```

## Development Guidelines

### Code Structure
- `/pages` - Next.js pages and API routes
- `/components` - React components
- `/server` - Express/NestJS backend code
- `/lib` - Shared utilities
- `/models` - Data models and types
- `/styles` - Global styles

### Testing
Run tests with:
```bash
npm test
```

### Deployment

#### Frontend (Vercel)
```bash
vercel --prod
```

#### Backend
```bash
# Build
npm run build:server

# Start
npm run start:server
```

## Anti-Fraud Mechanisms

- Voice activity verification
- IP/Device matching prevention
- Minimum call duration for rewards
- Rating allowed only after 2 minutes
- Captcha for signup

## Business Model

- Pro subscription ($4.99/month or $39.99/year)
- Paid in $CASHUAL tokens or USDC
- 50% of subscription revenue used to burn $CASHUAL tokens

## Future Roadmap

- Casual weeks with special rewards
- Creator program for Cashual influencers
- Open spaces (similar to X or Clubhouse)
