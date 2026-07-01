# Technical Design Document

## AI Campus Digital Twin – AI-Powered Smart Campus Navigation System

**Version:** 1.0 | **Prepared By:** Team Kero

---

## Overview

This document describes the technical architecture, component design, data models, API contracts, and key algorithms for the AI Campus Digital Twin platform. The design is derived directly from the 14 requirements in `requirements.md` and targets the following technology stack:

- **Frontend:** React.js (Web) + Flutter (Mobile — Android & iOS)
- **Backend:** FastAPI (Python)
- **AI:** OpenAI GPT-4o / Google Gemini via REST API
- **3D Engine:** Three.js (Web) / Unity WebGL (embedded view)
- **Navigation:** A* pathfinding on a weighted graph
- **Database:** MongoDB Atlas (primary) + Firebase Realtime Database (live status)
- **Auth:** Firebase Authentication + custom JWT middleware
- **Voice:** Google Speech-to-Text + Google Text-to-Speech
- **Cloud:** AWS (ECS, S3, CloudFront, ElastiCache) + Firebase
- **Real-time:** WebSockets via FastAPI + Firebase Realtime DB

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Layer                          │
│  React Web App        Flutter Mobile (Android / iOS)    │
│  Three.js 3D Viewer   Unity WebGL (embedded)            │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / WSS
┌────────────────────────▼────────────────────────────────┐
│                    API Gateway                           │
│  AWS API Gateway + WAF  (rate limiting, DDoS protection) │
└──┬─────────┬──────────┬──────────┬───────────┬──────────┘
   │         │          │          │           │
┌──▼──┐ ┌───▼───┐ ┌────▼───┐ ┌───▼────┐ ┌────▼────┐
│Auth │ │AI     │ │Nav     │ │Search  │ │Facility │
│Svc  │ │Svc    │ │Svc     │ │Svc     │ │Monitor  │
└──┬──┘ └───┬───┘ └────┬───┘ └───┬────┘ └────┬────┘
   │        │          │         │            │
┌──▼────────▼──────────▼─────────▼────────────▼────────┐
│                 Data Layer                             │
│  MongoDB Atlas          Firebase Realtime DB           │
│  (Users, Rooms,         (Occupancy, Room Status,       │
│   Faculty, Events,       Announcements, Alerts)        │
│   Buildings, Graph)                                    │
│                                                        │
│  Redis (ElastiCache)    AWS S3 + CloudFront            │
│  (Session cache,        (3D model assets, Floor        │
│   search index cache)    plan images, static files)    │
└───────────────────────────────────────────────────────┘
```

---

## Components and Interfaces

### 3.1 Auth Service

**Responsibilities:** User registration, login, JWT issuance, RBAC enforcement, MFA for admins, account lockout.

**Technology:** FastAPI + Firebase Auth + PyJWT

**Key design decisions:**
- Firebase Auth handles identity provider (email/password, Google SSO) and MFA (TOTP via Firebase)
- After Firebase verifies identity, the Auth Service issues a short-lived custom JWT (≤24h, HMAC-SHA256, 256-bit key) that carries `userId`, `role`, and `exp`
- All downstream services validate the JWT independently via a shared middleware — no central session store required
- Redis tracks failed login attempts per `userId` with a TTL of 15 minutes; after 5 consecutive failures, requests are rejected with HTTP 429
- Guest tokens are issued as anonymous Firebase tokens with a scoped JWT containing `role: "guest"` and 24h expiry

**RBAC permission matrix:**

| Feature | Student | Faculty | Administrator | Security_Staff | Guest |
|---|---|---|---|---|---|
| 3D Map (view) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Navigation | ✓ | ✓ | ✓ | ✓ | ✓ |
| AI Assistant | ✓ | ✓ | ✓ | ✓ | ✓ |
| Faculty Directory (full) | ✓ | ✓ | ✓ | ✓ | — |
| Faculty self-edit | — | ✓ | ✓ | — | — |
| Event create/edit | — | — | ✓ | — | — |
| Admin Console | — | — | ✓ | — | — |
| Emergency Module | ✓ | ✓ | ✓ | ✓ | ✓ |
| Visitor Portal | — | — | — | — | ✓ |

---

### 3.2 AI Assistant Service

**Responsibilities:** NLP query processing, intent classification, campus knowledge base Q&A, route intent handoff, voice transcription orchestration.

**Technology:** FastAPI + OpenAI GPT-4o (primary) / Gemini 1.5 Pro (fallback) + Google Speech-to-Text + Google Text-to-Speech

**Architecture:**

```
User Query (text or audio)
        │
        ▼
  [Speech-to-Text]  ← audio only (Google STT)
        │
        ▼
  [Intent Classifier]
  ├── navigation_intent → Navigation Service
  ├── entity_query → Campus Knowledge Base (MongoDB)
  ├── general_campus → LLM with RAG context
  └── unknown (< 70% confidence) → clarifying prompt
        │
        ▼
  [LLM (GPT-4o)]
  System prompt: campus knowledge context (RAG chunks)
  Max context window: 10 turns (sliding window)
        │
        ▼
  [Response]  → text to client
             → [Text-to-Speech] → audio response (if voice mode)
