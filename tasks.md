# Implementation Plan: AI Campus Digital Twin

## Overview

This plan covers the full implementation of the AI Campus Digital Twin platform across 15 task groups, spanning backend microservices (FastAPI), web frontend (React + Three.js), mobile app (Flutter), data layer (MongoDB + Firebase), and cloud infrastructure (AWS ECS + S3 + CloudFront). Tasks are ordered by dependency: infrastructure and data layer first, then backend services in parallel, then frontend clients, and finally load/security testing.

## Task Dependency Graph

```json
{
  "waves": [
    { "wave": 1, "tasks": ["1"] },
    { "wave": 2, "tasks": ["2"] },
    { "wave": 3, "tasks": ["3"] },
    { "wave": 4, "tasks": ["4", "5", "6", "7", "8", "9", "10", "14"] },
    { "wave": 5, "tasks": ["11", "12", "13"] },
    { "wave": 6, "tasks": ["15"] }
  ]
}
```

## Tasks

- [ ] 1. Project Scaffold & Infrastructure Setup
  - [ ] 1.1 Initialize monorepo structure with `/backend`, `/web`, `/mobile`, `/shared` directories and configure root-level tooling (ESLint, Prettier, pre-commit hooks)
  - [ ] 1.2 Create FastAPI backend project with Poetry, configure pyproject.toml with all required dependencies (fastapi, uvicorn, pymongo, firebase-admin, PyJWT, redis, networkx, openai)
  - [ ] 1.3 Create React web app with Vite + TypeScript, install Three.js, Tailwind CSS, React Router, and Zustand
  - [ ] 1.4 Create Flutter mobile project targeting Android and iOS, add dependencies (firebase_core, firebase_auth, geolocator, flutter_tts, speech_to_text, webview_flutter)
  - [ ] 1.5 Set up Docker Compose for local development (FastAPI, MongoDB, Redis, Firebase emulator)
  - [ ] 1.6 Configure GitHub Actions CI pipeline: lint → test → build for backend and web
  - **Requirements:** R12, R13

- [ ] 2. Data Layer & MongoDB Schema
  - [ ] 2.1 Create MongoDB Atlas cluster, configure VPC peering, IP allowlist, and field-level encryption for PII fields
  - [ ] 2.2 Implement Mongoose/PyMongo models for all 8 schemas: User, Room, Building, Faculty, Event, Navigation Graph Node, Audit Log, and Admin Notifications
  - [ ] 2.3 Create MongoDB Atlas Search index on Rooms, Facilities, Faculty, Departments, Events, and Parking Areas collections
  - [ ] 2.4 Create TTL index on `audit_logs` collection (90-day expiry on `timestamp` field)
  - [ ] 2.5 Seed navigation graph with sample campus nodes and edges covering all transition types (walk, elevator, ramp, staircase, outdoor)
  - [ ] 2.6 Configure Firebase Realtime Database schema for `/facilities/{facilityId}` and `/rooms/{roomId}/status` paths
  - [ ] 2.7 Seed campus knowledge base in MongoDB: create at least 50 CampusDocument records covering rooms, buildings, facilities, faculty bios, and FAQs; generate OpenAI `text-embedding-3-small` embeddings for each document; create MongoDB Atlas Vector Search index on the `embedding` field using cosine similarity
  - **Requirements:** R2, R5, R6, R11

