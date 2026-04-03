# Real-Time Communication

## Overview

Cashual uses three real-time channels:
1. **Socket.IO** — bidirectional messaging (chat namespace `/chat`, call namespace `/call`)
2. **SSE (Server-Sent Events)** — unidirectional notifications (`/sse/events`)
3. **WebRTC** — peer-to-peer audio/video (signaled via Socket.IO `/call`)

```mermaid
graph LR
    subgraph Client
        Browser
    end

    subgraph "Real-Time Channels"
        SocketChat["Socket.IO /chat<br/>Text messaging"]
        SocketCall["Socket.IO /call<br/>WebRTC signaling"]
        SSE["SSE /sse/events<br/>Notifications"]
        WebRTC["WebRTC P2P<br/>Audio/Video"]
    end

    Browser <-->|bidirectional| SocketChat
    Browser <-->|bidirectional| SocketCall
    Browser <--|unidirectional| SSE
    Browser <-->|peer-to-peer| WebRTC

    SocketChat <--> Redis
    SocketCall <--> Redis
    SSE <-- Redis
```

---

## Socket.IO Chat Namespace (`/chat`)

### Connection

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Socket.IO /chat
    participant R as Redis

    C->>S: Connect with JWT token
    S->>S: verifyToken(token)
    Note over S: Extract: senderId, receiverId,<br/>roomId, usernames
    S->>S: socket.join(roomId)
    S->>R: Subscribe to room channel
    S-->>C: Connected + joined room
```

**Authentication**: JWT token in connection auth header containing `senderId`, `receiverId`, `roomId`, `senderUsername`, `receiverUsername`.

### Events

#### Client → Server

| Event | Payload | Description |
|-------|---------|-------------|
| `join` | — | Join the room (auto-called on connect) |
| `leave` | — | Leave the room |
| `message` | `{ content, type, replyTo? }` | Send a message (text/image/gif/audio/video/file) |
| `user-event` | `{ eventType }` | Typing indicators and user actions |
| `call-request` | — | Request a voice call with chat partner |
| `call-request-accepted` | — | Accept incoming call request |
| `call-request-rejected` | — | Reject incoming call request |
| `emit_to_room` | `{ event, data }` | Emit custom event to other users in room |
| `broadcast_to_room` | `{ event, data }` | Broadcast custom event to all in room |

#### Server → Client

| Event | Payload | Description |
|-------|---------|-------------|
| `message` | `Message` | New message in room |
| `user-joined` | `{ userId, username }` | User joined the room |
| `user-left` | `{ userId, username }` | User left the room |
| `user-event` | `{ eventType, userId }` | User typing/action indicator |
| `call-request` | `{ from }` | Incoming call request |
| `call-request-accepted` | — | Call request accepted |
| `call-request-rejected` | — | Call request rejected |
| `friend-request` | `{ from }` | Friend request during chat |

### Message Flow

```mermaid
sequenceDiagram
    participant A as User A
    participant S as Socket.IO
    participant R as Redis Adapter
    participant Q as BullMQ
    participant DB as PostgreSQL
    participant B as User B

    A->>S: message { content, type }
    S->>R: Broadcast to room
    R->>S: Deliver to room members
    S-->>B: message event
    S->>Q: Queue DB persistence job
    Q->>DB: INSERT INTO text
```

---

## Socket.IO Call Namespace (`/call`)

### Connection & WebRTC Signaling

```mermaid
sequenceDiagram
    participant A as User A (Caller)
    participant S as Socket.IO /call
    participant B as User B (Callee)

    A->>S: Connect with JWT token
    B->>S: Connect with JWT token
    S->>S: Both in room, start signaling

    A->>S: send-offer (SDP offer)
    S->>B: offer (SDP offer)
    B->>S: answer (SDP answer)
    S->>A: answer (SDP answer)

    A->>S: add-ice-candidate
    S->>B: add-ice-candidate
    B->>S: add-ice-candidate
    S->>A: add-ice-candidate

    Note over A,B: WebRTC P2P connection established
    Note over A,B: Audio streams flow directly P2P
```

### Events

#### Client → Server

| Event | Payload | Description |
|-------|---------|-------------|
| `send-offer` | `{ sdp }` | WebRTC SDP offer |
| `answer` | `{ sdp }` | WebRTC SDP answer |
| `add-ice-candidate` | `{ candidate }` | ICE candidate for NAT traversal |
| `user-event` | `{ eventType, data }` | Custom user events |
| `friend-request` | `{ to }` | Send friend request during call |
| `lobby` | — | Enter waiting queue |

#### Server → Client

| Event | Payload | Description |
|-------|---------|-------------|
| `offer` | `{ sdp }` | Forwarded SDP offer |
| `answer` | `{ sdp }` | Forwarded SDP answer |
| `add-ice-candidate` | `{ candidate }` | Forwarded ICE candidate |
| `user-event` | `{ eventType, data }` | Custom user events |
| `friend-request` | `{ from }` | Friend request notification |
| `matched` | `{ roomId, token }` | Paired with another user |

### Call User Manager

The call namespace uses a queue-based matching system:

```mermaid
flowchart TD
    A[User connects to /call] --> B[Add to call queue]
    B --> C{Queue size >= 2?}
    C -->|No| D[Wait in lobby]
    C -->|Yes| E[Pop 2 users from queue]
    E --> F[Create room]
    F --> G[Generate JWT tokens]
    G --> H[Send 'matched' event to both]
    H --> I[WebRTC signaling begins]