```

**Session management:**
- Sessions stored in Redis with a 30-minute sliding TTL
- Session key: `ai_session:{userId}:{sessionId}`
- Each entry stores up to 10 message turns as a FIFO list; oldest evicted on overflow
- On session end or TTL expiry, Redis key is deleted — no persistent storage of conversation history

**RAG (Retrieval-Augmented Generation):**
- Campus knowledge base stored in MongoDB with vector embeddings (OpenAI `text-embedding-3-small`)
- At query time, top-5 semantically similar chunks retrieved and injected into the LLM system prompt
- This constrains responses to verified campus data and prevents hallucination

---

### 3.3 Navigation Service

**Responsibilities:** Graph-based pathfinding, accessible routing, real-time route recalculation, blocked passage detection.

**Technology:** FastAPI + NetworkX (A* algorithm) + Redis (graph cache)

**Campus Navigation Graph:**

```
Node: {
  id: string,          // e.g. "B2-101" (Block B, Floor 2, Room 101)
  type: "room" | "corridor" | "elevator" | "staircase" | "entrance" | "exit",
  building: string,
  floor: int,
  coordinates: { x: float, y: float, z: float },
  accessible: bool,    // false for staircases
  is_emergency_exit: bool,
  is_hazardous: bool
}

Edge: {
  from: node_id,
  to: node_id,
  weight: float,       // walking distance in meters
  type: "walk" | "elevator" | "ramp" | "staircase" | "outdoor",
  bidirectional: bool
}
```

**A* heuristic:** Euclidean distance in 3D coordinate space.

**Accessible routing:** Before running A*, filter graph to exclude all edges with `type: "staircase"` and nodes with `accessible: false`.

**Emergency routing:** Filter graph to exclude all nodes with `is_hazardous: true` before computing path to nearest `is_emergency_exit: true` node.

**Real-time recalculation:** The mobile client sends location updates via WebSocket every 3 seconds. When the user deviates from the active route or a node is marked blocked, the Navigator recomputes using the same A* instance from the nearest graph node to the destination.

---

### 3.4 3D Renderer

**Responsibilities:** Campus 3D visualization, floor plan views, route overlay, entity selection and highlighting.

**Technology:** Three.js (Web) / Unity WebGL (embedded in Flutter via WebView)

**Asset pipeline:**
- Campus 3D models authored in Blender, exported as `.glTF 2.0`
- Assets stored on AWS S3, served via CloudFront CDN
- Level-of-Detail (LOD): 3 LOD levels per building (high/medium/low) — Three.js switches automatically based on camera distance
- Floor plans stored as separate `.glTF` scenes loaded on demand

**Route overlay:**
- Route coordinates from Navigation Service projected onto the 3D scene as a `THREE.TubeGeometry` along the graph edge path
- Indoor segments: solid blue tube
- Outdoor segments: dashed green tube
- Updated within 500ms of new location data via WebSocket push

**Fallback:** If model fails to load in 10s, client falls back to pre-rendered SVG floor plan images stored on S3.

---

### 3.5 Search Service

**Responsibilities:** Full-text and fuzzy search across all campus entity types, result ranking, suggestions on no-match.

**Technology:** FastAPI + MongoDB Atlas Search (Lucene-based) + Redis (index cache)

**Search index entities:** Rooms, Facilities, Faculty, Departments, Events, Parking Areas

**Query pipeline:**
1. Validate input (reject empty/whitespace)
2. Attempt exact match first (MongoDB `$eq` on `name` field)
3. If no exact match → Atlas Search fuzzy query (`maxEdits: 2`, `prefixLength: 1`)
4. Score and rank: exact matches boosted 10×; partial matches ranked by Atlas score
5. Cap results: top 20 total, top 10 for fuzzy-only
6. On zero results: query top-5 most searched terms from Redis frequency log as suggestions

**Index availability:** If MongoDB Atlas Search returns a connection error, return HTTP 503 with `"Search is temporarily unavailable"` — no partial/cached results served.

---

### 3.6 Facility Monitor

**Responsibilities:** Real-time occupancy display, room availability tracking, Near Full indicators.

**Technology:** FastAPI + Firebase Realtime Database + WebSockets

**Data flow:**
```
IoT Sensors / Manual Update
        │
        ▼
  Firebase Realtime DB
  /facilities/{facilityId}/occupancy
  /rooms/{roomId}/status
        │
        ▼
  Client WebSocket subscription
  (Firebase onValue listener — push on change, max 60s staleness)
        │
        ▼
  UI renders live occupancy / status badge
