# 🆘 ReliefLink — Disaster Response Coordination Platform
### In-Depth Reference Guide

---

## 1. Project Vision

ReliefLink is a **real-time, offline-capable Progressive Web App** that coordinates disaster relief — mapping SOS requests to volunteers and resources. Citizens report needs, volunteers accept tasks, NGOs manage supply chains, and admins oversee everything on a live crisis map.

Think: *Google Crisis Response + Ushahidi + Splitwise for resources — built by you.*

---

## 2. System Architecture

```
┌──────────────────────────────────────────────────────────┐
│               CLIENTS (Angular PWA)                       │
│  Citizen │ Volunteer │ NGO Coordinator │ Admin            │
│  Service Worker + Workbox (offline caching)               │
│  IndexedDB (offline queue) + Mapbox GL                    │
└───────┬──────────────────────────────┬────────────────────┘
        │ REST / WebSocket             │ FCM Push
        ▼                              ▼
┌──────────────────┐         ┌──────────────────────┐
│   API Gateway    │         │  Firebase Cloud      │
│   (Nginx)        │         │  Messaging           │
└──────┬───────────┘         └──────────────────────┘
       │
  ┌────┴────────────────────────────────┐
  ▼                                     ▼
┌────────────────────┐      ┌────────────────────────┐
│  Crisis Service    │      │  Matching Service      │
│  (Requests, SOS,  │      │  (Auto-assign nearest  │
│   Geofences)      │      │   volunteer/resource)  │
└────────┬───────────┘      └───────────┬────────────┘
         │                              │
         ▼                              ▼
┌────────────────────────────────────────────────────┐
│                   MongoDB                           │
│  Users │ Requests │ Resources │ Tasks │ Incidents  │
│  (2dsphere geospatial indexes)                     │
└───────────────────────┬────────────────────────────┘
                        │
           ┌────────────┼──────────────┐
           ▼            ▼              ▼
┌──────────────┐ ┌────────────┐ ┌───────────────────┐
│  Redis       │ │  BullMQ    │ │  Cloud Storage    │
│  - Pub/Sub   │ │  - Image   │ │  (Damage photos)  │
│  - Crisis    │ │    analysis│ │                   │
│    state     │ │  - FCM     │ │                   │
│  - Live locs │ │    batch   │ └───────────────────┘
└──────────────┘ └──────┬─────┘
                        │
                        ▼
              ┌─────────────────────┐
              │ Google Vision API   │
              │ (Damage severity    │
              │  classification)    │
              └─────────────────────┘
```

---

## 3. Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Angular 17 PWA | Offline-capable, role dashboards |
| Maps | Mapbox GL JS | Crisis map with geospatial markers |
| Offline | Workbox + IndexedDB | Service worker caching strategies |
| State | NgRx | Complex multi-role state |
| Push | Firebase Cloud Messaging | Volunteer notifications |
| Backend | Node.js + Express | Microservices |
| Real-time | Socket.io + Redis | Live map updates |
| Database | MongoDB (geospatial) | 2dsphere indexed queries |
| Cache | Redis | Crisis state, live locations |
| Queue | BullMQ | Image jobs, FCM batch |
| AI | Google Vision API | Damage image classification |
| Storage | Cloudinary / GCS | Photo uploads |
| DevOps | Docker + Nginx | Containerized services |

---

## 4. MongoDB Schema Design

### User Model
```js
{
  _id: ObjectId,
  name: String,
  phone: String,
  email: String,
  passwordHash: String,
  role: enum['citizen','volunteer','ngo','admin'],
  isVerified: Boolean,
  skills: [String],           // ['medical','rescue','driving']
  fcmToken: String,           // Firebase push token
  location: {
    type: 'Point',
    coordinates: [lng, lat]   // 2dsphere index
  },
  availability: Boolean,      // Volunteer online/offline toggle
  tasksCompleted: Number,
  createdAt: Date
}
```

### HelpRequest Model
```js
{
  _id: ObjectId,
  type: enum['shelter','food','medical','rescue','water','other'],
  urgency: enum['critical','high','medium','low'],
  title: String,
  description: String,
  requester: ObjectId (ref: User),
  location: {
    type: 'Point',
    coordinates: [lng, lat]
  },
  address: String,
  numberOfPeople: Number,
  images: [String],           // Cloudinary URLs
  damageScore: Number,        // 0-1, from Google Vision
  damageSeverity: enum['none','minor','moderate','severe','critical'],
  status: enum['open','assigned','in_progress','fulfilled','escalated','closed'],
  assignedTo: ObjectId (ref: User),
  assignedAt: Date,
  fulfilledAt: Date,
  escalationThreshold: Number, // minutes before auto-escalate
  escalatedAt: Date,
  statusHistory: [{
    status: String,
    by: ObjectId,
    timestamp: Date,
    note: String
  }],
  createdAt: Date
}
```

