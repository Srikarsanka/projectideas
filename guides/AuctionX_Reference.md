# 🔨 AuctionX — Real-Time Live Auction Platform
### In-Depth Reference Guide

---

## 1. Project Vision

AuctionX is a **real-time live auction SaaS platform** where thousands of users can simultaneously bid on items with millisecond accuracy. It is engineered like a financial trading system — every bid is an atomic event, race conditions are prevented at the database level, and the entire bid history is immutably stored using **event sourcing via Kafka**.

Think: *eBay + Stock Exchange + Stripe Escrow, built by you.*

---

## 2. System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        CLIENTS (React)                        │
│         Browser ──── Socket.io ──── HTTP REST API             │
└──────────────┬────────────────────────────┬──────────────────┘
               │ WebSocket                  │ REST
               ▼                            ▼
┌──────────────────────┐       ┌────────────────────────┐
│   Socket.io Server   │       │   Express REST API     │
│  (Node.js Cluster)   │       │   (Node.js / Express)  │
└──────────┬───────────┘       └──────────┬─────────────┘
           │ Redis Adapter                 │
           ▼                              ▼
┌──────────────────┐         ┌────────────────────────┐
│   Redis Cluster  │◄────────│   MongoDB (Mongoose)   │
│ - Bid Locks      │         │ - Users, Auctions      │
│ - Pub/Sub        │         │ - Items, Bids          │
│ - Leaderboard    │         │ - Transactions         │
└──────────────────┘         └────────────────────────┘
           │
           ▼
┌──────────────────┐         ┌────────────────────────┐
│   Kafka Broker   │────────►│   Kafka Consumers      │
│ (Bid Event Log)  │         │ - Audit Trail Writer   │
└──────────────────┘         │ - Email Notifier       │
                             │ - Stripe Charge Worker │
                             └────────────────────────┘
                                        │
                                        ▼
                             ┌────────────────────────┐
                             │    Stripe API          │
                             │ (Escrow + Payouts)     │
                             └────────────────────────┘
```

---

## 3. Tech Stack (Full Detail)

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | React + Vite | Fast, component-based UI |
| State Management | Zustand | Lightweight, no boilerplate |
| Real-time UI | Socket.io Client | Bi-directional events |
| Styling | TailwindCSS | Rapid premium UI |
| Charts | Recharts | Bid history graphs |
| Backend | Node.js + Express | Non-blocking I/O for high concurrency |
| Real-time | Socket.io + Redis Adapter | Horizontal scaling of WebSockets |
| Database | MongoDB + Mongoose | Flexible schema for auction items |
| Cache + Locks | Redis | Atomic bid locking, leaderboard |
| Message Queue | Apache Kafka | Event sourcing, async processing |
| Payments | Stripe | Escrow, charges, payouts |
| Auth | JWT + Refresh Tokens | Stateless, secure |
| Media | Cloudinary | Auction item image uploads |
| Queue Jobs | Bull + Redis | Timer jobs, notifications |
| DevOps | Docker + Docker Compose | Local orchestration |

---

## 4. MongoDB Schema Design

### User Model
```js
{
  _id: ObjectId,
  name: String,
  email: String (unique),
  passwordHash: String,
  role: enum['buyer', 'seller', 'admin'],
  stripeCustomerId: String,
  stripeAccountId: String,      // For seller payouts
  walletBalance: Number,        // Escrow pre-auth amount
  watchlist: [ObjectId],        // Auction IDs
  createdAt: Date
}
```

### Auction Model
```js
{
  _id: ObjectId,
  title: String,
  description: String,
  images: [String],             // Cloudinary URLs
  seller: ObjectId (ref: User),
  category: String,
  condition: enum['new','used','refurbished'],
  startingPrice: Number,
  reservePrice: Number,         // Hidden minimum
  currentBid: Number,
  currentWinner: ObjectId,
  bidCount: Number,
  startTime: Date,
  endTime: Date,
  status: enum['upcoming','live','ended','cancelled'],
  antiSnipeExtension: Number,   // Extra seconds on last-minute bid
  bids: [ObjectId],
  createdAt: Date
}
```

### Bid Model (Event Sourcing Record)
```js
{
  _id: ObjectId,
  auction: ObjectId (ref: Auction),
  bidder: ObjectId (ref: User),
  amount: Number,
  timestamp: Date,
  isWinning: Boolean,
  ipAddress: String,
  kafkaOffset: Number           // Immutable reference
}
```

### Transaction Model
```js
{
  _id: ObjectId,
  auction: ObjectId,
  buyer: ObjectId,
  seller: ObjectId,
  amount: Number,
  stripePaymentIntentId: String,
  status: enum['escrowed','released','refunded'],
  createdAt: Date
}
```

---

## 5. Real-Time Bid Flow (Core Architecture)

This is the most critical part of AuctionX. Every bid goes through this exact flow:

```
User clicks "Place Bid"
        │
        ▼