- [ ] 3. Auth Service
  - [ ] 3.1 Implement `POST /api/v1/auth/login` — validate Firebase ID token, issue custom HMAC-SHA256 JWT (≤24h expiry, 256-bit key) containing `userId`, `role`, `exp`
  - [ ] 3.2 Implement `POST /api/v1/auth/guest` — issue anonymous Firebase token and scoped guest JWT with `role: "guest"` and 24h expiry
  - [ ] 3.3 Implement `POST /api/v1/auth/logout` and `POST /api/v1/auth/refresh`
  - [ ] 3.4 Implement JWT validation middleware shared across all services — reject expired, malformed, or revoked tokens with HTTP 401 before processing any payload
  - [ ] 3.5 Implement RBAC middleware — enforce per-endpoint role checks; return HTTP 403 for unauthorized roles
  - [ ] 3.6 Implement login lockout logic using Redis: increment failure counter per `userId` on each bad credential; lock for 15 minutes after 5 consecutive failures; return HTTP 429 with `Retry-After: 900`
  - [ ] 3.7 Enforce MFA requirement for Administrator accounts: block activation until TOTP configured via Firebase; return HTTP 403 with `error: "mfa_required"` if MFA not set up
  - [ ] 3.8 Write unit tests: JWT generation/validation, lockout counter at thresholds (4 failures = no lock, 5 = lock), RBAC permission matrix for all 5 roles × all protected endpoints
  - **Requirements:** R1, R13

- [ ] 4. Navigation Service
  - [ ] 4.1 Load navigation graph from MongoDB into NetworkX `DiGraph` at service startup; cache in Redis with 5-minute TTL
  - [ ] 4.2 Implement `filter_accessible(graph)` — remove all edges with `type: "staircase"` and nodes with `accessible: false`
  - [ ] 4.3 Implement `astar(graph, start_id, end_id, accessible_only)` using NetworkX with 3D Euclidean heuristic; return ordered list of node IDs and total distance
  - [ ] 4.4 Implement `POST /api/v1/nav/route` — validate nodes exist, run A*, return route steps with `instruction`, `node_id`, `distance_m`, `transition_type`, `has_accessible_path`; respond within 3 seconds
  - [ ] 4.5 Implement `PATCH /api/v1/nav/route/{route_id}/recalc` — accept new origin node, re-run A* from that node to original destination; complete within 2 seconds; return `recalc_failed: true` if timeout exceeded
  - [ ] 4.6 Implement `POST /api/v1/nav/emergency-route` (public) — filter graph to exclude `is_hazardous: true` nodes, find nearest `is_emergency_exit: true` node, compute route; fall back to floor plan response if computation exceeds 5 seconds
  - [ ] 4.7 Implement `GET /api/v1/emergency/nearest` (public) — return pre-computed nearest first-aid, fire exit, and security office for the given node_id from Redis O(1) lookup; recompute at startup
  - [ ] 4.8 Write unit tests: A* correctness on a 10-node sample graph, accessible filter removes all staircase edges, arrival detection at exactly 15m and 14m, no-path returns null, alternative route on blocked edge
  - **Requirements:** R4, R9

- [ ] 5. Search Service
  - [ ] 5.1 Implement `GET /api/v1/search` — validate query (reject empty/whitespace with HTTP 400); run exact match first (`$eq` on `name`); fall back to Atlas Search fuzzy query (`maxEdits: 2`, `prefixLength: 1`)
  - [ ] 5.2 Implement result ranking: boost exact matches 10×; cap total results at 20; cap fuzzy-only results at 10
  - [ ] 5.3 Implement zero-results path: query Redis frequency log for top 3–5 most searched terms; return as `suggestions` array
  - [ ] 5.4 Implement search index unavailability handler: return HTTP 503 with `"Search is temporarily unavailable"`; do not serve cached results
  - [ ] 5.5 Track search term frequency in Redis sorted set (`ZINCRBY search:freq <term>`) on every successful query
  - [ ] 5.6 Write unit tests: empty query rejection, exact-before-partial ranking, fuzzy match on misspelled input, index-unavailable 503 path, zero-results suggestion list length (3–5)
  - **Requirements:** R5

