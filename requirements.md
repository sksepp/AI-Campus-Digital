# Requirements Document

## AI Campus Digital Twin – AI-Powered Smart Campus Navigation System

**Version:** 1.0 | **Prepared By:** Team Kero

---

## Introduction

AI Campus Digital Twin is a smart campus navigation platform that creates a real-time 3D virtual model of a college campus. It enables students, faculty, visitors, and administrators to locate buildings, classrooms, labs, faculty offices, and campus facilities using AI-powered conversational search, interactive 3D navigation, and live facility information.

This document formally defines the 14 system requirements for the platform. All design components, API contracts, and implementation tasks trace back to these requirements using identifiers R1–R14.

### Product Goals

- Reduce average campus navigation time to under 3 seconds per search
- Provide indoor and outdoor turn-by-turn navigation with ≥ 95% route accuracy
- Enable natural-language and voice-based campus queries with ≥ 90% AI response accuracy
- Support 20,000 concurrent users with 99.9% uptime
- Meet WCAG 2.1 Level AA accessibility standards across all interfaces

### Target Users

| Role | Description |
|---|---|
| Student | Primary user; navigates classrooms, labs, exam halls, faculty offices |
| Faculty | Manages office profile and availability; receives student visits |
| Visitor | Guest user with no registration; navigates to public destinations |
| Administrator | Manages all campus data; broadcasts alerts and announcements |
| Security Staff | Accesses emergency module and campus map |
| Event Coordinator | Creates and manages campus events |

---

## Requirements

### R1 – Authentication and Authorization

The system shall provide secure user authentication supporting four roles: Student, Faculty, Administrator, and Security Staff, plus a Guest/Visitor mode requiring no registration.

**User Stories:**
- As a student, I want to log in with my college email so that I can access personalised campus navigation.
- As a visitor, I want to use the app without registering so that I can navigate the campus immediately on arrival.
- As an administrator, I want MFA enforced on my account so that admin operations are protected against unauthorised access.

#### R1.1
The system shall authenticate users via email/password and Google SSO through Firebase Authentication.

**Acceptance Criteria:**
- Email/password login returns a JWT on valid credentials
- Google SSO login returns a JWT on successful OAuth flow
- Invalid credentials return HTTP 401 with a generic error message (no username/password hint)

#### R1.2
The system shall issue a short-lived custom JWT (≤24 hours, HMAC-SHA256, 256-bit key) after successful Firebase authentication, carrying `userId`, `role`, and `exp` claims.

**Acceptance Criteria:**
- JWT payload contains `userId`, `role`, and `exp` fields
- JWT expiry is no longer than 24 hours from issuance
- JWT is signed with HMAC-SHA256 using a 256-bit key stored in AWS Secrets Manager

#### R1.3
All protected API endpoints shall validate the JWT before processing any request payload. Expired, malformed, or revoked tokens shall be rejected with HTTP 401.

**Acceptance Criteria:**
- Request with expired JWT returns HTTP 401 with `error: "token_expired"`
- Request with malformed JWT returns HTTP 401
- Request with valid JWT proceeds to endpoint processing

#### R1.4
Guest users shall receive an anonymous session token scoped to `role: "guest"` with a 24-hour expiry. No registration or personal data shall be required.

**Acceptance Criteria:**
- `POST /auth/guest` returns a JWT with `role: "guest"` and 24h expiry
- No email, name, or other personal data is collected or required
- Guest JWT grants access only to public endpoints defined in the RBAC matrix

#### R1.5
The system shall lock a user account for 15 minutes after 5 consecutive failed login attempts, returning HTTP 429 with `Retry-After: 900`.

**Acceptance Criteria:**
- 4 consecutive failures do not trigger lockout
- 5th consecutive failure triggers lockout; subsequent attempts return HTTP 429
- `Retry-After: 900` header is present on locked responses
- Lockout clears automatically after 15 minutes

#### R1.6
Every API endpoint shall enforce RBAC. A lower-privilege role accessing a higher-privilege endpoint shall receive HTTP 403.

**Acceptance Criteria:**
- Student JWT accessing Admin Console endpoint returns HTTP 403
- Guest JWT accessing Faculty contact fields receives a response with `contact_email` and `contact_phone` fields omitted entirely from the object
- Each role can access exactly the endpoints defined in the permission matrix