React validates (amount > currentBid)
        │
        ▼
HTTP POST /api/auctions/:id/bid
        │
        ▼
Express Middleware:
  - Authenticate JWT
  - Check auction is LIVE
  - Check bid > currentBid
        │
        ▼
Redis ATOMIC LOCK (SETNX):
  - Key: lock:auction:{auctionId}
  - TTL: 500ms
  - If lock exists → reject (another bid processing)
        │
        ▼
Redis transaction (MULTI/EXEC):
  - Update currentBid
  - Update currentWinner
        │
        ▼
MongoDB write (Bid document created)
        │
        ▼
Anti-snipe check:
  - If endTime - now < 30s → extend endTime by 30s
        │
        ▼
Kafka Producer → topic: "bids"
  - Payload: { auctionId, bidderId, amount, timestamp }
        │
        ▼
Socket.io broadcast to room: "auction:{id}"
  - Event: "bid:new"
  - Payload: { amount, bidder, timeLeft, bidCount }
        │
        ▼
All connected clients update UI instantly
```

---

## 6. Socket.io Events Reference

### Client → Server
| Event | Payload | Description |
|-------|---------|-------------|
| `join:auction` | `{ auctionId }` | Join auction room |
| `leave:auction` | `{ auctionId }` | Leave auction room |
| `bid:place` | `{ auctionId, amount }` | Place a bid (via socket) |
| `watchlist:add` | `{ auctionId }` | Add to watchlist |

### Server → Client
| Event | Payload | Description |
|-------|---------|-------------|
| `bid:new` | `{ amount, bidder, bidCount, timeLeft }` | New bid placed |
| `auction:ended` | `{ winner, finalPrice }` | Auction concluded |
| `auction:extended` | `{ newEndTime }` | Anti-snipe extension |
| `outbid:alert` | `{ currentBid, yourBid }` | You were outbid |
| `viewers:count` | `{ count }` | Current room viewers |

---

## 7. REST API Design

### Auth Routes
```
POST   /api/auth/register         → Register user
POST   /api/auth/login            → Login, returns JWT
POST   /api/auth/refresh          → Refresh access token
POST   /api/auth/logout           → Invalidate refresh token
```

### Auction Routes
```
GET    /api/auctions              → List all (filter, paginate, search)
GET    /api/auctions/:id          → Auction detail + bid history
POST   /api/auctions              → Create auction (seller only)
PUT    /api/auctions/:id          → Update auction (before start)
DELETE /api/auctions/:id          → Cancel auction
POST   /api/auctions/:id/bid      → Place a bid
GET    /api/auctions/:id/bids     → Full bid history
GET    /api/auctions/live         → Currently live auctions
GET    /api/auctions/upcoming     → Scheduled auctions
```

### User Routes
```
GET    /api/users/profile         → My profile
PUT    /api/users/profile         → Update profile
GET    /api/users/bids            → My bid history
GET    /api/users/auctions        → My listed auctions
GET    /api/users/watchlist       → My watchlist
POST   /api/users/watchlist/:id   → Add to watchlist
DELETE /api/users/watchlist/:id   → Remove from watchlist
```

### Payment Routes
```
POST   /api/payments/setup-intent        → Stripe SetupIntent (save card)
POST   /api/payments/escrow/:auctionId   → Pre-authorize winning bid
POST   /api/payments/release/:txnId      → Release escrow to seller
POST   /api/payments/refund/:txnId       → Refund buyer
GET    /api/payments/transactions        → My transaction history
```

---

## 8. Kafka Event Sourcing

Kafka acts as the **immutable audit log** for every bid ever placed.

### Topics
| Topic | Producers | Consumers |
|-------|----------|----------|
| `bids` | Bid API | Audit Writer, Email Worker |
| `auctions.ended` | Timer Worker | Stripe Worker, Notification Worker |
| `payments` | Stripe Webhook | Transaction Updater |

### Bid Event Schema
```json
{
  "eventType": "BID_PLACED",
  "auctionId": "64f3...",
  "bidderId": "64a1...",
  "amount": 1500.00,
  "previousBid": 1400.00,
  "timestamp": "2024-01-15T10:23:45.123Z",
  "metadata": {
    "ip": "192.168.1.1",
    "userAgent": "Mozilla/5.0..."
  }
}
```

---

## 9. Redis Patterns Used

### 1. Distributed Lock (Prevent Race Conditions)
```js
// Atomic bid lock - only one bid processed at a time per auction
const lock = await redis.set(
  `lock:auction:${auctionId}`,
  'locked',
  'NX',   // Only set if not exists
  'PX',   // Millisecond expiry
  500     // 500ms max lock duration
);
if (!lock) throw new Error('Another bid is being processed');
```

### 2. Leaderboard (Top Bidders)
```js
// Sorted set: score = total bid amount
await redis.zadd('leaderboard:global', totalBidAmount, userId);
const topBidders = await redis.zrevrange('leaderboard:global', 0, 9, 'WITHSCORES');
```

### 3. Auction State Cache
```js
// Cache live auction state (currentBid, timeLeft, bidCount)
await redis.hset(`auction:${id}`, {
  currentBid: 1500,
  currentWinner: 'userId123',
  bidCount: 47,
  endTime: timestamp
});
await redis.expire(`auction:${id}`, 86400); // 24hr
```

### 4. Rate Limiting (Prevent Bid Spam)
```js
// Allow max 5 bids per 10 seconds per user per auction
const key = `ratelimit:bid:${userId}:${auctionId}`;
const count = await redis.incr(key);
if (count === 1) await redis.expire(key, 10);
if (count > 5) throw new Error('Too many bids. Slow down.');
```

---

## 10. Stripe Escrow Flow

```
Winner is determined (auction ends)
            │
            ▼
