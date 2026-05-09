# Product Requirements Document
## FitMarket — A Marketplace App for Local Fitness Providers

**Version:** 1.0
**Date:** May 2026
**Status:** Ready for design and development

---

## 1. Product Vision

FitMarket is a mobile marketplace that connects independent, local fitness providers — small gym owners, yoga instructors, Zumba teachers, boxing coaches, meditation guides — with end customers in their city. It gives small fitness businesses a zero-effort digital presence (no need to build their own app), and gives customers one app to discover, subscribe to, and manage all their fitness memberships.

Think of it as **"Zomato for fitness"** — but with subscriptions instead of one-time orders, and digital check-ins instead of delivery.

### The problem we are solving

- Small gym owners and independent fitness providers cannot afford to build their own apps. They rely on paper registers, WhatsApp, and word-of-mouth.
- Customers in tier-1 and tier-2 Indian cities don't know which small gyms or classes exist near them. Discovery is broken.
- Existing apps like Cult.fit own their gyms — they are not a marketplace. There is no Zomato-equivalent for fitness providers in India yet.
- Customers who attend more than one type of class (e.g., gym + yoga + Zumba) currently juggle multiple WhatsApp groups, paper receipts, and reminders. There is no single place to manage all fitness subscriptions.

### What makes FitMarket different

- **Marketplace, not a chain** — we onboard existing local providers; we don't open our own centers.
- **Multi-discipline** — gyms, yoga, Zumba, boxing, meditation, dance, all in one app.
- **Light onboarding** — a gym owner needs only logo, photos, address, services, and plans to go live.
- **Owner-side mobile app** — the gym owner uses the same app (different role) to scan customer QR codes and manage subscribers. No separate hardware, no installed scanner.
- **Anti-leakage by design** — no free-text chat between members and owners. All needed information lives on the gym profile page itself, so commission revenue is protected from day one.

---

## 2. Target Users

### Primary Persona 1 — The Small Gym Owner ("Provider")
- Owns a single neighborhood gym, yoga studio, or Zumba class
- 25–50 attendees on average
- Currently uses paper registers or a basic Excel sheet
- Cannot afford a custom app or a full-time admin staff
- Wants more visibility, easier subscription tracking, and automatic reminders for renewals

### Primary Persona 2 — The Fitness Customer ("Member")
- 18–45 years old, urban or semi-urban
- May attend more than one type of class
- Wants to discover nearby options, compare plans, and manage all memberships in one place
- Currently pays in cash or UPI directly to the gym; no digital record

### Secondary Persona — The Platform Admin
- Created only by direct database entry for the first admin; subsequent admins via an admin-only API endpoint
- Verifies gym onboarding requests through admin screens inside the mobile app
- Can also onboard gyms manually on behalf of providers
- Suspends gyms or members in case of fraud or disputes

---

## 3. Differentiation vs Cult.fit and Similar Apps

| Dimension | Cult.fit | FitMarket |
|---|---|---|
| Business model | Owns its centers | Marketplace for independent providers |
| Onboarding | N/A — they are the gym | Self-service or admin-assisted |
| Discovery | Only Cult locations | Any enrolled local provider |
| Pricing power | Cult sets it | Each provider sets their own plans |
| Target | Urban metros, mid-premium | All cities including underserved tier-2/tier-3 |
| Owner tooling | Internal staff app | Public provider role inside the same app |

We are not competing with Cult.fit head-on. We are filling the gap below them — the long tail of small gyms that Cult will never own.

---

## 4. Phased Roadmap

The product ships in phases, not in one big release. Each phase ships only when the previous phase is stable and validated.

### V1 — MVP (the focus of this PRD)

**Goal:** Prove that gym owners will list themselves and customers will subscribe through the app. No revenue logic yet — just adoption.

Scope:
- Gym owner onboarding (self-serve + admin-onboarded)
- Customer mobile signup (phone OTP via Firebase Phone Auth)
- Gym discovery (nearby + browse all)
- Subscription request flow (no online payment)
- Single global rotating QR for check-in
- Owner-side subscriber management and QR scanner
- Admin role inside mobile app (first admin via DB; subsequent via API)
- Push notifications via Firebase Cloud Messaging

### V2 — Monetization
- Razorpay payment gateway integration
- Automatic split — platform commission + provider payout
- Auto-renewal and expiry reminders
- Refund and dispute flow
- Owner analytics dashboard (footfall, revenue, churn)
- Web-based admin panel (built when gym count crosses ~50)
- Mandatory KYC and bank details for providers

### V3 — Stickiness and Content
- In-app workout library (body-part-wise GIFs / videos via free or licensed APIs)
- Provider-uploaded private video lectures
- Trainer-led live sessions
- Push promotions and offers
- Referral program
- (Possible) masked-call communication via Exotel-style virtual numbers — only if absolutely needed, never free-text chat