### Resource Model
```js
{
  _id: ObjectId,
  name: String,               // "Food Packets", "Blankets"
  category: enum['food','water','medical','shelter','clothing','other'],
  quantity: Number,
  unit: String,               // 'kg', 'units', 'liters'
  location: {
    type: 'Point',
    coordinates: [lng, lat]
  },
  warehouseName: String,
  managedBy: ObjectId (ref: User),  // NGO user
  incomingShipments: [{
    source: String,
    quantity: Number,
    expectedAt: Date,
    status: enum['en_route','arrived']
  }],
  lastUpdated: Date
}
```

### Task Model
```js
{
  _id: ObjectId,
  request: ObjectId (ref: HelpRequest),
  volunteer: ObjectId (ref: User),
  status: enum['assigned','accepted','en_route','arrived','completed','abandoned'],
  assignedAt: Date,
  acceptedAt: Date,
  completedAt: Date,
  volunteerLocation: {
    type: 'Point',
    coordinates: [lng, lat]
  },
  eta: Number,                // minutes
  completionNote: String,
  completionPhoto: String
}
```

### Incident (Crisis Event) Model
```js
{
  _id: ObjectId,
  name: String,               // "Kerala Floods 2024"
  type: enum['flood','earthquake','cyclone','fire','other'],
  severity: enum['low','medium','high','critical'],
  affectedZone: {
    type: 'Polygon',
    coordinates: [[[lng,lat]]]  // GeoJSON polygon
  },
  center: { type: 'Point', coordinates: [lng, lat] },
  radius: Number,             // km
  status: enum['active','monitoring','closed'],
  coordinators: [ObjectId],
  startedAt: Date,
  closedAt: Date
}
```

---

## 5. REST API Design

### Auth Service
```
POST /auth/register
POST /auth/login
POST /auth/refresh
GET  /auth/me
PUT  /auth/fcm-token          → Update Firebase push token
PUT  /auth/availability       → Volunteer toggle online/offline
```

### Crisis Service
```
# Help Requests
GET    /requests                     → List (filter: type, urgency, status, near)
POST   /requests                     → Citizen: raise SOS
GET    /requests/:id                 → Detail
PUT    /requests/:id/status          → Update status
POST   /requests/:id/images          → Upload damage photos
GET    /requests/nearby?lat&lng&km   → Geospatial query

# Incidents
GET    /incidents                    → Active incidents
POST   /incidents                    → Admin: declare incident
PUT    /incidents/:id                → Update incident zone
GET    /incidents/:id/requests       → All requests in incident zone
GET    /incidents/:id/analytics      → Response time stats
```

### Matching Service
```
POST   /match/request/:id        → Auto-match nearest volunteer to request
POST   /match/bulk               → NGO: bulk assign tasks
GET    /match/suggestions/:id    → Get top 5 volunteer suggestions
```

### Volunteer Service
```
GET    /tasks/mine               → My assigned tasks
PUT    /tasks/:id/accept         → Accept task
PUT    /tasks/:id/status         → Update task status (en_route, arrived, done)
POST   /tasks/:id/location       → Driver: share live location during task
```

### Resource Service
```
GET    /resources                → All resources (filter by category, near)
POST   /resources                → NGO: add resource
PUT    /resources/:id            → Update quantity
POST   /resources/:id/shipment   → Log incoming supply truck
GET    /resources/allocate/:reqId → Auto-suggest best resource for request
```

---

## 6. Socket.io Events

### Client → Server
| Event | Payload | Description |
|-------|---------|-------------|
| `join:incident` | `{ incidentId }` | Join crisis room |
| `volunteer:location` | `{ lat, lng }` | Share location during task |
| `request:new` | `{ requestData }` | Citizen broadcasts SOS |

### Server → All in Incident Room
| Event | Payload | Description |
|-------|---------|-------------|
| `request:created` | `{ request }` | New SOS on map |
| `request:updated` | `{ requestId, status }` | Status change |
| `request:escalated` | `{ request }` | Auto-escalation alert |
| `volunteer:moved` | `{ volunteerId, lat, lng }` | Live volunteer position |
| `resource:updated` | `{ resource }` | Stock level changed |
| `task:assigned` | `{ task, volunteerId }` | Task dispatched |

