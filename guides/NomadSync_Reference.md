# 🗺️ NomadSync — Real-Time Collaborative Trip Planning SaaS
### In-Depth Reference Guide

---

## 1. Project Vision

NomadSync is a **real-time, collaborative trip planning platform** that solves the chaos of planning group travel. Imagine Google Docs + Notion + Splitwise + Google Maps — all fused into one beautifully designed application. Multiple users edit the same itinerary simultaneously, vote on activities, split expenses, and receive AI-powered destination suggestions — all in real-time, even with poor connectivity (offline-first with CRDTs).

Think: *Notion + Splitwise + Airbnb Experiences, built by you.*

---

## 2. System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        CLIENTS (React)                         │
│   Browser ─── Socket.io ─── HTTP REST ─── IndexedDB (offline)│
└────────┬──────────────────────────────────────┬───────────────┘
         │ WebSocket (real-time sync)            │ REST API
         ▼                                       ▼
┌─────────────────────┐              ┌───────────────────────┐
│   Socket.io Server  │              │   Express REST API    │
│ (Presence, CRDT ops)│              │   (CRUD, Auth, AI)    │
└──────────┬──────────┘              └──────────┬────────────┘
           │                                    │
           ▼                                    ▼
┌──────────────────────────────────────────────────────┐
│                    Redis                              │
│  - Online presence (who's editing which block)       │
│  - Session cache                                     │
│  - Rate limiting                                     │
│  - CRDT operation queue                              │
└──────────────────────────┬───────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────┐
│                    MongoDB                            │
│  - Users, Trips, Itineraries, Places, Expenses       │
│  - CRDT operation log (for conflict resolution)      │
│  - AI suggestion cache                               │
└──────────────────────────┬───────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
┌──────────────┐  ┌────────────────┐  ┌────────────┐
│  OpenAI API  │  │   Mapbox GL    │  │ Amadeus API│
│ (AI Suggest) │  │  (Maps/Routes) │  │(Flights $) │
└──────────────┘  └────────────────┘  └────────────┘
           │
           ▼
┌──────────────────┐
│   Cloudinary     │
│ (Trip photos,    │
│  place images)   │
└──────────────────┘
```

---

## 3. Tech Stack (Full Detail)

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | React + Vite | Fast SPA with hot reload |
| State Management | Zustand | Simple global state |
| Offline Sync | Yjs (CRDT library) | Conflict-free collaborative editing |
| Real-time | Socket.io Client | Presence, live edits, chat |
| Maps | Mapbox GL JS | Interactive maps with custom markers |
| Styling | TailwindCSS | Modern, utility-first |
| Drag & Drop | @dnd-kit | Itinerary card reordering |
| Charts | Recharts | Expense breakdown pie charts |
| Offline DB | IndexedDB (Dexie.js) | Client-side persistence |
| Backend | Node.js + Express | REST API server |
| Real-time | Socket.io | CRDT sync & presence |
| Database | MongoDB + Mongoose | Flexible nested documents |
| Cache | Redis | Presence, rate limits, sessions |
| Auth | JWT + Refresh Tokens | Stateless auth |
| AI | OpenAI GPT-4 | Trip suggestions, itinerary gen |
| Travel Data | Amadeus API | Real flight/hotel pricing |
| Media | Cloudinary | Photo uploads |
| Email | Nodemailer + Resend | Trip invitations, reminders |
| DevOps | Docker + Docker Compose | Full local setup |

---

## 4. Core Concept: CRDTs (Conflict-Free Replicated Data Types)

This is the most technically advanced part of NomadSync and the #1 differentiator.

### What Problem CRDTs Solve
When two users edit the same itinerary block simultaneously (one on WiFi, one on mobile with poor signal), traditional systems create conflicts. CRDTs mathematically guarantee that all edits **merge correctly without conflicts** — even if made offline.

### How We Use Yjs
```js
// Shared document (synced across all clients)
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'

const ydoc = new Y.Doc()
const provider = new WebsocketProvider('ws://localhost:1234', 'trip-{tripId}', ydoc)

// Shared itinerary as a Yjs Array
const itinerary = ydoc.getArray('itinerary')

// Add a day's activities
itinerary.push([{
  day: 1,
  activities: ydoc.getArray('day-1-activities')
}])

// This edit auto-syncs to all connected clients
// And merges correctly even with offline edits
```

### CRDT Sync Flow
```
User A edits (offline)         User B edits (online)
       │                              │
       ▼                              ▼
  Local Y.Doc update           Local Y.Doc update
  Stored in IndexedDB          Sent to server via WS
       │                              │
       ▼                              ▼
  User A comes online     Server broadcasts to all clients
       │                              │
       ▼                              ▼
  Y.Doc sync               User A receives & merges
  (automatic merge,         (no conflict, mathematically
   no conflict)              guaranteed correct result)
```

---

## 5. MongoDB Schema Design

### User Model
```js
{
  _id: ObjectId,
  name: String,
  email: String (unique),
  passwordHash: String,
  avatar: String,              // Cloudinary URL
  currency: String,            // Default: 'USD'
  timezone: String,
  trips: [ObjectId],
  createdAt: Date
}
```

### Trip Model
```js
{
  _id: ObjectId,
  title: String,
  description: String,
  coverImage: String,          // Cloudinary URL
  destination: {
    city: String,
    country: String,
    coordinates: {
      lat: Number,
      lng: Number
    }
  },
  startDate: Date,
  endDate: Date,
  owner: ObjectId (ref: User),
  members: [{
    user: ObjectId (ref: User),
    role: enum['owner', 'editor', 'viewer'],
    joinedAt: Date
  }],
  budget: {
    total: Number,
    currency: String,
    spent: Number
  },
  status: enum['planning', 'upcoming', 'ongoing', 'completed'],
  isPublic: Boolean,
  shareToken: String,          // For invite link
  crdtDocId: String,           // Yjs document identifier
  createdAt: Date
}
```

### Itinerary Day Model
```js
{
  _id: ObjectId,
  trip: ObjectId (ref: Trip),
  date: Date,
  dayNumber: Number,
  theme: String,               // e.g., "Beach Day", "Museum Tour"
  activities: [{
    id: String,                // UUID for CRDT reference
    title: String,
    description: String,
    type: enum['place','transport','meal','accommodation','activity'],
    location: {
      name: String,
      address: String,
      coordinates: { lat: Number, lng: Number },
      mapboxPlaceId: String
    },
    startTime: String,         // "09:00"
    endTime: String,           // "11:00"
    duration: Number,          // minutes
    cost: Number,
    votes: [{
      user: ObjectId,
      vote: enum['yes','no','maybe']
    }],
    status: enum['suggested','confirmed','rejected'],
    addedBy: ObjectId (ref: User),
    notes: String
  }],
  notes: String                // Day-level CRDT-synced notes
}
```

### Expense Model
```js
{
  _id: ObjectId,
  trip: ObjectId (ref: Trip),
  title: String,
  amount: Number,
  currency: String,
  paidBy: ObjectId (ref: User),
  category: enum['food','transport','accommodation','activity','shopping','other'],
  splitType: enum['equal','percentage','exact'],
  splits: [{
    user: ObjectId (ref: User),
    amount: Number,
    percentage: Number,
    isPaid: Boolean
  }],
  receiptImage: String,        // Cloudinary URL
  date: Date,
  createdAt: Date
}
```

### Settlement Model
```js
{
  _id: ObjectId,
  trip: ObjectId (ref: Trip),
  from: ObjectId (ref: User),
  to: ObjectId (ref: User),
  amount: Number,
  currency: String,
  status: enum['pending', 'confirmed'],
  settledAt: Date
}
```

### Vote/Poll Model
```js
{
  _id: ObjectId,
  trip: ObjectId (ref: Trip),
  question: String,
  options: [{
    id: String,
    text: String,
    votes: [ObjectId]          // User IDs who voted
  }],
  type: enum['single','multiple'],
  deadline: Date,
  createdBy: ObjectId,
  status: enum['open','closed']
}
```

---

## 6. REST API Design

### Auth Routes
```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
POST   /api/auth/refresh
GET    /api/auth/me
```

### Trip Routes
```
GET    /api/trips                    → My trips (paginated)
POST   /api/trips                    → Create trip
GET    /api/trips/:id                → Trip detail (with members)
PUT    /api/trips/:id                → Update trip info
DELETE /api/trips/:id                → Delete trip

POST   /api/trips/:id/invite         → Send email invite
POST   /api/trips/:id/join/:token    → Join via invite link
DELETE /api/trips/:id/members/:uid   → Remove member
PUT    /api/trips/:id/members/:uid   → Change member role
```

### Itinerary Routes
```
GET    /api/trips/:id/itinerary              → Full itinerary
POST   /api/trips/:id/itinerary/days         → Add a day
PUT    /api/trips/:id/itinerary/days/:dayId  → Update day
DELETE /api/trips/:id/itinerary/days/:dayId  → Remove day

POST   /api/trips/:id/itinerary/days/:dayId/activities     → Add activity
PUT    /api/trips/:id/itinerary/days/:dayId/activities/:aid → Update activity
DELETE /api/trips/:id/itinerary/days/:dayId/activities/:aid → Remove
POST   /api/trips/:id/itinerary/days/:dayId/activities/:aid/vote → Vote
```

### Expense Routes
```
GET    /api/trips/:id/expenses               → All expenses
POST   /api/trips/:id/expenses               → Add expense
PUT    /api/trips/:id/expenses/:eid          → Update expense
DELETE /api/trips/:id/expenses/:eid          → Delete expense

GET    /api/trips/:id/expenses/summary       → Who owes whom
POST   /api/trips/:id/expenses/settle        → Mark settlement
```

### AI Routes
```
POST   /api/ai/suggest-activities     → AI activity suggestions for destination
POST   /api/ai/generate-itinerary     → Full AI-generated day plan
POST   /api/ai/budget-estimate        → AI cost estimate for trip
```

### Map Routes
```
GET    /api/map/search?q=...          → Place search (Mapbox)
GET    /api/map/route?from=&to=       → Route between places
GET    /api/map/flights?from=&to=&date= → Flight prices (Amadeus)
```

---

## 7. Socket.io Events Reference

### Client → Server
| Event | Payload | Description |
|-------|---------|-------------|
| `trip:join` | `{ tripId }` | Join trip collaboration room |
| `trip:leave` | `{ tripId }` | Leave trip room |
| `presence:update` | `{ tripId, editingBlock }` | Update what user is editing |
| `activity:add` | `{ tripId, dayId, activity }` | Add activity (CRDT) |
| `activity:move` | `{ tripId, from, to }` | Reorder activity (drag&drop) |
| `vote:cast` | `{ pollId, optionId }` | Vote in a poll |
| `chat:send` | `{ tripId, message }` | Trip group chat |

### Server → Client
| Event | Payload | Description |
|-------|---------|-------------|
| `presence:members` | `[{ userId, name, editingBlock }]` | Who's online |
| `activity:added` | `{ dayId, activity }` | New activity added |
| `activity:updated` | `{ dayId, activityId, changes }` | Activity edited |
| `activity:moved` | `{ from, to }` | Itinerary reordered |
| `expense:added` | `{ expense }` | New expense logged |
| `vote:updated` | `{ pollId, results }` | Vote count updated |
| `chat:message` | `{ userId, message, timestamp }` | Chat message |
| `member:joined` | `{ user }` | New member joined trip |

---

## 8. Expense Splitting Algorithm

### Equal Split
```js
function splitEqual(totalAmount, memberIds) {
  const perPerson = totalAmount / memberIds.length;
  return memberIds.map(userId => ({
    user: userId,
    amount: Math.round(perPerson * 100) / 100 // round to 2 decimals
  }));
}
```

### Settlement Calculation (Who Owes Whom)
```js
function calculateSettlements(expenses, members) {
  // Step 1: Calculate net balance per person
  const balances = {};
  members.forEach(m => balances[m._id] = 0);

  expenses.forEach(expense => {
    // Payer gets credit
    balances[expense.paidBy] += expense.amount;
    // Each split person owes their share
    expense.splits.forEach(split => {
      balances[split.user] -= split.amount;
    });
  });

  // Step 2: Simplify debts (greedy algorithm)
  const creditors = []; // positive balance (owed money)
  const debtors = [];   // negative balance (owe money)

  Object.entries(balances).forEach(([userId, bal]) => {
    if (bal > 0) creditors.push({ userId, amount: bal });
    if (bal < 0) debtors.push({ userId, amount: -bal });
  });

  const settlements = [];
  while (creditors.length && debtors.length) {
    const creditor = creditors[0];
    const debtor = debtors[0];
    const amount = Math.min(creditor.amount, debtor.amount);

    settlements.push({
      from: debtor.userId,
      to: creditor.userId,
      amount: Math.round(amount * 100) / 100
    });

    creditor.amount -= amount;
    debtor.amount -= amount;

    if (creditor.amount === 0) creditors.shift();
    if (debtor.amount === 0) debtors.shift();
  }

  return settlements;
}
```

---

## 9. AI Integration (OpenAI GPT-4)

### Activity Suggestion Prompt
```js
async function suggestActivities(destination, duration, preferences) {
  const prompt = `
    You are a travel expert. Suggest 10 activities for a trip to ${destination}.
    Trip duration: ${duration} days
    Traveler preferences: ${preferences.join(', ')}
    
    Return a JSON array with this exact structure:
    [
      {
        "title": "Activity Name",
        "type": "food|activity|accommodation|transport",
        "description": "Brief description",
        "estimatedCost": 50,
        "currency": "USD",
        "duration": 120,
        "bestTime": "morning|afternoon|evening",
        "coordinates": { "lat": 0.0, "lng": 0.0 }
      }
    ]
  `;

  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [{ role: 'user', content: prompt }],
    response_format: { type: 'json_object' }
  });

  return JSON.parse(response.choices[0].message.content);
}
```

### Full Itinerary Generator
```js
async function generateItinerary(tripDetails) {
  const { destination, startDate, endDate, budget, members, preferences } = tripDetails;
  
  const prompt = `
    Create a detailed day-by-day travel itinerary for a group trip.
    
    Destination: ${destination}
    Dates: ${startDate} to ${endDate}
    Group size: ${members} people
    Total budget: $${budget} USD
    Preferences: ${preferences}
    
    Return JSON with this structure:
    {
      "days": [
        {
          "dayNumber": 1,
          "date": "2024-01-15",
          "theme": "Arrival & Old City",
          "activities": [...]
        }
      ],
      "estimatedTotalCost": 1200,
      "packingTips": [...],
      "localTips": [...]
    }
  `;
  // ...
}
```

---

## 10. Offline-First with IndexedDB

```js
// Using Dexie.js (IndexedDB wrapper)
import Dexie from 'dexie';