### V4 (Future)
- Personal trainer marketplace
- Diet / nutrition plans
- Wearables integration
- AI workout recommendations

The rest of this PRD covers V1 only in detail.

---

## 5. V1 Feature List

### 5.1 Authentication and Roles

- Single app, two user-selectable roles at signup: **Member** or **Provider**
- Login via mobile number + OTP using **Firebase Phone Auth** (free tier — up to 10K verifications/month)
- The Spring Boot backend verifies Firebase ID tokens via Firebase Admin SDK and issues its own session JWT for all subsequent API calls
- A third role, **Admin**, exists in the data model but is never selectable from the UI:
  - The first admin is created by direct database update, one time only
  - All subsequent admins are promoted via an admin-only API endpoint (`POST /api/v1/admin/users/{id}/promote`) which requires an existing admin's JWT
  - There is no "create admin" path that does not pass through either the database or an existing admin's authority
- Profile basics: name, email (optional), photo (optional), city

### 5.2 Provider (Gym Owner) Features

**Onboarding**

Required fields, all collected in one form:
- Gym name, owner name, phone, city
- Full address with map pin
- Gym logo (single image)
- Gym photos (up to 5)
- Description (free text — provider can describe their gym in their own words)
- Services offered (multi-select: Gym, Yoga, Zumba, Boxing, Meditation, Dance, Other)
- Opening time and closing time
- Trainers available (yes/no, count if yes)
- Equipment list (multi-select: Treadmill, Dumbbells, Cables, Free Weights, Mats, Cardio Machines, Other)
- Parking available (yes/no)
- Women-only hours (text — e.g., "10am-12pm" or "None")
- Dress code (free text)

Status flow: `pending_verification` → `active` (after admin approval) → can also be `suspended` or `rejected`.

Bank details field exists in the schema but is optional in V1 (mandatory in V2).

**Subscription Plan Setup**
- Provider can create multiple plans: e.g., "1 month — ₹1500", "3 months — ₹4000", "1 year — ₹12000"
- Fields per plan: name, duration (days), price, description, active/inactive toggle

**Subscriber Management**
- List of all customers who have requested or active subscriptions
- Two tabs: **Pending Requests** | **Active Members**
- For each pending request: Approve or Reject button. Approve = mark active, set start/end dates. Reject = optional reason
- For each active member: see plan, start date, expiry date, days remaining

**QR Scanner (Check-in)**
- A "Scan" button on the home screen
- Opens the camera, scans the customer's QR
- Backend verifies (a) the QR token is fresh and signed correctly, (b) this user has an active subscription at this gym (the scanning gym is identified by the logged-in provider)
- Shows customer name, photo, plan, expiry status. Marks check-in. If expired, not subscribed, or token stale, shows clear red warning

**Provider Profile Page (public-facing)**
- This is what customers see when they tap the gym in the discovery list
- Shows logo, photos, description, services, address with map, opening hours, equipment list, parking info, women-only hours, dress code, trainer info, plans, and a "Request to Join" button

### 5.3 Customer (Member) Features

**Discovery**
- Home screen shows nearby gyms first (sorted by distance using device location), then a "Browse All" section below
- Filters: service type (Gym / Yoga / Zumba / etc.), price range, distance
- Search bar (search by gym name or service)

**Gym Detail Page**
- Photos, logo, description, services, address with map, opening hours, equipment list, parking info, women-only hours, dress code, trainer info, list of available plans
- "Request to Join" CTA — opens plan selection

**Subscription Request**
- Customer selects a plan → confirms → request goes to gym owner as `pending`
- Customer sees a card on their My Subscriptions screen with status `Pending Approval`
- After owner approves, status flips to `Active`. Owner is expected to collect payment offline (cash / UPI) and mark active in the app
- Customer gets a push notification on approval

**My Subscriptions Screen**
- Shows one card per active or pending subscription
- Each card has: gym logo, gym name, service, plan name, days remaining, status badge
- Tap a card → see full subscription detail (plan history, check-in count, expiry)

**My QR Code (Global Rotating)**
- A single QR code is permanently visible on the customer's home/dashboard screen
- The customer does not navigate into any specific subscription to display it
- The QR encodes: `user_id + timestamp + signed_token` where the token is signed with a per-user secret
- The QR rotates every 60 seconds (regenerated client-side using the server-issued signing secret)
- When a gym owner scans it:
  - Backend verifies the token's signature and freshness
  - Backend looks up: does this `user_id` have an active subscription at the scanning gym (the gym is identified by the logged-in provider)?
  - Returns success or appropriate error (expired token, no subscription at this gym, expired subscription)
