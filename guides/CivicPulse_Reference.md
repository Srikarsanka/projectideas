# 🏙️ CivicPulse — Hyperlocal Community Crisis & Collaboration Platform
### In-Depth Reference Guide

---

## 1. Project Vision

CivicPulse is a **real-time hyperlocal civic platform** — a geo-tagged incident reporting system, aid coordination engine, community governance tool, and volunteer network all in one. Built for India's urban-rural gaps where WhatsApp chaos and government bureaucracy leave citizens without coordinated help.

Think: *Nextdoor + FixMyStreet + Consul (Madrid's civic platform) — built for South Asia, by you.*

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENTS (Angular 17 PWA)                      │
│  Citizen │ Volunteer │ NGO │ Ward Official │ Platform Admin      │
│  Leaflet.js Map │ WebRTC (simple-peer) │ D3.js Analytics        │
│  Service Worker (offline) │ FCM Push Notifications              │
└──────────┬──────────────────────────────────────────────────────┘
           │ REST / WebSocket
           ▼
┌──────────────────────────────────────────────────────────────┐
│          Node.js / Express API Gateway (Port 80)             │
│   JWT Auth · Rate Limiting · Role Guard · Request Routing    │
└────┬──────────┬───────────┬──────────┬───────────┬───────────┘
     │          │           │          │           │
     ▼          ▼           ▼          ▼           ▼
┌─────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────┐
│  Alert  │ │Resource│ │Community│ │Analytics│ │AI Layer │
│ Service │ │Service │ │Service  │ │Service  │ │Service  │
│(Crisis, │ │(Aid,   │ │(Polls,  │ │(Heatmap,│ │(OpenAI  │
│ Geo)    │ │Matching│ │Govern.) │ │ Trends) │ │Triage)  │
└────┬────┘ └───┬────┘ └───┬─────┘ └────┬───┘ └────┬────┘
     │          │           │             │          │
     └──────────┴───────────┴─────────────┴──────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
     ┌─────────────┐ ┌───────────┐ ┌──────────────┐
     │  MongoDB    │ │  Redis    │ │  BullMQ      │
     │  (Geospatial│ │  Pub/Sub  │ │  - FCM fan-out│
     │   indexes)  │ │  Socket.io│ │  - AI jobs   │
     └─────────────┘ │  adapter  │ │  - Emails    │
                     └───────────┘ └──────┬───────┘
                                          │
                         ┌────────────────┼──────────┐
                         ▼                ▼          ▼
                  ┌────────────┐ ┌──────────────┐ ┌──────────┐
                  │ Firebase   │ │ OpenAI/Claude│ │Cloudinary│
                  │ Cloud Msg  │ │ API (Triage) │ │ (Photos) │
                  └────────────┘ └──────────────┘ └──────────┘
```

---

## 3. Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Angular 17 PWA + NgRx | 5-role dashboards, offline |
| Maps | Leaflet.js | Incident map with geo markers |
| Real-time | Socket.io | Geo-scoped incident broadcast |
| Video | WebRTC (simple-peer) | Crisis coordination rooms |
| Analytics | D3.js | Heatmaps, trend charts |
| Push | Firebase Cloud Messaging | Volunteer/citizen alerts |
| Backend | Node.js + Express | API Gateway + 5 microservices |
| Database | MongoDB (2dsphere) | Geospatial incident queries |
| Cache/PubSub | Redis | Socket.io adapter, hot data |
| Queue | BullMQ | Notifications, AI jobs, emails |
| AI | OpenAI / Claude API | Severity triage + categorization |
| Storage | Cloudinary | Incident photo evidence |
| DevOps | Docker + Docker Compose | Full local orchestration |

---

## 4. MongoDB Schema Design

### User Model
```js
{
  _id: ObjectId,
  name: String,
  email: String,
  phone: String,
  passwordHash: String,
  role: enum['citizen','volunteer','ngo','ward_official','admin'],
  isVerified: Boolean,
  ward: String,               // e.g., "Ward 47, Bengaluru"
  locality: String,
  location: { type: 'Point', coordinates: [lng, lat] },
  fcmToken: String,
  reputationScore: Number,    // Weighted trust score
  availability: Boolean,      // Volunteers only
  skills: [String],
  createdAt: Date
}
```

### Incident (Report) Model
```js
{
  _id: ObjectId,
  title: String,
  description: String,
  category: enum['road','water','medical','fire','flood','crime','other'],
  severity: enum['critical','high','medium','low'],
  aiSeverityScore: Number,    // 0-1 from OpenAI triage
  aiCategory: String,         // AI-assigned category
  images: [String],
  reporter: ObjectId,
  location: { type: 'Point', coordinates: [lng, lat] },
  address: String,
  ward: String,
  upvotes: [ObjectId],
  comments: [{
    user: ObjectId, text: String, createdAt: Date
  }],
  status: enum['open','in_progress','escalated','resolved','closed'],
  assignedTo: ObjectId,
  resolvedBy: ObjectId,
  escalatedAt: Date,
  resolvedAt: Date,
  statusHistory: [{ status: String, by: ObjectId, timestamp: Date }],
  notifiedRadius: Number,     // km radius used for socket broadcast
  createdAt: Date
}
```

### AidRequest Model
```js
{
  _id: ObjectId,
  requester: ObjectId,
  type: enum['food','medicine','transport','labour','shelter','other'],
  description: String,
  quantity: Number,
  urgency: enum['critical','high','medium'],
  location: { type: 'Point', coordinates: [lng, lat] },
  status: enum['open','matched','fulfilled','cancelled'],
  matchedVolunteer: ObjectId,
  matchedAt: Date,
  fulfilledAt: Date,
  incident: ObjectId,         // Optional link to incident
  createdAt: Date
}
```

### Proposal (Governance) Model
```js
{
  _id: ObjectId,
  title: String,
  description: String,
  proposer: ObjectId,
  ward: String,
  category: enum['infrastructure','environment','safety','social'],
  status: enum['draft','open','quorum_reached','passed','rejected','implemented'],
  votes: [{
    user: ObjectId,
    vote: enum['yes','no','abstain'],
    timestamp: Date
  }],
  quorumRequired: Number,     // Minimum votes to proceed
  deadline: Date,
  resourcePledges: [{
    user: ObjectId, amount: Number, type: String
  }],
  linkedOfficer: ObjectId,    // Ward official assigned
  officialResponse: String,
  createdAt: Date
}
```

### VolunteerTask Model
```js
{
  _id: ObjectId,
  volunteer: ObjectId,
  aidRequest: ObjectId,
  incident: ObjectId,
  status: enum['assigned','accepted','en_route','completed','abandoned'],
  assignedAt: Date, acceptedAt: Date, completedAt: Date,
  completionNote: String,
  completionPhoto: String,
  responseTimeMinutes: Number,
  rating: Number              // Citizen rates volunteer (1-5)
}
```

### ReputationEvent Model
```js
{
  _id: ObjectId,
  user: ObjectId,
  eventType: enum['task_completed','accurate_report','fast_response',
                  'false_report','task_abandoned'],
  points: Number,             // +/- weighted points
  relatedEntity: ObjectId,
  timestamp: Date
}
```

---

## 5. REST API Design

### Auth & Users
```
POST  /auth/register
POST  /auth/login
POST  /auth/refresh
GET   /auth/me
PUT   /auth/fcm-token
PUT   /auth/availability      → Volunteer toggle
GET   /users/:id/reputation   → Public reputation score
```

### Alert Service (Incidents)
```
GET   /incidents              → List (filter: ward, severity, category, status)
POST  /incidents              → Create with photo upload
GET   /incidents/:id          → Detail + comments
POST  /incidents/:id/upvote
POST  /incidents/:id/comment
PUT   /incidents/:id/status   → Official/volunteer update
GET   /incidents/nearby?lat&lng&km → Geospatial $near query
GET   /incidents/ward/:ward   → All in a ward
```

### Resource Service (Aid)
```
GET   /aid                    → List open requests near user
POST  /aid                    → Create aid request
PUT   /aid/:id/status
POST  /aid/match/:id          → Trigger matching algorithm
GET   /resources              → NGO resource inventory
POST  /resources              → Add resource stock
PUT   /resources/:id
```

### Community Service (Governance)
```
GET   /proposals              → All (filter by ward, status)
POST  /proposals              → Submit proposal
GET   /proposals/:id
POST  /proposals/:id/vote     → Cast vote
POST  /proposals/:id/pledge   → Pledge resource
PUT   /proposals/:id/response → Ward official official response
GET   /proposals/:id/report   → Generate governance PDF
```

### Analytics Service
```
GET   /analytics/heatmap?ward=&category=  → Incident density data
GET   /analytics/resolution-rates         → By ward, by category
GET   /analytics/volunteer-response       → Avg response times
GET   /analytics/trends?days=30           → Time-series trends
GET   /analytics/top-volunteers           → Leaderboard
```

### AI Service
```
POST  /ai/triage              → Analyse text + image → severity + category
POST  /ai/summarise/:incidentId → Generate incident summary
```

---

## 6. Socket.io — Geo-Scoped Rooms

The key design: **rooms are keyed by ward or geohash cell**, not individual users. Every incident update broadcasts only to users in the same geographic area.

```js
// Server: join locality room on connect
socket.on('join:locality', ({ ward, lat, lng }) => {
  socket.join(`ward:${ward}`);
  // Also join a 2km geohash cell for finer granularity
  const cell = geohash.encode(lat, lng, 5); // ~5km precision
  socket.join(`geo:${cell}`);
});

// Broadcast new incident to ward room
async function broadcastIncident(incident) {
  io.to(`ward:${incident.ward}`).emit('incident:new', incident);
}

// Broadcast to geo radius (multiple cells)
async function broadcastToRadius(lat, lng, radiusKm, event, data) {
  const cells = geohash.neighbors(geohash.encode(lat, lng, 5));
  cells.forEach(cell => io.to(`geo:${cell}`).emit(event, data));
}
```

### Socket Events Reference

| Direction | Event | Payload |
|-----------|-------|---------|
| C→S | `join:locality` | `{ ward, lat, lng }` |
| S→Ward | `incident:new` | `{ incident }` |
| S→Ward | `incident:updated` | `{ incidentId, status }` |
| S→Ward | `incident:escalated` | `{ incident }` |
| S→Volunteer | `task:assigned` | `{ task, aidRequest }` |
| S→Ward | `proposal:vote` | `{ proposalId, counts }` |
| C→S | `crisis:join` | `{ incidentId }` — WebRTC room |
| S→Crisis | `peer:signal` | `{ from, signal }` — WebRTC |

---

## 7. BullMQ — Job Queues

```js
// Queues defined
const notificationQueue = new Queue('notifications', { connection: redis });
const aiTriageQueue     = new Queue('ai-triage', { connection: redis });
const emailQueue        = new Queue('emails', { connection: redis });
const escalationQueue   = new Queue('escalation', { connection: redis });

// On new incident → add AI triage job + notification job
async function onIncidentCreated(incident) {
  await aiTriageQueue.add('triage', { incidentId: incident._id });
  await notificationQueue.add('fan-out', {
    type: 'NEW_INCIDENT', ward: incident.ward, incidentId: incident._id
  }, { priority: incident.severity === 'critical' ? 1 : 5 });
  await escalationQueue.add('check', { incidentId: incident._id },
    { delay: 30 * 60 * 1000 }); // 30 min escalation threshold
}
```

### Workers
```js
// AI Triage Worker
new Worker('ai-triage', async (job) => {
  const incident = await Incident.findById(job.data.incidentId);
  const result = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{
      role: 'user',
      content: `Classify this civic incident. Return JSON with:
        severity (critical/high/medium/low), category, summary (1 sentence).
        Title: ${incident.title}. Description: ${incident.description}`
    }],
    response_format: { type: 'json_object' }
  });
  const { severity, category, summary } = JSON.parse(result.choices[0].message.content);
  await Incident.findByIdAndUpdate(incident._id, {
    aiSeverityScore: { critical:1, high:0.75, medium:0.5, low:0.25 }[severity],
    aiCategory: category, aiSummary: summary, severity
  });
  io.to(`ward:${incident.ward}`).emit('incident:updated', { incidentId: incident._id, severity, category });
}, { connection: redis });