#### R1.7
Administrator accounts shall require MFA (TOTP via Firebase). Admins without MFA configured shall receive HTTP 403 with `error: "mfa_required"`.

**Acceptance Criteria:**
- Admin login without MFA configured returns HTTP 403 with `error: "mfa_required"`
- Admin login with valid TOTP code proceeds and issues JWT
- Non-admin roles are unaffected by MFA enforcement

---

### R2 – AI Assistant

The system shall provide a conversational AI assistant capable of understanding natural-language campus queries and generating accurate, grounded responses.

**User Stories:**
- As a student, I want to ask "Where is the AI lab?" in natural language so that I get directions without memorising room numbers.
- As a visitor, I want to speak my query aloud so that I can navigate hands-free.
- As a user, I want the AI to admit when it doesn't know something so that I don't get directed to the wrong place.

#### R2.1
The AI assistant shall classify user queries into four intents: `navigation_intent`, `entity_query`, `general_campus`, and `unknown`.

**Acceptance Criteria:**
- "Take me to Block B" is classified as `navigation_intent`
- "What are Dr. Kumar's office hours?" is classified as `entity_query`
- "What time does the library open?" is classified as `general_campus`
- Ambiguous input below 70% confidence is classified as `unknown`

#### R2.2
For `navigation_intent` queries, the assistant shall hand off to the Navigation Service and return a route in the response.

**Acceptance Criteria:**
- AI chat response for a navigation query includes a `route_id` field
- The route is retrievable from `GET /nav/route/{route_id}`
- If navigation fails, the response includes a human-readable error message

#### R2.3
For `entity_query` and `general_campus` queries, the assistant shall use RAG with the campus knowledge base to produce grounded responses.

**Acceptance Criteria:**
- Response cites only data present in the campus knowledge base
- Top-5 relevant knowledge base chunks are retrieved and injected into the LLM prompt
- Response does not contain information from outside the campus knowledge base

#### R2.4
The AI assistant shall NOT claim a specific room, building, faculty member, or facility exists unless that entity is present in the campus knowledge base.

**Acceptance Criteria:**
- Query about a non-existent room returns "I don't have information about that location" style response
- Response never fabricates a room number, building name, or faculty name
- Automated test with a made-up entity name confirms no hallucinated affirmative response

#### R2.5
When query classification confidence is below 70%, the assistant shall return a clarifying prompt instead of a response or route.

**Acceptance Criteria:**
- Confidence of exactly 0.70 proceeds to a response
- Confidence of 0.69 returns a clarifying prompt with no route or entity data
- Clarifying prompt asks the user to rephrase or be more specific

#### R2.6
The assistant shall maintain a session of up to 10 turns. On the 11th turn the oldest turn is evicted. Sessions expire after 30 minutes of inactivity.

**Acceptance Criteria:**
- Turn 1 through 10 are all retained in session context
- On turn 11, turn 1 is evicted and turns 2–11 are retained
- A session with no activity for 30 minutes returns an empty context on next query
- `DELETE /ai/session` immediately clears the session

#### R2.7
The assistant shall support voice input (Google STT) and voice output (Google TTS). Voice responses shall complete within 4 seconds.

**Acceptance Criteria:**
- Audio file submitted to `POST /ai/voice` returns transcribed text and audio response
- End-to-end voice round-trip completes in ≤ 4 seconds (P95)
- On transcription failure, response contains `error: "transcription_failed"` and display message "Could not understand audio"

#### R2.8
Text responses shall return within 3 seconds. If GPT-4o is unavailable, fail over to Gemini 1.5 Pro within 2 seconds. If both LLMs are unavailable, the system shall return HTTP 503 with message "AI assistant is temporarily unavailable."

**Acceptance Criteria:**
- `POST /ai/chat` returns a response in ≤ 3 seconds at P95
- When GPT-4o endpoint returns a 5xx error, Gemini is called within 2 seconds
- User receives a response regardless of which LLM handled it; no error shown for transparent failover
- When both GPT-4o and Gemini return errors, `POST /ai/chat` returns HTTP 503 with `"AI assistant is temporarily unavailable. Please try again."`

---

### R3 – 3D Campus Map

