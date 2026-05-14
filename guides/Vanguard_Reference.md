# 🚛 Project Vanguard — Fleet & Logistics Orchestration Engine
### In-Depth Reference Guide

---

## 1. Project Vision

Vanguard is a **real-time fleet management SaaS** for small to mid-sized logistics companies. It tracks vehicles live on a map, auto-assigns deliveries, optimizes routes, triggers geo-fence alerts, and predicts delays using AI — all in one unified platform with role-specific dashboards for Dispatchers, Drivers, and Customers.

Think: *A lighter Uber Freight + Amazon Logistics dashboard, built by you.*

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              CLIENTS                                         │
│   Angular (Dispatcher) ─── Angular (Driver mobile view)     │
│   Angular (Customer portal)                                  │
└──────┬────────────────────────────────┬─────────────────────┘
       │ REST / WebSocket               │ REST
       ▼                                ▼
┌─────────────────┐          ┌──────────────────────┐
│  API Gateway    │          │   Auth Microservice   │
│  (Nginx/Express)│          │   (JWT + RBAC)        │
└────────┬────────┘          └──────────────────────┘
         │
    ┌────┴───────────────────────────────┐
    ▼                                    ▼
┌───────────────────┐        ┌───────────────────────┐
│ Tracking Service  │        │  Dispatch Service     │
│ (Socket.io)       │        │  (Orders, Routes,     │
│ Redis GEO         │        │   Assignments)        │
└────────┬──────────┘        └──────────┬────────────┘
         │                              │
         ▼                              ▼
┌────────────────────────────────────────────────────┐
│                   MongoDB                           │
│  Users │ Vehicles │ Orders │ Routes │ Geofences    │
└────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────┐     ┌──────────────────────┐
│  Redis             │     │  Python ML Service   │
│  - GEO (live loc)  │     │  (Delay Prediction)  │
│  - Pub/Sub         │     │  Flask + scikit-learn│
│  - Session cache   │     └──────────────────────┘
└────────────────────┘
         │
    ┌────┴──────────────────┐
    ▼                       ▼
┌──────────────┐   ┌────────────────────┐
│ Mapbox API   │   │  Bull Queue        │
│ (Routes,     │   │  (Offline sync,    │
│  Geocoding)  │   │   Notifications)   │
└──────────────┘   └────────────────────┘
```

---

## 3. Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Angular 17 + NgRx | Complex state, role dashboards |
| Maps | Leaflet.js | Vehicle tracking map |
| Mobile UI | Angular (responsive) | Driver mobile view |
| API Gateway | Nginx | Route to microservices |
| Auth Service | Node.js + Express | JWT, RBAC |
| Tracking Service | Node.js + Socket.io | Real-time location |
| Dispatch Service | Node.js + Express | Orders, assignment |
| ML Service | Python + Flask | Delay prediction |
| Database | MongoDB | Persistent data |
| Cache | Redis GEO + Pub/Sub | Live locations |
| Queue | Bull + Redis | Offline sync jobs |
| Maps API | Mapbox / Google Maps | Route optimization |
| DevOps | Docker + Docker Compose | Microservice orchestration |

---

## 4. MongoDB Schema Design

### User Model
```js
{
  _id: ObjectId,
  name: String,
  email: String,
  passwordHash: String,
  role: enum['dispatcher', 'driver', 'customer', 'admin'],
  phone: String,
  assignedVehicle: ObjectId,    // Driver only
  createdAt: Date
}
```

### Vehicle Model
```js
{
  _id: ObjectId,
  plateNumber: String,
  type: enum['bike','van','truck'],
  capacity: Number,             // kg
  driver: ObjectId (ref: User),
  status: enum['idle','en_route','at_warehouse','offline'],
  currentLocation: {
    type: 'Point',
    coordinates: [lng, lat]
  },
  fuelLevel: Number,
  lastSeen: Date
}
```

### Order Model
```js
{
  _id: ObjectId,
  trackingId: String,           // Customer-facing code
  customer: ObjectId (ref: User),
  pickup: {
    address: String,
    coordinates: [lng, lat],
    warehouseId: ObjectId
  },
  dropoff: {
    address: String,
    coordinates: [lng, lat]
  },
  weight: Number,
  priority: enum['standard','express','same_day'],
  status: enum['pending','assigned','picked_up','in_transit','delivered','failed'],
  assignedVehicle: ObjectId,
  assignedDriver: ObjectId,
  estimatedDelivery: Date,
  actualDelivery: Date,
  deliveryProof: String,        // Photo URL
  statusHistory: [{
    status: String,
    timestamp: Date,
    coordinates: [lng, lat],
    note: String
  }],
  createdAt: Date
}
```

### Route Model
```js
{
  _id: ObjectId,
  vehicle: ObjectId,
  driver: ObjectId,
  orders: [ObjectId],
  waypoints: [{
    orderId: ObjectId,
    coordinates: [lng, lat],
    address: String,
    estimatedArrival: Date,
    actualArrival: Date,
    status: enum['pending','completed','skipped']
  }],
  totalDistance: Number,        // km
  totalDuration: Number,        // minutes
  optimizedPolyline: String,    // Encoded route
  startedAt: Date,
  completedAt: Date
}
```

### Geofence Model
```js
{
  _id: ObjectId,
  name: String,                 // "Warehouse A"
  type: enum['warehouse','restricted','delivery_zone'],
  center: { lat: Number, lng: Number },
  radius: Number,               // meters
  alertOnEnter: Boolean,
  alertOnExit: Boolean,
  notifyRoles: [String],        // ['dispatcher', 'customer']
  active: Boolean
}
```

### DeliveryEvent Model (Analytics)
```js
{
  _id: ObjectId,
  order: ObjectId,
  vehicle: ObjectId,
  driver: ObjectId,
  estimatedDuration: Number,    // minutes
  actualDuration: Number,
  distanceKm: Number,
  weatherCondition: String,
  trafficLevel: enum['low','medium','high'],
  delayMinutes: Number,
  date: Date,
  dayOfWeek: Number,
  hour: Number
}
```

---

## 5. REST API Design

### Auth Service (Port 5001)
```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /auth/me
```

### Dispatch Service (Port 5002)
```
# Orders
GET    /orders                        → List (filter by status, driver, date)
POST   /orders                        → Create order
GET    /orders/:id                    → Order detail + status history
PUT    /orders/:id/status             → Update status (driver)
POST   /orders/:id/assign             → Assign vehicle manually
POST   /orders/auto-assign            → Trigger auto-assignment algorithm
GET    /orders/tracking/:trackingId   → Customer public tracking

