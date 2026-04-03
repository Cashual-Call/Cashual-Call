# API Reference

Base URL: `http://localhost:8080`

All authenticated endpoints require a JWT token in the `Authorization: Bearer <token>` header.

---

## Health Check

### `GET /health`

Returns system metrics and online user count.

**Response:**
```json
{
  "timestamp": "2026-04-03T12:00:00.000Z",
  "totalUserCount": 42,
  "uptime": { "seconds": 3600, "formatted": "1h 0m 0s" },
  "memory": { "total": "128MB", "used": "64MB", "external": "8MB", "rss": "150MB" },
  "cpu": { "user": "1200ms", "system": "300ms" },
  "environment": "production"
}
```

---

## Authentication

### `ALL /api/auth/*`

Handled by Better-Auth. Supports:
- Magic link sign-in (email via Resend)
- Google OAuth
- Twitter OAuth
- Discord OAuth
- Anonymous sign-in (auto-generated username)
- Username plugin
- Admin plugin

---

## User Routes (`/api/v1/users`)

### `GET /api/v1/users/avatars`
Returns list of available avatar URLs.

**Auth**: None

**Response:** `Avatar[]`

---

### `GET /api/v1/users/points`
Get user points within an optional date range.

**Auth**: None  
**Query**: `userId`, `startDate?`, `endDate?`

---

### `GET /api/v1/users/points-by-date`
Get user's points grouped by date.

**Auth**: Required

---

### `GET /api/v1/users/rankings`
Get leaderboard rankings for today.

**Auth**: None

---

### `GET /api/v1/users/lucky-winner`
Get the latest lucky winner entry.

**Auth**: None  
**Cache**: 7200s

---

## Friend Routes (`/api/v1/users/friends`)

### `GET /api/v1/users/friends`
Get full friends list (accepted, pending sent, pending received).

**Auth**: Required

**Response:**
```json
{
  "accepted": [{ "id": "...", "username": "...", "avatarUrl": "...", "friendshipId": "..." }],
  "pending_sent": [...],
  "pending_received": [...]
}
```

---

### `GET /api/v1/users/friends/pending`
Get incoming friend requests only.

**Auth**: Required

---

### `GET /api/v1/users/friends/suggestions`
Get suggested friends (random users not in friend network).

**Auth**: Required  
**Query**: `limit?` (default: 10)

---

### `POST /api/v1/users/friends/:friendId`
Send a friend request. Auto-accepts if incoming request from that user exists.

**Auth**: Required

---

### `POST /api/v1/users/friends/accept/:friendshipId`
Accept a pending friend request.

**Auth**: Required

---

### `POST /api/v1/users/friends/reject/:friendshipId`
Reject a pending friend request (deletes it).

**Auth**: Required

---

### `DELETE /api/v1/users/friends/:friendId`
Remove a friend or cancel pending request.

**Auth**: Required

---

### `POST /api/v1/users/friends/:friendId/cancel`
Cancel a sent friend request.

**Auth**: Required

---

### `GET /api/v1/users/friends/:friendId/status`
Check friendship status with a user.

**Auth**: Required

**Response:**
```json
{
  "areFriends": true,
  "status": "accepted"
}
```

---

### `POST /api/v1/users/friends/:friend/chat`
Start a 1-on-1 chat with a friend. Requires accepted friendship.

**Auth**: Required

---

### `POST /api/v1/users/friends/:friend/call`
Initiate a voice/video call with a friend.

**Auth**: Required

---

## Search & Matching Routes (`/api/v1/search`)

### `POST /api/v1/search/call/start-search/:userId`
Add user to the call matching queue.

**Auth**: Required  
**Body**: `{ interests?: string[] }`

---

### `POST /api/v1/search/call/stop-search/:userId`
Remove user from the call matching queue.

**Auth**: Required

---

### `POST /api/v1/search/call/heartbeat/:userId`
Update activity timestamp for call search.

**Auth**: Required

---

### `POST /api/v1/search/chat/start-search/:userId`
Add user to the chat matching queue.

**Auth**: Required  
**Body**: `{ interests?: string[] }`

---

### `POST /api/v1/search/chat/stop-search/:userId`
Remove user from the chat matching queue.

**Auth**: Required

---

### `POST /api/v1/search/chat/heartbeat/:userId`
Update activity timestamp for chat search.

**Auth**: Required

---

### `POST /api/v1/search/public-room`
Create a JWT token for public room access.

**Auth**: Required

---

### `POST /api/v1/search/chat/start-direct`
Initiate a direct chat request to a specific user. Stored in Redis with 10-minute TTL.

**Auth**: Required  
**Body**: `{ targetUserId: string }`

---

### `POST /api/v1/search/chat/accept-direct`
Accept a direct chat request. Creates room and generates tokens.

**Auth**: Required  
**Body**: `{ requestId: string }`

---

### `POST /api/v1/search/chat/decline-direct`
Decline a direct chat request.

**Auth**: Required  
**Body**: `{ requestId: string }`

---

### `GET /api/v1/search/`
Search users by username (top 20, cached).