Stripe PaymentIntent created (amount = winning bid)
            │
            ▼
Buyer card charged (pre-authorized during bidding)
            │
            ▼
Funds held in Stripe escrow (not yet to seller)
            │
            ▼
Seller ships item → marks as "shipped"
            │
            ▼
Buyer confirms receipt (or auto-confirm after 7 days)
            │
            ▼
Stripe Transfer to seller's connected account
(minus platform fee: 5%)
```

---

## 11. Anti-Sniping Algorithm

```js
// In bid processing middleware
const SNIPE_WINDOW = 30; // seconds
const EXTENSION = 30;    // seconds to add

const auction = await Auction.findById(auctionId);
const timeLeft = (auction.endTime - Date.now()) / 1000;

if (timeLeft < SNIPE_WINDOW) {
  // Extend auction end time
  auction.endTime = new Date(Date.now() + EXTENSION * 1000);
  await auction.save();
  
  // Notify all clients in the room
  io.to(`auction:${auctionId}`).emit('auction:extended', {
    newEndTime: auction.endTime,
    reason: 'Last-minute bid detected'
  });
}
```

---

## 12. Frontend Pages & Components

```
src/
├── pages/
│   ├── Home.jsx              → Browse/search live & upcoming auctions
│   ├── AuctionDetail.jsx     → MAIN PAGE: live bidding interface
│   ├── CreateAuction.jsx     → Seller: create new auction
│   ├── Dashboard.jsx         → Seller: manage listings, analytics
│   ├── Profile.jsx           → Buyer: bids, watchlist, wins
│   ├── Transactions.jsx      → Payment history
│   └── Admin.jsx             → Admin: manage all auctions/users
│
├── components/
│   ├── BidPanel.jsx          → Live bid input + current bid display
│   ├── BidHistory.jsx        → Real-time scrolling bid feed
│   ├── AuctionTimer.jsx      → Countdown (turns red in last 30s)
│   ├── AuctionCard.jsx       → Auction preview card
│   ├── BidGraph.jsx          → Recharts bid price over time
│   ├── ViewerCount.jsx       → "142 watching" indicator
│   └── OutbidAlert.jsx       → Toast notification when outbid
│
├── store/
│   ├── useAuctionStore.js    → Zustand: current auction state
│   └── useAuthStore.js       → Zustand: user/auth state
│
├── socket/
│   └── socketClient.js       → Socket.io connection singleton
│
└── services/
    ├── api.js                → Axios instance with interceptors
    └── stripeService.js      → Stripe.js helpers