# Vehicles
GET    /vehicles                      → All vehicles + live status
POST   /vehicles                      → Register vehicle
PUT    /vehicles/:id                  → Update vehicle info
GET    /vehicles/:id/history          → Location history

# Routes
GET    /routes/:vehicleId             → Current active route
POST   /routes/optimize               → Generate optimized route
PUT    /routes/:id/waypoint/:wid      → Mark waypoint done

# Geofences
GET    /geofences                     → All geofences
POST   /geofences                     → Create geofence
PUT    /geofences/:id                 → Update
DELETE /geofences/:id                 → Delete

# Analytics
GET    /analytics/overview            → KPI summary
GET    /analytics/delays              → Delay trends
POST   /analytics/predict             → Call ML service for prediction
```

### Tracking Service (Port 5003) — WebSocket only
```
WS     ws://localhost:5003
```

---

## 6. Socket.io Events

### Driver → Server
| Event | Payload | Description |
|-------|---------|-------------|
| `driver:connect` | `{ driverId, vehicleId }` | Driver comes online |
| `location:update` | `{ vehicleId, lat, lng, speed, heading }` | GPS ping (every 5s) |
| `order:status` | `{ orderId, status, lat, lng, note }` | Update delivery status |
| `driver:offline` | `{ vehicleId }` | Driver goes offline |

### Server → Dispatcher
| Event | Payload | Description |
|-------|---------|-------------|
| `vehicle:moved` | `{ vehicleId, lat, lng, speed }` | Live map update |
| `geofence:alert` | `{ vehicleId, geofenceId, type, timestamp }` | Enter/Exit alert |
| `order:updated` | `{ orderId, status, location }` | Delivery status change |
| `driver:online` | `{ driverId, vehicleId }` | Driver connected |
| `driver:offline` | `{ driverId, vehicleId }` | Driver disconnected |

### Server → Customer
| Event | Payload | Description |
|-------|---------|-------------|
| `delivery:location` | `{ lat, lng, eta }` | Live driver position |
| `delivery:status` | `{ status, message }` | Status notification |

---

## 7. Redis Geospatial Tracking

Redis `GEO` commands store and query vehicle locations with O(log N) speed.

```js
// Store live vehicle location
await redis.geoadd('vehicles:live', lng, lat, vehicleId);

