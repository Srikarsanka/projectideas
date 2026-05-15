# 🗺️ NomadSync — Real-Time Collaborative Trip Planning SaaS
### Built with MEAN Stack (MongoDB · Express · Angular 18 · Node.js)

---

## 1. Project Vision

NomadSync is a **real-time, collaborative trip planning platform** that solves the chaos of planning group travel. Imagine Google Docs + Notion + Splitwise + Google Maps — all fused into one beautifully designed application. Multiple users edit the same itinerary simultaneously, vote on activities, split expenses, and receive AI-powered destination suggestions — all in real-time, even with poor connectivity (offline-first with CRDTs).

Think: *Notion + Splitwise + Airbnb Experiences, built by you.*

---

## 2. System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                     CLIENTS (Angular 18)                       │
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

## 3. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | **Angular 18** | Production-grade SPA with strong typing |
| State Management | **Angular Services + RxJS** | Reactive streams, no extra library needed |
| Offline Sync | **Yjs (CRDT library)** | Conflict-free collaborative editing |
| Real-time | **Socket.io Client** | Presence, live edits, chat |
| Maps | **Mapbox GL JS** | Interactive maps with custom markers |
| Styling | **TailwindCSS** | Modern, utility-first |
| Drag & Drop | **Angular CDK DragDrop** | Built into Angular, zero extra dependency |
| Charts | **Chart.js** | Expense breakdown pie charts |
| Offline DB | **IndexedDB (Dexie.js)** | Client-side persistence |
| Backend | **Node.js + Express** | REST API server |
| Real-time | **Socket.io** | CRDT sync & presence |
| Database | **MongoDB + Mongoose** | Flexible nested documents |
| Cache | **Redis** | Presence, rate limits, sessions |
| Auth | **JWT + Refresh Tokens** | Stateless auth |
| AI | **OpenAI GPT-4** | Trip suggestions, itinerary gen |
| Travel Data | **Amadeus API** | Real flight/hotel pricing |
| Media | **Cloudinary** | Photo uploads |
| Email | **Nodemailer** | Trip invitations, reminders |
| DevOps | **Docker + Docker Compose** | Full local setup |

> **Angular vs React:** The backend, CRDT logic, Socket.io, and all algorithms are framework-agnostic. Angular's built-in DI, RxJS, and CDK make it a cleaner fit for a complex real-time app — no third-party state library needed.

---

## 4. Core Concept: CRDTs (Conflict-Free Replicated Data Types)

This is the most technically advanced part of NomadSync and the #1 portfolio differentiator.

### What Problem CRDTs Solve
When two users edit the same itinerary block simultaneously (one on WiFi, one on mobile with poor signal), traditional systems create conflicts. CRDTs mathematically guarantee that all edits **merge correctly without conflicts** — even if made offline.

### How We Use Yjs (inside an Angular Service)

```typescript
// src/app/crdt/crdt.service.ts
import { Injectable } from '@angular/core';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

@Injectable({ providedIn: 'root' })
export class CrdtService {
  private ydoc = new Y.Doc();
  private provider!: WebsocketProvider;

  connect(tripId: string): void {
    this.provider = new WebsocketProvider(
      'ws://localhost:1234',
      `trip-${tripId}`,
      this.ydoc
    );
  }

  getItinerary(): Y.Array<any> {
    return this.ydoc.getArray('itinerary');
  }

  getDayNotes(dayId: string): Y.Text {
    return this.ydoc.getText(`notes-${dayId}`);
  }

  destroy(): void {
    this.provider?.destroy();
    this.ydoc.destroy();
  }
}
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
    coordinates: { lat: Number, lng: Number }
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
  theme: String,
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
    startTime: String,
    endTime: String,
    duration: Number,          // minutes
    cost: Number,
    votes: [{ user: ObjectId, vote: enum['yes','no','maybe'] }],
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
  options: [{ id: String, text: String, votes: [ObjectId] }],
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

POST   /api/trips/:id/itinerary/days/:dayId/activities      → Add activity
PUT    /api/trips/:id/itinerary/days/:dayId/activities/:aid → Update activity
DELETE /api/trips/:id/itinerary/days/:dayId/activities/:aid → Remove
POST   /api/trips/:id/itinerary/days/:dayId/activities/:aid/vote → Vote
```