The system shall render an interactive 3D digital twin of the campus using real building geometry, floor plans, and room data.

**User Stories:**
- As a student, I want to rotate and zoom the 3D campus map so that I can orient myself before navigating.
- As a visitor, I want to click a building and see its opening hours so that I know if it's accessible.
- As a user with low vision, I want a high-contrast map mode so that I can distinguish buildings clearly.

#### R3.1
The 3D viewer shall support pan, zoom (10%–500%), and rotate via both touch and pointer input.

**Acceptance Criteria:**
- Pinch gesture zooms in/out on touch devices
- Scroll wheel zooms in/out on desktop
- Drag rotates the camera on both touch and pointer
- Zoom is clamped between 10% and 500% of default view

#### R3.2
Buildings shall be rendered as `.glTF 2.0` models with 3 LOD levels switching by camera distance.

**Acceptance Criteria:**
- High-LOD model renders when camera distance < 50m
- Medium-LOD renders between 50m and 200m
- Low-LOD renders beyond 200m
- LOD switching is seamless with no visible pop-in

#### R3.3
Selecting a building displays name, opening hours, accessible entrances, and floor list within 1 second.

**Acceptance Criteria:**
- Building detail panel appears within 1 second of click/tap
- Panel contains building name, opening hours string, list of accessible entrance labels, and clickable floor list
- Panel is keyboard-accessible and closeable via Escape key

#### R3.4
Floor plan view shall load per-floor geometry on demand and display room labels. Unmapped floors show "No room data available".

**Acceptance Criteria:**
- Clicking a floor loads that floor's `.glTF` scene on demand
- Each room mesh shows a label with room number and name
- A floor with no data shows the message "No room data available"
- Floor scene loads within 3 seconds on standard network

#### R3.5
Selecting a room or facility highlights it and shows a details panel.

**Acceptance Criteria:**
- Selected mesh receives an outline highlight via `THREE.OutlinePass`
- Details panel shows: name, type, capacity, accessibility status
- Only one entity is highlighted at a time; selecting another clears the previous highlight

#### R3.6
Active navigation route overlaid on 3D scene: indoor = solid blue tube, outdoor = dashed green tube. Overlay updates within 500ms of new location push.

**Acceptance Criteria:**
- Route tube renders within 500ms of receiving updated location via WebSocket
- Indoor edges render as solid blue `THREE.TubeGeometry`
- Outdoor edges render as dashed green `THREE.TubeGeometry`
- Route overlay clears when navigation ends or is cancelled

#### R3.7
If 3D model fails to load in 10 seconds, show error banner and fall back to SVG floor plan. No WebGL = static 2D map.

**Acceptance Criteria:**
- After 10 seconds without a successful model load, an error banner appears and the SVG floor plan loads from S3
- On a device with no WebGL support, the static 2D campus map is shown with text-based search
- Both fallback states are fully functional for search and navigation initiation

---

### R4 – Navigation Engine

The system shall compute real-time navigation routes between any two campus points, supporting indoor, outdoor, and multi-floor segments.

**User Stories:**
- As a student, I want turn-by-turn directions to my exam hall so that I arrive on time even in an unfamiliar block.
- As a wheelchair user, I want an accessible route that avoids stairs so that I can navigate independently.
- As a mobile user, I want my route to recalculate automatically if I take a wrong turn so that I stay on track.

#### R4.1
Navigation engine shall use A* on a weighted directed graph with 3D Euclidean heuristic.

**Acceptance Criteria:**
- Route from node A to node B returns the shortest weighted path
- Path computed on a 10-node test graph matches the known optimal path
- Heuristic uses `sqrt((x2-x1)² + (y2-y1)² + (z2-z1)²)`

#### R4.2
Accessible routing mode excludes all staircase edges and inaccessible nodes. Accessible route contains zero staircase edges.

**Acceptance Criteria:**
- `POST /nav/route` with `accessible: true` returns a path with zero staircase-type edges
- If no accessible path exists, response includes `has_accessible_path: false` and optionally offers the standard route
- `filter_accessible(graph)` unit test confirms all staircase edges removed

#### R4.3
Route steps include turn-by-turn instructions, node ID, distance in metres, and transition type.