- A screenshot of the QR is invalid after 60 seconds, preventing sharing

**Profile**
- Edit name, photo, city
- Logout

### 5.4 Admin Features (inside the mobile app)

The admin role lives inside the same mobile app. There is no separate admin app and no web panel in V1.

When a user with `role = admin` logs in, they see admin-only screens instead of the member home:

- **Pending Gym Queue** — list of gyms with status `pending_verification`. Tap a gym → see all submitted info → Approve or Reject (with reason)
- **Manual Gym Onboarding** — admin can fill out the gym onboarding form on behalf of a provider who is not tech-comfortable. After submission, admin can immediately approve
- **Search** — search by gym name, member name, or phone number. View any record
- **Suspend / Reactivate** — flip the status of any gym or member
- **Promote to Admin** — admin can promote any existing user to admin via this screen (calls the `/admin/users/{id}/promote` endpoint)
- **Stats card** — total gyms (active/pending/suspended), total members, total active subscriptions, check-ins this week

For raw data access, exports, and emergency fixes, the developer uses TablePlus connected to Supabase Postgres. No mobile UI is built for those.

---

## 6. Critical User Flows

### Flow A — Gym Owner Self-Onboarding
1. Download app → choose role "I am a gym owner"
2. Enter phone → OTP verification via Firebase Phone Auth
3. Fill onboarding form (name, address, services, photos, plans, equipment, parking, dress code, etc.)
4. Submit → status `pending_verification`
5. Admin reviews on mobile admin screens → approves
6. Owner gets push notification → "You're live on FitMarket"
7. Owner can now appear in customer discovery and accept subscription requests

### Flow B — Customer Subscribes to a Gym (offline payment)
1. Customer logs in via OTP
2. Allows location access → sees nearby gyms
3. Taps a gym → views detail (with all profile info) → taps "Request to Join"
4. Selects a plan → confirms
5. Subscription is created with status `pending`
6. Owner sees the request, contacts the customer offline (phone number visible only to provider for the pending request), collects cash/UPI, then taps "Approve" in app
7. Status flips to `active`. Customer gets push notification
8. Customer can now show their global QR on home screen at the gym to check in

### Flow C — Daily Check-in (Global Rotating QR)
1. Customer arrives at gym. Opens app — QR is right on home screen
2. Owner opens app → taps Scan → scans QR
3. Backend verifies token freshness + signature + subscription validity at this specific gym
4. App shows customer's name, photo, plan, status (green = active, red = expired/invalid/wrong gym)
5. Check-in is recorded with timestamp
6. QR has already rotated to a new value — even if the owner screenshots it, it's invalid in 60 seconds

### Flow D — Admin Promotes a New Admin
1. Existing admin logs in → opens admin home
2. Searches for the user by phone or name
3. Taps "Promote to Admin" on the user's record
4. Confirmation prompt → confirms
5. Backend updates `users.role = 'admin'` for that user
6. User, on next login, sees admin home instead of member home

---

## 7. Tech Stack

| Layer | Choice | Free tier limit |
|---|---|---|
| Mobile framework | Flutter (latest stable) | Free forever |
| Mobile language | Dart | Free forever |
| Backend language | Java 21 | Free forever |
| Backend framework | Spring Boot 3.x | Free forever |
| Database | PostgreSQL 16 (hosted on Supabase) | 500 MB storage, 2 GB bandwidth/month free |
| ORM | Spring Data JPA + Hibernate | Free forever |
| DB migrations | Flyway | Free forever |
| Auth (OTP) | Firebase Phone Auth | 10K verifications/month free |
| Backend session | Spring Security + JWT | Free forever |
| File storage | Cloudflare R2 | 10 GB storage + 1M reads/month free |
| Push notifications | Firebase Cloud Messaging | Unlimited free |
| Maps | Google Maps SDK for Flutter | $200/month free credit |
| QR generation (mobile) | `qr_flutter` package | Free |
| QR scanning (mobile) | `mobile_scanner` package | Free |
| API documentation | Springdoc OpenAPI (auto-generates Swagger UI) | Free |
| Backend hosting | Render free tier | Free with cold starts; $7/mo for always-on |
| Source control | GitHub | Free for personal repos |
| CI/CD | GitHub Actions | 2000 minutes/month free |
| Local DB GUI | TablePlus | Free version available |
| API testing | Postman | Free |

**Note on geospatial:** Postgres has built-in earthdistance/cube extensions sufficient for V1 "nearby" queries via Haversine formula. PostGIS is only needed when gym count crosses 10K+.

---

## 8. Data Model

PostgreSQL schema. All tables have `id` (UUID, primary key), `created_at`, `updated_at` unless noted.