### Expense Routes
```
GET    /api/trips/:id/expenses          → All expenses
POST   /api/trips/:id/expenses          → Add expense
PUT    /api/trips/:id/expenses/:eid     → Update expense
DELETE /api/trips/:id/expenses/:eid     → Delete expense

GET    /api/trips/:id/expenses/summary  → Who owes whom
POST   /api/trips/:id/expenses/settle   → Mark settlement
```

### AI Routes
```
POST   /api/ai/suggest-activities   → AI activity suggestions for destination
POST   /api/ai/generate-itinerary   → Full AI-generated day plan
POST   /api/ai/budget-estimate      → AI cost estimate for trip
```

### Map Routes
```
GET    /api/map/search?q=...              → Place search (Mapbox)
GET    /api/map/route?from=&to=           → Route between places
GET    /api/map/flights?from=&to=&date=   → Flight prices (Amadeus)
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

### Angular Socket Service
```typescript
// src/app/services/socket.service.ts
import { Injectable } from '@angular/core';
import { Socket } from 'ngx-socket-io';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SocketService {
  constructor(private socket: Socket) {}

  joinTrip(tripId: string): void {
    this.socket.emit('trip:join', { tripId });
  }

  updatePresence(tripId: string, editingBlock: string): void {
    this.socket.emit('presence:update', { tripId, editingBlock });
  }

  onActivityAdded(): Observable<any> {
    return this.socket.fromEvent('activity:added');
  }

  onPresenceUpdate(): Observable<any> {
    return this.socket.fromEvent('presence:members');
  }
}
```

---

## 8. Expense Splitting Algorithm

### Equal Split
```typescript
function splitEqual(totalAmount: number, memberIds: string[]) {
  const perPerson = totalAmount / memberIds.length;
  return memberIds.map(userId => ({
    user: userId,
    amount: Math.round(perPerson * 100) / 100
  }));
}
```

### Settlement Calculation (Who Owes Whom)
```typescript
function calculateSettlements(expenses: any[], members: any[]) {
  // Step 1: Calculate net balance per person
  const balances: Record<string, number> = {};
  members.forEach(m => balances[m._id] = 0);

  expenses.forEach(expense => {
    balances[expense.paidBy] += expense.amount;
    expense.splits.forEach((split: any) => {
      balances[split.user] -= split.amount;
    });
  });

  // Step 2: Simplify debts (greedy algorithm)
  const creditors: { userId: string; amount: number }[] = [];
  const debtors: { userId: string; amount: number }[] = [];

  Object.entries(balances).forEach(([userId, bal]) => {
    if (bal > 0) creditors.push({ userId, amount: bal });
    if (bal < 0) debtors.push({ userId, amount: -bal });
  });

  const settlements: any[] = [];
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

```typescript
// src/app/services/ai.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AiService {
  constructor(private http: HttpClient) {}

  suggestActivities(destination: string, duration: number, preferences: string[]): Observable<any[]> {
    return this.http.post<any[]>('/api/ai/suggest-activities', {
      destination, duration, preferences
    });
  }

  generateItinerary(tripDetails: any): Observable<any> {
    return this.http.post<any>('/api/ai/generate-itinerary', tripDetails);
  }

  estimateBudget(tripDetails: any): Observable<any> {
    return this.http.post<any>('/api/ai/budget-estimate', tripDetails);
  }
}
```

---

## 10. Offline-First with IndexedDB

```typescript
// src/app/services/offline-db.service.ts
import { Injectable } from '@angular/core';
import Dexie from 'dexie';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class OfflineDbService extends Dexie {
  trips!: Dexie.Table<any, string>;
  itinerary!: Dexie.Table<any, string>;
  expenses!: Dexie.Table<any, string>;
  pendingOps!: Dexie.Table<any, number>;

  constructor(private http: HttpClient) {
    super('NomadSyncDB');
    this.version(1).stores({
      trips:      'id, title, updatedAt',
      itinerary:  'id, tripId, date',
      expenses:   'id, tripId, date',
      pendingOps: '++id, tripId, type, createdAt'
    });

    window.addEventListener('online', () => this.syncPendingOps());
  }

  async cacheTripData(trip: any): Promise<void> {
    await this.trips.put({ ...trip, cachedAt: Date.now() });
  }

  async queueOperation(tripId: string, type: string, payload: any): Promise<void> {
    await this.pendingOps.add({ tripId, type, payload, createdAt: Date.now() });
  }

  async syncPendingOps(): Promise<void> {
    const ops = await this.pendingOps.toArray();
    for (const op of ops) {
      await this.http.post(`/api/trips/${op.tripId}/sync`, op).toPromise();
      await this.pendingOps.delete(op.id);
    }
  }
}
```

---

## 11. Angular Project Structure

```
src/
├── app/
│   ├── pages/
│   │   ├── home/                   → Landing + my trips dashboard
│   │   ├── create-trip/            → New trip wizard
│   │   ├── trip-dashboard/         → Overview: members, budget, map
│   │   ├── itinerary/              → MAIN: day-by-day planner (CRDT)
│   │   ├── map-view/               → Interactive map of all activities
│   │   ├── expenses/               → Expense tracker + settlement
│   │   ├── polls/                  → Group voting interface
│   │   ├── trip-chat/              → Real-time group chat
│   │   └── profile/                → User settings
│   │
│   ├── components/
│   │   ├── itinerary/
│   │   │   ├── day-card/           → One day's activities (CDK draggable)
│   │   │   ├── activity-card/      → Single activity with vote buttons
│   │   │   ├── activity-form/      → Add/edit activity modal
│   │   │   └── timeline-view/      → Day view as timeline
│   │   ├── map/
│   │   │   ├── trip-map/           → Mapbox map with all places
│   │   │   ├── place-marker/       → Custom map pin
│   │   │   └── route-layer/        → Day route polyline
│   │   ├── expenses/
│   │   │   ├── expense-form/       → Add expense with split config
│   │   │   ├── expense-summary/    → Who owes whom table
│   │   │   └── expense-chart/      → Chart.js pie chart by category
│   │   ├── presence/
│   │   │   ├── online-avatars/     → Who's online in trip
│   │   │   └── editing-indicator/  → "Priya is editing Day 2..."
│   │   └── shared/
│   │       ├── invite-modal/       → Email invite + shareable link
│   │       └── currency-selector/
│   │
│   ├── services/
│   │   ├── auth.service.ts         → JWT login, register, refresh
│   │   ├── trip.service.ts         → Trip CRUD + HTTP calls
│   │   ├── socket.service.ts       → Socket.io wrapper (ngx-socket-io)
│   │   ├── crdt.service.ts         → Yjs document + provider
│   │   ├── ai.service.ts           → OpenAI API calls via backend
│   │   ├── mapbox.service.ts       → Place search + routing
│   │   ├── expense.service.ts      → Expense logic + settlement
│   │   └── offline-db.service.ts   → Dexie.js wrapper
│   │
│   ├── store/                      → Angular Signals or RxJS BehaviorSubjects
│   │   ├── trip.store.ts
│   │   ├── presence.store.ts
│   │   └── auth.store.ts
│   │
│   ├── guards/
│   │   └── auth.guard.ts
│   │
│   ├── interceptors/
│   │   └── jwt.interceptor.ts      → Auto-attach Bearer token
│   │
│   ├── models/
│   │   ├── trip.model.ts
│   │   ├── user.model.ts
│   │   ├── itinerary.model.ts
│   │   └── expense.model.ts
│   │
│   └── app.routes.ts               → Angular standalone routing
```

---

## 12. Angular ↔ Concept Mappings

| React/Other | Angular Equivalent |
|------------|-------------------|
| Zustand store | Angular Service + `BehaviorSubject` / Signals |
| React Context | `providedIn: 'root'` Injectable Service |
| `useEffect` | `ngOnInit` / `ngOnDestroy` lifecycle hooks |
| `useState` | `signal()` (Angular 17+) or class property |
| React Router | `@angular/router` with `app.routes.ts` |
| Axios interceptor | `HttpInterceptor` for JWT attachment |
| `@dnd-kit` | `@angular/cdk/drag-drop` (built-in) |
| Recharts | Chart.js with `ng2-charts` wrapper |
| `socket.io-client` hooks | `ngx-socket-io` Angular wrapper |

---

## 13. Development Phases

### Phase 1: Foundation (Week 1–2)
- [ ] Angular 18 + Express project setup
- [ ] MongoDB models: User, Trip, Itinerary, Expense
- [ ] JWT auth: register, login, refresh + `HttpInterceptor`
- [ ] Trip CRUD: create, list, detail, delete
- [ ] Member invite via email (Nodemailer)
- [ ] Join via invite link (token-based)

### Phase 2: Real-Time Collaboration (Week 3–4)
- [ ] Socket.io server with trip rooms
- [ ] `ngx-socket-io` Angular integration
- [ ] Yjs CRDT wired inside `CrdtService`
- [ ] Live presence system (who's editing what)
- [ ] Real-time activity add/edit/delete sync
- [ ] Angular CDK drag-drop reordering
- [ ] Collaborative day notes (Yjs Text type)

### Phase 3: Maps & Places (Week 5)
- [ ] Mapbox GL JS integration (vanilla JS inside Angular component)
- [ ] Place search via Mapbox Geocoding API
- [ ] Interactive map: markers per activity
- [ ] Route drawing between day activities
- [ ] Distance + travel time calculation

### Phase 4: Expenses & Splitting (Week 6)
- [ ] Expense form with split types (Reactive Forms)
- [ ] Settlement calculation algorithm
- [ ] Expense categories + Chart.js pie chart
- [ ] Multi-currency support
- [ ] Receipt photo upload (Cloudinary)

### Phase 5: AI & Polls (Week 7)
- [ ] OpenAI: activity suggestions per destination
- [ ] OpenAI: full itinerary generator
- [ ] OpenAI: budget estimate
- [ ] Poll/voting system for group decisions
- [ ] Amadeus API: flight price lookup

### Phase 6: Offline-First (Week 8)
- [ ] Dexie.js setup as Angular service
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
- [ ] Performance: `OnPush` change detection, lazy-loaded routes

---

## 14. Environment Variables

```env
# Server
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:4200

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

> **Note:** Angular dev server runs on port `4200` by default (not `5173` like Vite).

---

## 15. Full Folder Structure

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
├── frontend/                        ← Angular 18 project
│   ├── src/
│   │   ├── app/
│   │   │   ├── pages/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   ├── models/
│   │   │   └── app.routes.ts
│   │   ├── environments/
│   │   │   ├── environment.ts
│   │   │   └── environment.prod.ts
│   │   └── main.ts
│   ├── angular.json
│   ├── tailwind.config.js
│   └── Dockerfile
│
└── docker-compose.yml
```

---

## 16. Docker Compose

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
    ports: ["4200:4200"]       # Angular default port
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

## 17. Why This Impresses Interviewers

| Concept | Implementation in NomadSync |
|---------|---------------------------|
| **CRDTs** | Yjs wired inside Angular service — Google Docs level conflict resolution |
| **Offline-First** | Dexie.js as Angular service + sync queue — enterprise UX pattern |
| **Real-Time Presence** | Socket.io + RxJS Observables showing live editing state |
| **Graph Algorithm** | Expense settlement via greedy debt simplification algorithm |
| **AI Integration** | GPT-4 for context-aware travel suggestions |
| **Maps Engineering** | Mapbox routing, custom layers, geocoding |
| **Complex Data Model** | Nested relational data in MongoDB with CRDT references |
| **Angular Best Practices** | HttpInterceptor, OnPush CD, Guards, Reactive Forms, Signals |
| **Multi-Currency** | Real-world financial UX with forex handling |

---

## 18. Getting Started

```bash
# Clone the repo
git clone https://github.com/your-username/nomadsync.git
cd nomadsync

# Start all services with Docker
docker-compose up -d

# Or run manually:

# Backend
cd backend && npm install && npm run dev

# Frontend (Angular)
cd frontend && npm install && ng serve

# App runs at http://localhost:4200
```

---

*NomadSync — Built with MEAN Stack by Sanka Saran Sai Krishna Srikar*