**Acceptance Criteria:**
- Each step in the response contains: `instruction` (string), `node_id`, `distance_m` (float), `transition_type`
- `transition_type` is one of: walk, elevator, ramp, staircase, outdoor
- Instructions are human-readable (e.g. "Turn left at the corridor junction")

#### R4.4
Route generation ≤ 3 seconds (P95). Recalculation from new origin ≤ 2 seconds.

**Acceptance Criteria:**
- `POST /nav/route` P95 latency ≤ 3 seconds under normal load
- `PATCH /nav/route/{id}/recalc` P95 latency ≤ 2 seconds
- If recalculation exceeds 2 seconds, response includes `recalc_failed: true` and returns last valid route segment

#### R4.5
Mobile client sends WebSocket location updates every 3 seconds. On route deviation, system automatically recomputes.

**Acceptance Criteria:**
- Client sends location ping every 3 seconds while navigation is active
- Server detects deviation when user's current node is not on the active route
- Recalculation is triggered automatically without user interaction

#### R4.6
Arrival detected when user is within 15 metres of destination coordinates.

**Acceptance Criteria:**
- At approximately 15m from destination (±3m GPS tolerance), arrival is triggered
- At 20m from destination, arrival is not triggered
- On arrival, a destination-reached notification is shown and navigation ends

---

### R5 – Search

The system shall provide full-text and fuzzy search across all campus entity types with sub-2-second response time.

**User Stories:**
- As a student, I want to search "comupter lab" (misspelled) and still find the Computer Lab so that typos don't block me.
- As a user, I want the top result to be the exact match so that I don't have to scroll through partial matches.
- As a user with no results, I want suggestions for what to search instead so that I'm not left with a dead end.

#### R5.1
Search shall cover: Rooms, Facilities, Faculty, Departments, Events, and Parking Areas.

**Acceptance Criteria:**
- A query for a room name returns Room entities
- A query for a faculty name returns Faculty entities
- A query for an event title returns Event entities
- Response includes `entity_type` field on every result

#### R5.2
Exact match attempted first; fall back to fuzzy search (max 2 edits, prefix length 1).

**Acceptance Criteria:**
- "Computer Lab" returns exact match before any partial matches
- "Computar Lab" (1 edit) returns the correct result via fuzzy match
- "Cmoputer Lab" (2 edits) returns the correct result via fuzzy match
- 3+ edits return no fuzzy results (or very low score)

#### R5.3
Exact matches ranked 10× higher. Results capped at 20 total, 10 for fuzzy-only.

**Acceptance Criteria:**
- Exact match always appears as the first result when one exists
- Total results never exceed 20
- A fuzzy-only result set never exceeds 10

#### R5.4
Zero results returns 3–5 suggestions from most frequently searched terms.

**Acceptance Criteria:**
- Search returning zero results includes a non-empty `suggestions` array
- `suggestions` contains between 3 and 5 items
- Suggestions are drawn from the Redis frequency log, not hardcoded

#### R5.5
Empty or whitespace-only queries rejected with HTTP 400.

**Acceptance Criteria:**
- `GET /search?q=` returns HTTP 400 with `error: "query_required"`
- `GET /search?q=   ` (spaces only) returns HTTP 400
- Non-empty query of at least one non-whitespace character proceeds

#### R5.6
Search index unavailable returns HTTP 503. No stale cached results served.

**Acceptance Criteria:**
- When Atlas Search is unreachable, response is HTTP 503 with message "Search is temporarily unavailable"
- No results from cache are returned in the 503 response
- Client displays inline "Search is temporarily unavailable" error state

#### R5.7
Search response time ≤ 2 seconds at P95.

**Acceptance Criteria:**
- Under simulated load of 1,000 concurrent search requests, P95 latency remains ≤ 2 seconds
- Response time measured from request receipt to first byte of response

---

### R6 – Live Facility Status

The system shall display real-time occupancy and availability for campus facilities via live IoT sensor feeds or manual updates.

**User Stories:**
- As a student, I want to see if the library is near capacity before walking there so that I don't waste time.
- As a student, I want to see "Status Unavailable" clearly when the live feed is down so that I know not to rely on that data.
- As an admin, I want to configure the "Near Full" threshold per facility so that different spaces have appropriate alerts.

#### R6.1
Facilities displayed: parking, library, labs, cafeteria, classrooms.