// Get vehicle location
const loc = await redis.geopos('vehicles:live', vehicleId);

// Find all vehicles within 5km of a warehouse
const nearby = await redis.georadius(
  'vehicles:live', warehouseLng, warehouseLat, 5, 'km', 'WITHCOORD', 'ASC'
);

// Pub/Sub: broadcast location update to dispatcher room
await redis.publish('tracking:channel', JSON.stringify({
  vehicleId, lat, lng, speed, timestamp
}));
```

---

## 8. Auto-Assignment Algorithm

```js
async function autoAssign(order) {
  const { pickup, weight } = order;

  // Step 1: Get all idle vehicles from Redis GEO near pickup
  const nearbyVehicles = await redis.georadius(
    'vehicles:live',
    pickup.coordinates[0],
    pickup.coordinates[1],
    50, 'km', 'WITHCOORD', 'ASC'
  );

  // Step 2: Filter by capacity and status
  const eligible = await Promise.all(
    nearbyVehicles.map(async ([vehicleId]) => {
      const v = await Vehicle.findById(vehicleId);
      return v.capacity >= weight && v.status === 'idle' ? v : null;
    })
  );
  const filtered = eligible.filter(Boolean);
  if (!filtered.length) throw new Error('No vehicles available');

  // Step 3: Pick closest vehicle (first in GEO radius result = nearest)
  const best = filtered[0];

  // Step 4: Assign
  order.assignedVehicle = best._id;
  order.assignedDriver = best.driver;
  order.status = 'assigned';
  await order.save();

  // Step 5: Notify driver via Socket.io
  io.to(`driver:${best.driver}`).emit('order:assigned', order);

  return order;
}
```

---

## 9. Geofencing Logic

```js
// Run every time a location:update event arrives
function checkGeofences(vehicleId, lat, lng, previousLat, previousLng) {
  const geofences = await Geofence.find({ active: true });

  geofences.forEach(fence => {
    const wasInside = isInsideCircle(previousLat, previousLng, fence);
    const isInside  = isInsideCircle(lat, lng, fence);

    if (!wasInside && isInside && fence.alertOnEnter) {
      triggerAlert(vehicleId, fence, 'ENTER');
    }
    if (wasInside && !isInside && fence.alertOnExit) {
      triggerAlert(vehicleId, fence, 'EXIT');
    }
  });
}

function isInsideCircle(lat, lng, fence) {
  const d = haversineDistance(lat, lng, fence.center.lat, fence.center.lng);
  return d <= fence.radius;
}

function haversineDistance(lat1, lng1, lat2, lng2) {
  const R = 6371000; // Earth radius in meters
  const dLat = toRad(lat2 - lat1);
  const dLng = toRad(lng2 - lng1);
  const a = Math.sin(dLat/2)**2 +
            Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLng/2)**2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}
```

---

## 10. Offline Sync (Driver Dead Zones)

```js
// DRIVER SIDE (Angular + IndexedDB via Dexie.js)

// When driver updates status offline:
async function updateStatus(orderId, status, coords) {
  if (!navigator.onLine) {
    // Save to local queue
    await offlineDB.pendingUpdates.add({
      orderId, status, coords, timestamp: Date.now()
    });
    showUI('Saved offline. Will sync when connected.');
    return;
  }
  await api.put(`/orders/${orderId}/status`, { status, coords });
}

// Auto-sync when back online
window.addEventListener('online', async () => {
  const pending = await offlineDB.pendingUpdates.toArray();
  for (const update of pending) {
    await api.put(`/orders/${update.orderId}/status`, update);
    await offlineDB.pendingUpdates.delete(update.id);
  }
  showUI('Synced ' + pending.length + ' updates.');
});
```

---

## 11. AI Delay Prediction (Python Microservice)

### Flask Service (Port 5004)
```python
# train.py - Train once, save model
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import LabelEncoder
import joblib, pandas as pd

df = pd.read_csv('delivery_events.csv')
le = LabelEncoder()
df['traffic_encoded'] = le.fit_transform(df['trafficLevel'])

X = df[['distanceKm','hour','dayOfWeek','traffic_encoded']]
y = df['delayMinutes']

model = LinearRegression()
model.fit(X, y)
joblib.dump(model, 'delay_model.pkl')
joblib.dump(le, 'label_encoder.pkl')
```

```python
# app.py - Prediction API
from flask import Flask, request, jsonify
import joblib, numpy as np