### Server → Specific Volunteer
| Event | Payload | Description |
|-------|---------|-------------|
| `task:new` | `{ task, request }` | New task assigned |
| `task:cancelled` | `{ taskId }` | Task cancelled |

---

## 7. Resource Matching Engine

```js
async function matchNearestVolunteer(requestId) {
  const request = await HelpRequest.findById(requestId);
  const [lng, lat] = request.location.coordinates;

  // Find available volunteers within 10km using MongoDB 2dsphere
  const volunteers = await User.find({
    role: 'volunteer',
    availability: true,
    location: {
      $near: {
        $geometry: { type: 'Point', coordinates: [lng, lat] },
        $maxDistance: 10000  // meters
      }
    }
  }).limit(5);

  if (!volunteers.length) {
    // Escalate: no volunteers nearby
    await escalateRequest(request);
    return null;
  }

  // Pick closest (first result from $near is nearest)
  const best = volunteers[0];

  // Create task
  const task = await Task.create({
    request: requestId,
    volunteer: best._id,
    status: 'assigned',
    assignedAt: new Date()
  });

  // Notify volunteer via FCM
  await sendFCMNotification(best.fcmToken, {
    title: '🆘 New Task Assigned',
    body: `${request.type} request nearby — ${request.urgency} urgency`,
    data: { taskId: task._id.toString() }
  });

  // Notify room via Socket.io
  io.to(`incident:${request.incident}`).emit('task:assigned', { task, volunteerId: best._id });

  return task;
}
```

---

## 8. Auto-Escalation (BullMQ)

```js
// When a request is created, schedule escalation check
import { Queue, Worker } from 'bullmq';

const escalationQueue = new Queue('escalation', { connection: redis });

// Schedule check after threshold minutes
async function scheduleEscalation(request) {
  await escalationQueue.add('check', { requestId: request._id }, {
    delay: request.escalationThreshold * 60 * 1000
  });
}

// Worker: if still unfulfilled, escalate
new Worker('escalation', async (job) => {
  const req = await HelpRequest.findById(job.data.requestId);
  if (['open', 'assigned'].includes(req.status)) {
    req.status = 'escalated';
    req.escalatedAt = new Date();
    await req.save();

    // Push to NGO/Admin dashboards
    io.to('ngo-room').emit('request:escalated', req);
    io.to('admin-room').emit('request:escalated', req);

    // FCM to all NGO coordinators
    await batchFCMToRole('ngo', {
      title: '⚠️ Escalated Request',
      body: `${req.type} request unmet for ${req.escalationThreshold} mins`
    });
  }
}, { connection: redis });
```

---

## 9. AI Damage Assessment

```js
// BullMQ worker: process image after upload
import vision from '@google-cloud/vision';
const client = new vision.ImageAnnotatorClient();

new Worker('image-analysis', async (job) => {
  const { requestId, imageUrl } = job.data;

  // Call Google Vision API
  const [result] = await client.labelDetection(imageUrl);
  const labels = result.labelAnnotations.map(l => l.description.toLowerCase());

  // Score damage severity
  let score = 0;
  if (labels.some(l => l.includes('rubble') || l.includes('collapse'))) score += 0.4;
  if (labels.some(l => l.includes('flood') || l.includes('water'))) score += 0.3;
  if (labels.some(l => l.includes('fire') || l.includes('smoke'))) score += 0.4;
  if (labels.some(l => l.includes('debris'))) score += 0.2;
  score = Math.min(score, 1.0);

  const severity =
    score > 0.8 ? 'critical' :
    score > 0.6 ? 'severe' :
    score > 0.4 ? 'moderate' :
    score > 0.2 ? 'minor' : 'none';

  await HelpRequest.findByIdAndUpdate(requestId, { damageScore: score, damageSeverity: severity });

  // Notify room of updated severity
  io.to(`request:${requestId}`).emit('request:updated', { requestId, damageSeverity: severity });

}, { connection: redis });
```

---

## 10. Offline-First PWA

### Service Worker Strategy (Workbox)
```js
// sw.js (registered in Angular)
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';

// Cache map tiles (Mapbox) — CacheFirst
registerRoute(
  ({ url }) => url.hostname.includes('mapbox.com'),
  new CacheFirst({ cacheName: 'mapbox-tiles', plugins: [
    new ExpirationPlugin({ maxEntries: 500, maxAgeSeconds: 7 * 86400 })
  ]})
);

// API calls — NetworkFirst (fallback to cache)
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache' })
);

// Static assets — StaleWhileRevalidate
registerRoute(
  ({ request }) => request.destination === 'script',
  new StaleWhileRevalidate({ cacheName: 'static-assets' })
);
```