**Acceptance Criteria:**
- `GET /facilities` returns entries for each of the five facility types
- Each entry includes `name`, `type`, `capacity`, `current_occupancy`, `status`, `feed_available`

#### R6.2
"Near Full" shown when `(current_occupancy / capacity) * 100 >= occupancy_threshold`. Default threshold 90%.

**Acceptance Criteria:**
- At 90% occupancy, "Near Full" badge is shown
- At 89% occupancy, "Near Full" badge is not shown
- Admin can update `occupancy_threshold` via `PATCH /facilities/{id}`
- Updated threshold takes effect on next occupancy push

#### R6.3
Status pushed via Firebase Realtime DB listeners. Max staleness 60 seconds.

**Acceptance Criteria:**
- Client receives occupancy update within 60 seconds of a change in Firebase
- Firebase `onValue` listener is active while the facility screen is open
- No polling; updates are push-based

#### R6.4
IoT feed not updated in 120 seconds is marked stale: `feed_available: false`.

**Acceptance Criteria:**
- `feed_available` becomes `false` when `last_updated` is more than 120 seconds ago
- The field is updated in Firebase Realtime DB, not just on the client
- Downstream clients immediately see the updated `feed_available: false`

#### R6.5
`feed_available: false` → client removes all occupancy/status data and shows "Status Unavailable".

**Acceptance Criteria:**
- No occupancy numbers or status badges visible when `feed_available` is `false`
- "Status Unavailable" text is shown per affected facility
- When feed recovers (`feed_available: true`), live data is immediately re-displayed

#### R6.6
Firebase connection lost → client immediately clears all live facility data and shows "Status Unavailable" per facility.

**Acceptance Criteria:**
- On Firebase disconnection event, all occupancy values are removed from UI within 1 second
- "Status Unavailable" badge shown for every facility
- On reconnection, live data resumes displaying automatically

---

### R7 – Faculty Locator

The system shall provide a searchable faculty directory with office location, availability, and contact information.

**User Stories:**
- As a student, I want to find Dr. Sharma's office and get directions so that I can attend office hours without getting lost.
- As a faculty member, I want to update my own office hours so that students always see accurate availability.
- As a visitor, I want to see faculty names and departments but not personal contact details so that PII is protected.

#### R7.1
Faculty profile displays: name, department, designation, office room, office hours, contact email, phone, availability status.

**Acceptance Criteria:**
- `GET /faculty/{id}` response contains all 8 fields
- Contact email and phone are omitted when requester has `role: "guest"`
- Availability status is one of: `available`, `unavailable`

#### R7.2
If `office_room_id` is null, display "Location not available" and suppress navigation option.

**Acceptance Criteria:**
- Faculty profile with `office_room_id: null` shows "Location not available"
- No navigation button is rendered when office room is null
- Faculty with a valid `office_room_id` shows a "Navigate" button that launches turn-by-turn directions

#### R7.3
Faculty can update their own office hours and contact info. Updates propagate to Search and AI knowledge base within 5 minutes.

**Acceptance Criteria:**
- `PATCH /faculty/{id}` with faculty's own JWT updates `office_hours` and contact fields
- Search results for that faculty member reflect the update within 5 minutes
- AI assistant responses about that faculty member reflect the update within 5 minutes

#### R7.4
Guest/Visitor users shall not see faculty contact email and phone.

**Acceptance Criteria:**
- `GET /faculty/{id}` with guest JWT returns profile without `contact_email` and `contact_phone`
- Fields are omitted from the response object entirely, not returned as null or empty string

#### R7.5
Faculty can set availability status to `available` or `unavailable`.

**Acceptance Criteria:**
- `PATCH /faculty/{id}/status` with valid faculty JWT updates status in MongoDB
- Updated status is visible in `GET /faculty/{id}` response immediately after update
- Only `available` and `unavailable` are accepted values; other values return HTTP 400

---

### R8 – Event Navigation

The system shall display campus events and enable navigation directly to event venues.

**User Stories:**
- As a student, I want to find today's workshops and navigate to the venue so that I don't miss them.
- As a user who bookmarked an event, I want a push notification when it's cancelled so that I can adjust my plans.
- As an event coordinator, I want created events to appear in search within 2 minutes so that students find them quickly.