const db = new Dexie('NomadSyncDB');
db.version(1).stores({
  trips: 'id, title, updatedAt',
  itinerary: 'id, tripId, date',
  expenses: 'id, tripId, date',
  pendingOps: '++id, tripId, type, createdAt' // offline queue
});

// Save trip data locally
async function cacheTripData(trip) {
  await db.trips.put({ ...trip, cachedAt: Date.now() });
}

// Queue operation when offline
async function queueOperation(tripId, type, payload) {
  await db.pendingOps.add({
    tripId, type, payload, createdAt: Date.now()
  });
}

// Sync pending ops when back online
async function syncPendingOps() {
  const ops = await db.pendingOps.toArray();
  for (const op of ops) {
    await api.post(`/trips/${op.tripId}/sync`, op);
    await db.pendingOps.delete(op.id);
  }
}

// Listen for connectivity
window.addEventListener('online', syncPendingOps);
```

---

## 11. Frontend Pages & Components

```
src/
├── pages/
│   ├── Home.jsx                → Landing + my trips dashboard
│   ├── CreateTrip.jsx          → New trip wizard
│   ├── TripDashboard.jsx       → Overview: members, budget, map
│   ├── Itinerary.jsx           → MAIN: day-by-day planner (CRDT)
│   ├── MapView.jsx             → Interactive map of all activities
│   ├── Expenses.jsx            → Expense tracker + settlement
│   ├── Polls.jsx               → Group voting interface
│   ├── TripChat.jsx            → Real-time group chat
│   └── Profile.jsx             → User settings
│
├── components/
│   ├── itinerary/
│   │   ├── DayCard.jsx         → One day's activities (draggable)
│   │   ├── ActivityCard.jsx    → Single activity with vote buttons
│   │   ├── ActivityForm.jsx    → Add/edit activity modal
│   │   └── TimelineView.jsx    → Day view as timeline
│   ├── map/
│   │   ├── TripMap.jsx         → Mapbox map with all places
│   │   ├── PlaceMarker.jsx     → Custom map pin
│   │   └── RouteLayer.jsx      → Day route polyline
│   ├── expenses/
│   │   ├── ExpenseForm.jsx     → Add expense with split config
│   │   ├── ExpenseSummary.jsx  → Who owes whom table
│   │   └── ExpenseChart.jsx    → Pie chart by category
│   ├── presence/
│   │   ├── OnlineAvatars.jsx   → Who's online in trip
│   │   └── EditingIndicator.jsx → "Priya is editing Day 2..."
│   └── shared/
│       ├── InviteModal.jsx     → Email invite + shareable link
│       └── CurrencySelector.jsx
│
├── store/
│   ├── useTripStore.js         → Zustand: trip state
│   ├── usePresenceStore.js     → Online members
│   └── useAuthStore.js
│
├── crdt/
│   ├── ydoc.js                 → Yjs document setup
│   └── syncProvider.js        → WebSocket sync provider
│
└── services/
    ├── api.js
    ├── mapboxService.js
    └── offlineDB.js            → Dexie.js wrapper