### Offline Action Queue (IndexedDB)
```js
// When offline, queue actions locally
async function submitRequest(data) {
  if (!navigator.onLine) {
    await offlineDB.pendingActions.add({
      type: 'CREATE_REQUEST', payload: data, createdAt: Date.now()
    });
    showBanner('Saved offline. Will sync when connected.');
    return;
  }
  await api.post('/requests', data);
}

// Background sync on reconnect
self.addEventListener('sync', async (event) => {
  if (event.tag === 'sync-pending') {
    event.waitUntil(syncPendingActions());
  }
});

async function syncPendingActions() {
  const actions = await offlineDB.pendingActions.toArray();
  for (const action of actions) {
    if (action.type === 'CREATE_REQUEST') {
      await api.post('/requests', action.payload);
    }
    await offlineDB.pendingActions.delete(action.id);
  }
}
```

---

## 11. Firebase Cloud Messaging (FCM)

```js
// Send to single device
async function sendFCMNotification(fcmToken, { title, body, data }) {
  await admin.messaging().send({
    token: fcmToken,
    notification: { title, body },
    data,
    android: { priority: 'high' },
    apns: { payload: { aps: { sound: 'default' } } }
  });
}

// Batch send to role (NGOs, Admins)
async function batchFCMToRole(role, notification) {
  const users = await User.find({ role, fcmToken: { $exists: true } });
  const tokens = users.map(u => u.fcmToken);

  // FCM allows max 500 per batch
  for (let i = 0; i < tokens.length; i += 500) {
    await admin.messaging().sendEachForMulticast({
      tokens: tokens.slice(i, i + 500),
      notification
    });
  }
}
```

---

## 12. Angular Frontend Structure

```
src/
├── app/
│   ├── core/
│   │   ├── guards/           → AuthGuard, RoleGuard
│   │   ├── interceptors/     → JWT, offline detector
│   │   └── services/
│   │       ├── auth.service.ts
│   │       ├── socket.service.ts
│   │       ├── fcm.service.ts
│   │       └── offline.service.ts
│   │
│   ├── store/                → NgRx
│   │   ├── requests/         → actions, reducer, effects, selectors
│   │   ├── resources/
│   │   ├── tasks/
│   │   └── incident/
│   │
│   ├── modules/
│   │   ├── crisis-map/       → Live Mapbox map (all roles)
│   │   │   ├── map.component.ts
│   │   │   ├── request-marker/
│   │   │   ├── volunteer-marker/
│   │   │   └── resource-marker/
│   │   │
│   │   ├── citizen/
│   │   │   ├── sos-form/     → Report need (offline-capable)
│   │   │   └── track-status/ → Watch request status
│   │   │
│   │   ├── volunteer/
│   │   │   ├── task-list/    → Assigned tasks
│   │   │   ├── task-detail/  → Navigation + status update
│   │   │   └── availability/ → Toggle online
│   │   │
│   │   ├── ngo/
│   │   │   ├── dashboard/    → All requests + stats
│   │   │   ├── resources/    → Manage supply
│   │   │   └── bulk-assign/  → Mass task dispatch
│   │   │
│   │   └── admin/
│   │       ├── incidents/    → Declare/manage incidents
│   │       ├── analytics/    → Response time, bottlenecks
│   │       └── users/        → Verify volunteers
│   │
│   └── shared/
│       ├── offline-banner/   → "You're offline" indicator
│       ├── urgency-badge/    → Color-coded urgency label
│       └── timeline/         → Status history
│
├── manifest.webmanifest      → PWA manifest
└── ngsw-config.json          → Angular service worker config
```

---

## 13. Development Phases

### Phase 1: Foundation (Week 1–2)
- [ ] Angular PWA setup (ng add @angular/pwa)
- [ ] Node.js services: Auth + Crisis
- [ ] MongoDB models + 2dsphere indexes
- [ ] JWT auth with role-based guards
- [ ] Docker Compose: MongoDB + Redis

### Phase 2: Crisis Map & Real-Time (Week 3–4)
- [ ] Mapbox GL JS integration in Angular
- [ ] Socket.io: incident rooms, live updates
- [ ] Citizen: SOS form → marker appears on map
- [ ] Color-coded urgency markers
- [ ] Volunteer location sharing via socket

### Phase 3: Matching & Tasks (Week 5)
- [ ] MongoDB $near geospatial matching
- [ ] Auto-assign nearest available volunteer
- [ ] Task CRUD (accept, update, complete)
- [ ] FCM: notify volunteer on task assign
- [ ] Volunteer mobile-responsive task view

