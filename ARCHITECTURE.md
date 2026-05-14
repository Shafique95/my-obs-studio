# Streaming App — Full Stack Architecture

## Tech Stack Overview

| Layer | Technology |
|-------|-----------|
| **Frontend** | Flutter (Android, iOS, Windows, macOS, Linux) |
| **Backend** | Go (Golang) + Fiber framework |
| **Database** | PostgreSQL + GORM |
| **Media Server** | SRS (Simple Realtime Server) |
| **Cache** | Redis |
| **Real-time** | WebSocket (Gorilla WebSocket) |
| **Auth** | JWT |
| **Deployment** | Docker + Railway / Fly.io |

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   FRONTEND (Flutter)                         │
│          Android | iOS | Windows | macOS | Linux             │
│                                                              │
│   ┌────────────┐  ┌─────────────┐  ┌──────────────────────┐ │
│   │ Auth Screen│  │  Home/Feed  │  │  Go Live / Watch     │ │
│   └────────────┘  └─────────────┘  └──────────────────────┘ │
└────────────────────────┬─────────────────────────────────────┘
                         │
              REST API (HTTP/HTTPS)
              WebSocket (Live Chat)
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                   BACKEND (Go + Fiber)                       │
│                                                              │
│   ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌─────────────┐  │
│   │  Auth   │  │  Stream  │  │  User   │  │    Chat     │  │
│   │ Handler │  │  Handler │  │ Handler │  │  WebSocket  │  │
│   └────┬────┘  └────┬─────┘  └────┬────┘  └──────┬──────┘  │
│        └────────────┴─────────────┴───────────────┘         │
│                          │                                   │
│                    Service Layer                             │
│                          │                                   │
│              ┌───────────┴────────────┐                      │
│              │                        │                      │
│         ┌────▼─────┐           ┌──────▼──────┐              │
│         │PostgreSQL│           │    Redis    │              │
│         │  (GORM)  │           │   (Cache)   │              │
│         └──────────┘           └─────────────┘              │
└──────────────────────────────────────────────────────────────┘
                         │
              RTMP Stream Push
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                  MEDIA SERVER (SRS)                          │
│                                                              │
│   Port 1935 (RTMP)  ←──── Flutter App pushes stream         │
│   Port 8080 (HLS)   ────▶  Viewers watch via HLS            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Backend Folder Structure (Go)

```
backend/
├── cmd/
│   └── main.go                  ← Entry point
├── internal/
│   ├── auth/
│   │   ├── handler.go           ← Login / Register endpoints
│   │   ├── service.go           ← Business logic
│   │   └── middleware.go        ← JWT verification
│   ├── stream/
│   │   ├── handler.go           ← Stream key, start/stop
│   │   └── service.go
│   ├── user/
│   │   ├── handler.go           ← Profile, follow
│   │   └── service.go
│   └── chat/
│       └── websocket.go         ← Live chat via WebSocket
├── pkg/
│   ├── database/
│   │   └── postgres.go          ← DB connection
│   ├── cache/
│   │   └── redis.go             ← Redis connection
│   └── config/
│       └── config.go            ← Env variables
├── go.mod
└── go.sum
```

---

## Flutter Frontend Folder Structure

```
frontend/
├── lib/
│   ├── main.dart
│   ├── core/
│   │   ├── constants/
│   │   ├── theme/
│   │   └── router/
│   ├── features/
│   │   ├── auth/
│   │   │   ├── screens/
│   │   │   │   ├── login_screen.dart
│   │   │   │   └── register_screen.dart
│   │   │   └── providers/
│   │   ├── stream/
│   │   │   ├── screens/
│   │   │   │   ├── go_live_screen.dart    ← RTMP push
│   │   │   │   └── watch_screen.dart      ← HLS player
│   │   │   └── services/
│   │   │       └── rtmp_service.dart
│   │   ├── home/
│   │   │   └── screens/
│   │   │       └── home_screen.dart       ← Live stream feed
│   │   ├── chat/
│   │   │   └── services/
│   │   │       └── chat_service.dart      ← WebSocket
│   │   └── profile/
│   │       └── screens/
│   │           └── profile_screen.dart
│   └── shared/
│       └── widgets/
├── android/
├── ios/
├── windows/
├── macos/
└── linux/
```

---

## Database Schema (PostgreSQL)

```sql
-- Users table
users
├── id          UUID PRIMARY KEY
├── username    VARCHAR UNIQUE
├── email       VARCHAR UNIQUE
├── password    VARCHAR (hashed)
├── avatar_url  VARCHAR
└── created_at  TIMESTAMP

-- Streams table
streams
├── id          UUID PRIMARY KEY
├── user_id     UUID → users.id
├── title       VARCHAR
├── stream_key  VARCHAR UNIQUE
├── is_live     BOOLEAN
├── viewer_count INT
└── started_at  TIMESTAMP

-- Followers table
followers
├── follower_id  UUID → users.id
└── following_id UUID → users.id
```

---

## API Endpoints (Go Backend)

```
POST   /api/auth/register       ← নতুন account
POST   /api/auth/login          ← Login, JWT token পাওয়া

GET    /api/streams             ← সব live stream
POST   /api/streams/start       ← Stream শুরু করা
POST   /api/streams/stop        ← Stream বন্ধ করা
GET    /api/streams/:id         ← একটা stream এর details

GET    /api/users/:id           ← Profile দেখা
PUT    /api/users/profile       ← Profile update

WS     /ws/chat/:stream_id      ← Live chat WebSocket
```

---

## Media Server (SRS) — Docker Setup

```yaml
# docker-compose.yml
version: '3'
services:
  srs:
    image: ossrs/srs:5
    ports:
      - "1935:1935"   # RTMP
      - "8080:8080"   # HLS / HTTP
    restart: always

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: streaming_app
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
      - srs
```

---

## Stream Flow

```
১. User Flutter app খোলে
        ↓
২. Go Backend থেকে stream_key নেয়
        ↓
৩. Flutter → RTMP → SRS Server (1935)
        ↓
৪. SRS → HLS convert করে (8080)
        ↓
৫. Viewers Flutter app-এ HLS player দিয়ে দেখে
        ↓
৬. Live chat WebSocket দিয়ে চলে
```

---

## Development Roadmap

- [ ] **Phase 1** — Go Backend: Auth + JWT
- [ ] **Phase 2** — Go Backend: Stream API + Stream Key
- [ ] **Phase 3** — SRS Media Server Docker setup
- [ ] **Phase 4** — Flutter: Auth screens
- [ ] **Phase 5** — Flutter: Go Live screen (RTMP push)
- [ ] **Phase 6** — Flutter: Watch screen (HLS player)
- [ ] **Phase 7** — WebSocket Live Chat
- [ ] **Phase 8** — Deploy to production