```

---

## 12. Development Phases

### Phase 1: Foundation (Week 1–2)
- [ ] Vite React + Express project setup
- [ ] MongoDB models: User, Trip, Itinerary, Expense
- [ ] JWT auth: register, login, refresh
- [ ] Trip CRUD: create, list, detail, delete
- [ ] Member invite via email (Nodemailer)
- [ ] Join via invite link (token-based)

### Phase 2: Real-Time Collaboration (Week 3–4)
- [ ] Socket.io server with trip rooms
- [ ] Yjs CRDT integration (Vite + y-websocket)
- [ ] Live presence system (who's editing what)
- [ ] Real-time activity add/edit/delete sync
- [ ] Drag & drop reordering (@dnd-kit)
- [ ] Collaborative day notes (Yjs text type)

### Phase 3: Maps & Places (Week 5)
- [ ] Mapbox GL JS integration
- [ ] Place search via Mapbox Geocoding API
- [ ] Interactive map: markers per activity
- [ ] Route drawing between day activities
- [ ] Distance + travel time calculation

### Phase 4: Expenses & Splitting (Week 6)
- [ ] Expense add form with split types
- [ ] Settlement calculation algorithm
- [ ] Expense categories + pie chart
- [ ] Multi-currency support
- [ ] Receipt photo upload (Cloudinary)

### Phase 5: AI & Polls (Week 7)
- [ ] OpenAI: activity suggestions per destination
- [ ] OpenAI: full itinerary generator
- [ ] OpenAI: budget estimate
- [ ] Poll/voting system for group decisions
- [ ] Amadeus API: flight price lookup

### Phase 6: Offline-First (Week 8)
- [ ] IndexedDB setup with Dexie.js
- [ ] Cache trip data locally on load
- [ ] Queue ops when offline
- [ ] Auto-sync on reconnect
- [ ] Offline indicator in UI

### Phase 7: Polish & DevOps (Week 9–10)
- [ ] PDF export: full itinerary (pdfmake)
- [ ] Trip sharing (public view link)
- [ ] Email reminders before trip
- [ ] Docker Compose setup
- [ ] Rate limiting + input validation
- [ ] Performance: React.memo, lazy loading

---

## 13. Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:5173

# MongoDB
MONGO_URI=mongodb://localhost:27017/nomadsync

# JWT
JWT_SECRET=your_secret
JWT_REFRESH_SECRET=your_refresh_secret

# Redis
REDIS_URL=redis://localhost:6379

# OpenAI
OPENAI_API_KEY=sk-...

# Mapbox
MAPBOX_ACCESS_TOKEN=pk.eyJ1...

# Amadeus (Flights)
AMADEUS_CLIENT_ID=...
AMADEUS_CLIENT_SECRET=...

# Cloudinary
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

# Email
SMTP_HOST=smtp.gmail.com
SMTP_USER=your@gmail.com
SMTP_PASS=your_app_password
```