### Phase 4: Offline PWA (Week 6)
- [ ] Workbox caching strategies
- [ ] IndexedDB offline action queue
- [ ] Background sync registration
- [ ] Offline SOS form + map tile caching
- [ ] Online/offline banner in Angular

### Phase 5: AI + Escalation (Week 7)
- [ ] Cloudinary photo upload on SOS form
- [ ] BullMQ: Google Vision image analysis job
- [ ] Update request severity from AI result
- [ ] BullMQ: escalation timer jobs
- [ ] FCM batch notifications on escalation

### Phase 6: NGO Supply Chain (Week 8)
- [ ] Resource CRUD (NGO role)
- [ ] Incoming shipment logging
- [ ] Auto-allocate resource to nearest critical request
- [ ] Supply stock level updates on map

### Phase 7: Analytics + Admin (Week 9)
- [ ] Post-incident analytics: avg response time
- [ ] Bottleneck detection: request type heatmap
- [ ] Admin: verify volunteers, manage incidents
- [ ] Export analytics report (CSV/PDF)

### Phase 8: Polish & DevOps (Week 10)
- [ ] Nginx API gateway + rate limiting
- [ ] Docker multi-stage builds
- [ ] PWA install prompt
- [ ] Input validation + error handling
- [ ] README + architecture diagram

---

## 14. Folder Structure

```
relieflink/
├── services/
│   ├── auth/
│   │   ├── src/
│   │   │   ├── models/User.js
│   │   │   ├── routes/auth.routes.js
│   │   │   ├── middleware/rbac.js
│   │   │   └── server.js
│   │   └── Dockerfile
│   │
│   ├── crisis/
│   │   ├── src/
│   │   │   ├── models/
│   │   │   ├── routes/
│   │   │   ├── controllers/
│   │   │   ├── socket/crisisSocket.js
│   │   │   ├── workers/
│   │   │   │   ├── escalationWorker.js
│   │   │   │   └── imageAnalysisWorker.js
│   │   │   └── server.js
│   │   └── Dockerfile
│   │
│   └── matching/
│       ├── src/
│       │   ├── engine/matchEngine.js
│       │   ├── routes/match.routes.js
│       │   └── server.js
│       └── Dockerfile
│
├── frontend/                   → Angular PWA
├── nginx.conf
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

  auth-service:
    build: ./services/auth
    ports: ["5001:5001"]
    depends_on: [mongodb]

  crisis-service:
    build: ./services/crisis
    ports: ["5002:5002"]
    depends_on: [mongodb, redis]

  matching-service:
    build: ./services/matching
    ports: ["5003:5003"]
    depends_on: [mongodb, redis]

  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    volumes: [./nginx.conf:/etc/nginx/nginx.conf]

  frontend:
    build: ./frontend
    ports: ["4200:4200"]

volumes:
  mongo_data:
```

---

## 16. Environment Variables

```env
# Auth Service
PORT=5001
MONGO_URI=mongodb://localhost:27017/relieflink
JWT_SECRET=...
JWT_REFRESH_SECRET=...

# Crisis Service
PORT=5002
REDIS_URL=redis://localhost:6379
GOOGLE_APPLICATION_CREDENTIALS=./gcloud-key.json
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...
FIREBASE_PROJECT_ID=...

# Matching Service
PORT=5003
MONGO_URI=mongodb://localhost:27017/relieflink
REDIS_URL=redis://localhost:6379

# Angular
NG_APP_API_URL=http://localhost:80
NG_APP_SOCKET_URL=http://localhost:5002
NG_APP_MAPBOX_TOKEN=pk.eyJ1...
NG_APP_FIREBASE_CONFIG={"apiKey":"..."}
```

---

## 17. Why This Impresses Interviewers

| Concept | Implementation in ReliefLink |
|---------|------------------------------|
| **Geospatial Queries** | MongoDB 2dsphere + $near for proximity matching |
| **Offline-First PWA** | Workbox caching + IndexedDB + Background Sync |
| **Real-Time Systems** | Socket.io crisis rooms with live map updates |
| **AI Integration** | Google Vision API for damage severity scoring |
| **Push Notifications** | Firebase Cloud Messaging with batch support |
| **Microservices** | 3 decoupled Node.js services + Nginx gateway |
| **Social Impact** | Real-world problem — makes interviewers care |
| **BullMQ Jobs** | Async escalation timers + image processing |
| **Resource Algorithms** | Geospatial matching of needs to supplies |

---

*End of ReliefLink Reference Guide*