// Notification Fan-out Worker
new Worker('notifications', async (job) => {
  const { type, ward, incidentId } = job.data;
  const users = await User.find({ ward, fcmToken: { $exists: true } });
  const tokens = users.map(u => u.fcmToken);
  for (let i = 0; i < tokens.length; i += 500) {
    await admin.messaging().sendEachForMulticast({
      tokens: tokens.slice(i, i+500),
      notification: { title: '🚨 New incident in your area', body: type }
    });
  }
}, { connection: redis });
```

---

## 8. Reputation Score Algorithm

```js
const WEIGHTS = {
  task_completed:    +10,
  accurate_report:   +5,
  fast_response:     +8,   // Responded in < 15 min
  task_abandoned:    -15,
  false_report:      -20,
  citizen_rating:    (r) => (r - 3) * 4  // Rating 1-5 → -8 to +8
};

async function updateReputation(userId, eventType, meta = {}) {
  let points = typeof WEIGHTS[eventType] === 'function'
    ? WEIGHTS[eventType](meta.rating)
    : WEIGHTS[eventType];

  await ReputationEvent.create({ user: userId, eventType, points });

  // Recalculate weighted rolling score
  const events = await ReputationEvent.find({ user: userId })
    .sort({ timestamp: -1 }).limit(50);

  // Recent events weighted more (decay factor)
  const score = events.reduce((sum, e, i) => {
    const decay = Math.pow(0.95, i);
    return sum + (e.points * decay);
  }, 0);

  await User.findByIdAndUpdate(userId, {
    reputationScore: Math.max(0, Math.round(score))
  });
}
```

---

## 9. Community Governance Module

```js
// Vote on proposal with quorum check
async function castVote(proposalId, userId, vote) {
  const proposal = await Proposal.findById(proposalId);
  if (new Date() > proposal.deadline) throw new Error('Voting closed');

  // Upsert vote (one vote per user)
  const existing = proposal.votes.find(v => v.user.equals(userId));
  if (existing) existing.vote = vote;
  else proposal.votes.push({ user: userId, vote, timestamp: new Date() });

  // Check quorum
  const yesCnt = proposal.votes.filter(v => v.vote === 'yes').length;
  if (yesCnt >= proposal.quorumRequired) {
    proposal.status = 'quorum_reached';
    // Notify linked ward official
    await notificationQueue.add('official-alert', {
      officialId: proposal.linkedOfficer,
      proposalId: proposal._id,
      message: `Proposal "${proposal.title}" reached quorum in your ward`
    });
  }
  await proposal.save();
  io.to(`ward:${proposal.ward}`).emit('proposal:vote', {
    proposalId, counts: { yes: yesCnt, total: proposal.votes.length }
  });
}
```

---

## 10. WebRTC Crisis Rooms (simple-peer)

```js
// Server: signalling relay for WebRTC
socket.on('crisis:join', ({ incidentId }) => {
  socket.join(`crisis:${incidentId}`);
  socket.to(`crisis:${incidentId}`).emit('peer:new', { from: socket.id });
});