---

## 14. Full Folder Structure

```
nomadsync/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── db.js
│   │   │   └── redis.js
│   │   ├── models/
│   │   │   ├── User.js
│   │   │   ├── Trip.js
│   │   │   ├── ItineraryDay.js
│   │   │   ├── Expense.js
│   │   │   ├── Settlement.js
│   │   │   └── Poll.js
│   │   ├── routes/
│   │   │   ├── auth.routes.js
│   │   │   ├── trip.routes.js
│   │   │   ├── itinerary.routes.js
│   │   │   ├── expense.routes.js
│   │   │   ├── ai.routes.js
│   │   │   └── map.routes.js
│   │   ├── controllers/
│   │   │   ├── auth.controller.js
│   │   │   ├── trip.controller.js
│   │   │   ├── itinerary.controller.js
│   │   │   ├── expense.controller.js
│   │   │   └── ai.controller.js
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js
│   │   │   └── errorHandler.js
│   │   ├── socket/
│   │   │   ├── socketServer.js
│   │   │   └── tripSocket.js
│   │   ├── services/
│   │   │   ├── openai.service.js
│   │   │   ├── mapbox.service.js
│   │   │   └── email.service.js
│   │   └── server.js
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── store/
│   │   ├── crdt/
│   │   └── services/
│   └── Dockerfile
│
└── docker-compose.yml
```