```

---

## 13. Development Phases

### Phase 1: Foundation (Week 1–2)
- [ ] Initialize MERN project structure (Vite + Express)
- [ ] MongoDB connection + Mongoose models (User, Auction, Bid)
- [ ] JWT Auth: register, login, refresh token
- [ ] Basic auction CRUD (create, list, detail)
- [ ] Docker Compose: MongoDB + Redis + Kafka

### Phase 2: Real-Time Core (Week 3–4)
- [ ] Socket.io server setup with rooms
- [ ] Redis adapter for Socket.io clustering
- [ ] Bid placement API with Redis atomic lock
- [ ] Real-time bid broadcasting to auction room
- [ ] Anti-snipe timer extension logic
- [ ] AuctionTimer component (countdown)

### Phase 3: Kafka & Event Sourcing (Week 5)
- [ ] Kafka producer on every bid
- [ ] Kafka consumer: write bids to MongoDB (audit log)
- [ ] Kafka consumer: send outbid email notifications
- [ ] Bull queue: auction end timer jobs

### Phase 4: Payments (Week 6)
- [ ] Stripe Connect: seller onboarding
- [ ] Stripe SetupIntent: save buyer payment method
- [ ] Escrow: charge winner when auction ends
- [ ] Release funds to seller after confirmation
- [ ] Stripe webhooks for payment events

### Phase 5: Advanced Features (Week 7–8)
- [ ] AI price prediction (OpenAI API based on category/condition)
- [ ] Watchlist with smart notifications
- [ ] Leaderboard (top bidders using Redis sorted sets)
- [ ] Seller analytics dashboard
- [ ] Admin panel: manage flagged auctions

### Phase 6: Polish & DevOps (Week 9–10)
- [ ] Rate limiting (Redis + express-rate-limit)
- [ ] Input validation (Joi / Zod)
- [ ] Error handling middleware
- [ ] Cloudinary image upload
- [ ] Docker multi-stage builds
- [ ] Environment configuration
- [ ] README + Architecture diagram

---

## 14. Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:5173

# MongoDB
MONGO_URI=mongodb://localhost:27017/auctionx

# JWT
JWT_SECRET=your_super_secret_key
JWT_REFRESH_SECRET=your_refresh_secret
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Redis
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=auctionx-server
KAFKA_GROUP_ID=auctionx-consumers

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Cloudinary
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

# Email (Nodemailer)
SMTP_HOST=smtp.gmail.com
SMTP_USER=your@gmail.com
SMTP_PASS=your_app_password
```

---

## 15. Folder Structure

```
auctionx/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── db.js           → MongoDB connection
│   │   │   ├── redis.js        → Redis client
│   │   │   └── kafka.js        → Kafka producer/consumer setup
│   │   ├── models/
│   │   │   ├── User.js
│   │   │   ├── Auction.js
│   │   │   ├── Bid.js
│   │   │   └── Transaction.js
│   │   ├── routes/
│   │   │   ├── auth.routes.js
│   │   │   ├── auction.routes.js
│   │   │   ├── user.routes.js
│   │   │   └── payment.routes.js
│   │   ├── controllers/
│   │   │   ├── auth.controller.js
│   │   │   ├── auction.controller.js
│   │   │   ├── bid.controller.js
│   │   │   └── payment.controller.js
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js
│   │   │   ├── rateLimiter.js
│   │   │   └── errorHandler.js
│   │   ├── socket/
│   │   │   ├── socketServer.js   → Socket.io setup
│   │   │   └── auctionSocket.js  → Room/event handlers
│   │   ├── workers/
│   │   │   ├── bidConsumer.js    → Kafka bid consumer
│   │   │   ├── emailWorker.js    → Kafka email consumer
│   │   │   └── auctionTimer.js  → Bull: end auction jobs
│   │   └── server.js
│   ├── .env
│   ├── package.json
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── store/
│   │   ├── socket/
│   │   └── services/
│   ├── index.html
│   ├── vite.config.js
│   └── Dockerfile
│
└── docker-compose.yml
```

---

## 16. Docker Compose Setup

```yaml
version: '3.8'
services:
  mongodb:
    image: mongo:7
    ports: ["27017:27017"]
    volumes: [mongo_data:/data/db]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  backend:
    build: ./backend
    ports: ["5000:5000"]
    depends_on: [mongodb, redis, kafka]
    env_file: ./backend/.env

  frontend:
    build: ./frontend
    ports: ["5173:5173"]
    depends_on: [backend]

volumes:
  mongo_data:
```

---

## 17. Why This Impresses Interviewers

| Concept | Implementation in AuctionX |
|---------|--------------------------|
| **Distributed Systems** | Redis distributed locks prevent race conditions |
| **Event Sourcing** | Kafka stores every bid as an immutable event |
| **Real-Time at Scale** | Socket.io with Redis adapter = horizontal scaling |
| **Financial Engineering** | Stripe escrow + payout flow like a fintech app |
| **Concurrency** | Atomic Redis operations under high load |
| **Message Queues** | Kafka async processing decouples services |
| **Anti-patterns avoided** | No polling, no naive DB writes under concurrent load |

---

*End of AuctionX Reference Guide*