socket.on('peer:signal', ({ to, signal }) => {
  io.to(to).emit('peer:signal', { from: socket.id, signal });
});

// Angular client
import SimplePeer from 'simple-peer';

function joinCrisisRoom(incidentId) {
  socket.emit('crisis:join', { incidentId });
  socket.on('peer:new', ({ from }) => {
    const peer = new SimplePeer({ initiator: true, stream: localStream });
    peer.on('signal', signal => socket.emit('peer:signal', { to: from, signal }));
    peer.on('stream', stream => addRemoteVideo(stream));
  });
  socket.on('peer:signal', ({ from, signal }) => {
    const peer = new SimplePeer({ stream: localStream });
    peer.signal(signal);
    peer.on('stream', stream => addRemoteVideo(stream));
  });
}
```

---

## 11. Angular Frontend Structure

```
src/app/
├── core/
│   ├── guards/         → AuthGuard, RoleGuard(['ngo','admin'])
│   ├── interceptors/   → JWT, offline detector
│   └── services/
│       ├── auth.service.ts
│       ├── socket.service.ts
│       ├── geo.service.ts
│       └── offline.service.ts
│
├── store/              → NgRx
│   ├── incidents/      → actions, effects, reducer, selectors
│   ├── aid/
│   ├── proposals/
│   └── auth/
│
├── modules/
│   ├── map/            → Leaflet crisis map (ALL roles see this)
│   │   ├── incident-marker/
│   │   ├── filter-panel/
│   │   └── map.component.ts
│   │
│   ├── citizen/        → Lazy loaded
│   │   ├── report-incident/
│   │   ├── my-reports/
│   │   └── proposals/
│   │
│   ├── volunteer/      → Lazy loaded
│   │   ├── task-board/
│   │   ├── task-detail/
│   │   └── reputation/
│   │
│   ├── ngo/            → Lazy loaded
│   │   ├── resource-manager/
│   │   ├── bulk-tasks/
│   │   └── campaigns/
│   │
│   ├── official/       → Lazy loaded
│   │   ├── ward-dashboard/
│   │   ├── escalated-alerts/
│   │   └── governance-inbox/
│   │
│   └── admin/          → Lazy loaded
│       ├── analytics/  → D3.js heatmaps
│       ├── users/
│       └── moderation/
│
└── shared/
    ├── offline-banner/
    ├── severity-badge/
    ├── incident-card/
    └── crisis-room/    → WebRTC component