### `users`
```
id                UUID PK
phone             VARCHAR(15) UNIQUE NOT NULL
name              VARCHAR(100)
email             VARCHAR(150)
photo_url         TEXT
role              ENUM('member', 'provider', 'admin') NOT NULL
city              VARCHAR(100)
fcm_token         TEXT
qr_signing_secret VARCHAR(64)  -- per-user secret for signing rotating QR tokens
status            ENUM('active', 'suspended') DEFAULT 'active'
created_at        TIMESTAMP
updated_at        TIMESTAMP
```

### `gyms`
```
id                  UUID PK
owner_user_id       UUID FK → users.id
name                VARCHAR(150) NOT NULL
description         TEXT
logo_url            TEXT
photo_urls          TEXT[]
services            TEXT[]   -- e.g., ['gym', 'yoga']
address             TEXT
city                VARCHAR(100)
latitude            DECIMAL(10, 7)
longitude           DECIMAL(10, 7)
opening_time        TIME
closing_time        TIME
trainers_available  BOOLEAN
trainer_count       INT
equipment           TEXT[]
parking_available   BOOLEAN
women_only_hours    VARCHAR(100)
dress_code          TEXT
status              ENUM('pending_verification', 'active', 'suspended', 'rejected')
created_at          TIMESTAMP
updated_at          TIMESTAMP
```

### `plans`
```
id              UUID PK
gym_id          UUID FK → gyms.id
name            VARCHAR(100)
duration_days   INT
price           DECIMAL(10, 2)
description     TEXT
is_active       BOOLEAN DEFAULT TRUE
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### `subscriptions`
```
id               UUID PK
user_id          UUID FK → users.id
gym_id           UUID FK → gyms.id
plan_id          UUID FK → plans.id
status           ENUM('pending', 'active', 'expired', 'rejected', 'cancelled')
start_date       DATE
end_date         DATE
approved_at      TIMESTAMP
rejection_reason TEXT
created_at       TIMESTAMP
updated_at       TIMESTAMP
```

### `checkins`
```
id              UUID PK
subscription_id UUID FK → subscriptions.id
user_id         UUID FK → users.id
gym_id          UUID FK → gyms.id
checked_in_at   TIMESTAMP
```

### `bank_details` (schema present, optional in V1; mandatory in V2)
```
id              UUID PK
gym_id          UUID FK → gyms.id
account_holder  VARCHAR(150)
account_number  VARCHAR(30)
ifsc            VARCHAR(15)
upi_id          VARCHAR(100)
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

---

## 9. API Surface

All endpoints are JSON over HTTPS. JWT in `Authorization: Bearer <token>` header. Documented via Swagger UI auto-generated by Springdoc.

**Auth**
- `POST /api/v1/auth/firebase-verify` — accepts Firebase ID token (from mobile app), returns FitMarket session JWT
- `POST /api/v1/auth/refresh` — refresh JWT

**User**
- `GET /api/v1/users/profile` — get logged-in user's profile
- `PATCH /api/v1/users/profile` — update logged-in user's profile (name, photo, city, email)

**Gyms — Provider**
- `POST /api/v1/gyms` — create gym (provider only)
- `PATCH /api/v1/gyms/{id}` — update own gym
- `GET /api/v1/gyms/{id}/subscribers?status=pending|active` — list subscribers

**Gyms — Customer (public)**
- `GET /api/v1/gyms/nearby?lat=&lng=&radius_km=` — nearby first
- `GET /api/v1/gyms?service=&city=&page=` — browse all
- `GET /api/v1/gyms/{id}` — gym detail

**Plans**
- `POST /api/v1/gyms/{gym_id}/plans` — provider creates
- `GET /api/v1/gyms/{gym_id}/plans` — list
- `PATCH /api/v1/plans/{id}`
- `DELETE /api/v1/plans/{id}`

**Subscriptions**
- `POST /api/v1/subscriptions` — customer requests
- `GET /api/v1/subscriptions/me` — customer's own subscriptions
- `POST /api/v1/subscriptions/{id}/approve` — provider approves
- `POST /api/v1/subscriptions/{id}/reject` — provider rejects

**Check-in**
- `GET /api/v1/qr/token` — customer fetches a fresh signed QR token (called by client every 60s)
- `POST /api/v1/checkins/scan` — provider posts the scanned QR payload, backend verifies and records

**File Uploads**
- `POST /api/v1/uploads/presign` — returns a pre-signed Cloudflare R2 URL for direct mobile upload (gym logos, photos, profile pictures)