```

---

## Server-Sent Events (`/sse/events`)

### Connection

```mermaid
sequenceDiagram
    participant C as Client (EventSource)
    participant S as SSE Endpoint
    participant R as Redis Pub/Sub
    participant P as Presence Service

    C->>S: GET /sse/events (with auth)
    S->>S: Verify JWT token
    S->>P: incrementPresence()
    S->>R: Subscribe to sse:user:{userId}
    S->>S: Send unsent notifications
    S-->>C: Connection established

    loop Every notification
        R-->>S: New message on channel
        S-->>C: data: {notification JSON}
    end

    C->>S: Connection closed
    S->>P: decrementPresence()
    S->>R: Unsubscribe
```

### Event Format

```
event: notification
data: {"id":"uuid","type":"MATCH_FOUND","title":"Match Found!","message":"You've been matched","data":{"roomId":"...","token":"..."},"priority":"NORMAL"}

event: notification
data: {"id":"uuid","type":"FRIEND_REQUEST","title":"Friend Request","message":"user123 wants to be friends","data":{"from":"user123","friendshipId":"..."},"priority":"NORMAL"}
```

### Notification Types

| Type | Trigger | Client Action |
|------|---------|---------------|
| `MATCH_FOUND` | Match cron pairs users | Redirect to `/chat` or `/call` |
| `FRIEND_REQUEST` | User sends friend request | Toast + refetch friendships |
| `FRIEND_ACCEPTED` | Friend request accepted | Toast notification |
| `NEW_MESSAGE` | Direct chat request | Alert with accept/decline |
| `CALL_INCOMING` | Friend initiates call | Incoming call dialog |
| `CALL_MISSED` | Call not answered | Toast notification |
| `SYSTEM_ANNOUNCEMENT` | Admin broadcast | Alert modal |
| `POINTS_EARNED` | Points awarded | Toast notification |
| `ACHIEVEMENT_UNLOCKED` | Achievement earned | Toast notification |
| `SUBSCRIPTION_EXPIRING` | Pro expiring soon | Warning toast |
| `SUBSCRIPTION_EXPIRED` | Pro expired | Warning toast |

### Reconnection Strategy

The SSE client implements:
1. **Auto-reconnect** with exponential backoff
2. **Visibility API** — pauses when tab is hidden
3. **Heartbeat checking** — detects stale connections
4. **Last event ID** — resumes from last received event

---

## Matching Algorithm

```mermaid
flowchart TD
    Start[Match Cron Runs<br/>Every 3 seconds] --> Lock[Acquire Redlock]
    Lock --> Cleanup[Cleanup inactive users<br/>30s timeout]
    Cleanup --> GetUsers[Get all available users<br/>from Redis ZSET]
    GetUsers --> CalcInterests[Calculate common interests<br/>for every pair]
    CalcInterests --> Sort[Sort pairs by<br/>interest score DESC]
    Sort --> Greedy[Greedy matching:<br/>Take highest score pair]
    Greedy --> Check{Both users<br/>still unmatched?}
    Check -->|Yes| Match[Create match]
    Check -->|No| Skip[Skip pair]
    Match --> Next{More pairs?}
    Skip --> Next
    Next -->|Yes| Greedy
    Next -->|No| Remaining{Unmatched<br/>users left?}
    Remaining -->|Yes| Random[Random pair<br/>remaining users]
    Remaining -->|No| Done[Release lock]
    Random --> Done

    Match --> CreateRoom[Create Room in DB]
    CreateRoom --> GenTokens[Generate JWT tokens<br/>for both users]
    GenTokens --> InitState[Initialize room state<br/>for heartbeat]
    InitState --> StoreRedis[Store match data<br/>in Redis]
    StoreRedis --> Notify[Send MATCH_FOUND<br/>notification via SSE]

    style Start fill:#4CAF50
    style Done fill:#4CAF50
```

### Anti-Spam Protection

- **7-second cooldown**: Same pair cannot be matched within 7 seconds
- **30-second timeout**: Inactive users removed from pool
- **Distributed lock**: Only one server instance runs matching at a time

---

## Heartbeat System

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Heartbeat API
    participant RS as RoomStateService
    participant PS as PointService
    participant Cron as Heartbeat Cron

    loop Every few seconds
        C->>API: POST /api/v1/heartbeat
        API->>RS: heartbeat(roomId, userId)
        RS->>RS: Update timestamp
        RS->>RS: Increment count
        alt count % 10 == 0
            RS->>PS: addPoints(userId, points)
        end
        RS-->>API: { heartbeatCount, otherUserState }
        API-->>C: 200 OK
    end

    loop Every 10 seconds
        Cron->>RS: makeDisconnect()
        RS->>RS: Check all rooms
        Note over RS: If no heartbeat in 10s,<br/>mark user disconnected
        Cron->>RS: removeDisconnectedUsers()
        Note over RS: If both disconnected,<br/>clean up room state
    end
```

### Points Awarded

Points are awarded every 10 heartbeats (~30-50 seconds of activity):
- Points vary by room type (chat vs call)
- Tracked in `PointActivity` table
- Contribute to daily leaderboard

---

## Horizontal Scaling

```mermaid
graph TB
    subgraph "Server Instance 1"
        S1["Socket.IO Server"]
    end

    subgraph "Server Instance 2"
        S2["Socket.IO Server"]
    end

    subgraph "Redis"
        Adapter["Socket.IO Redis Adapter"]
        PubSub["Pub/Sub Channels"]
        State["Shared State"]
    end

    S1 <--> Adapter
    S2 <--> Adapter
    S1 <--> PubSub
    S2 <--> PubSub
    S1 <--> State
    S2 <--> State

    C1["Client A<br/>(connected to S1)"] <--> S1
    C2["Client B<br/>(connected to S2)"] <--> S2
```

Socket.IO uses the Redis adapter, so messages sent by a client connected to Instance 1 are automatically forwarded to clients on Instance 2 via Redis pub/sub.