- [ ] 6. AI Assistant Service
  - [ ] 6.1 Implement intent classifier using GPT-4o with `logprobs=True`; compute confidence from log probabilities; return `IntentResult(intent, confidence)`; return `intent: "unknown"` when confidence < 0.70
  - [ ] 6.2 Implement RAG pipeline: embed incoming query with `text-embedding-3-small`; retrieve top-5 semantically similar chunks from MongoDB vector index; inject into GPT-4o system prompt
  - [ ] 6.3 Implement `POST /api/v1/ai/chat` — classify intent → route to Navigator (navigation), MongoDB lookup (entity_query), or RAG LLM (general); return within 3 seconds; fall back to Gemini 1.5 Pro if GPT-4o unavailable
  - [ ] 6.4 Implement session management: store up to 10 turns as FIFO list in Redis (`ai_session:{userId}:{sessionId}`); evict oldest turn on overflow; set 30-minute sliding TTL; delete key on explicit session end or TTL expiry
  - [ ] 6.5 Implement `POST /api/v1/ai/voice` — accept multipart audio, transcribe via Google STT, process through chat pipeline, synthesise response via Google TTS; return text + audio within 4 seconds; return `"Could not understand audio"` on transcription failure
  - [ ] 6.6 Implement `DELETE /api/v1/ai/session` — delete Redis session key immediately
  - [ ] 6.7 Write unit tests: confidence gate at exactly 0.70 (pass) and 0.69 (unknown), session eviction at turn 11 drops turn 1, 30-minute inactivity cleanup, fabricated entity response blocked when entity not in knowledge base
  - **Requirements:** R2, R14

- [ ] 7. Facility Monitor Service
  - [ ] 7.1 Implement `GET /api/v1/facilities` and `GET /api/v1/facilities/{id}` — return live data from Firebase Realtime DB; include `feed_available` field in every response
  - [ ] 7.2 Implement `PATCH /api/v1/facilities/{id}` (Admin only) — update `status`, `occupancy_threshold`, `operating_hours` in Firebase and MongoDB; enforce RBAC
  - [ ] 7.3 Implement client-side Firebase `onValue` listener in React and Flutter — subscribe to `/facilities/{id}/occupancy` and `/rooms/{id}/status`; render live badge; detect Firebase disconnection and immediately clear all occupancy/status display, show `"Status Unavailable"` per facility
  - [ ] 7.4 Implement Near Full indicator logic on client: compute `(current_occupancy / capacity) * 100 >= occupancy_threshold`; show badge when true, remove when false
  - [ ] 7.5 Write unit tests: Near Full triggers at 90% and not at 89%, feed-unavailable clears all display values, 60-second max staleness (mock stale feed), status update propagates to client within 60 seconds
  - **Requirements:** R6

- [ ] 8. Faculty Directory Service
  - [ ] 8.1 Implement `GET /api/v1/faculty` and `GET /api/v1/faculty/{id}` — return name, office Room, department, office hours, contact info; omit contact fields from Guest responses
  - [ ] 8.2 Implement `PATCH /api/v1/faculty/{id}` — enforce `role == "faculty" AND sub == id` or `role == "admin"`; update `office_hours` and contact fields in MongoDB; publish update event to Redis pub/sub channel `faculty:updates`
  - [ ] 8.3 Implement `PATCH /api/v1/faculty/{id}/status` — set `status: "available" | "unavailable"`; update MongoDB; return updated profile
  - [ ] 8.4 Implement subscriber in Search Service and AI RAG pipeline listening to `faculty:updates` — re-index faculty document within 5 minutes of update
  - [ ] 8.5 Return `"Location not available"` and suppress navigation option in response when `office_room_id` is null
  - [ ] 8.6 Write unit tests: faculty self-edit allowed, cross-user edit blocked (HTTP 403), student attempting edit blocked, null office_room_id suppresses nav option, status update reflected within 5 minutes via pub/sub
  - **Requirements:** R7, R10