**Admin** (requires `role = admin`)
- `GET /api/v1/admin/gyms?status=pending_verification`
- `POST /api/v1/admin/gyms/{id}/approve`
- `POST /api/v1/admin/gyms/{id}/reject`
- `POST /api/v1/admin/gyms/{id}/suspend`
- `POST /api/v1/admin/users/{id}/suspend`
- `POST /api/v1/admin/users/{id}/promote` — promotes user to admin role
- `GET /api/v1/admin/stats`
- `GET /api/v1/admin/search?query=` — search users and gyms

---

## 10. Security Notes

- All API endpoints require JWT except `firebase-verify` (which itself validates the Firebase ID token)
- Spring Security role-based access control: `@PreAuthorize("hasRole('PROVIDER')")`, `@PreAuthorize("hasRole('ADMIN')")` etc.
- The `qr_signing_secret` per user is generated server-side, stored in DB (encrypted at rest by the DB), and shared with the client over HTTPS once on first login. Used to sign QR tokens client-side. Server verifies on scan
- All file uploads (logos, photos) go through pre-signed R2 URLs, never as multipart through the Spring backend (avoids large payload load on backend)
- Phone numbers of customers are visible to providers only for active or pending subscriptions — not for browsing
- No API endpoint mutates the `role` field except the explicit admin-only `/admin/users/{id}/promote` endpoint
- The very first admin must be created by direct DB update; there is no API path to bootstrap the first admin
- Rate limiting on Firebase OTP verification handled by Firebase itself; backend additionally rate-limits the `/firebase-verify` endpoint (3 attempts per phone per hour)
- All passwords / secrets / API keys are stored in environment variables on Render, never in code or Git
- HTTPS enforced everywhere (Render provides this automatically)

---

## 11. Success Metrics

V1 is about proving adoption, not revenue. Targets to track from day one:

- Number of gyms onboarded — target: 20 verified gyms in first city (Bengaluru) in 3 months
- Number of customers signed up — target: 500
- Number of active subscriptions — target: 150
- Number of check-ins per week per active gym — target: avg 30
- Gym owner retention — target: 80% still active after 30 days
- Customer retention — target: 40% check in at least once a week

If these numbers look healthy, V2 (payments) is justified. If they don't, iterate on V1 before adding complexity.

---

## 12. Open Questions and Risks

**Open questions to resolve before V2:**
- Commission rate per booking (industry standard: 8–15%)
- Refund policy when a gym shuts down mid-subscription
- Cancellation policy for customers
- Payout frequency to gym owners (weekly? bi-weekly?)

**Risks:**
- **Cold-start problem** — empty marketplace. Mitigation: personally onboard the first 10–20 gyms in one neighborhood before launching publicly
- **Owner adoption friction** — small gym owners may not be tech-comfortable. Mitigation: keep onboarding under 5 minutes; support both self-serve and admin-onboarded paths
- **No-show / fake check-ins** — owner could mark friends as members. Mitigation: not a real concern in V1 (no money flowing); revisit in V2
- **Trust gap** — customers may not trust an unknown app. Mitigation: clearly show "verified" badge after admin approval; show photos and plans transparently
- **Render free tier cold starts** — backend sleeps after 15 min idle, first request takes ~30s to wake. Mitigation: acceptable for V1 testing; upgrade to $7/mo when real users join

---

## 13. Out of Scope for V1