```

---

## 12. Offline PWA Setup

```json
// ngsw-config.json
{
  "index": "/index.html",
  "assetGroups": [{
    "name": "app",
    "installMode": "prefetch",
    "resources": { "files": ["/favicon.ico", "/*.css", "/*.js"] }
  }],
  "dataGroups": [{
    "name": "api-incidents",
    "urls": ["/incidents*"],
    "cacheConfig": {
      "strategy": "freshness",
      "maxSize": 200,
      "maxAge": "1h",
      "timeout": "3s"
    }
  }]
}
```

```js
// Offline action queue (IndexedDB via Dexie)
async function submitIncident(data) {
  if (!navigator.onLine) {
    await offlineDB.pendingReports.add({ data, timestamp: Date.now() });
    return { queued: true };
  }
  return api.post('/incidents', data);
}
window.addEventListener('online', async () => {
  const pending = await offlineDB.pendingReports.toArray();
  for (const p of pending) {
    await api.post('/incidents', p.data);
    await offlineDB.pendingReports.delete(p.id);
  }
});
```

---

## 13. D3.js Analytics Heatmap

```ts
// Angular analytics component
import * as d3 from 'd3';

drawHeatmap(data: IncidentDensity[]) {
  const svg = d3.select('#heatmap').append('svg').attr('width', 800).attr('height', 500);
  const color = d3.scaleSequential(d3.interpolateOrRd)
    .domain([0, d3.max(data, d => d.count)!]);

  svg.selectAll('rect').data(data).enter().append('rect')
    .attr('x', d => d.col * 20)
    .attr('y', d => d.row * 20)
    .attr('width', 18).attr('height', 18)
    .attr('fill', d => color(d.count))
    .append('title').text(d => `${d.ward}: ${d.count} incidents`);
}
```

---

## 14. Development Phases (6 Months)

### Month 1 — Foundation
- [ ] Angular PWA + NgRx setup, 5 lazy modules
- [ ] Node.js API Gateway + Auth Service (JWT + RBAC)
- [ ] MongoDB models + 2dsphere indexes
- [ ] RoleGuard: route protection per role
- [ ] Leaflet.js map: plot static incidents
- [ ] Docker Compose: MongoDB + Redis + all services

### Month 2 — Real-Time Core
- [ ] Socket.io geo-scoped rooms (ward-based)
- [ ] Live incident broadcast on new report
- [ ] Citizen: SOS form → appears on map live
- [ ] Upvote / comment on incidents
- [ ] FCM token registration in Angular

### Month 3 — Queues & Notifications
- [ ] BullMQ: notification fan-out worker
- [ ] BullMQ: escalation timer jobs
- [ ] FCM batch send to ward users
- [ ] Aid request + volunteer matching ($near)
- [ ] VolunteerTask lifecycle (assign → accept → complete)

### Month 4 — AI Triage
- [ ] Cloudinary photo upload on incident form
- [ ] BullMQ: AI triage job on incident create
- [ ] OpenAI/Claude API severity classification
- [ ] UI: show AI-assigned severity badge
- [ ] Reputation score system + events

### Month 5 — Governance + WebRTC
- [ ] Proposal CRUD + voting + quorum logic
- [ ] Ward official governance inbox
- [ ] PDF report generation (pdfmake)
- [ ] WebRTC crisis room (simple-peer)
- [ ] D3.js analytics heatmap (admin)

### Month 6 — Polish & Deploy
- [ ] Offline PWA: ngsw-config + IndexedDB queue
- [ ] Winston structured logging
- [ ] Rate limiting + input validation (Zod)
- [ ] Docker multi-stage production builds
- [ ] Deploy: Railway / Render + MongoDB Atlas
- [ ] README + architecture diagram + demo video

---

## 15. Environment Variables

```env
# API Gateway
PORT=3000
JWT_SECRET=...
JWT_REFRESH_SECRET=...