- [ ] 9. Event Manager Service
  - [ ] 9.1 Implement `POST /api/v1/events` (Admin/Event_Coordinator) — create event, write to MongoDB with `status: "published"`, trigger Search Service re-index via MongoDB change stream; complete indexing within 2 minutes
  - [ ] 9.2 Implement `GET /api/v1/events` and `GET /api/v1/events/{id}` — return events sorted by `start_datetime` ascending; include venue Room details and navigation option
  - [ ] 9.3 Implement `PATCH /api/v1/events/{id}` and `DELETE /api/v1/events/{id}` — update listing within 2 minutes; send FCM push notification to all users in `bookmarked_by` array on update or cancellation
  - [ ] 9.4 Implement `POST /api/v1/events/{id}/bookmark` — add/remove `userId` from `bookmarked_by` array (toggle); return updated bookmark count
  - [ ] 9.5 Write unit tests: non-admin/coordinator create blocked (HTTP 403), event visible in search within 2 minutes of creation, FCM notification sent to bookmarked users on cancellation, navigation option present when venue Room has graph_node_id
  - **Requirements:** R8

- [ ] 10. Admin Console Service
  - [ ] 10.1 Implement `POST /api/v1/admin/rooms`, `PATCH /api/v1/admin/rooms/{id}`, `DELETE /api/v1/admin/rooms/{id}` — enforce Admin-only RBAC; validate field constraints (name ≤100 chars, capacity 1–10000)
  - [ ] 10.2 Implement cascade deletion for Room: wrap in MongoDB transaction — (1) remove Navigation Graph edges/nodes, (2) delete Events with matching `venue_room_id`, (3) trigger Search re-index removal; on transaction failure, roll back and write to `admin_notifications`
  - [ ] 10.3 Implement `POST /api/v1/admin/announcements` — validate text ≤500 characters; broadcast to all active sessions via Firebase Realtime DB push; display to all logged-in users within 2 minutes
  - [ ] 10.4 Implement `GET /api/v1/admin/audit-log` — return paginated audit log entries from MongoDB; enforce Admin-only access
  - [ ] 10.5 Write audit log entry on every mutating Room or Facility operation: record `admin_id`, `action`, `entity_type`, `entity_id`, `field_changed`, `previous_value`, `new_value`, UTC `timestamp`
  - [ ] 10.6 Write unit tests: non-admin access returns HTTP 403, cascade deletion removes all 3 artifact types, partial cascade failure triggers rollback and notification, announcement > 500 chars rejected, audit log entry written on create/update/delete
  - **Requirements:** R11

- [ ] 11. 3D Renderer (Web)
  - [ ] 11.0 Prepare 3D asset pipeline: source or author campus building models in Blender (or import from CAD/architectural drawings), export each building as `.glTF 2.0` with 3 LOD meshes (high/medium/low), export per-floor geometry as separate `.glTF` scenes, upload all assets to S3 under `/3d-assets/buildings/` and `/3d-assets/floors/` prefixes — this sub-task is a hard prerequisite for all other 11.x tasks
  - [ ] 11.1 Implement Three.js scene loader — fetch `.glTF 2.0` campus model from CloudFront; implement 10-second load timeout; on timeout display error banner and load fallback SVG floor plan from S3
  - [ ] 11.2 Implement 3 LOD levels per building using `THREE.LOD` — switch automatically based on camera distance thresholds
  - [ ] 11.3 Implement pan, zoom (10%–500%), and rotate camera controls using `THREE.OrbitControls`; support both touch and pointer input
  - [ ] 11.4 Implement building click handler — on selection display panel with name, opening hours, accessible entrances, and floor list within 1 second
  - [ ] 11.5 Implement floor plan view — load per-floor `.glTF` scene on demand; display Room labels (number + name); show `"No room data available"` for unmapped floors
  - [ ] 11.6 Implement entity highlight on click — apply `THREE.OutlinePass` to selected Room or Facility mesh; display details panel (name, type, capacity, accessibility status)
  - [ ] 11.7 Implement route overlay — project Navigation Service route steps onto 3D scene as `THREE.TubeGeometry`; solid blue for indoor, dashed green for outdoor; update within 500ms of WebSocket location push
  - [ ] 11.8 Implement high-contrast mode toggle — swap material colours to WCAG 2.1 AA compliant palette; ensure all map elements meet 4.5:1 contrast ratio
  - **Requirements:** R3, R14