These will tempt scope creep — explicitly deferring:
- Online payments (V2)
- Free-text chat between members and providers — never; gym profile shows all needed info
- Web admin panel (V2, when gym count > ~50)
- Workout video library (V3)
- Live trainer sessions (V3)
- Provider analytics dashboard (V2)
- Multi-language support (V2+)
- Dark mode (V2+)
- Web app for customers (V3+)
- Wearables / fitness tracker integration (V4)
- Custom domain (deferred until launch — Render's free subdomain is fine for V1)

---

## 14. Decision Log

A record of "we chose X over Y because Z" — so future contributors and Claude Code understand the why behind each choice.

| Decision | Choice | Rejected alternative | Reason |
|---|---|---|---|
| Mobile framework | Flutter | React Native, native iOS+Android | Single codebase, mature ecosystem |
| Backend language | Java 21 + Spring Boot | Node.js, Python, Firebase functions | Real SQL transactions needed for V2 payments, no vendor lock-in |
| Database | PostgreSQL | MongoDB, Firestore | Entities deeply relational (subscriptions tie users + gyms + plans); joins matter; transactions matter for V2 |
| QR strategy | Single global rotating QR | Per-subscription QR | Better UX, security solved by 60s token rotation, multi-gym ambiguity solved by scanner identity |
| Communication | Profile fields only | Free-text chat, FAQ section, action buttons | All info on the profile prevents disintermediation entirely; no chat means no leakage path |
| Admin creation | First via DB; rest via API | Admin signup flow with secret code | Eliminates privilege-escalation surface; matches industry standard (Stripe, Razorpay, Linear) |
| OTP provider | Firebase Phone Auth | MSG91, Twilio | Free up to 10K/month; switch to MSG91 in V2 if needed |
| File storage | Cloudflare R2 | AWS S3, Firebase Storage | ~10x cheaper outbound bandwidth; same S3-compatible API |
| Backend hosting | Render | AWS EC2, Heroku | Free tier, zero ops, GitHub auto-deploy |
| DB hosting | Supabase Postgres | AWS RDS, self-hosted | Free 500MB tier, managed backups, easy connection from anywhere |
| Push | Firebase Cloud Messaging | OneSignal, custom | Free unlimited; integrates well with Firebase Auth |
| Profile API path | `/users/profile` | `/users/me` | Semantically clearer for a public API |

---

## 15. Cost & Hosting

### One-time costs

| Item | Cost | Required for V1? |
|---|---|---|
| Google Play Console developer account | ₹2,000 (one-time) | Yes — required to publish on Play Store |
| Apple Developer Program | ₹8,300/year | No — defer to V2 |

### Monthly running cost: ₹0

All chosen services have free tiers that comfortably cover V1 testing volume:

| Service | What it does | Free tier limit | Approx V1 usage |
|---|---|---|---|
| Render | Hosts Spring Boot backend | 512MB RAM, sleeps after 15 min idle | Within limits |
| Supabase Postgres | Hosts database | 500 MB storage, 2 GB bandwidth/month | Within limits at 500 users |
| Cloudflare R2 | Hosts gym photos and logos | 10 GB storage, 1M reads/month | Way within limits |
| Firebase Phone Auth | Sends OTP | 10K verifications/month | ~2-3K expected |
| Firebase Cloud Messaging | Push notifications | Unlimited | Unlimited |
| Google Maps | Map display, geocoding | $200/month free credit | <$10 in V1 |
| GitHub | Source control | Free for personal repos | Free |
| GitHub Actions | CI/CD | 2000 minutes/month | <500 min expected |

### When you'd start paying

- Render: $7/month when you want to eliminate cold starts (recommended once real users start)
- Supabase: $25/month when DB exceeds 500 MB (probably 10K+ users)
- Cloudflare R2: $0.015/GB stored beyond 10 GB (negligible until 100+ active gyms)
- Firebase Phone Auth: $0.06/verification beyond 10K/month
- Apple Developer: ₹8,300/year when launching on iOS

**Principle:** the architectural choices stay the same forever — Flutter, Spring Boot, Postgres, R2. What changes when you grow is the tier of each service, which is a billing change, not a code rewrite.

---

## 16. Deployment Architecture

### Local development setup

While building, everything runs on your laptop:

- Spring Boot backend → `http://localhost:8080`
- PostgreSQL → installed locally or via Docker, accessed at `localhost:5432`
- Flutter app → runs on Android emulator (Android Studio) or iOS simulator (Xcode), or physical device via USB
- TablePlus → connects to local Postgres for data inspection
- Postman → tests backend endpoints

### Cloud production setup

When deployed, each piece moves to a managed service:

- Spring Boot backend → **Render** (auto-deploys from GitHub)
- PostgreSQL database → **Supabase** (managed, backed up, accessible from anywhere)
- Gym photos / logos → **Cloudflare R2** bucket
- OTP / authentication → **Firebase Phone Auth**
- Push notifications → **Firebase Cloud Messaging**
- Mobile app → **Google Play Store** (Android), App Store (iOS — V2)

### Architecture diagram

```
┌─────────────────────┐
│   User's Phone      │
│   (Flutter App)     │
│                     │
│   - Member          │
│   - Provider        │
│   - Admin           │
└──────────┬──────────┘
           │
           │  HTTPS
           │  (REST API calls + JWT auth)
           │
           ├──────────────────────────┐
           │                          │
           │                          │  Direct OTP
           │                          │  verification
           ▼                          ▼
┌─────────────────────┐     ┌─────────────────────┐
│   Render            │     │   Firebase          │
│   (Spring Boot      │◄────│   - Phone Auth      │
│    backend)         │     │   - Cloud Messaging │
│                     │     │     (push)          │
│   - REST API        │     └─────────────────────┘
│   - JWT issuance    │              ▲
│   - Business logic  │              │
└──┬───────────┬──────┘              │
   │           │                     │
   │           │ Sends push          │
   │           └─────────────────────┘
   │
   │  SQL queries          File uploads/reads
   │  (JDBC)               (S3-compatible API)
   ▼                       ▼
┌──────────────┐     ┌──────────────┐
│  Supabase    │     │  Cloudflare  │
│  (Postgres)  │     │  R2          │
│              │     │  (gym photos,│
│  - users     │     │   logos)     │
│  - gyms      │     └──────────────┘
│  - plans     │
│  - subs      │
│  - checkins  │
└──────────────┘

         │ Maps & geocoding (direct from app)
         ▼
┌──────────────┐
│  Google Maps │
│  SDK         │
└──────────────┘
```

### Example: how a single user request flows through the system

A customer browses nearby gyms:

1. User opens FitMarket app on phone (downloaded from Play Store)
2. App reads device location, calls `GET https://fitmarket-backend.onrender.com/api/v1/gyms/nearby?lat=12.97&lng=77.59&radius_km=5` with JWT in headers
3. Render receives the request, runs Spring Boot code
4. Spring Boot validates the JWT, then queries Supabase Postgres: `SELECT * FROM gyms WHERE status='active' ORDER BY distance(lat,lng) LIMIT 20`
5. Postgres returns gym rows; Spring Boot serializes them as JSON, sends back to phone
6. Phone gets the response, fetches each gym's photos directly from Cloudflare R2 URLs (which are stored as strings in the gym rows)
7. Map view renders using Google Maps SDK on the phone
8. User sees gym list + map

Every piece does its own job. None overlap.

### Deployment flow

1. **Write code on laptop.** Spring Boot backend + Flutter app + local Postgres. Everything works locally
2. **Push backend code to GitHub.** Create repo `fitmarket-backend`, push code. Same for `fitmarket-mobile` (separate repo)
3. **Set up Supabase.** Sign up at supabase.com, create project, get connection string. Connect TablePlus to verify access
4. **Set up Cloudflare R2.** Create bucket `fitmarket-uploads`, get API credentials
5. **Set up Firebase project.** Enable Phone Authentication and Cloud Messaging. Get Android and iOS config files for the Flutter app + a service account JSON for the Spring backend
6. **Connect Render to GitHub backend repo.** Render reads code, builds Spring Boot, runs it. Pass environment variables (Supabase URL, R2 keys, Firebase service account, JWT secret) via the Render dashboard
7. **Update Flutter app's API base URL** from `http://localhost:8080` to the Render URL (e.g., `https://fitmarket-backend.onrender.com`)
8. **Build Android app.** Run `flutter build appbundle` → produces `app-release.aab`. Sign in to Play Console (₹2,000 one-time), upload, fill listing, submit for review (1–7 days)
9. **(Later) Build iOS app** when budget allows. `flutter build ipa` → upload via Xcode to App Store Connect

### Environment variables (configured in Render dashboard)

```
DATABASE_URL                     # Supabase Postgres connection string
JWT_SECRET                       # Random 64-char string for signing JWTs
R2_ACCESS_KEY_ID                 # Cloudflare R2 access key
R2_SECRET_ACCESS_KEY             # Cloudflare R2 secret
R2_BUCKET_NAME                   # fitmarket-uploads
R2_ACCOUNT_ID                    # Cloudflare account ID
FIREBASE_SERVICE_ACCOUNT_JSON    # Firebase admin SDK credentials (for verifying OTP tokens + sending FCM)
GOOGLE_MAPS_API_KEY              # If used server-side; otherwise lives in Flutter app
SPRING_PROFILES_ACTIVE           # 'production'
```

### CI/CD with GitHub Actions

- On every push to `main` branch of `fitmarket-backend`:
  - Run tests
  - Build Spring Boot Docker image
  - Render auto-detects the push and deploys the new build
- On every push to `main` branch of `fitmarket-mobile`:
  - Run Flutter tests
  - (Optional) Auto-build APK and upload to Play Store internal testing track

### Updating the app post-launch

- **Backend changes:** push new code to GitHub → Render rebuilds and redeploys automatically. Users feel nothing
- **Mobile app changes:** rebuild `.aab` / `.ipa`, upload to Play Store / App Store, users get notified to update

---

## 17. Build Sprint Plan

Six 2-week sprints. Total V1 build time: ~12 weeks. Each sprint ends with a working, testable feature set.

### Sprint 1 (Weeks 1–2) — Foundation
- Spring Boot project scaffold with Java 21, Maven/Gradle, Spring Web, Spring Security, Spring Data JPA, Flyway
- Postgres set up locally + Supabase project created
- Firebase project created, Phone Auth enabled
- Spring backend `/auth/firebase-verify` endpoint that validates Firebase ID tokens and issues FitMarket JWT
- Flutter app scaffold with bottom navigation, routing
- Flutter login screen with Firebase Phone Auth + role selection (Member / Provider)
- User profile create/fetch (`/users/profile`)
- Render deployment working with one test endpoint live

### Sprint 2 (Weeks 3–4) — Provider Onboarding
- Provider onboarding form (all fields from section 5.2)
- Cloudflare R2 set up + `POST /uploads/presign` endpoint
- Flutter image picker + upload to R2 via pre-signed URLs (logo + up to 5 photos)
- `POST /gyms` endpoint with all profile fields
- `PATCH /gyms/{id}` for editing
- Admin home screen (mobile) with pending gym queue
- `POST /admin/gyms/{id}/approve` and `/reject` endpoints + admin mobile UI
- First admin manually created in DB to test the full flow

### Sprint 3 (Weeks 5–6) — Discovery
- `GET /gyms/nearby` endpoint with Haversine distance query in Postgres
- `GET /gyms` endpoint with filters (service, city, pagination)
- `GET /gyms/{id}` endpoint with full gym detail
- Flutter customer home screen — nearby section + browse all section
- Filter UI (service type, distance, price)
- Search bar (by name)
- Gym detail page rendering all profile fields

### Sprint 4 (Weeks 7–8) — Subscriptions
- Plans CRUD (`POST/GET/PATCH/DELETE /plans`) for providers
- Plan management UI in provider section
- `POST /subscriptions` for customers requesting to join
- `GET /subscriptions/me` for customer's My Subscriptions screen
- Provider Subscriber Management UI (Pending + Active tabs)
- `POST /subscriptions/{id}/approve` and `/reject` endpoints + provider UI
- Push notifications via FCM when subscription is approved/rejected

### Sprint 5 (Weeks 9–10) — Check-in System
- `qr_signing_secret` generation on first login (server) + secure delivery to client (client stores in secure storage)
- Client-side rotating QR generation every 60 seconds using `qr_flutter`
- Customer home screen showing the global QR prominently
- `GET /qr/token` endpoint for refreshing the signing key if needed
- Provider scanner UI using `mobile_scanner`
- `POST /checkins/scan` endpoint — verifies token signature, freshness, subscription validity at the scanning gym
- Check-in success/failure UI on provider side (green / red state)
- `checkins` table writes on successful scan

### Sprint 6 (Weeks 11–12) — Admin Polish + Launch Prep
- Admin search (`GET /admin/search`)
- Admin suspend/reactivate for gyms and members
- `POST /admin/users/{id}/promote` endpoint + admin UI
- Admin stats card
- Manual gym onboarding by admin (admin fills form on behalf of provider)
- App icon, splash screen, branding pass
- Privacy policy + terms of service pages (required for Play Store)
- Play Store listing — screenshots, description, content rating
- Internal testing with 5–10 friends as members + 2–3 friendly gym owners
- Bug fixes from internal testing
- Submit to Play Store

### Post-launch (Beta)
- Onboard 10–20 gyms in one Bengaluru neighborhood
- Open invitations to ~100 customers in that neighborhood
- Watch metrics from section 11
- Collect feedback, iterate
- Once validated, plan V2 (payments)

---

## Appendix A — Why no payment gateway in V1

Adding Razorpay in V1 doubles the build time and adds:
- KYC for every gym (mandatory under RBI rules)
- Settlement logic
- Refund flows
- Tax compliance (GST)
- Dispute handling

None of this is needed to validate the core hypothesis: "Will small gym owners list themselves, and will customers subscribe?" If the answer is yes in V1 with offline payments, V2 with online payments is a clear, justified investment.

---

## Appendix B — How payment will work in V2 (preview)

When V2 ships with Razorpay:
- Customer pays in-app via Razorpay (UPI, card, netbanking)
- Money lands in your Razorpay account
- Razorpay Route splits the amount: `gym_share = amount - platform_commission`
- Settlement to gym's bank account happens automatically on a schedule (e.g., weekly)
- Bank details and KYC become mandatory on gym onboarding
- This is exactly how Swiggy, Zomato, and Uber operate

This is for V2 planning only — not V1 scope.

---

## Appendix C — Anti-leakage strategy (why no chat, ever)

The single biggest revenue risk for any marketplace is disintermediation: members and providers connecting directly, then transacting outside the platform to avoid commission. Once a member has a provider's WhatsApp number, you've lost that customer's lifetime value.

Defenses, baked in from V1:

1. **No free-text chat. Anywhere. Ever.** Even in V3, if any communication is added, it's masked calls only (Exotel-style virtual numbers).
2. **Phone number visibility is gated.** Members never see provider phone numbers in browsing. Providers only see member phone numbers for pending or active subscriptions, which already implies a transaction is happening.
3. **All info on the profile.** The gym profile shows everything a member could need — equipment, parking, hours, dress code, women-only hours, trainers, plans, photos. There's no reason to "ask a question" because the answer is already there.
4. **ToS clause (V2 onwards).** Off-platform transactions = account suspension. Enforceable when transaction records exist.

This is the same playbook as Airbnb, Urban Company, and Upwork. They learned it the hard way; FitMarket gets to start with it.