#### R8.1
Events include: workshops, exams, seminars, college events. Each record shows: title, type, venue room, start datetime, end datetime, and organiser name.

**Acceptance Criteria:**
- `GET /events` response includes all 6 fields per event: title, type, venue, start_datetime, end_datetime, organiser
- Events are sorted by `start_datetime` ascending
- `type` is one of: seminar, workshop, exam, college_event, other

#### R8.2
Events appear in search results within 2 minutes of creation or update.

**Acceptance Criteria:**
- After `POST /events`, the event appears in `GET /search?q={title}` within 2 minutes
- After `PATCH /events/{id}`, updated title/description is searchable within 2 minutes
- Achieved via MongoDB change stream triggering Search Service re-index

#### R8.3
Users can bookmark events. Bookmarked users receive FCM push on update or cancellation.

**Acceptance Criteria:**
- `POST /events/{id}/bookmark` adds user to `bookmarked_by` array (idempotent toggle)
- On `PATCH /events/{id}`, FCM notification sent to all users in `bookmarked_by`
- On `DELETE /events/{id}`, FCM notification sent to all users in `bookmarked_by`
- Users not in `bookmarked_by` receive no notification

#### R8.4
Events with a mapped venue room include a navigation button that launches turn-by-turn directions.

**Acceptance Criteria:**
- Event with a valid `venue_room_id` that has a `graph_node_id` shows a "Navigate to Venue" button
- Clicking the button initiates navigation from user's current location to the venue node
- Event with `venue_room_id: null` does not show a navigation button

---

### R9 – Emergency Guidance

The system shall provide immediate emergency exit routes, first-aid locations, security contacts, and campus-wide alert broadcasting.

**User Stories:**
- As anyone on campus, I want to see the nearest fire exit immediately when I open the emergency screen so that I can react without delay.
- As an admin, I want to broadcast an emergency alert to all active users so that everyone is informed within 10 seconds.
- As a mobile user receiving an alert, I want a full-screen overlay so that I cannot miss the emergency notification.

#### R9.1
Emergency Module shows nearest fire exit, first-aid point, and security office within 1 second of opening.

**Acceptance Criteria:**
- `GET /emergency/nearest?node_id={id}` returns in ≤ 1 second (O(1) Redis lookup)
- Response includes nearest fire exit, first-aid location, and security office node IDs and names
- Pre-computation runs at service startup; no real-time graph traversal at query time

#### R9.2
Emergency route excludes `is_hazardous: true` nodes and routes to nearest `is_emergency_exit: true` node.

**Acceptance Criteria:**
- `POST /emergency/route` response path contains zero nodes with `is_hazardous: true`
- Destination is always the nearest reachable `is_emergency_exit: true` node
- If all emergency exits are unreachable, response returns floor plan fallback

#### R9.3
If emergency route computation exceeds 5 seconds, fall back to floor plan with all exits marked.

**Acceptance Criteria:**
- Route computation that takes > 5 seconds returns a floor plan response, not a timeout error
- Floor plan response includes positions of all `is_emergency_exit: true` nodes on the current floor
- Client renders the floor plan fallback with exit markers visible

#### R9.4
Admin can broadcast campus-wide emergency alert delivered via FCM to all active sessions in last 5 minutes within ≤ 10 seconds.

**Acceptance Criteria:**
- `POST /emergency/alert` fans out FCM to all tokens in `active_sessions` Redis set
- Delivery completes within 10 seconds for the full active session set
- Only sessions active within the last 5 minutes receive the broadcast

#### R9.5
Mobile client shows full-screen alert overlay on receiving emergency broadcast.

**Acceptance Criteria:**
- FCM notification triggers a full-screen overlay component
- Overlay cannot be dismissed by back gesture; requires explicit acknowledgement
- Overlay displays alert message text, nearest exit direction, and emergency contact number

---

### R10 – Visitor Mode

The system shall provide an unauthenticated visitor experience with simplified navigation to key public campus destinations.

**User Stories:**
- As a parent visiting for admissions, I want to navigate to the Admission Block without creating an account so that I can find it quickly.
- As a visitor, I want the AI assistant to help me navigate within the public areas so that I don't need a human guide.