- [ ] 12. Flutter Mobile App
  - [ ] 12.1 Implement login screen with Firebase Auth (email/password + Google SSO); handle MFA prompt for Administrator accounts; show guest/visitor mode entry point
  - [ ] 12.2 Implement home dashboard with role-based navigation menu (AI Chat FAB, 3D Map, Search, Faculty, Events, Facility Status, Emergency, Admin Console for admins)
  - [ ] 12.3 Embed Unity WebGL 3D campus viewer in Flutter via `webview_flutter`; implement JS bridge for route overlay and entity selection events
  - [ ] 12.4 Implement real-time navigation screen — show step-by-step instructions, WebSocket location updates every 3 seconds, arrival detection at 15m threshold, destination-reached notification
  - [ ] 12.5 Implement AI Chat screen — text input + voice input toggle (speech_to_text), display response text + play TTS audio (flutter_tts), maintain 10-turn session context display
  - [ ] 12.6 Implement Visitor Mode screen — display simplified map with only key destinations (departments, principal's office, admission block, parking, reception); suppress staff-only locations and Faculty contact PII
  - [ ] 12.7 Implement Emergency Module screen — display nearest first-aid, fire exit, security office within 1 second of open; safe exit route button; full-screen alert overlay on FCM emergency broadcast
  - [ ] 12.8 Implement facility status screen with Firebase real-time listener — show Near Full badge, Occupied/Available room status, Status Unavailable on feed loss
  - **Requirements:** R1, R2, R3, R4, R6, R9, R10, R14

- [ ] 13. React Web App
  - [ ] 13.1 Implement auth pages: login, forgot password, guest entry; integrate Firebase Auth SDK; store JWT in httpOnly cookie
  - [ ] 13.2 Implement main layout with sidebar navigation, AI Chat floating action button, and role-gated menu items
  - [ ] 13.3 Implement Search page — input with debounce (300ms), display ranked results with entity type badges, navigation button for mappable entities, inline `"Search is temporarily unavailable"` error state
  - [ ] 13.4 Implement Faculty Directory page — list view with search filter, profile modal showing office Room, hours, status; navigation button; suppress contact fields for Guest role
  - [ ] 13.5 Implement Events page — upcoming events list with date/venue/type; bookmark toggle; navigation button per event; filter by date/type/keyword
  - [ ] 13.6 Implement Admin Console — Room CRUD form with field validation (name ≤100 chars, capacity 1–10000), Facility status editor, announcement composer (≤500 chars), paginated audit log table
  - [ ] 13.7 Implement WCAG 2.1 AA accessibility throughout: semantic HTML, ARIA labels, keyboard navigation, skip links, focus management on modals; run axe-core in CI
  - **Requirements:** R1, R5, R7, R8, R11, R14

- [ ] 14. Emergency Module Backend
  - [ ] 14.1 Implement startup pre-computation: for each graph node compute the 3 nearest `is_emergency_exit`, nearest `first_aid`, and nearest `security_office` nodes using BFS; store in Redis hash `emergency:nearest:{nodeId}`
  - [ ] 14.2 Implement `POST /api/v1/emergency/alert` (Admin only) — read `active_sessions` Redis set (sessions active in last 5 minutes), fan out FCM notification to all registered tokens; target ≤10 second delivery
  - [ ] 14.3 Maintain `active_sessions` Redis set: add session token on login/activity; expire entries after 5 minutes of inactivity using Redis EXPIRE
  - [ ] 14.4 Write unit tests: nearest-exit lookup returns correct pre-computed node, alert fan-out hits all tokens in active_sessions set, inactive sessions excluded from broadcast, route computation falls back to floor plan on 5-second timeout
  - **Requirements:** R9

- [ ] 15. Performance, Security & Deployment
  - [ ] 15.1 Write Dockerfile for FastAPI services (multi-stage build); create ECS task definitions for all 9 microservices with minimum task counts per design
  - [ ] 15.2 Configure ECS Auto Scaling: scale out when CPU > 70%; configure ALB target group with health checks; disable sticky sessions
  - [ ] 15.3 Configure AWS WAF rules: SQL injection protection, XSS filter, rate limit 1000 req/min per IP
  - [ ] 15.4 Store all secrets (DB credentials, API keys, JWT signing key) in AWS Secrets Manager; configure 90-day automatic rotation for JWT signing key; inject via ECS environment at runtime
  - [ ] 15.5 Configure S3 bucket as private; configure CloudFront distribution with signed URL policy for 3D model assets and floor plan SVGs
  - [ ] 15.6 Configure MongoDB Atlas automated backups with ≤24-hour interval; document and test restore procedure to verify ≤1-hour RTO
  - [ ] 15.7 Run k6 load test at 20,000 concurrent users — verify search P95 ≤ 2s, route P95 ≤ 3s; verify HTTP 503 returned (not crash) when load exceeds capacity; verify active sessions preserved
  - [ ] 15.8 Run OWASP ZAP scan against staging environment; fix all high/medium severity findings before production deploy
  - **Requirements:** R12, R13

## Notes

- All backend services are FastAPI microservices deployed on AWS ECS Fargate. Each service has its own Dockerfile and task definition.
- MongoDB Atlas Search index (Task 2.3) must be created before the Search Service (Task 5) and Event Manager (Task 9) are implemented, as both depend on it.
- The campus knowledge base seed and vector index (Task 2.7) must be complete before the AI Assistant Service (Task 6) can be implemented and tested end-to-end.
- The Navigation Graph seed data (Task 2.5) must be in place before the Navigation Service (Task 4) and Emergency Module (Task 14) can be tested end-to-end.
- Firebase Realtime Database schema (Task 2.6) must be configured before the Facility Monitor (Task 7) and Emergency alert broadcast (Task 14.2) are implemented.
- The JWT middleware (Task 3.4) and RBAC middleware (Task 3.5) are shared dependencies — all other services import them before implementing their own endpoints.
- The 3D Renderer (Task 11) and Flutter mobile app (Task 12) both consume the Navigation Service WebSocket for live route updates; Navigation Service (Task 4) must be stable before these are integrated.
- Within Wave 4, Task 8 sub-task 8.4 (faculty re-index listener) depends on the Search Service (Task 5) being implemented first. Task 6 (AI) depends on Task 2.7 (knowledge base seed). These are intra-wave sequential dependencies.
- Task 11.0 (3D asset pipeline) is a hard prerequisite for all other Task 11 sub-tasks and should be started in parallel with Wave 1, not after Wave 4 completes.
- Load and security testing (Task 15) should run against a full staging environment with all services deployed; it is the final gate before production.
- WCAG 2.1 AA compliance (Task 13.7) requires manual screen reader testing in addition to automated axe-core scans — plan for dedicated QA time before release.
- Firebase must be upgraded to the Blaze plan (RK-9) before Task 7 development begins to avoid the 100-connection Spark plan limit.

---

## Effort Estimates

Rough estimates for a team of 3–4 engineers working in parallel. Times assume stack familiarity and exclude code review cycles.

| Task | Description | Estimated Effort | Wave |
|---|---|---|---|
| 1 | Project Scaffold & Infrastructure | 1 day | 1 |
| 2 | Data Layer & MongoDB Schema | 2 days | 2 |
| 3 | Auth Service | 2 days | 3 |
| 4 | Navigation Service | 3 days | 4 |
| 5 | Search Service | 2 days | 4 |
| 6 | AI Assistant Service | 4 days | 4 |
| 7 | Facility Monitor Service | 2 days | 4 |
| 8 | Faculty Directory Service | 1.5 days | 4 |
| 9 | Event Manager Service | 1.5 days | 4 |
| 10 | Admin Console Service | 2 days | 4 |
| 11 | 3D Renderer (Web) | 4 days | 5 |
| 12 | Flutter Mobile App | 5 days | 5 |
| 13 | React Web App | 3 days | 5 |
| 14 | Emergency Module Backend | 1 day | 4 |
| 15 | Performance, Security & Deployment | 2 days | 6 |
| **Total** | | **~35 developer-days** | |

For a 3-person team working in parallel, the critical path is approximately **12–14 calendar days** for a functional demo.

**Critical path:** Task 1 → Task 2 → Task 3 → Task 4 → Task 11/12 → Task 15

The AI Assistant Service (Task 6) and 3D Renderer (Task 11) are the highest-risk tasks due to external API dependencies and 3D asset availability. Start these earliest in their respective waves.

---

## Definition of Done

A task is **Done** only when all of the following are true:

### Code Quality
- [ ] All sub-tasks implemented and committed
- [ ] Code passes lint (ruff for Python, ESLint for TypeScript/Dart)
- [ ] No type errors (mypy for Python, tsc --noEmit for TypeScript)
- [ ] No hardcoded secrets, credentials, or API keys in source code

### Testing
- [ ] All unit tests specified in the task are written and passing
- [ ] Test coverage ≥ 80% for new code (pytest-cov / vitest coverage)
- [ ] Integration test for the task's primary user flow passes in Docker Compose

### Requirements Traceability
- [ ] Every requirement ID listed under the task is demonstrably satisfied
- [ ] Any unsatisfied requirement is documented as a known gap with justification

### API Contracts
- [ ] All new endpoints match request/response schemas from design.md Section 5
- [ ] All new endpoints return correct HTTP status codes per the design.md Error Handling section
- [ ] All new endpoints enforce RBAC per the permission matrix in design.md Section 3.1

### Performance
- [ ] Task latency targets are met under local Docker Compose load
- [ ] No new N+1 query patterns introduced in MongoDB or Firebase calls

### Security
- [ ] No SQL/NoSQL injection vectors introduced
- [ ] All user-supplied inputs validated before use
- [ ] No PII logged in stdout

### Accessibility (Frontend tasks only)
- [ ] axe-core scan on new views returns zero WCAG 2.1 AA violations
- [ ] All new interactive elements are keyboard-navigable
- [ ] All new text elements meet 4.5:1 colour contrast ratio

### Documentation
- [ ] Public API changes reflected in design.md
- [ ] Any deviation from the design documented with rationale
- [ ] Open Questions resolved during this task updated in design.md Section 10

---

## Risk Register

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| RK-1 | OpenAI API rate limits or outages during demo | Medium | High | Gemini 1.5 Pro fallback in Task 6.3; test fallback path before demo |
| RK-2 | 3D campus model not available in time | High | High | Start Blender/CAD work in parallel with Wave 1; have low-poly placeholder by Wave 3 |
| RK-3 | Campus GIS/floor plan data not provided by college | Medium | High | Resolve in Week 1; fall back to manually digitised building outlines if needed |
| RK-4 | Indoor positioning inaccuracy makes real-time nav unusable | High | Medium | QR checkpoint fallback always available; remove live rerouting from demo scope if needed |
| RK-5 | Firebase Realtime DB free-tier limits hit during load test | Low | Medium | Upgrade to Blaze plan before Task 15; mock the facility feed in demo if needed |
| RK-6 | WebGL performance on demo device below 30 FPS | Medium | Medium | Reduce LOD thresholds; disable route animation; use SVG floor plan fallback on weak hardware |
| RK-7 | MongoDB Atlas Search index build time exceeds demo timeline | Low | High | Trigger index build in Task 2.3 immediately; complete before Task 5 begins |
| RK-8 | AWS costs exceed budget during development | Medium | Low | Use Free Tier + Spot instances for dev; set billing alarm at $50; use Docker Compose locally |
| RK-9 | Firebase Spark plan 100 simultaneous connection limit hit during development or demo | High | High | Upgrade Firebase project to Blaze (pay-as-you-go) plan before Task 7 begins; Spark limit is per-project and will be hit with even a small test team using Realtime DB WebSocket connections |