**Auth**: Required  
**Query**: `q` (search term)

---

## Friend Chat Routes (`/api/v1/friend-chat`)

### `GET /api/v1/friend-chat/rooms/:roomId/messages`
Fetch messages from a friend chat room.

**Auth**: Required  
**Query**: `limit?` (default: 100, max: 500)

---

### `POST /api/v1/friend-chat/rooms/:roomId/messages`
Send a message to a friend chat room.

**Auth**: Required

**Body:**
```json
{
  "content": "Hello!",
  "type": "text",
  "replyTo": "optional-message-id"
}
```

Supported types: `text`, `image`, `gif`, `audio`, `video`, `file`

---

## Payment Routes (`/api/v1/payment`)

### `POST /api/v1/payment/success`
Handle successful payment. Updates user Pro status and creates subscription.

**Auth**: Required  
**Body**: `{ plan: "MONTHLY" | "YEARLY", paymentId?: string }`

---

### `POST /api/v1/payment/error`
Log a payment error for tracking.

**Auth**: Required

---

### `POST /api/v1/payment/webhook`
Handle Helio webhook callbacks. Verifies webhook signature.

**Auth**: None (signature verification)

---

### `GET /api/v1/payment/status`
Get current subscription status.

**Auth**: Required

**Response:**
```json
{
  "isPro": true,
  "proEnd": "2026-05-03T00:00:00.000Z",
  "isActive": true,
  "subscription": { "plan": "MONTHLY", "startedAt": "...", "expiresAt": "..." }
}
```

---

## Report Routes (`/api/v1/reports`)

### `POST /api/v1/reports/`
Create a user report. Prevents self-reports and duplicates.

**Auth**: Required  
**Body**: `{ reportedUserId: string, reason: string }`

---

### `GET /api/v1/reports/`
Get paginated reports list.

**Auth**: Required  
**Query**: `skip?`, `take?`, `orderBy?`

---

### `GET /api/v1/reports/stats`
Get report statistics (total, today, week, month, top 10 reported).

**Auth**: Required

---

### `GET /api/v1/reports/:id`
Get a single report by ID.

**Auth**: Required

---

### `GET /api/v1/reports/reporter/:reporterId`
Get all reports made by a user.

**Auth**: Required

---

### `GET /api/v1/reports/reported/:reportedUserId`
Get all reports against a user.

**Auth**: Required

---

### `DELETE /api/v1/reports/:id`
Delete a report and invalidate caches.

**Auth**: Required

---

## Rating Routes (`/api/v1/ratings`)

### `POST /api/v1/ratings/`
Create or update a 1-5 star rating. Prevents self-rating.

**Auth**: Required  
**Body**: `{ ratedUserId: string, rating: 1-5 }`

---

### `GET /api/v1/ratings/user/:userId`
Get paginated ratings for a user with summary statistics.

**Auth**: Required  
**Query**: `skip?`, `take?`

---

### `GET /api/v1/ratings/me/summary`
Get authenticated user's rating summary (average and count).

**Auth**: Required

---

### `GET /api/v1/ratings/me/given`
Get ratings given by the authenticated user.

**Auth**: Required  
**Query**: `skip?`, `take?`

---

## History Routes (`/api/v1/history`)

### `GET /api/v1/history/chat`
Get chat room history.

**Auth**: Required  
**Pro**: Required

---

### `GET /api/v1/history/call`
Get call history.

**Auth**: Required  
**Pro**: Required

---

### `GET /api/v1/history/global-chats`
Get global chat messages.

**Auth**: None

---

## Upload Routes (`/api/v1/upload`)

### `GET /api/v1/upload/presigned-url`
Generate an AWS S3 presigned URL for file upload (1-hour expiry).

**Auth**: None  
**Query**: `fileType` (MIME type)

**Response:**
```json
{
  "uploadUrl": "https://s3.amazonaws.com/...",
  "fileUrl": "https://s3.amazonaws.com/bucket/uuid.ext"
}
```

---

## Heartbeat Routes (`/api/v1/heartbeat`)

### `POST /api/v1/heartbeat/`
Update room state heartbeat. Detects disconnections and awards points.

**Auth**: Required  
**Body**: `{ roomId: string, token: string }`

---

## SSE Routes (`/sse`)

### `GET /sse/events`
Server-Sent Events endpoint for real-time notifications.

**Auth**: Required (via query param or header)

**Events:**
```
data: {"type":"MATCH_FOUND","data":{"roomId":"...","token":"..."}}

data: {"type":"FRIEND_REQUEST","data":{"from":"username"}}

data: {"type":"NEW_MESSAGE","data":{"roomId":"...","content":"..."}}
```

---

## Admin Routes

### `GET /admin/queues`
BullMQ admin dashboard for monitoring message and match job queues.

---

## WebSocket Namespaces

See [Real-Time Communication](./REALTIME.md) for complete WebSocket event documentation.

| Namespace | Purpose |
|-----------|---------|
| `/chat` | Text messaging, typing indicators, call requests |
| `/call` | WebRTC signaling (SDP offer/answer, ICE candidates) |