app = Flask(__name__)
model = joblib.load('delay_model.pkl')
le = joblib.load('label_encoder.pkl')

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json
    traffic = le.transform([data['trafficLevel']])[0]
    features = np.array([[
        data['distanceKm'],
        data['hour'],
        data['dayOfWeek'],
        traffic
    ]])
    delay = model.predict(features)[0]
    return jsonify({ 'predictedDelayMinutes': round(float(delay), 1) })
```

---

## 12. Route Optimization (Mapbox)

```js
// Optimize waypoint order using Mapbox Optimization API
async function optimizeRoute(vehicleLocation, orderCoordinates) {
  const origin = `${vehicleLocation.lng},${vehicleLocation.lat}`;
  const waypoints = orderCoordinates
    .map(c => `${c.lng},${c.lat}`)
    .join(';');

  const url = `https://api.mapbox.com/optimized-trips/v1/mapbox/driving/
    ${origin};${waypoints}
    ?roundtrip=false
    &source=first
    &destination=last
    &access_token=${MAPBOX_TOKEN}`;

  const res = await axios.get(url);
  const trip = res.data.trips[0];

  return {
    optimizedWaypoints: res.data.waypoints, // Sorted order
    totalDistance: trip.distance / 1000,    // km
    totalDuration: trip.duration / 60,      // minutes
    polyline: trip.geometry                 // For map rendering
  };
}
```

---

## 13. Angular Frontend Structure

```
src/
├── app/
│   ├── core/
│   │   ├── guards/             → AuthGuard, RoleGuard
│   │   ├── interceptors/       → JWT interceptor
│   │   └── services/
│   │       ├── auth.service.ts
│   │       ├── socket.service.ts    → Socket.io wrapper
│   │       └── api.service.ts
│   │
│   ├── store/                  → NgRx
│   │   ├── vehicles/
│   │   │   ├── vehicles.actions.ts
│   │   │   ├── vehicles.reducer.ts
│   │   │   ├── vehicles.effects.ts  → API + Socket side effects
│   │   │   └── vehicles.selectors.ts
│   │   ├── orders/
│   │   └── app.state.ts
│   │
│   ├── modules/
│   │   ├── dispatcher/
│   │   │   ├── dashboard/           → KPI cards, live map
│   │   │   ├── orders/              → Order management table
│   │   │   ├── vehicles/            → Fleet list
│   │   │   └── geofences/           → Geofence manager
│   │   │
│   │   ├── driver/
│   │   │   ├── my-route/            → Today's route list
│   │   │   ├── delivery-update/     → Update status form
│   │   │   └── offline-indicator/   → Connectivity status
│   │   │
│   │   └── customer/
│   │       ├── track-order/         → Live tracking page
│   │       └── order-history/
│   │
│   └── shared/
│       ├── map/                     → Leaflet map component
│       ├── vehicle-marker/          → Custom map pin
│       └── timeline/                → Status history timeline
```

---

## 14. NgRx State Shape

```ts
interface AppState {
  auth: {
    user: User | null;
    token: string | null;
    role: 'dispatcher' | 'driver' | 'customer';
  };
  vehicles: {
    all: Vehicle[];
    liveLocations: { [vehicleId: string]: Coordinates };
    loading: boolean;
  };
  orders: {
    list: Order[];
    selected: Order | null;
    filters: OrderFilters;
    loading: boolean;
  };
  tracking: {
    connectedDrivers: string[];
    geofenceAlerts: GeofenceAlert[];
  };
}
```

---

## 15. Development Phases

### Phase 1: Foundation (Week 1–2)
- [ ] Angular 17 project + NgRx setup
- [ ] Node.js Auth microservice (JWT + RBAC)
- [ ] MongoDB models: User, Vehicle, Order
- [ ] Role-based routing in Angular (RoleGuard)
- [ ] Docker Compose: MongoDB + Redis

### Phase 2: Real-Time Tracking (Week 3–4)
- [ ] Socket.io Tracking Service
- [ ] Redis GEO: store + query vehicle locations
- [ ] `location:update` every 5s from driver
- [ ] Leaflet.js map with moving vehicle markers
- [ ] NgRx Effects: pipe socket events into store

### Phase 3: Dispatch & Orders (Week 5)
- [ ] Order CRUD (Dispatch Service)
- [ ] Auto-assignment algorithm (proximity + capacity)
- [ ] Manual assignment by dispatcher
- [ ] Driver: view assigned orders, update status
- [ ] Customer: public order tracking page

### Phase 4: Route Optimization (Week 6)
- [ ] Mapbox Optimization API integration
- [ ] Display optimized route polyline on Leaflet
- [ ] Estimated arrival per waypoint
- [ ] Reroute on order status change

### Phase 5: Geofencing (Week 7)
- [ ] Geofence CRUD (Dispatch Service)
- [ ] Draw geofence circles on Leaflet map
- [ ] Haversine check on every location update
- [ ] Real-time alert via Socket.io to dispatcher
- [ ] Email/push notification via Bull queue

### Phase 6: Offline Sync & AI (Week 8)
- [ ] IndexedDB with Dexie.js in Angular
- [ ] Offline status update queue
- [ ] Auto-sync on reconnect
- [ ] Python Flask ML service (delay prediction)
- [ ] Analytics dashboard: delay trends, KPIs

### Phase 7: Polish & DevOps (Week 9–10)
- [ ] Nginx API Gateway config
- [ ] Docker multi-stage builds per service
- [ ] Rate limiting + input validation
- [ ] Mobile-responsive driver UI
- [ ] Delivery proof: photo upload (Cloudinary)
- [ ] README + architecture diagram

---

## 16. Environment Variables

```env
# Auth Service
PORT=5001
JWT_SECRET=...
JWT_REFRESH_SECRET=...
MONGO_URI=mongodb://localhost:27017/vanguard