#### R10.1
Visitor mode requires no registration. Guest session token issued automatically.

**Acceptance Criteria:**
- Opening the app without logging in offers "Enter as Visitor" option
- Tapping it calls `POST /auth/guest` and stores the guest JWT automatically
- No form, email, or any personal data is requested

#### R10.2
Visitor map displays only: departments, principal's office, admission block, parking, cafeteria, reception.

**Acceptance Criteria:**
- Visitor map renders exactly the 6 public destination categories
- Staff offices, faculty offices, and internal rooms are not visible on the visitor map
- Attempting to navigate to a non-public location is blocked with an appropriate message

#### R10.3
Staff-only locations and faculty contact PII not visible in Visitor mode.

**Acceptance Criteria:**
- Faculty contact email and phone are absent from all visitor-accessible responses
- Staff office nodes are excluded from the visitor navigation graph view
- Guest JWT cannot call any endpoint that returns restricted data

#### R10.4
Visitors can use AI assistant, 3D map, search, and navigation within restricted scope.

**Acceptance Criteria:**
- AI assistant responds to navigation queries within public destination scope
- Search returns only public entities for guest JWT requests
- 3D map renders for visitors but room detail panels for restricted rooms are suppressed

---

### R11 – Admin Console

The system shall provide an Administrator interface for managing campus data, monitoring the system, and broadcasting announcements.

**User Stories:**
- As an admin, I want to add a new room and have it immediately available in navigation so that the map stays current.
- As an admin, I want all my changes logged so that I can audit who changed what and when.
- As an admin, I want to broadcast a text announcement to all active users so that I can communicate important information instantly.

#### R11.1
Admins can create, update, and delete Rooms and Facilities. Room name ≤ 100 chars; capacity 1–10,000.

**Acceptance Criteria:**
- `POST /admin/rooms` with valid data creates a room and returns HTTP 201
- Room name > 100 characters returns HTTP 400
- Capacity < 1 or > 10,000 returns HTTP 400
- Non-admin JWT returns HTTP 403 on all admin room endpoints

#### R11.2
Deleting a Room atomically cascades: remove graph edges/nodes, delete related events, trigger search removal. Failure rolls back and alerts admin.

**Acceptance Criteria:**
- After successful `DELETE /admin/rooms/{id}`, no navigation graph edges reference that room
- After successful deletion, no events have `venue_room_id` matching that room
- After successful deletion, room no longer appears in search results
- If any cascade step fails, all steps are rolled back; room still exists; alert written to `admin_notifications`

#### R11.3
Every mutating Room or Facility operation writes an audit log with full change details. Retained 90 days.

**Acceptance Criteria:**
- Create, update, and delete operations each produce one audit log entry
- Entry contains: `admin_id`, `action`, `entity_type`, `entity_id`, `field_changed`, `previous_value`, `new_value`, UTC `timestamp`
- Entries older than 90 days are automatically deleted via MongoDB TTL index

#### R11.4
Admins can broadcast text announcements ≤ 500 chars. Appears to all active users within 2 minutes.

**Acceptance Criteria:**
- Announcement > 500 characters returns HTTP 400 with `error: "announcement_too_long"`
- Valid announcement is visible to all logged-in users within 2 minutes via Firebase push
- Announcement is not stored permanently; it is a broadcast-only event

#### R11.5
Admin Console accessible only to `role: "admin"`. All other roles receive HTTP 403.

**Acceptance Criteria:**
- Faculty, Student, Security Staff, and Guest JWTs all receive HTTP 403 on admin endpoints
- Admin JWT successfully accesses all admin endpoints
- RBAC middleware rejects non-admin requests before any business logic executes

---

### R12 – Performance and Scalability

The system shall meet defined latency targets and support high concurrency without degradation.

**Acceptance Criteria (all measured at P95 under load):**
- Search response ≤ 2 seconds
- Route generation ≤ 3 seconds
- Auth login ≤ 2 seconds
- System supports 20,000 concurrent users without latency degradation
- Above capacity: HTTP 503 returned rather than crash or hang
- System availability ≥ 99.9% measured monthly
- MongoDB Atlas backups at ≤ 24-hour intervals; RTO ≤ 1 hour (tested and documented)

---

### R13 – Security