# Alert Service
PORT=5001
MONGO_URI=mongodb://localhost:27017/civicpulse
REDIS_URL=redis://localhost:6379

# Community Service
PORT=5002
MONGO_URI=mongodb://localhost:27017/civicpulse

# Analytics Service
PORT=5003
MONGO_URI=mongodb://localhost:27017/civicpulse

# AI Service
PORT=5004
OPENAI_API_KEY=sk-...

# All services share:
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...
FIREBASE_PROJECT_ID=...
GOOGLE_APPLICATION_CREDENTIALS=./gcloud-key.json

# Angular
NG_APP_API_URL=http://localhost:3000
NG_APP_SOCKET_URL=http://localhost:5001
NG_APP_MAPBOX_TOKEN=pk.eyJ1...
NG_APP_FIREBASE_CONFIG={"apiKey":"..."}
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
  gateway:
    build: ./gateway
    ports: ["3000:3000"]
    depends_on: [mongodb, redis]
  alert-service:
    build: ./services/alert
    ports: ["5001:5001"]
    depends_on: [mongodb, redis]
  resource-service:
    build: ./services/resource
    ports: ["5002:5002"]
    depends_on: [mongodb, redis]
  community-service:
    build: ./services/community
    ports: ["5003:5003"]
    depends_on: [mongodb]
  analytics-service:
    build: ./services/analytics
    ports: ["5004:5004"]
    depends_on: [mongodb]
  ai-service:
    build: ./services/ai
    ports: ["5005:5005"]
  frontend:
    build: ./frontend
    ports: ["4200:4200"]
volumes:
  mongo_data:
```

---

## 17. Why Interviewers Will Be Impressed

| Concept | CivicPulse Implementation |
|---------|--------------------------|
| **Geospatial DB** | MongoDB 2dsphere, `$near`, `$geoWithin` queries |
| **Geo-scoped Real-time** | Socket.io rooms keyed by ward / geohash cell |
| **Multi-tenant RBAC** | 5 roles, route guards, server-side enforcement |
| **Queue Architecture** | BullMQ fan-out — the #1 backend pattern |
| **AI Pipeline** | OpenAI triage integrated into ingestion flow |
| **Offline-First PWA** | Service Worker + IndexedDB sync queue |
| **Graph Algorithm** | Reputation score with decay weighting |
| **Governance Engine** | Quorum voting — mirrors real civic tech (Consul) |
| **WebRTC** | P2P crisis coordination rooms |
| **D3.js Analytics** | MongoDB aggregation → visual heatmaps |
| **Social Impact** | Problem every interviewer understands instantly |

---

*End of CivicPulse Reference Guide*