```

**Near Full threshold:** Stored per-facility in MongoDB as `occupancy_threshold` (default 90%). Computed on client: `(current / capacity) * 100 >= threshold`.

**Feed unavailability:** If Firebase connection is lost, client removes all occupancy/availability data from display and shows `"Status Unavailable"` badge. No stale values shown.

---

### 3.7 Faculty Directory

**Responsibilities:** Faculty profile storage, search, self-service updates, availability status.

**Technology:** FastAPI + MongoDB

**Profile schema (see Data Models section 4.3)**

**Self-service update:** Faculty users can PATCH `/faculty/{id}` for `office_hours` and `contact`. JWT role check enforces `role == "faculty" AND sub == id` or `role == "admin"`.

**Office hours propagation:** Updates written to MongoDB and pushed to a Redis pub/sub channel `faculty:updates`. Subscribed services (Search, AI RAG embeddings) re-index within 5 minutes.

---

### 3.8 Event Manager

**Responsibilities:** Event CRUD, visibility publishing, bookmark notifications.

**Technology:** FastAPI + MongoDB + Firebase Cloud Messaging (FCM)

**Event publish flow:**
1. Admin/Event_Coordinator creates event via POST `/events`
2. Written to MongoDB with `status: "published"`
3. Search Service re-indexes event within 2 minutes via change stream
4. FCM sends push notification to bookmarked users on update/cancellation

---

### 3.9 Emergency Module

**Responsibilities:** Display nearest emergency points, compute safe exit routes, broadcast alerts.

**Technology:** FastAPI + Navigation Service (internal call) + FCM

**Nearest point computation:** Pre-computed at server startup — for each node in the navigation graph, the 3 nearest `is_emergency_exit`, nearest `first_aid`, and nearest `security_office` nodes are stored in Redis. Lookup is O(1) at runtime.

**Alert broadcast:** Admin triggers POST `/emergency/alert`. Server fans out via FCM to all tokens with a session active in the last 5 minutes (tracked in Redis set `active_sessions`). Target delivery: ≤10 seconds.

---

### 3.10 Admin Console

**Responsibilities:** Room/Facility CRUD, audit logging, announcements, cascade deletion.

**Technology:** React (web-only, Vite + TypeScript) + FastAPI

**Audit log:** Every mutating operation on Room or Facility records writes an entry to a dedicated MongoDB `audit_logs` collection with TTL index of 90 days.

**Cascade deletion:** Deletion of a Room triggers:
1. Remove all Navigation Graph edges/nodes referencing that Room
2. Remove all Events with `venue_room_id == roomId`
3. Trigger Search Service re-index (remove document)
4. All steps executed as a MongoDB transaction; if any step fails, the transaction rolls back and an alert is written to `admin_notifications`.

---

## Data Models

### 4.1 User

```json
{
  "_id": "ObjectId",
  "firebase_uid": "string",
  "email": "string",
  "name": "string",
  "role": "student | faculty | admin | security_staff",
  "department": "string",
  "student_id": "string | null",
  "is_active": "boolean",
  "mfa_enabled": "boolean",
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

### 4.2 Room

```json
{
  "_id": "ObjectId",
  "room_number": "string",
  "name": "string (max 100 chars)",
  "building_id": "ObjectId",
  "floor": "integer",
  "capacity": "integer (1–10000)",
  "type": "classroom | lab | office | exam_hall | auditorium | library | cafeteria | other",
  "is_accessible": "boolean",
  "floor_plan_id": "ObjectId | null",
  "graph_node_id": "string",
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

### 4.3 Faculty

```json
{
  "_id": "ObjectId",
  "user_id": "ObjectId",
  "name": "string",
  "department": "string",
  "designation": "string",
  "office_room_id": "ObjectId | null",
  "office_hours": "string (HH:MM–HH:MM format)",
  "contact_email": "string",
  "contact_phone": "string",
  "status": "available | unavailable",
  "updated_at": "ISODate"
}
```

### 4.4 Building

```json
{
  "_id": "ObjectId",
  "name": "string",
  "code": "string",
  "total_floors": "integer",
  "accessible_entrances": ["string"],
  "opening_hours": "string (HH:MM–HH:MM format)",
  "coordinates": { "lat": "float", "lng": "float" },
  "model_asset_url": "string (S3 URL)"
}
```

### 4.5 Event

```json
{
  "_id": "ObjectId",
  "title": "string",
  "description": "string",
  "type": "seminar | workshop | exam | college_event | other",
  "venue_room_id": "ObjectId",
  "organizer_id": "ObjectId",
  "start_datetime": "ISODate",
  "end_datetime": "ISODate",
  "status": "published | cancelled | draft",
  "bookmarked_by": ["ObjectId"],
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

### 4.6 Facility (Firebase Realtime DB)

```json
{
  "facilityId": {
    "name": "string",
    "type": "library | cafeteria | parking | lab | classroom",
    "capacity": "integer",
    "current_occupancy": "integer",
    "status": "open | closed | maintenance",
    "operating_hours": "string",
    "occupancy_threshold": "integer (1–100)",
    "feed_available": "boolean",
    "last_updated": "timestamp"
  }
}
```

### 4.7 Navigation Graph Node (MongoDB)

```json
{
  "_id": "string (node_id)",
  "type": "room | corridor | elevator | staircase | entrance | exit | first_aid | security_office",
  "building_id": "ObjectId",
  "floor": "integer",
  "coordinates": { "x": "float", "y": "float", "z": "float" },
  "accessible": "boolean",
  "is_emergency_exit": "boolean",
  "is_hazardous": "boolean",
  "edges": [
    {
      "to": "string (node_id)",
      "weight": "float",
      "type": "walk | elevator | ramp | staircase | outdoor"
    }
  ]
}
```

### 4.8 Audit Log (MongoDB, TTL 90 days)

```json
{
  "_id": "ObjectId",
  "admin_id": "ObjectId",
  "action": "create | update | delete",
  "entity_type": "room | facility",
  "entity_id": "ObjectId",
  "field_changed": "string",
  "previous_value": "any",
  "new_value": "any",
  "timestamp": "ISODate"
}
```

### 4.9 Campus Knowledge Base Document (MongoDB, with vector index)

Used by the AI Assistant RAG pipeline. Each document represents a discrete chunk of campus information that can be embedded and retrieved semantically.

```json
{
  "_id": "ObjectId",
  "content": "string (the raw text chunk, max 500 tokens)",
  "source_type": "room | building | faculty | facility | event | policy | faq",
  "source_id": "ObjectId | null",
  "title": "string (human-readable label for this chunk)",
  "embedding": "[float] (1536-dimensional vector, OpenAI text-embedding-3-small)",
  "embedding_model": "string (e.g. text-embedding-3-small)",
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

**Vector index:** MongoDB Atlas Vector Search index on the `embedding` field using cosine similarity. At query time, top-5 nearest neighbours are retrieved using `$vectorSearch` aggregation pipeline.

**Re-indexing triggers:**
- Faculty profile updated → re-embed the faculty's knowledge chunk via Redis pub/sub `faculty:updates`
- Room or Building created/updated → re-embed the room/building chunk via Admin Service
- Event published → embed new event chunk via Event Manager change stream

**RAG retrieval failure:** If MongoDB Atlas Vector Search is unavailable, the AI Service falls back to keyword-based Atlas Search on the `content` field. If both are unavailable, the AI Service returns HTTP 503 with `"AI assistant is temporarily unavailable."` No hallucinated responses are generated.

---

## API Design

All endpoints are prefixed with `/api/v1`. All requests require `Authorization: Bearer <JWT>` except public endpoints marked *(public)*.

### 5.1 Auth API

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login` | Authenticate with email/password; returns JWT |
| POST | `/auth/guest` | Issue guest session token *(public)* |
| POST | `/auth/logout` | Invalidate session |
| POST | `/auth/refresh` | Refresh JWT before expiry |

### 5.2 AI Assistant API

| Method | Endpoint | Description |
|---|---|---|
| POST | `/ai/chat` | Submit text query; returns AI response |
| POST | `/ai/voice` | Submit audio (multipart); returns text + audio response |
| DELETE | `/ai/session` | Explicitly end session and clear context |

**POST /ai/chat request:**
```json
{
  "session_id": "string",
  "message": "string",
  "user_location": { "node_id": "string" }
}
```

**Response:**
```json
{
  "response_text": "string",
  "intent": "navigation | entity_query | general | unknown",
  "route_id": "string | null",
  "confidence": "float"
}
```

### 5.3 Navigation API

| Method | Endpoint | Description |
|---|---|---|
| POST | `/nav/route` | Compute route between two nodes |
| GET | `/nav/route/{route_id}` | Retrieve computed route |
| PATCH | `/nav/route/{route_id}/recalc` | Recalculate from new location |
| POST | `/nav/emergency-route` | Compute safe exit route *(public)* |

**POST /nav/route request:**
```json
{
  "origin_node_id": "string",
  "destination_node_id": "string",
  "accessible": "boolean"
}
```

**Response:**
```json
{
  "route_id": "string",
  "steps": [
    {
      "instruction": "string",
      "node_id": "string",
      "distance_m": "float",
      "transition_type": "walk | elevator | ramp | staircase | outdoor"
    }
  ],
  "total_distance_m": "float",
  "estimated_time_sec": "integer",
  "has_accessible_path": "boolean"
}
```

### 5.4 Search API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/search?q={query}&type={type}` | Search all or filtered entity types |

**Response:**
```json
{
  "results": [
    {
      "entity_type": "room | faculty | facility | event | department | parking",
      "id": "string",
      "name": "string",
      "description": "string",
      "location": { "building": "string", "floor": "integer", "node_id": "string" },
      "score": "float"
    }
  ],
  "total": "integer",
  "suggestions": ["string"]
}
```

### 5.5 Facility Status API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/facilities` | List all facilities with live status |
| GET | `/facilities/{id}` | Get single facility status |
| PATCH | `/facilities/{id}` | Update status/hours (Admin only) |

### 5.6 Faculty API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/faculty` | List all faculty |
| GET | `/faculty/{id}` | Get faculty profile |
| PATCH | `/faculty/{id}` | Update own hours/contact (Faculty/Admin) |
| PATCH | `/faculty/{id}/status` | Set availability status |

### 5.7 Events API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/events` | List upcoming events |
| GET | `/events/{id}` | Get event detail |
| POST | `/events` | Create event (Admin/Event_Coordinator) |
| PATCH | `/events/{id}` | Update event |
| DELETE | `/events/{id}` | Cancel event |
| POST | `/events/{id}/bookmark` | Bookmark event |

### 5.8 Emergency API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/emergency/nearest` | Get nearest aid, exit, security *(public)* |
| POST | `/emergency/route` | Compute safe exit route *(public)* |
| POST | `/emergency/alert` | Broadcast alert (Admin only) |

### 5.9 Admin API

| Method | Endpoint | Description |
|---|---|---|
| POST | `/admin/rooms` | Create room |
| PATCH | `/admin/rooms/{id}` | Update room |
| DELETE | `/admin/rooms/{id}` | Delete room + cascade |
| GET | `/admin/audit-log` | Retrieve audit log (paginated) |
| POST | `/admin/announcements` | Broadcast announcement |

---

## Key Algorithms

### 6.1 A* Pathfinding

The following is pseudocode showing the algorithm logic. The actual implementation uses NetworkX's built-in `astar_path` with a custom heuristic rather than a manual priority queue, but the logic is equivalent.

```python
# Actual implementation — uses NetworkX astar_path
import networkx as nx
from math import sqrt

def heuristic(node_id: str, goal_id: str, graph: nx.DiGraph) -> float:
    a = graph.nodes[node_id]["coordinates"]
    b = graph.nodes[goal_id]["coordinates"]
    return sqrt((a["x"]-b["x"])**2 + (a["y"]-b["y"])**2 + (a["z"]-b["z"])**2)

def astar(graph: nx.DiGraph, start_id: str, end_id: str, accessible_only: bool = False) -> list[str] | None:
    working_graph = filter_accessible(graph) if accessible_only else graph
    try:
        path = nx.astar_path(
            working_graph,
            start_id,
            end_id,
            heuristic=lambda u, v: heuristic(u, v, working_graph),
            weight="weight"
        )
        return path
    except nx.NetworkXNoPath:
        return None  # No path found
    except nx.NodeNotFound:
        return None  # Invalid node ID

def filter_accessible(graph: nx.DiGraph) -> nx.DiGraph:
    # Remove staircase edges and inaccessible nodes
    filtered = graph.copy()
    staircase_edges = [
        (u, v) for u, v, d in filtered.edges(data=True)
        if d.get("type") == "staircase"
    ]
    filtered.remove_edges_from(staircase_edges)
    inaccessible_nodes = [
        n for n, d in filtered.nodes(data=True)
        if not d.get("accessible", True)
    ]
    filtered.remove_nodes_from(inaccessible_nodes)
    return filtered
```

### 6.2 Arrival Detection (Client-Side)

```dart
// Flutter — runs on each GPS/BLE location update
void checkArrival(LatLng userPos, LatLng destPos) {
  double distanceMeters = Geolocator.distanceBetween(
    userPos.latitude, userPos.longitude,
    destPos.latitude, destPos.longitude,
  );
  if (distanceMeters <= 15.0) {
    navigationController.onArrived();
  }
}
```

### 6.3 AI Confidence Gate

```python
async def classify_intent(query: str) -> IntentResult:
    response = await openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": INTENT_CLASSIFIER_PROMPT},
            {"role": "user", "content": query}
        ],
        logprobs=True
    )
    intent = response.choices[0].message.content
    confidence = compute_confidence(response.choices[0].logprobs)
    if confidence < 0.70:
        return IntentResult(intent="unknown", confidence=confidence)
    return IntentResult(intent=intent, confidence=confidence)
```

---

## Infrastructure and Deployment

```
┌──────────────────────────────────────────────────────┐
│  AWS Cloud                                           │
│                                                      │
│  Route 53 (DNS)                                      │
│       │                                              │
│  CloudFront (CDN) ── S3 (3D assets, floor plans)     │
│       │                                              │
│  API Gateway + WAF                                   │
│       │                                              │
│  ECS Fargate (FastAPI microservices)                 │
│  ├── auth-service      (2 tasks min)                 │
│  ├── ai-service        (4 tasks min)                 │
│  ├── nav-service       (2 tasks min)                 │
│  ├── search-service    (2 tasks min)                 │
│  ├── facility-service  (2 tasks min)                 │
│  ├── faculty-service   (1 task min)                  │
│  ├── event-service     (1 task min)                  │
│  ├── emergency-service (2 tasks min)                 │
│  └── admin-service     (1 task min)                  │
│       │                                              │
│  ElastiCache Redis (session, graph cache)            │
│  MongoDB Atlas (primary database)                    │
│  Firebase (Auth, Realtime DB, FCM)                   │
└──────────────────────────────────────────────────────┘
```

**Scaling:** ECS Auto Scaling on CPU > 70%. Target group: 20,000 concurrent users. ALB with sticky sessions disabled (stateless JWT).

**CI/CD:** GitHub Actions → Docker build → ECR → ECS rolling deploy. Zero-downtime deploys via blue/green on critical services.

---

## Security Design

- All inter-service communication inside VPC; only API Gateway exposed publicly
- WAF rules: SQL injection, XSS, rate limiting (1000 req/min per IP)
- Secrets (DB credentials, API keys) stored in AWS Secrets Manager; injected as env vars at runtime
- S3 buckets private; assets served only via CloudFront signed URLs
- MongoDB Atlas: VPC peering, IP allowlist, field-level encryption for PII fields
- JWT signing key rotated every 90 days via automated Secrets Manager rotation

---

## Screen Flow

```
Splash Screen
    │
    ├─► Login Screen ──► Role-based Home Dashboard
    │       │                    │
    │       └─► Forgot Password  ├─► AI Chat (floating button)
    │                            ├─► 3D Campus Map
    └─► Guest / Visitor Mode     │       └─► Floor Plan View
                │                │               └─► Room Detail
                └─► Visitor Map  ├─► Search Results → Nav
                                 ├─► Faculty Directory
                                 ├─► Events List → Nav
                                 ├─► Facility Status
                                 ├─► Emergency Module
                                 └─► Admin Console (Admin only)
```

---

## Requirements Traceability

| Requirement | Design Component(s) |
|---|---|
| R1 – Authentication | Auth Service, Firebase Auth, Redis (lockout), JWT middleware |
| R2 – AI Assistant | AI Service, GPT-4o, Google STT/TTS, Redis (session) |
| R3 – 3D Map | Three.js/Unity, S3/CloudFront, 3D Renderer component |
| R4 – Navigation | Navigation Service, A* algorithm, Navigation Graph (MongoDB) |
| R5 – Search | Search Service, MongoDB Atlas Search, Redis (cache) |
| R6 – Facility Status | Facility Monitor, Firebase Realtime DB, WebSockets |
| R7 – Faculty Locator | Faculty Directory Service, MongoDB, Redis pub/sub |
| R8 – Events | Event Manager Service, MongoDB, FCM |
| R9 – Emergency | Emergency Service, Navigation Service, FCM, Redis (active sessions) |
| R10 – Visitor Mode | Visitor Portal (frontend), Auth Service (guest token), RBAC |
| R11 – Admin Console | Admin Service, MongoDB (audit log TTL), cascade transaction |
| R12 – Performance | ECS Auto Scaling, ElastiCache, MongoDB Atlas, CloudFront |
| R13 – Security | WAF, TLS 1.2+, AES-256, Secrets Manager, JWT |
| R14 – Accessibility | WCAG 2.1 AA (frontend), AI_Assistant (voice mode), Navigator (accessible route) |

---

## Correctness Properties

The following invariants must hold at all times during system operation:

### Property 1: Route Validity
A Route returned by the Navigator SHALL only contain edges present in the current Navigation Graph. No Route SHALL pass through a node marked `is_hazardous: true` unless explicitly requested without hazard exclusions.

**Validates: Requirements 4.1, 4.2, 9.2**

### Property 2: Accessible Route Purity
An Accessible_Route SHALL contain zero staircase-type edges. If the A* algorithm produces a path containing a staircase edge with `accessible_only=true`, it is a bug.

**Validates: Requirements 4.2, 14.4**

### Property 3: RBAC Completeness
Every API endpoint must have an explicit role check. A missing role check that allows a lower-privilege role to access a higher-privilege operation is a correctness violation.

**Validates: Requirements 1.6, 11.6**

### Property 4: No Stale Facility Data
If `feed_available` is `false` for a Facility, the client MUST NOT display any occupancy or status value for that Facility. Displaying cached values when the feed is down violates R6.6.

**Validates: Requirements 6.6**

### Property 5: AI Response Grounding
The AI Assistant MUST NOT return a response claiming a specific room, building, or faculty member exists unless that entity is present in the campus knowledge base. Fabricated campus data is a correctness violation.

**Validates: Requirements 2.4, 2.5**

### Property 6: Cascade Integrity
Deleting a Room MUST atomically remove all associated Navigation Graph edges, Events, and Search index entries. A Room deletion that leaves orphaned routes or events is a correctness violation.

**Validates: Requirements 11.5**

### Property 7: JWT Non-Reuse
An expired or revoked JWT MUST NOT grant access to any protected resource. The Auth Service MUST reject all requests bearing an invalid JWT before processing any request payload.

**Validates: Requirements 1.3, 13.7**

---

## Error Handling

### Auth Service
- Invalid credentials → HTTP 401 with generic message (no username/password hint)
- Account locked → HTTP 429 with `Retry-After: 900` header (15 minutes)
- Expired JWT → HTTP 401 with `error: "token_expired"`
- MFA not configured (Admin) → HTTP 403 with `error: "mfa_required"`

### AI Assistant Service
- LLM API timeout (>5s) → Return HTTP 503, display `"AI assistant is temporarily unavailable. Please try again."`
- Voice transcription failure → HTTP 200 with `error: "transcription_failed"`, display `"Could not understand audio"`
- Confidence < 70% → HTTP 200 with clarifying prompt, no route or entity data returned
- Primary LLM (GPT-4o) unavailable → Automatic failover to Gemini 1.5 Pro within 2s
- Both GPT-4o and Gemini unavailable → HTTP 503, display `"AI assistant is temporarily unavailable. Please try again."`
- RAG vector search unavailable → Fall back to Atlas keyword search on `content` field; if that also fails → HTTP 503

### Navigation Service
- No path found → HTTP 200 with `route: null` and `error: "no_path_available"`
- No accessible path found → HTTP 200 with `has_accessible_path: false`; offer standard route
- Graph load failure → HTTP 503; client displays `"Navigation is temporarily unavailable"`
- Route recalculation timeout (>3s) → Return last valid route segment; flag `recalc_failed: true`

### 3D Renderer (Client-Side)
- Model load timeout (>10s) → Display error banner and load fallback SVG floor plan from S3
- WebGL not supported → Display static 2D campus map with text-based search only
- Floor plan missing → Display `"No room data available"` message for that floor

### Search Service
- Empty/whitespace query → HTTP 400, `"error": "query_required"`
- Search index unavailable → HTTP 503, `"Search is temporarily unavailable"`, no cached results
- Zero results → HTTP 200, empty results array, 3–5 suggestions from frequency log

### Facility Monitor
- Firebase Realtime DB disconnection → Remove all live data from UI; show `"Status Unavailable"` per facility
- IoT feed stale (>120s without update) → Set `feed_available: false`; trigger Status Unavailable display

### Emergency Module
- Route computation timeout (>5s) → Skip route; display floor plan with all exits marked
- No reachable exit → Display floor plan with all exits marked and text instruction
- Location unavailable → Display most recently visited building floor plan

### Admin Console
- Cascade deletion partial failure → Roll back full transaction; log failure; notify Admin via in-app alert
- Non-admin access attempt → HTTP 403, `"error": "forbidden"`
- Announcement > 500 characters → HTTP 400, `"error": "announcement_too_long"`

---

## Testing Strategy

### Unit Tests
- **Auth Service:** JWT generation/validation, RBAC permission matrix, lockout counter logic, MFA state machine
- **Navigation Service:** A* correctness on sample graphs, accessible-only filtering, arrival detection (15m threshold), blocked edge handling
- **AI Service:** Intent classification with mock LLM responses, confidence gate at 70%, session eviction (10-turn sliding window), 30-minute inactivity cleanup
- **Search Service:** Empty query rejection, fuzzy match ranking, exact-before-partial ordering, index-unavailable error path
- **Facility Monitor:** Near Full threshold at exactly 90% and 91%, feed-unavailable state transition, 60s refresh cycle

### Integration Tests
- Login → JWT → protected route access for each Role
- AI chat → navigation intent → Navigation Service route returned and rendered on 3D map
- Admin creates room → Search index updated → room appears in search results within 2 minutes
- Admin deletes room → cascade removes events + graph edges → search returns no results
- Facility feed disconnects → all occupancy badges switch to "Status Unavailable"
- Emergency alert broadcast → FCM push delivered to active session users

### Performance Tests
- Load test: 20,000 simulated concurrent users using k6
  - Search P95 latency target: ≤ 2 seconds
  - Route generation P95 latency target: ≤ 3 seconds
  - Auth login P95 latency target: ≤ 2 seconds
- Stress test: Ramp to 25,000 users — verify HTTP 503 returned for overflow, active sessions preserved
- 3D Renderer: FPS benchmark on minimum-spec device (WebGL 2.0, 4GB RAM, 1080p) — target ≥ 30 FPS

### Security Tests
- JWT with expired `exp` claim → verify HTTP 401 on all protected endpoints
- JWT with tampered payload (re-signed with wrong key) → verify rejection
- Student JWT accessing Admin Console → verify HTTP 403
- Guest JWT accessing Faculty contact info → verify field not returned
- SQL injection / NoSQL injection in search query → verify sanitisation and no data leak
- Brute-force login: 5 failed attempts → verify lockout; attempt 6 within lockout window → verify rejection

### Accessibility Tests (Manual + Automated)
- axe-core automated scan on all web views — target zero WCAG 2.1 AA violations
- Screen reader walkthrough (NVDA + Chrome, VoiceOver + Safari) on Search, Navigation, and AI Chat flows
- High-contrast mode: verify all 3D map elements meet 4.5:1 contrast ratio
- Voice command flow: end-to-end test with Google STT on Android and iOS


---

## Monitoring and Observability

### 9.1 Logging

All FastAPI services emit structured JSON logs to stdout. ECS captures stdout and forwards to AWS CloudWatch Logs.

**Log format (every request):**
```json
{
  "timestamp": "ISO8601",
  "service": "auth-service",
  "level": "INFO | WARN | ERROR",
  "request_id": "uuid4",
  "user_id": "string | null",
  "method": "POST",
  "path": "/api/v1/auth/login",
  "status_code": 200,
  "duration_ms": 142,
  "error": "string | null"
}
```

**Log retention:** INFO/WARN retained 30 days; ERROR retained 90 days.

**Sensitive data policy:** `Authorization` headers, JWT payloads, and PII fields (email, phone) are never written to logs. `user_id` logs only the MongoDB ObjectId, never email or name.

### 9.2 Metrics

Metrics published to AWS CloudWatch via Embedded Metric Format (EMF) from each FastAPI service.

| Metric | Unit | Published By | Alarm Threshold |
|---|---|---|---|
| `RequestDuration` | Milliseconds | All services | P95 > 3000ms |
| `ErrorRate` | Percent | All services | > 1% over 5 min |
| `ActiveSessions` | Count | AI Service | — (informational) |
| `NavRouteComputeTime` | Milliseconds | Nav Service | P95 > 3000ms |
| `SearchLatency` | Milliseconds | Search Service | P95 > 2000ms |
| `AIConfidenceLow` | Count | AI Service | > 100 in 5 min |
| `EmergencyAlertDelivery` | Milliseconds | Emergency Service | > 10000ms |
| `FacilityFeedStaleness` | Seconds | Facility Monitor | > 120s |
| `CascadeDeleteFailure` | Count | Admin Service | Any occurrence |

### 9.3 Distributed Tracing

AWS X-Ray is enabled on all ECS tasks. The FastAPI `aws-xray-sdk` middleware injects trace IDs into every request. Cross-service calls propagate the `X-Amzn-Trace-Id` header so a full request chain (e.g. AI chat → Navigation → Search) is traceable end-to-end in the X-Ray console.

### 9.4 Alerting

CloudWatch Alarms are configured for every metric threshold in section 9.2. All alarms publish to an SNS topic. SNS delivers to:

- **Email:** ops team distribution list (all alarms)
- **PagerDuty:** P1 alarms only — `ErrorRate > 5%`, `EmergencyAlertDelivery > 10s`, `CascadeDeleteFailure`

Alarm runbooks are stored in `/docs/runbooks` in the repository. Each runbook maps an alarm to: probable cause, diagnostic steps, and resolution actions.

### 9.5 Dashboards

A CloudWatch Dashboard named `ai-campus-digital-twin-ops` contains:

- Request volume and error rate (all services, last 1h)
- P50/P95/P99 latency per service (last 1h)
- Active WebSocket connections (Nav + Facility services)
- Redis memory utilisation and eviction rate
- MongoDB Atlas connection pool utilisation
- ECS task health: running vs desired count per service

---

## Glossary

| Term | Definition |
|---|---|
| **Digital Twin** | A real-time virtual model of a physical campus that mirrors the state of buildings, rooms, and facilities |
| **RAG** | Retrieval-Augmented Generation — relevant knowledge base chunks injected into an LLM prompt to ground responses in verified data |
| **LOD** | Level of Detail — 3D models rendered at lower geometric resolution as camera distance increases, preserving frame rate |
| **A\*** | A-star — graph search algorithm that finds the shortest path between two nodes using a heuristic to guide exploration |
| **JWT** | JSON Web Token — compact, signed token carrying user identity and role claims between client and server |
| **RBAC** | Role-Based Access Control — access model where permissions are assigned to roles and users are assigned roles |
| **FCM** | Firebase Cloud Messaging — Google service for sending push notifications to Android and iOS devices |
| **RTO** | Recovery Time Objective — maximum acceptable time to restore service after a failure |
| **WCAG** | Web Content Accessibility Guidelines — international standard defining accessibility requirements for web content |
| **WAF** | Web Application Firewall — security layer that filters and blocks malicious HTTP requests |
| **glTF** | GL Transmission Format — royalty-free 3D file format optimised for web transmission and runtime loading |
| **BLE** | Bluetooth Low Energy — short-range wireless protocol used for indoor positioning beacons |
| **FIFO** | First In, First Out — queue structure where the oldest item is evicted first |
| **TTL** | Time to Live — expiry duration applied to cache entries or database records |
| **ECS** | Elastic Container Service — AWS managed container orchestration service |
| **CDN** | Content Delivery Network — geographically distributed servers that cache and serve static assets close to users |
| **Navigation Graph** | The weighted directed graph representing all walkable paths, rooms, corridors, staircases, elevators, and outdoor routes across campus |
| **Node (graph)** | A point in the Navigation Graph representing a physical location (room, corridor junction, elevator landing, exit, etc.) |
| **Edge (graph)** | A weighted connection between two graph nodes representing a traversable path segment |
| **Intent** | The classified purpose of a user query: `navigation_intent`, `entity_query`, `general_campus`, or `unknown` |
| **Sliding TTL** | A cache expiry that resets on each access, so entries persist as long as they are actively used |
| **P95 latency** | The 95th percentile response time — 95% of requests complete within this duration |
| **Cascade deletion** | Deletion of a parent record that atomically removes all child records referencing it |
| **Feed** | A live data stream from IoT sensors or manual updates supplying real-time occupancy to the Facility Monitor |

---

## Open Questions

These questions require a decision before the corresponding components can be finalised. Each is tagged with the requirement and task it blocks.

| # | Question | Options | Blocks | Owner | Status |
|---|---|---|---|---|---|
| OQ-1 | Which indoor positioning technology will be used for sub-room accuracy on mobile? | (A) BLE beacons + trilateration, (B) Wi-Fi fingerprinting, (C) QR checkpoint scan-only | R4.5, Task 12.4 | Team Kero | Open |
| OQ-2 | Will the 3D campus model be built from scratch in Blender or sourced from a CAD export of existing architectural drawings? | (A) Blender from scratch, (B) CAD import + cleanup, (C) Photogrammetry scan | R3.2, Task 11.1 | Team Kero | Open |
| OQ-3 | What is the source of truth for initial campus data (rooms, buildings, floor plans)? | (A) Existing college GIS/CAD data, (B) Manual digitisation from photos/blueprints | R4, Task 2.5 | College Admin | Open |
| OQ-4 | Should the AI assistant retain conversation history across sessions (persistent memory) or only within a single session (current design)? | (A) Session-only — current design, (B) Persistent per-user history in MongoDB | R2.6, Task 6.4 | Team Kero | Open |
| OQ-5 | What is the target deployment environment for the demo — local Docker Compose, AWS staging, or live production? | (A) Local Docker, (B) AWS staging, (C) Live production URL | Task 15 | Team Kero | Open |
| OQ-6 | Will IoT sensors be physically installed for the demo, or will occupancy data be simulated via a mock feed script? | (A) Real IoT sensors, (B) Mock feed simulator script | R6, Task 7 | Team Kero | Open |
| OQ-7 | Does the college have an existing student/faculty identity system (LDAP, SSO) that Firebase Auth should federate with, or will standalone email/password be used? | (A) Federate with college SSO, (B) Standalone Firebase accounts | R1.1, Task 3.1 | College IT | Open |
| OQ-8 | What is the minimum target device specification for the Flutter mobile app? (Affects LOD thresholds, WebGL quality, BLE support.) | (A) Android 8+ / iOS 13+, (B) Android 10+ / iOS 15+ | R3.7, Task 12.3 | Team Kero | Open |
