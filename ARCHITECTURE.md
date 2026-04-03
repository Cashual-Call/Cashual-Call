# Architecture

## System Overview

```mermaid
graph TB
    subgraph Client["Frontend (Next.js 16)"]
        Browser["Browser"]
        SSE["SSE Client"]
        SocketClient["Socket.IO Client"]
        WebRTC["WebRTC Peer"]
    end

    subgraph Server["Backend (Express + Bun)"]
        API["REST API<br/>/api/v1/*"]
        Auth["Better-Auth<br/>/api/auth/*"]
        WSChat["Socket.IO /chat"]
        WSCall["Socket.IO /call"]
        SSEEndpoint["SSE /sse/events"]
        Cron["Cron Jobs"]
    end

    subgraph Data["Data Layer"]
        PG["PostgreSQL 16"]
        Redis["Redis 7"]
        S3["AWS S3"]
        BullMQ["BullMQ"]
    end

    subgraph External["External Services"]
        Polar["Polar (Payments)"]
        Helio["Helio (Solana Payments)"]
        Resend["Resend (Email)"]
        OAuth["OAuth Providers<br/>Google, Twitter, Discord"]
    end

    Browser -->|HTTP| API
    Browser -->|HTTP| Auth
    SSE -->|EventSource| SSEEndpoint
    SocketClient -->|WebSocket| WSChat
    SocketClient -->|WebSocket| WSCall
    WebRTC -->|P2P Audio| WebRTC

    API --> PG
    API --> Redis
    Auth --> PG
    Auth --> Redis
    WSChat --> Redis
    WSCall --> Redis
    SSEEndpoint --> Redis
    Cron --> PG
    Cron --> Redis

    API --> S3
    API --> BullMQ
    BullMQ --> PG

    API --> Polar
    API --> Helio
    Auth --> Resend
    Auth --> OAuth
```

## Request Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Express
    participant M as Middleware
    participant R as Router
    participant Ctrl as Controller
    participant S as Service
    participant DB as PostgreSQL
    participant Cache as Redis

    C->>E: HTTP Request
    E->>M: CORS, Helmet, Morgan, Rate Limit
    M->>R: Route matching
    R->>M: Auth middleware (verifyToken)
    M->>Ctrl: Authorized request
    Ctrl->>Cache: Check cache
    alt Cache hit
        Cache-->>Ctrl: Cached data
    else Cache miss
        Ctrl->>S: Business logic
        S->>DB: Query
        DB-->>S: Result
        S->>Cache: Store in cache
        S-->>Ctrl: Data
    end
    Ctrl-->>C: JSON Response
```

## Middleware Pipeline

```mermaid
graph LR
    A[Request] --> B[CORS]
    B --> C[Helmet]
    C --> D[Morgan Logger]
    D --> E[Rate Limiter<br/>Redis Store]
    E --> F{Path?}
    F -->|/api/auth/*| G[Better-Auth Handler]
    F -->|/api/v1/*| H[JSON Body Parser]
    H --> I[URL Encoded Parser]
    I --> J[Error Handler]
    J --> K[Route Handler]
    K --> L{Needs Auth?}
    L -->|Yes| M[verifyToken]
    M --> N{Needs Pro?}
    N -->|Yes| O[requirePro]
    O --> P[Controller]
    N -->|No| P
    L -->|No| P
```

## Infrastructure

```mermaid
graph TB
    subgraph Docker["Docker Compose"]
        PG["PostgreSQL 16<br/>Port: 5432<br/>DB: app_db"]
        Redis["Redis 7<br/>Port: 6379<br/>AOF Persistence"]
    end

    subgraph Volumes["Persistent Storage"]
        PGVol["postgres_data"]
        RedisVol["redis_data"]
    end

    PG --> PGVol
    Redis --> RedisVol

    subgraph Backend["Backend Server (Bun)"]
        Express["Express App"]
        SocketIO["Socket.IO Server"]
        Cron["Cron Workers"]
    end

    subgraph Frontend["Frontend (Vercel)"]
        NextJS["Next.js 16<br/>App Router"]
    end

    Express --> PG
    Express --> Redis
    SocketIO --> Redis
    Cron --> PG
    Cron --> Redis
    NextJS --> Express
    NextJS --> SocketIO
```

## Redis Usage Patterns

Redis serves multiple roles in the architecture:

| Pattern | Usage | Key Format |
|---------|-------|-----------|
| **Cache** | API response caching (TTL-based) | `cache:{resource}:{id}` |
| **Pub/Sub** | Real-time notifications via SSE | `sse:user:{userId}` |
| **ZSET** | Available user pool for matching | `available:{type}:users` |
| **ZSET** | Interest-based user indexing | `available:{type}:interest:{interest}` |
| **Hash** | User metadata during search | `available:{type}:user:{userId}` |
| **List** | Friend chat message storage | `friend-chat:{roomId}:messages` |
| **List** | Call queue for WebRTC pairing | `call:queue` |
| **String** | Room state / heartbeat tracking | `room-state:{roomId}` |
| **String** | Match data (JWT, room info) | `match:{userId}` |
| **String** | Rate limiter counters | `rl:{ip}` |
| **Adapter** | Socket.IO horizontal scaling | Internal Socket.IO channels |
| **Redlock** | Distributed cron job locking | `lock:{cronName}` |
| **Store** | Better-Auth session storage | `auth:session:{token}` |

## Distributed Execution

```mermaid
graph TB
    subgraph Instance1["Server Instance 1"]
        C1["Cron: Match"]
        C2["Cron: Heartbeat"]
        C3["Cron: Subscription"]
    end

    subgraph Instance2["Server Instance 2"]
        C4["Cron: Match"]
        C5["Cron: Heartbeat"]
        C6["Cron: Subscription"]
    end

    subgraph Redis["Redis (Redlock)"]
        L1["lock:match-cron<br/>TTL: 1.9s"]
        L2["lock:heartbeat-cron<br/>TTL: 28s"]
        L3["lock:subscription-cron<br/>TTL: 50s"]
    end

    C1 -->|Acquire lock| L1
    C4 -->|Acquire lock| L1
    C2 -->|Acquire lock| L2
    C5 -->|Acquire lock| L2
    C3 -->|Acquire lock| L3
    C6 -->|Acquire lock| L3

    style C1 fill:#4CAF50
    style C5 fill:#4CAF50
    style C3 fill:#4CAF50
    style C4 fill:#f44336
    style C2 fill:#f44336
    style C6 fill:#f44336
```

Only one instance wins each lock. Others skip execution, preventing duplicate processing.

## Rate Limiting

- **Window**: 5 minutes (configurable via `RATE_LIMIT_WINDOW_MS`)
- **Max Requests**: 300 per window (configurable via `RATE_LIMIT_MAX`)
- **Store**: Redis-backed for distributed consistency
- **Headers**: Standard rate limit headers (`RateLimit-*`)

## Graceful Shutdown

```mermaid
sequenceDiagram
    participant OS as SIGTERM
    participant App as Application
    participant DB as Prisma
    participant R as Redis
    participant Cron as Cron Jobs
    participant HTTP as HTTP Server

    OS->>App: SIGTERM signal
    App->>DB: $disconnect()
    App->>R: pubClient.quit()
    App->>R: subClient.quit()
    App->>Cron: matchCleanup()
    App->>Cron: heartbeatCleanup()
    App->>Cron: subscriptionCleanup()
    App->>HTTP: server.close()
    HTTP-->>App: Closed
    App->>OS: process.exit(0)
```