The system shall protect user data, prevent unauthorised access, and defend against common web application threats.

**Acceptance Criteria:**
- All endpoints accessible only over HTTPS/WSS (TLS 1.2+)
- MongoDB PII fields (email, phone) encrypted at field level with AES-256
- No secrets hardcoded in source code or version control; all stored in AWS Secrets Manager
- JWT signing key rotated every 90 days via automated Secrets Manager rotation
- AWS WAF enforces: SQL injection protection, XSS filtering, rate limit 1,000 req/min/IP
- S3 buckets are private; 3D assets served only via CloudFront signed URLs
- Expired or revoked JWT rejected on every protected endpoint before payload processing
- OWASP ZAP scan on staging shows zero high-severity and zero medium-severity findings

---

### R14 – Accessibility

The system shall meet WCAG 2.1 Level AA standards across all user-facing interfaces.

**Acceptance Criteria:**
- All web views pass automated axe-core scans with zero WCAG 2.1 AA violations in CI
- All interactive elements keyboard-navigable; focus managed correctly on modal open/close
- All 3D map elements in high-contrast mode meet 4.5:1 contrast ratio
- AI assistant supports voice commands on both Android and iOS
- Screen reader walkthrough (NVDA + Chrome; VoiceOver + Safari) passes on Search, Navigation, and AI Chat flows

---

## Glossary

| Term | Definition |
|---|---|
| **Digital Twin** | Real-time virtual model of the physical campus mirroring buildings, rooms, and facilities |
| **RAG** | Retrieval-Augmented Generation — knowledge base chunks injected into LLM prompt to ground responses |
| **LOD** | Level of Detail — 3D models at lower resolution as camera distance increases to maintain frame rate |
| **A\*** | A-star — graph search algorithm finding shortest path using a heuristic |
| **JWT** | JSON Web Token — signed token carrying user identity and role claims |
| **RBAC** | Role-Based Access Control — permissions assigned to roles, users assigned to roles |
| **FCM** | Firebase Cloud Messaging — Google push notification service for Android and iOS |
| **RTO** | Recovery Time Objective — maximum acceptable time to restore service after failure |
| **WCAG** | Web Content Accessibility Guidelines — international accessibility standard |
| **WAF** | Web Application Firewall — filters and blocks malicious HTTP requests |
| **Navigation Graph** | Weighted directed graph of all walkable paths across campus |
| **Intent** | Classified purpose of an AI query: navigation, entity lookup, general info, or unknown |
| **P95 latency** | 95th percentile response time — 95% of requests complete within this duration |
| **Cascade deletion** | Atomic deletion of a parent record removing all referencing child records |
| **Feed** | Live data stream from IoT sensors supplying real-time occupancy to Facility Monitor |
| **Sliding TTL** | Cache expiry that resets on each access, persisting entries while actively used |

---

## Out of Scope (Phase 1)

| Feature | Reason for Deferral |
|---|---|
| Student attendance tracking | Requires academic management system integration |
| Fee payment portal | Requires payment gateway and PCI-DSS compliance |
| Hostel management | Separate operational domain |
| Learning Management System | Academic content delivery is a separate product |
| Online examination | Requires proctoring and exam integrity systems |
| AR navigation | Requires ARCore/ARKit; deferred to Phase 2 |
| Timetable integration | Requires live sync with college scheduling system |
| Multi-language support | Phase 1 targets English only |
| IoT smart parking guidance | Requires hardware installation beyond current scope |
| Personalised AI recommendations | Requires behaviour tracking and ML model; Phase 2 feature |

---

## Constraints

### Technical
- Backend: Python (FastAPI) only
- Web frontend: React.js with TypeScript only
- Mobile: Flutter only; no native iOS/Android
- Cloud: AWS + Firebase only

### Business
- Must run on standard college campus network with no special hardware beyond smartphones
- Third-party APIs (OpenAI, Google Speech, Firebase) must operate within free-tier or low-cost tiers for the demo phase
- Campus data must be obtained with explicit institutional permission

### Compliance
- Personal data (name, email, student ID) handled per applicable data protection regulations
- Faculty contact PII must not be exposed to unauthenticated (Guest) users

---

## Change Log

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-06-30 | Team Kero | Initial version — 14 requirements with user stories and acceptance criteria |