---

## 15. Docker Compose

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

  backend:
    build: ./backend
    ports: ["5000:5000"]
    depends_on: [mongodb, redis]
    env_file: ./backend/.env

  frontend:
    build: ./frontend
    ports: ["5173:5173"]
    depends_on: [backend]

  # Yjs WebSocket server (CRDT sync)
  yjs-server:
    image: node:20-alpine
    working_dir: /app
    command: npx y-websocket
    ports: ["1234:1234"]
    environment:
      PORT: 1234

volumes:
  mongo_data:
```

---

## 16. Why This Impresses Interviewers

| Concept | Implementation in NomadSync |
|---------|---------------------------|
| **CRDTs** | Yjs for conflict-free multi-user editing — Google Docs level |
| **Offline-First** | IndexedDB + sync queue — enterprise-grade UX pattern |
| **Real-Time Presence** | Socket.io showing who's editing what live |
| **Graph Algorithm** | Expense settlement uses a greedy debt simplification algorithm |
| **AI Integration** | GPT-4 for context-aware travel suggestions |
| **Maps Engineering** | Mapbox routing, custom layers, and geocoding |
| **Complex Data Model** | Nested, relational data in MongoDB with CRDT references |
| **Multi-Currency** | Real-world financial UX with forex handling |

---

*End of NomadSync Reference Guide*