# Tracking Service
PORT=5003
REDIS_URL=redis://localhost:6379

# Dispatch Service
PORT=5002
MONGO_URI=mongodb://localhost:27017/vanguard
REDIS_URL=redis://localhost:6379
MAPBOX_TOKEN=pk.eyJ1...
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

# ML Service
PORT=5004

# Angular
VITE_API_GATEWAY=http://localhost:80
VITE_SOCKET_URL=http://localhost:5003
VITE_MAPBOX_TOKEN=pk.eyJ1...
```

---

## 17. Docker Compose

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

  dispatch-service:
    build: ./services/dispatch
    ports: ["5002:5002"]
    depends_on: [mongodb, redis]

  tracking-service:
    build: ./services/tracking
    ports: ["5003:5003"]
    depends_on: [redis]

  ml-service:
    build: ./services/ml
    ports: ["5004:5004"]

  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    volumes: [./nginx.conf:/etc/nginx/nginx.conf]
    depends_on: [auth-service, dispatch-service]

  frontend:
    build: ./frontend
    ports: ["4200:4200"]
    depends_on: [nginx]

volumes:
  mongo_data:
```

---

## 18. Folder Structure

```
vanguard/
├── services/
│   ├── auth/
│   │   ├── src/
│   │   │   ├── models/User.js
│   │   │   ├── routes/auth.routes.js
│   │   │   ├── controllers/auth.controller.js
│   │   │   ├── middleware/rbac.middleware.js
│   │   │   └── server.js
│   │   └── Dockerfile
│   │
│   ├── dispatch/
│   │   ├── src/
│   │   │   ├── models/
│   │   │   ├── routes/
│   │   │   ├── controllers/
│   │   │   ├── algorithms/
│   │   │   │   ├── autoAssign.js
│   │   │   │   └── routeOptimizer.js
│   │   │   └── server.js
│   │   └── Dockerfile
│   │
│   ├── tracking/
│   │   ├── src/
│   │   │   ├── socket/trackingSocket.js
│   │   │   ├── geo/geofenceChecker.js
│   │   │   └── server.js
│   │   └── Dockerfile
│   │
│   └── ml/
│       ├── app.py
│       ├── train.py
│       ├── delay_model.pkl
│       └── Dockerfile
│
├── frontend/                    → Angular app
├── nginx.conf
└── docker-compose.yml
```

---

## 19. Why Interviewers Love This

| Concept | How Vanguard Demonstrates It |
|---------|------------------------------|
| **Microservices** | 4 independent services with clear boundaries |
| **Geospatial Data** | Redis GEO commands + Haversine math |
| **Real-Time Systems** | Socket.io tracking at 5s intervals |
| **Algorithm Design** | Auto-assignment (proximity + capacity) |
| **Offline-First** | IndexedDB sync queue for dead zones |
| **ML Integration** | Python microservice with regression model |
| **API Gateway** | Nginx routing across services |
| **Angular + NgRx** | Enterprise-grade state management |
| **Relevant to companies** | Uber, Swiggy, Amazon, FedEx all do this |

---

*End of Project Vanguard Reference Guide*
