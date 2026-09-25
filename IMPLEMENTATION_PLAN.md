# NZ Car Rental Marketplace: Implementation & Design Plan

Working name: **DriveShare NZ** (placeholder until the client confirms the brand; it cannot be the launch name, because "DriveShare" is already used by a US peer-to-peer car rental service, section 16 item 1)
Source: [project_requirements.md](project_requirements.md), the client's *NZ Peer-to-Peer Car Rental Marketplace – Website Specification (Sept 2026)*. It is the source of truth for requirements and is referred to as "spec §N" below.
Client-facing timeline: [MILESTONES.md](MILESTONES.md) (30-day delivery). Every milestone item and every section of the spec (§1–§32) is mapped to this plan in [section 10](#10-requirements-coverage), and every individual requirement (each bullet and step in the spec) is traced in [Appendix A](#appendix-a-requirement-traceability). Spec items scheduled after launch have a roadmap in [section 10.5](#105-post-launch-roadmap). The spec's questions for the development team (§31) are answered in [section 18](#18-answers-to-the-clients-questions-spec-31).
Stack: **React + Vite** (TypeScript) for the website, **MongoDB** for all data, and a small **Node.js + Express** API between them (see [section 1](#1-technology-stack)). No Redis, no migration tool, no monorepo tooling. Hosted on **AWS** in Sydney as one Docker container service (see [section 13](#13-deployment-architecture-aws)).
Design goal: a **professional, luxurious UI** that feels **fast**, with smooth, polished animations and completely original branding (see [section 12](#12-design-implementation))

---

## 1. Technology Stack

### 1.1 Core stack

| Part | Choice | Role |
|---|---|---|
| Website | **React 19 + Vite + TypeScript**, React Router | Single-page app. Fast dev server with hot reload and fast production builds. SEO is handled as described in 1.4. |
| Database | **MongoDB 8 (MongoDB Atlas)** + **Mongoose** | All application data: listings, bookings, payments, messages. Also holds login sessions, the background job queue, rate-limit counters and realtime sync, so **no Redis is needed** (section 4). Built-in geospatial queries for location search, and multi-document transactions for bookings and payments. |
| API | **Node.js 22 + Express + TypeScript** | The browser cannot connect to MongoDB safely, so a small API keeps the database credentials, Stripe secret keys and business rules on the server. In production the same Express server also serves the React build, so the whole site is **one deployable service**. Future iOS and Android apps reuse the same REST API (spec §26, section 11.1). |

### 1.2 Libraries and services

| Layer | Choice | Why |
|---|---|---|
| Styling | **Tailwind CSS** + **shadcn/ui** (Radix primitives) | Accessible building blocks, fully restyled to the luxury brand so nothing looks like a default template |
| Animation | **Motion** (`motion/react`, formerly Framer Motion) + the browser's **View Transitions API** | Spring physics, gestures, layout and shared-element animations. Native, low-cost page transitions. |
| Client data | TanStack Query, React Hook Form + Zod | Caching, optimistic UI, validation shared with the API |
| Background jobs | **MongoDB `jobs` collection** + a job runner inside the API process | Emails, SMS, reminders, booking expiry, payouts, extra charges (section 4) |
| Realtime | **Socket.IO** + `@socket.io/mongo-adapter` | Live messaging and in-app notifications, in sync across several API instances through MongoDB |
| Payments | **Stripe** (Payment Intents + **Stripe Connect Express** + **Radar**) | Supports NZD, Apple Pay and Google Pay. Connect handles Host payouts to NZ bank accounts. Radar screens payments for fraud. |
| Identity | **Stripe Identity** (ID document + selfie) for Guests and Hosts, with manual review by support staff as fallback | Identity, driver licence and Host identity checks (spec §22) |
| Email | **Resend** + **React Email** templates | Transactional email, with branded templates written as React components |
| SMS | Twilio | Phone verification codes, pickup/return reminders, and time-sensitive alerts such as new booking requests |
| File storage | Cloudinary | Image resizing, WebP/AVIF, CDN (spec §25). Private documents use signed URLs. |
| Maps / places | Google Places Autocomplete (restricted to `country: nz`) and Place Details, with one session token per search so each search is billed once, combined with our own destinations and airports. **Maps Static API** for the approximate-area map on listings (a shaded circle, never a pin). Directions open in the phone's own maps app through a standard maps link. | City, suburb, destination and airport search, NZ address formatting (spec §21), a listing map that does not reveal the address (spec §22) |
| Help and support | Support tickets and help articles stored in MongoDB | Help centre, contact form and the support staff inbox, with no third-party helpdesk needed at launch |
| Monitoring | Sentry + structured logs (Pino) sent to AWS CloudWatch, with CloudWatch alarms | Error monitoring, logging and alerts (spec §25) |
| Performance | Lighthouse CI + bundle size check in CI, real-user Core Web Vitals in Sentry | The build fails if a page breaks the speed budget (section 12.5) |
| Analytics | GA4 + Google Search Console | Analytics, conversion tracking and search monitoring (spec §24) |
| Documents and helpers | `@react-pdf/renderer` (receipt and earnings statement PDFs), `otplib` (staff two-factor codes), `@asteasolutions/zod-to-openapi` (OpenAPI docs from the shared Zod schemas), `libphonenumber-js` (phone numbers) | Receipts (spec §17), secure staff sign-in (spec §25), API docs for the future apps (spec §26) |
| Testing | Vitest, Supertest, mongodb-memory-server, Playwright, **k6** | Unit, API (against an in-memory MongoDB replica set) and end-to-end tests, plus a load test on staging before launch (section 9, Day 26) |
| CI/CD | GitHub Actions + Dependabot | Lint, typecheck, test and build on every push; builds the Docker image and deploys it to AWS (section 13.3). Dependabot opens pull requests for dependency security updates. |
| Hosting | **AWS, Sydney** (`ap-southeast-2`): ECS on Fargate, Application Load Balancer, CloudFront + AWS WAF, S3, Route 53, ECR, Secrets Manager, CloudWatch. Defined as code with **AWS CDK** (TypeScript). | One Docker image runs the whole app on managed containers, in the same region as the Atlas cluster (section 13) |

### 1.3 Deliberately left out
- **No Redis.** MongoDB covers everything Redis would have done: background jobs, rate limits, Socket.IO sync between instances, and sessions (section 4).
- **No migration tool.** Mongoose schemas define the structure, indexes are synced from the schemas on deploy, and schema changes are additive (section 3).
- **No monorepo tooling** (Turborepo, pnpm, Nx). Plain **npm workspaces**, built into npm, share code between the client and the server (section 2).
- **No separate worker or web server.** Background jobs run inside the API process, and Express serves the website. There is one container image and one ECS service to deploy (section 13).

### 1.4 SEO with a plain React app
A Vite React app renders in the browser. Vehicle, city and destination pages must still be indexable and show correct link previews (spec §24), so the Express server does the following:

1. **Server-injected metadata for every public route.** Before sending `index.html`, Express writes a unique page title, meta description, canonical URL, Open Graph tags (social sharing previews) and JSON-LD structured data into it. Static pages use a fixed table in `server/src/seo/pages.ts`: Home, Browse Cars, How It Works, Become a Host, Safety, Insurance / Protection, FAQs, Help, About Us, Contact Us, and the legal pages (Terms & Conditions, Privacy Policy, Cancellation Policy, Host Agreement, Guest Agreement). Vehicle pages (`/cars/:slug`) use the vehicle record. City and destination landing pages (`/rental/:city`) use the `destinations` collection: the 5 launch cities are seeded, and admins can add more destinations without code changes.
2. **Sitemap and robots.** `sitemap.xml` is generated from the static pages, active vehicles and destinations. `robots.txt` and `noindex` tags keep private areas (account, host, admin, checkout, login, sign-up) out of search results.
3. **Search Results and empty pages.** Search Results URLs (`/search?…`, one for every mix of place, dates and filters) are `noindex, follow`, so search engines index the clean city, destination and vehicle pages instead of endless filter combinations. A city or destination page with no cars yet still loads normally, with a "No cars here yet" message and a Become a Host prompt, never an empty or error page.

Google renders JavaScript, so the page body (specs, reviews) is indexed after rendering. Indexing is monitored in Search Console from launch. If coverage is poor, a prerendering service (e.g. Prerender.io) can be added for crawlers without changing the app.

---

## 2. Project Structure & Tooling

The whole product lives in **one Git repository** with three folders: `client` (the React app), `server` (the Express API) and `shared` (code both of them use). They are **npm workspaces**, so one `npm install` sets up everything, there is one lockfile, and the client and server import `shared` like a normal package. Because the Zod schemas, types and pricing live in `shared`, the website and the API cannot drift apart: if a shared schema change breaks either side, the typecheck fails in the same pull request.

### 2.1 Tooling

| Concern | Choice | Why |
|---|---|---|
| Packages | **npm workspaces** (`client`, `server`, `shared`) | Built into npm, no extra tools. One lockfile, and `shared` is linked into both apps. |
| TypeScript | Strict mode. One `tsconfig.base.json` at the root, extended by each folder | The same compiler rules everywhere |
| Website build | **Vite** | Dev server with hot reload; proxies `/api` and `/socket.io` to Express so the browser talks to one origin and auth cookies work without CORS setup |
| Server dev and build | `tsx watch` in development, **tsup** for production | tsup bundles the server, its `seed` and `sync-indexes` scripts, and the `shared` code they use into `server/dist` (`noExternal: ['@driveshare/shared']`), so production runs plain `node` without `tsx` |
| Production image | Multi-stage **Dockerfile** at the root | The same image runs on staging, on production and on a developer's machine (section 13.2) |
| Lint and format | One ESLint flat config + Prettier at the root | Includes a rule that stops `client` from importing anything in `server` |
| Node version | Node 22 LTS, pinned in `.nvmrc` and `engines` | The same runtime locally, in CI and in production |
| Local database | MongoDB 8 in `docker-compose.yml`, run as a **single-node replica set** | Transactions need a replica set. Without Docker, `npm run db:dev` starts one in memory with mongodb-memory-server. |

### 2.2 Layout

```
driveshare/
├── client/                        # @driveshare/client: React + Vite single-page app
│   ├── src/
│   │   ├── routes/
│   │   │   ├── public/            # home, cars (Browse Cars), search, cars/:slug, how-it-works,
│   │   │   │                      # become-a-host, safety, insurance, faq, help, about, contact
│   │   │   ├── legal/             # terms, privacy, cancellation-policy, host-agreement, guest-agreement
│   │   │   ├── rental/            # SEO landing pages (rental/:city)
│   │   │   ├── auth/              # login, signup, verify-email, reset-password
│   │   │   ├── checkout/          # booking flow
│   │   │   ├── account/           # Guest dashboard
│   │   │   ├── host/              # Host dashboard, Host application and vehicle onboarding
│   │   │   └── admin/             # Admin and support staff portal
│   │   ├── components/ui/         # design system: restyled shadcn/ui components + motion building blocks
│   │   ├── features/              # booking/, vehicle/, search/, dashboard/, messaging/, inspection/,
│   │   │                          # support/: marketplace components and hooks
│   │   ├── lib/                   # API client, auth, TanStack Query client, Socket.IO client
│   │   ├── router.tsx             # route tree, lazy-loaded routes, role guards
│   │   └── main.tsx
│   ├── index.html
│   └── vite.config.ts             # dev proxy for /api and /socket.io
├── server/                        # @driveshare/server: Express REST API + Socket.IO + job runner
│   ├── src/
│   │   ├── modules/               # auth, users, hosts, vehicles, search, availability, bookings, payments,
│   │   │                          # payouts, messages, reviews, inspections, incidents, verification,
│   │   │                          # notifications, support, help, moderation, risk, admin, cms.
│   │   │                          # Each has model.ts, service.ts, routes.ts
│   │   ├── jobs/                  # MongoDB job queue: queue.ts, runner.ts, one handler per job type
│   │   ├── emails/                # React Email templates
│   │   ├── integrations/          # Stripe, mailer (Resend / console), Twilio, Cloudinary, Google Places, logger
│   │   ├── middleware/            # auth, roles and permissions, rate limits, error handler, audit log
│   │   ├── realtime/              # Socket.IO server + MongoDB adapter
│   │   ├── seo/                   # meta tag injection, sitemap.xml, robots.txt
│   │   ├── db.ts                  # Mongoose connection + transaction helper
│   │   ├── env.ts                 # environment variables, validated with Zod at startup
│   │   └── server.ts              # /api/v1, Socket.IO, /healthz, graceful shutdown, and the client build in production
│   ├── scripts/                   # seed.ts, sync-indexes.ts
│   └── test/
├── shared/                        # @driveshare/shared: Zod schemas (API contracts), types, enums, pricing,
│                                  # cancellation policy, design tokens, NZD, date and NZ address formatters.
│                                  # Runs in the browser and in Node.
├── e2e/                           # Playwright tests against the client + API
├── infra/                         # AWS CDK app (TypeScript): VPC, load balancer, ECS, CloudFront, WAF, S3, ECR,
│                                  # secrets, alarms. Own package.json, not an npm workspace, so app installs and
│                                  # image builds never download the CDK.
├── .github/workflows/             # ci.yml (checks on every push), deploy.yml (staging and production, section 13.3)
├── Dockerfile                     # production image (section 13.2)
├── .dockerignore
├── docker-compose.yml             # local MongoDB (single-node replica set)
├── package.json                   # npm workspaces + root scripts
├── tsconfig.base.json
└── .nvmrc
```

### 2.3 How the parts connect

```
client  → shared
server  → shared
shared  → zod only (no Node built-ins, no DOM, no Mongoose)
```

- **The client never imports from `server`.** An ESLint rule blocks it, and `client` does not list `server` as a dependency, so Mongoose, Stripe secret keys and other server code cannot reach the browser bundle.
- **`shared` runs anywhere.** Only Zod, types and pure functions, so the future native apps (spec §26) can use it unchanged.
- **One request path in the API:** `routes.ts` (input checked with the Zod schemas from `shared`) → `service.ts` (business rules, unit tested) → `model.ts` (Mongoose). Background job handlers call the same services, so a request-to-book that expires in a job follows exactly the same rules as one the Host declines in the dashboard.
- **`shared` has no build step.** Its `exports` point straight at the TypeScript source, and Vite (client), `tsx` (server in development) and tsup (server build) compile it. A change in `shared` shows up immediately in `npm run dev`.
- **Design tokens are defined once** in `shared/src/tokens.ts`. They generate the CSS variables and Tailwind theme for the client (12.2) and are imported by the email templates, so emails match the site exactly.

```jsonc
// package.json (root)
{
  "name": "driveshare",
  "private": true,
  "workspaces": ["client", "server", "shared"],
  "scripts": {
    "dev": "concurrently -n client,server \"npm run dev -w client\" \"npm run dev -w server\"",
    "build": "npm run build -w client && npm run build -w server",
    "start": "node server/dist/server.js",
    "lint": "eslint .",
    "typecheck": "npm run typecheck --workspaces --if-present",
    "test": "npm test --workspaces --if-present",
    "e2e": "playwright test",
    "seed": "npm run seed -w server",
    "db:indexes": "npm run db:indexes -w server"
  }
}

// shared/package.json
{
  "name": "@driveshare/shared",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./pricing": "./src/pricing.ts",
    "./tokens": "./src/tokens.ts"
  },
  "dependencies": { "zod": "^4.0.0" }
}
```

`client` and `server` list `"@driveshare/shared": "*"` in their dependencies, and npm links the local folder.

### 2.4 Daily workflow

```bash
npm install                  # client, server and shared, one lockfile
docker compose up -d         # MongoDB (single-node replica set); or: npm run db:dev (no Docker)
npm run seed                 # NZ cities, airports, destinations, 20 demo vehicles, test users, FAQs, help articles
npm run dev                  # client :5173 (Vite), server :4000 (API, Socket.IO, job runner)
```

| Command | What it does |
|---|---|
| `npm run dev -w client` | Starts only the React app |
| `npm test -w server` | Runs the server tests (API and services against an in-memory replica set) |
| `npm run lint && npm run typecheck && npm test` | All checks, the same as CI |
| `npm run e2e` | Playwright end-to-end tests |
| `npm run email:dev -w server` | Previews every email template in the browser |
| `npm run db:indexes` | Creates or updates the MongoDB indexes from the Mongoose schemas |
| `docker build -t driveshare .` | Builds the production image locally, the same one deployed to AWS |
| `npx cdk diff -c env=staging` (in `infra/`) | Shows what a CDK update would change in AWS, before `cdk deploy` applies it |
| `npm install stripe -w server` | Adds a dependency to one workspace |

### 2.5 Environment variables

- `server/.env.example` and `client/.env.example` are committed; `.env` files are ignored. At startup the server checks its variables against a Zod schema in `src/env.ts` and exits with a clear message if one is missing or invalid.
- Only `VITE_`-prefixed variables reach the browser bundle. Secrets are never given that prefix.
- On AWS, the staging and production values are kept in **AWS Secrets Manager** (one secret per environment) and passed to the container as environment variables when it starts, so `env.ts` checks them the same way. Nothing secret is built into the Docker image.

---

## 3. Data Model (MongoDB collections)

Each collection's Mongoose model lives in its module, for example `server/src/modules/vehicles/vehicle.model.ts`.

**Embed or reference:** data that is always read with its parent and stays small (photos, documents, delivery options, line items, refunds, inspection photos, incident events, ticket messages) is embedded. Data that grows without limit or is queried on its own (bookings, messages, availability blocks, reviews, notifications, audit logs, jobs) gets its own collection.

```
users              _id, email (unique, lowercase), phone (E.164), passwordHash, firstName, lastName, dob, avatarUrl,
                   roles[] (GUEST|HOST|ADMIN|SUPPORT), permissions[] (e.g. REFUNDS for authorised support staff),
                   emailVerifiedAt, phoneVerifiedAt, status (ACTIVE|SUSPENDED), suspendedReason,
                   stripeCustomerId, favouriteVehicleIds[], blockedUserIds[], notificationPrefs (incl. marketing
                   opt-in), lastSearch { place, lat, lng, startAt, endAt } (estimated totals in Saved cars),
                   riskFlags[] { code, detail, createdAt, clearedBy }, mfa { totpSecret (encrypted), enabledAt }
                   (required for admin and support staff), closedAt (account closed and anonymised), createdAt
                   ├─ identityVerification { status (NONE|PENDING|APPROVED|REJECTED), provider, providerRef,
                   │                         verifiedAt, reviewedBy }                    (Guests and Hosts)
                   ├─ driverLicence { number (encrypted), numberHash (keyed HMAC: finds the same licence on
                   │                  another account, see Key rules), version, country, class (NZ_FULL|
                   │                  NZ_RESTRICTED|NZ_LEARNER|OVERSEAS), englishProof? (IDP|APPROVED_TRANSLATION),
                   │                  issuedAt, expiry, status, reviewedBy }
                   ├─ agreements[] { type (TERMS|PRIVACY|GUEST|HOST), version, acceptedAt, ip }
                   └─ hostProfile { status (APPLIED|APPROVED|REJECTED|SUSPENDED), appliedAt, reviewedBy,
                                    reviewNotes, stripeAccountId, payoutsEnabled, bio, responseRate,
                                    tripCount, rating {avg, count}, feesOwedCents (Host cancellation fees
                                    not yet deducted from a payout), gstRegistered, gstNumber? (GST
                                    treatment of the Host's rental and commission invoices, section 5) }
sessions           userId, refreshTokenHash, userAgent, ip, expiresAt (TTL index: deleted on expiry)
vehicles           _id, hostId, slug (unique), regoPlate, vin?, chassisNo? (NZ imports often have a chassis
                   number instead of a 17-character VIN; one of the two is required), make, model, year, variant,
                   transmission, bodyType (HATCHBACK|SEDAN|WAGON|SUV|UTE|VAN|PEOPLE_MOVER|COUPE|CONVERTIBLE),
                   fuelType (PETROL|DIESEL|HYBRID|PHEV|EV), seats, doors, features[], wofExpiry, regoExpiry,
                   cofExpiry? (vehicles that need a Certificate of Fitness instead of a WOF),
                   rucValidToKm? (Road User Charges licence end reading: diesel, EV and PHEV vehicles),
                   powertrain { engineCc, cylinders, description, evRangeKm, batteryKwh },
                   fuelPolicy (SAME_LEVEL|FULL: return at the level collected, or full; EVs use battery %),
                   kmAllowancePerDay, unlimitedKm, petFriendly, childSeat,
                   pricing { dailyCents, weeklyDiscountPct, monthlyDiscountPct, extraKmCents },
                   rules { minDays, maxDays, minNoticeHours, bufferHours, instantBook,
                           cancellationTier (one of the tiers allowed in platformSettings, see Key rules) },
                   status (DRAFT|UNDER_REVIEW|CHANGES_REQUESTED|REJECTED|ACTIVE|INACTIVE|SUSPENDED),
                   reviewNotes, onboardingStep,
                   location { type: "Point", coordinates: [lng, lat] }, suburb, city, region,
                   rating { avg, count }, tripCount, bookingSeq (used to serialise bookings, see below)
                   ├─ photos[]               { type (FRONT|REAR|DRIVER|PASSENGER|INTERIOR|DASH|BOOT|TYRES|DAMAGE),
                   │                           url, order, qualityFlag (OK|LOW_RES|DARK|BLURRY|ADMIN_FLAGGED),
                   │                           status (PENDING|APPROVED|REJECTED) }
                   ├─ documents[]            { type (REGO|WOF|COF|RUC|INSURANCE|OWNER_CONSENT|OTHER), url, expiry,
                   │                           status (PENDING|VERIFIED|REJECTED), reviewedBy }
                   ├─ deliveryOptions[]      { _id, type (PICKUP|DELIVERY|AIRPORT|CUSTOM), label, address,
                   │                           airportCode, feeCents, radiusKm, instructions (e.g. where to meet
                   │                           at the airport; shown once the booking is confirmed) }
                   ├─ recurringRules[]       { daysOfWeek[], startTime, endTime }   (e.g. unavailable weekdays 8am–6pm)
                   └─ maintenanceReminders[] { title, dueAt?, dueOdometer?, notes, doneAt }
availabilityBlocks vehicleId, startAt, endAt, reason (BOOKED|HOLD|HOST_BLOCK|RECURRING|BUFFER|ADMIN), bookingId?,
                   expiresAt? (HOLD only: dates held during checkout or while a request waits for the Host, 8.2)
bookings           _id, ref (DS-XXXXXX, unique), vehicleId, guestId, hostId, startAt, endAt,
                   pickupOptionId, pickupAddress?, returnOptionId, returnAddress?,
                   protectionPlan { code, name, priceCents, excessCents, coverSummary, mandatory } (copy),
                   status (PAYMENT_PENDING|PENDING|CONFIRMED|ACTIVE|COMPLETED|CANCELLED|DECLINED|EXPIRED)
                   (lifecycle in section 8.2), requestExpiresAt?,
                   vehicleSnapshot { title, photoUrl, regoPlate },
                   terms { fuelPolicy, kmAllowancePerDay, unlimitedKm, extraKmCents } (copied when booked),
                   price { subtotalCents, deliveryCents, serviceFeeCents, protectionCents, gstCents, totalCents,
                           hostPayoutCents, platformFeeCents },
                   cancellationPolicy, cancelledBy, cancelledAt, cancellationReason (incl. GUEST_NO_SHOW|
                   HOST_NO_SHOW|PLATFORM), cancellationFeeCents
                   ├─ lineItems[]     { code, label, amountCents, mandatory }
                   ├─ statusHistory[] { status, at, by, reason }   (every status change, incl. admin edits)
                   └─ extraCharges[]  { type (EXTRA_KM|FUEL|CLEANING|LATE_RETURN|DAMAGE|TOLL|FINE|OTHER),
                                        description, amountCents, incidentId?, addedBy, paymentId, status }
payments           bookingId, type (BOOKING|EXTRA_CHARGE), stripePaymentIntentId (unique), amountCents,
                   status (PENDING|AUTHORISED|SUCCEEDED|FAILED|REFUNDED|PARTIALLY_REFUNDED), method, failureReason
                   ├─ refunds[] { amountCents, reason, issuedBy, fundedBy (PLATFORM|HOST), stripeRefundId,
                   │              createdAt }
                   └─ dispute?  { stripeDisputeId, reason, status, dueBy }       (card chargebacks)
payouts            hostId, bookingId, amountCents, stripeTransferId, status (SCHEDULED|HELD|PAID|FAILED),
                   holdReason? (INCIDENT|DISPUTE|PAYOUT_SETUP|TRIP_NOT_STARTED|SUSPENDED), scheduledFor, paidAt
                   └─ deductions[] { type (HOST_CANCELLATION_FEE|HOST_FUNDED_REFUND|OTHER), bookingId, amountCents }
threads            bookingId (unique), participantIds[], lastMessageAt
messages           threadId, senderId, body, attachments[], systemGenerated, readAt, createdAt
reviews            bookingId, authorId, subjectId, direction (GUEST_TO_HOST|HOST_TO_GUEST), overall,
                   communication, pickupReturn, cleanliness (Guest → Host: vehicle cleanliness and condition),
                   care (Host → Guest: vehicle care and Guest behaviour), body, status (PUBLISHED|PENDING|HIDDEN),
                   revealAt (when both sides have reviewed or the review window closes), moderation { reason, by, at }
conditionReports   bookingId, stage (CHECK_IN|CHECK_OUT), odometer, fuelOrBatteryPct, notes,
                   damagePins[] { x, y, note, isNew }, confirmedByGuestAt, confirmedByHostAt
                   └─ photos[] { angle (FRONT|REAR|DRIVER_SIDE|PASSENGER_SIDE|WHEELS|WINDSCREEN|INTERIOR|
                                 DASHBOARD|DAMAGE), url, takenAt, exifTakenAt, lat, lng }
incidents          caseRef (unique, e.g. IN-XXXXXX), bookingId, reporterId, type (DAMAGE|ACCIDENT|THEFT|BREAKDOWN|CLEANING|
                   FUEL|LATE_RETURN|NO_SHOW|TOLL|FINE|DISPUTE|OTHER), description,
                   status (OPEN|INVESTIGATING|AWAITING_RESPONSE|RESOLVED|CLOSED), assignedTo
                   └─ events[] { actorId, action, note, attachments[], visibility (BOTH|GUEST|HOST|INTERNAL),
                                 createdAt }   (full audit trail, append only)
reports            reporterId, targetType (USER|MESSAGE|REVIEW|VEHICLE), targetId, reason, note,
                   status (OPEN|ACTIONED|DISMISSED), handledBy
supportTickets     ref (unique), userId? (or name + email from the contact form), bookingId?, subject,
                   category (incl. PRIVACY for requests to access or correct data, or to close an account),
                   status (OPEN|PENDING|RESOLVED), assignedTo
                   └─ messages[] { authorId, body, attachments[], internal, createdAt }
helpArticles       slug, title, body (Markdown), category, audience (GUEST|HOST|ALL), published, order
notifications      userId, type, channel (EMAIL|SMS|IN_APP; PUSH is added later, 10.5 R1), payload,
                   status (QUEUED|SENT|DELIVERED|FAILED), providerRef, error?, sentAt, readAt
jobs               type, payload, runAt, status (QUEUED|RUNNING|DONE|FAILED|CANCELLED), attempts, maxAttempts,
                   uniqueKey?, refId? (e.g. a bookingId), lockedAt, lockedBy, lastError, finishedAt   (section 4)
auditLogs          actorId, action, entity, entityId, before, after, ip, createdAt
stripeEvents       eventId (unique), type, processedAt   (webhook idempotency)
faqs               question, answer, category, audience (GUEST|HOST|ALL), showOnHome, order
cmsBlocks          key (unique), content, version   (homepage hero and sections, featured vehicles, footer links
                   incl. social links, legal pages in Markdown)
destinations       slug (unique), city, region, intro, heroImage, location, airports[], featured, order
                   (landing pages; featured destinations are the homepage tiles)
platformSettings   one document: fees, GST rate, cancellation tiers (and which ones Hosts may choose), Host
                   cancellation fee, Guest cancellation fee share, protection plans, driver eligibility rules,
                   verification requirements, required vehicle documents and photo angles, whether a VIN or chassis
                   number is required, review rules (window, reveal), late-return grace period, damage-report window
                   after check-out, homepage review threshold, data retention periods, risk-flag thresholds, search
                   limits (maximum trip length, radius range), message thread read-only period, SMS quiet hours
```

Two libraries also keep small collections of their own in the same database: the rate limiter's counters (expired by a TTL index) and the Socket.IO adapter's event collection (section 4).

**Key indexes**
- `vehicles`: `{ location: "2dsphere", status: 1 }`, `{ slug: 1 }` unique, `{ hostId: 1 }`, `{ make: 1, model: 1 }`
- `availabilityBlocks`: `{ vehicleId: 1, startAt: 1, endAt: 1 }`
- `bookings`: `{ ref: 1 }` unique, `{ guestId: 1, startAt: -1 }`, `{ hostId: 1, status: 1, startAt: -1 }`
- `messages`: `{ threadId: 1, createdAt: 1 }`
- `sessions`: `{ expiresAt: 1 }` with `expireAfterSeconds: 0`
- `jobs`: `{ status: 1, runAt: 1 }`, `{ uniqueKey: 1 }` unique (only where set), `{ refId: 1 }`, `{ finishedAt: 1 }` with a 30-day TTL
- `notifications`: `{ userId: 1, readAt: 1, createdAt: -1 }`
- `supportTickets`: `{ ref: 1 }` unique, `{ status: 1, updatedAt: -1 }`; `reports`: `{ status: 1, createdAt: -1 }`
- `users`: `{ email: 1 }` unique, `{ "driverLicence.numberHash": 1 }`, `{ "hostProfile.status": 1 }` (Host application queue)
- `reviews`: `{ bookingId: 1, direction: 1 }` unique (one review each way per trip), `{ subjectId: 1, status: 1, createdAt: -1 }`
- `payments`: `{ stripePaymentIntentId: 1 }` unique, `{ bookingId: 1 }`, `{ status: 1, createdAt: -1 }`; `payouts`: `{ hostId: 1, status: 1, scheduledFor: -1 }`, `{ bookingId: 1 }`
- `conditionReports`: `{ bookingId: 1, stage: 1 }` unique; `incidents`: `{ caseRef: 1 }` unique, `{ bookingId: 1 }`, `{ status: 1, updatedAt: -1 }`
- `threads`: `{ bookingId: 1 }` unique, `{ participantIds: 1, lastMessageAt: -1 }`; `stripeEvents`: `{ eventId: 1 }` unique
- `auditLogs`: `{ entity: 1, entityId: 1, createdAt: -1 }`, `{ actorId: 1, createdAt: -1 }`; `destinations` and `helpArticles`: `{ slug: 1 }` unique

**Indexes and schema changes, without migrations**
- Indexes are declared in the Mongoose schemas. `autoIndex` is on in development. In staging and production it is off, and the index sync (`syncIndexes()` for every model) runs in each deploy as a one-off ECS task, before the new version starts (section 13.3).
- Schema changes are **additive**: a new field is optional or has a default in the Mongoose schema, so existing documents keep working without being rewritten.
- Fields are not renamed or given a new meaning after launch. A replaced field is left unused and removed later.
- The seed script uses upserts, so it can be re-run on development and staging at any time.

**Key rules**
- **Money** is stored as integer cents, in NZD only. A Mongoose validator rejects values that are not integers.
- **Times** are stored as UTC `Date` values and displayed in `Pacific/Auckland` as `DD/MM/YYYY` and `h:mm a`.
- **Addresses** use one structured NZ format everywhere (Host pickup points, delivery and airport options, booking addresses): unit, street number and name, suburb, town or city, region, postcode and coordinates, filled in from Google Places or typed by hand. The shared address formatter displays them in NZ order, e.g. "12 Queen Street, Auckland Central, Auckland 1010" (spec §21, §23).
- **Rating and trip totals** on vehicles and Host profiles are updated when a review is published or a trip completes, so search results need no extra lookups.
- **Location search** uses MongoDB's built-in `2dsphere` index. The search service first finds the vehicles with a block overlapping the requested dates, then runs one `$geoNear` aggregation with the radius, the filters and `_id: { $nin: blockedIds }`, sorted and paginated. `$geoNear` also returns each vehicle's distance from the searched location, shown on the card ("4.2 km away"). Without dates (Browse Cars), the availability step is skipped.
- **Airport search (spec §21, §23):** when the searched place is an airport, the results also include cars outside the radius that deliver to that airport (`deliveryOptions.airportCode`), and their estimated total includes the airport delivery fee. For airport searches, `$geoNear` uses a wider regional distance, and a following `$match` keeps only cars inside the radius or delivering to that airport.
- **Expired documents:** search and booking skip vehicles whose WOF (or CoF, for a vehicle that needs one) or rego expires before the trip ends, and Hosts are reminded 30 and 7 days before expiry (section 4.3). For diesel, EV and PHEV vehicles, the Host records the end reading of the current Road User Charges licence; each check-out odometer reading is compared with it, and the Host is reminded when it is close.
- **Terms fixed at booking:** a booking keeps a copy of the listing terms (fuel policy, kilometre allowance, extra-km price), the protection plan and the cancellation tier as they were when it was made. Later changes by the Host, and fee or protection-plan changes by an admin, apply only to new quotes. Post-trip charges always use the booking's copy.
- **Location and contact privacy (spec §22):** public pages show the car's suburb and an approximate map area, never the exact address. The exact pickup address, delivery or airport instructions, and each party's phone number are shown to the other party once the booking is confirmed. The number plate is not shown publicly: the listing shows the rego and WOF status ("current", with the expiry month), and the confirmed Guest sees the plate in the booking details so they can find the car.
- **Public and private files:** approved listing photos, destination images and CMS images are public and served from the CDN. Vehicle documents, message attachments, inspection photos, incident evidence and support ticket attachments are private Cloudinary assets, shown only through short-lived signed URLs to the people allowed to see them. ID document images stay with Stripe Identity and are not copied to our storage.
- **Cancellation tier for each listing:** the Host picks one of the tiers the client allows (onboarding step 4), and the listing page, checkout and booking show it. The booking keeps a copy (see "Terms fixed at booking"). If the client chooses one policy for every car, the choice is hidden and the default tier applies (section 16, item 3).
- **Contact details in messages (spec §13, §22):** until a booking is confirmed, phone numbers, email addresses and links typed into messages are masked ("contact details are shared once the booking is confirmed"), matching what each party can see (section 6.2). A thread stays open for 30 days after the trip ends (a setting), then becomes read-only, except while an incident on the booking is open.
- **Audit trails are append-only (spec §15, §25):** the app has no code path that edits or deletes `auditLogs` entries or incident `events`. Only the retention job removes entries past their retention period (section 14).
- **Same person, several accounts (spec §22):** a licence number is compared through its keyed hash when it is saved. A licence already on another account raises a risk flag for admins rather than an error, so support can merge a genuine re-registration.
- **Changes to live listings (listing moderation, spec §22):** changes to price, discounts, availability, trip rules and delivery options apply immediately. New photos and documents stay `PENDING` until support staff approve them; meanwhile the listing stays live with the photos and documents already approved. Changing the rego, VIN or chassis number, make, model or year sends the listing back to `UNDER_REVIEW`, and it is hidden from search until it is approved again.
- **Recurring availability:** a Host's recurring rules (e.g. "unavailable every weekday 8am–6pm") are expanded into `RECURRING` blocks for the next 12 months whenever the rules change, and topped up monthly. Search, quotes and bookings then use the same single overlap check.
- **Double-booking prevention.** MongoDB has no range-overlap constraint, so every calendar write (bookings, Host blocks, recurring blocks, admin overrides) goes through one function that runs inside a transaction:
  ```ts
  // server/src/modules/availability/availability.service.ts: the only code path that writes availabilityBlocks
  await mongoose.connection.transaction(async (session) => {
    // 1. Write to the vehicle document first. Two bookings for the same car now conflict,
    //    and MongoDB aborts one and retries it automatically.
    await Vehicle.updateOne({ _id: vehicleId }, { $inc: { bookingSeq: 1 } }, { session });
    // 2. Check the requested range (including the buffer) against existing blocks
    const clash = await AvailabilityBlock.exists({
      vehicleId, startAt: { $lt: endWithBuffer }, endAt: { $gt: startAt },
    }).session(session);
    if (clash) throw new ConflictError('Vehicle not available');   // → 409 Conflict
    // 3. Insert the booking, the booked block and the buffer block
    ...
  });
  ```
  When the second booking is retried, it sees the first booking's block and returns `409 Conflict`. Transactions need a replica set: Atlas provides one, and docker-compose runs MongoDB locally as a single-node replica set. An automated test sends 20 simultaneous bookings for one car and checks that exactly one succeeds. An admin override (block or unblock dates) uses the same function and is written to the audit log. The `HOLD` blocks written when checkout starts (section 8.2) also go through this function, so held dates can't be double-booked either.

**Validation rules** (Zod schemas in `shared`, so the website and the API check exactly the same things)
- **Vehicle:** a number plate of 1–6 letters and numbers (stored in capitals without spaces) that is not on another active listing; a 17-character VIN (no I, O or Q) or, for an import without one, a chassis number; a year that is not in the future (next year's models allowed); seats, doors, prices and discounts within the limits in `platformSettings`, with minimum days no more than maximum days; WOF (or CoF) and rego expiry dates in the future when the listing is submitted; every required photo angle and document present before it can be submitted. The registered owner on the rego document is the Host, or the Host uploads the owner's written consent (`OWNER_CONSENT`); support staff check this during document review.
- **Search:** a pick-up time that is not in the past and a return after it; a trip length within the maximum in settings; a radius within the range in settings; price and year ranges with the minimum no higher than the maximum; a minimum rating of 1–5. Unknown filter values are ignored instead of failing the search. A search with no place covers all of NZ.
- **Booking:** a start time in the future and at least the listing's minimum notice away; an end time after the start; a length within the listing's minimum and maximum days; pickup and return options that belong to the car, with delivery addresses inside the option's radius; no overlap with any block, including the buffer; WOF (or CoF) and rego valid until the trip ends. The Guest meets the eligibility rules in settings (age, licence class, years held), holds a licence that is valid until the trip ends, is not suspended, is not the car's Host, and has not been blocked by the Host.
- **People and addresses:** a valid email address, unique regardless of capital letters; phone numbers checked and stored in E.164 (`libphonenumber-js`, +64 by default); an NZ licence number of 2 letters and 6 digits, with a 3-digit version number; a date of birth that meets the minimum age; passwords of at least 10 characters that are not on a common-password list; NZ postcodes of 4 digits and a region from the list of NZ regions; coordinates for every pickup and delivery point.
- **Content and files:** messages of up to 2,000 characters; ratings of 1–5 whole stars in every category; review text of up to 1,000 characters; photos as JPEG, PNG, WebP or HEIC (iPhone photos are accepted and converted) and documents as PDF or photos, up to 15 MB each and at least the minimum photo resolution in settings.
- **Money:** integer cents in NZD, never negative; a refund can never be more than what was paid less earlier refunds; a payout never goes below zero (anything left over is carried to the next payout as fees owed).
- **Inspections:** only the booking's Guest or Host (or support staff completing a trip, section 8.2); every required angle photographed; odometer readings are whole numbers, and the check-out reading is never lower than the check-in reading (a check-in reading below the car's last check-out reading is flagged to support); fuel or battery level 0–100 %; check-in no earlier than 2 hours before the start time.
- **Reviews and incidents:** a review only from the Guest or Host of a `COMPLETED` booking, once in each direction, within the review window. An incident only from the booking's Guest or Host (or staff), with a type and a description; a damage report must arrive within the damage-report window after check-out unless staff open it.
- **Extra charges:** a positive amount. Extra kilometres are calculated from the condition reports, never typed in. Every other charge is linked to a resolved incident and stays within any limit in the Guest Agreement.

---

## 4. Background Jobs & Realtime (MongoDB, no Redis)

### 4.1 What MongoDB handles

| Need | How it works |
|---|---|
| Background jobs (emails, SMS, reminders, expiries, payouts, extra charges) | `jobs` collection + a job runner inside the API process (4.2) |
| Live messaging and notifications across several server instances | Socket.IO with `@socket.io/mongo-adapter` (4.4) |
| Rate limiting (login, sign-up, OTP, password reset, contact form, messages) | `express-rate-limit` with a MongoDB store; counters expire through a TTL index |
| Login sessions | `sessions` collection with a TTL index (section 6) |
| Caching | A short in-memory cache per instance (60 s) for homepage content, featured vehicles and popular searches, and short CloudFront caching of public GET responses (section 13.4). Availability is never cached: it is always re-checked at quote and booking time. |

### 4.2 Job queue

```ts
// server/src/jobs/queue.ts: enqueue upserts by uniqueKey, so a reminder is never created twice
await enqueue('booking.expireRequest', { bookingId }, {
  runAt: addHours(new Date(), 24),
  uniqueKey: `expire-request:${bookingId}`,
  refId: bookingId,
});

// server/src/jobs/runner.ts: every instance polls every 5 s and claims one due job at a time
const job = await Job.findOneAndUpdate(
  { status: 'QUEUED', runAt: { $lte: new Date() } },
  { $set: { status: 'RUNNING', lockedAt: new Date(), lockedBy: instanceId }, $inc: { attempts: 1 } },
  { sort: { runAt: 1 }, new: true },
);
```

- **One instance per job.** `findOneAndUpdate` is atomic, so when two API instances poll at the same moment, only one of them gets a given job.
- **Retries:** a failed job is re-queued with exponential backoff (1 min, 5 min, 25 min, 2 h) up to `maxAttempts`, then marked `FAILED`. Failed jobs are listed in the admin dashboard with a Retry button.
- **Crash recovery:** a job stuck in `RUNNING` for more than 10 minutes (its instance crashed) is put back in the queue. On a normal deploy or scale-in, the runner shuts down cleanly and puts back any job it cannot finish (section 13.4), so this only covers real crashes.
- **Cancelling:** when a booking is cancelled, all its queued jobs are cancelled by `refId`.
- **Handlers are idempotent:** each checks the current state before acting (for example, a payout handler skips a booking that already has a transfer), so a retried job never does the work twice.
- **Recurring jobs:** after a daily job runs, it schedules its next run with a dated `uniqueKey` (e.g. `daily.hostReminders:2026-10-01`). The unique index means several instances create it only once.
- **Cleanup:** finished jobs are deleted after 30 days by the TTL index on `finishedAt`.
- **Setting:** `RUN_JOBS=true|false` per instance. At launch every instance runs jobs. At larger scale the runner runs as its own ECS service from the same image (section 13.6).

### 4.3 Job list

| Job | When it runs | What it does |
|---|---|---|
| `email.send` | Immediately | Renders the React Email template and sends it through Resend. Up to 5 attempts. |
| `sms.send` | Immediately | Sends an SMS through Twilio |
| `booking.expirePaymentHold` | 30 min after checkout starts | Releases the held dates if payment was not completed |
| `booking.expireRequest` | 24 h after a request-to-book | Expires the request, releases the payment authorisation, emails both parties |
| `reminder.pickup` | 24 h and 2 h before the trip starts | Email + SMS + a system message in the booking chat |
| `reminder.return` | 2 h before the trip ends | Email + SMS + a system message in the booking chat |
| `trip.reviewRequest` | After check-out | Asks both parties to leave a review |
| `trip.extraCharges` | After check-out | Calculates extra kilometres (section 5) and charges the Guest's saved card. If the charge fails, the Guest gets a link to pay, retries follow, and support is alerted. |
| `payout.transfer` | 24 h after the trip starts | Creates the Stripe transfer to the Host, if check-in shows the trip went ahead. If check-in is missing, an incident or card dispute is open, or the Host has not finished payout setup, the payout is held (with the reason) and re-checked daily. The Host's share of a kept Guest cancellation fee (including a Guest no-show) needs no check-in and is paid at the same time. |
| `messages.unreadEmail` | 10 min after a message | Emails the recipient if the message is still unread (and SMS if they opted in) |
| `availability.expandRecurring` | When recurring rules change, and monthly | Rebuilds `RECURRING` blocks for the next 12 months through the availability service |
| `daily.hostReminders` | Daily at 9:00 am NZ time | WOF, rego and insurance reminders at 30 and 7 days before expiry, plus Host-set maintenance reminders when due. Also flags confirmed bookings whose car's WOF, CoF, rego or RUC licence expires before the trip ends (section 8.2). |
| `daily.dataRetention` | Daily | Deletes data that has passed its retention period, such as ID images 90 days after verification (section 14) |
| `trip.startCheck` | 1 h and 2 h after the trip start | If check-in has not been done, reminds both parties, then alerts support (possible no-show, section 8.2) |
| `trip.returnCheck` | At the return time plus the grace period, and 24 h later | Late return: reminds the Guest and tells the Host, who can report it. If check-out is still missing after 24 h, alerts support to complete the trip (section 8.2). |
| `reviews.reveal` | When a booking's review window closes | Publishes any review still waiting for the other side's review (section 9, Days 21–22) |

### 4.4 Realtime
- Socket.IO runs on the same Express server. Users join a room for their own notifications and one for each booking chat they are part of.
- `@socket.io/mongo-adapter` passes events between API instances through a MongoDB collection, so a message sent to one instance reaches a user connected to another.
- Job handlers run in the same process, so they emit notifications through the same Socket.IO server.
- If the socket disconnects, TanStack Query refetches messages and notifications when the connection returns, so nothing is lost.
- With 2 or more app tasks behind the load balancer, the client connects over WebSocket first. HTTP long-polling is only a fallback for networks that block WebSockets, and load balancer stickiness keeps a long-polling client on one task (section 13.4).

---

## 5. Pricing Engine (`shared/src/pricing.ts`)

One function, imported as `@driveshare/shared/pricing`, is shared by the client (for display) and the API (the source of truth):

```
days            = ceil((end - start) / 24h)          counted on NZ wall-clock time (see "Trip days" below)
base            = dailyPrice × days
discount        = days ≥ 28 ? monthlyDiscount : days ≥ 7 ? weeklyDiscount : 0
rental          = base − discount
delivery        = pickup option fee + return option fee
serviceFee      = rental × PLATFORM_GUEST_FEE_PCT   (from platformSettings)
protection      = selected protection plan (optional / mandatory per plan config)
total (incl GST)= rental + delivery + serviceFee + protection
gstComponent    = total × 3/23                       (NZ GST 15%, prices shown GST-inclusive)
hostPayout      = rental + delivery − HOST_COMMISSION_PCT × rental

After the trip (unless unlimitedKm):
extraKm         = max(0, (checkOutOdometer − checkInOdometer) − kmAllowancePerDay × days)
extraKmCharge   = extraKm × extraKmCents             (Host receives it less HOST_COMMISSION_PCT)
```

Checkout shows **Mandatory** and **Optional** line items in separate groups (spec §7), with the total in NZD. All fee percentages and protection plans are admin-configurable. The cancellation policy engine (`shared/src/policies.ts`) calculates refunds and cancellation fees the same way on both sides. When the Guest cancels, it applies the booking's cancellation tier. When the Host cancels, the Guest gets a full refund and the Host cancellation fee from `platformSettings` applies (section 8). How GST applies to the Host's rental amount and to the platform's fees is confirmed by the client's accountant (section 16) before the pricing engine is finalised.

- **Trip days:** `days` is counted on NZ wall-clock time (`Pacific/Auckland`), so a 10 am to 10 am trip is one day even when a daylight-saving change makes it 23 or 25 hours long. Any other part day counts as a full day.
- **Estimated totals:** the estimated total on cards, listings and Saved cars comes from this same function and includes every mandatory charge (rental, service fee, mandatory protection and GST), so the Guest never meets a surprise fee at checkout (spec §29, transparent pricing). Delivery is added when the Guest chooses it; airport searches already include the airport delivery fee (section 3).
- **Guest cancellation fees:** when a Guest cancels and part of the payment is kept under the booking's tier, the kept rental is shared like any rental: the Host receives it less `HOST_COMMISSION_PCT` in their next payout, and the platform keeps its share. The split is confirmed at the kickoff (section 16, item 3).
- **Discounts shown:** a weekly or monthly discount is its own line in the price breakdown (for example "Weekly discount −$42"), so the Guest sees the saving.
- **Card processing fees:** Stripe's fees are paid by the platform out of its service fee and commission, unless the client decides otherwise (section 16, item 2). Guests never see them. Stripe does not return its fee when a payment is refunded, so the admin revenue report shows processing fees as a platform cost.
- **GST and Hosts (spec §23):** whether GST applies to a Host's rental depends on the Host's GST registration (recorded in the Host profile) and on the accountant's advice (section 16, item 7). The pricing function takes the Host's GST status as an input, so the confirmed rules change one function, and the breakdown, receipts and earnings statements follow.
- **Security deposit:** if the client decides to take one (section 16, item 4), it is shown as a separate refundable hold, never inside the total payable. Card holds expire after about 7 days, so the hold mechanics (when it is placed, and how long trips are covered) are designed once the amount and rules are decided.

---

## 6. Authentication, Roles & Permissions

### 6.1 Sign-in and sessions
- Email and password (bcrypt), with an email verification link and a password reset link sent by email.
- Mobile verification by SMS one-time code (Twilio Verify). Phone numbers are stored in E.164 format. The input defaults to +64 but accepts overseas numbers, so international visitors can verify too (spec §23).
- Short-lived JWT access token (15 min) and a rotating refresh token (30 days), both in `httpOnly`, `Secure`, `SameSite=Lax` cookies. Refresh tokens are stored hashed in the `sessions` collection, and a TTL index removes them when they expire. Mobile apps can use the same tokens via the `Authorization` header.
- **Sign in during checkout:** a Guest who is not logged in can sign in or create an account inside the booking flow without losing their selected car, dates and options (spec §7 step 6).
- **Agreements:** Terms and Privacy are accepted at sign-up, the Guest Agreement at checkout and the Host Agreement in the Host application. Each acceptance is saved with the document version, time and IP address. When a legal document changes, users accept the new version at their next sign-in.
- Rate limiting on auth routes (MongoDB store, section 4.1), keyed on the visitor's real IP behind CloudFront and the load balancer (section 13.4), and account lockout after repeated failures.
- **When email and mobile are verified (spec §22):** the mobile number is verified by SMS code inside checkout and inside the Host application, without leaving the page. The email link is sent at sign-up; it does not block a first checkout (keeping checkout fast, spec §20), but it must be clicked before the first trip starts or the Host application is approved. Final rules are part of section 16, item 5.
- **Account changes:** a new email address is verified before it replaces the old one; a new phone number needs a new SMS code; changing the password signs out every other session and sends the Password changed email.
- **Staff two-factor sign-in (spec §25):** admin and support accounts must set up an authenticator app (TOTP) before the admin portal opens, and enter a code at each sign-in. A lost authenticator is reset by another admin, and the reset is written to the audit log.

### 6.2 Roles and permissions (spec §2)

| Role | Can do |
|---|---|
| **Guest** | Search, book and pay, message Hosts, check in and check out, report incidents, review Hosts, manage their own account |
| **Host** | Everything a Guest can do. Once the Host application is approved: list and manage vehicles, calendar, bookings, earnings and payouts, and review Guests. One user can be both Guest and Host. |
| **Support / Operations staff** | The admin portal with limited access: verification queue, users (view, suspend), bookings (view, edit status, calendar override), incidents and disputes, support tickets, review and report moderation. Refunds only with the `REFUNDS` permission. No fee settings, content management or staff management. |
| **Administrator** | Everything, including fees and platform settings, homepage content, FAQs and help articles, reports, staff roles and permissions, and the audit log |

- Role and permission middleware on every route: `requireRole('HOST')`, `requirePermission('REFUNDS')`.
- React Router route guards hide pages from users without the right role. The API enforces the same rules and is the real security boundary.
- Every admin and support write action goes through the `auditLog()` middleware.
- **What each party can see (privacy, spec §22):** a Host sees a Guest's first name, photo, verification status (verified or not, never the documents or licence number), rating and trip count; the Guest's phone number appears only on a confirmed booking. A Guest sees the Host's first name, photo, rating, trips and response rate, and the exact pickup address and phone number once the booking is confirmed (section 3). Contact details typed into messages are masked until then (section 3). Support staff open a message thread only from a report, incident or support ticket, and each opening is written to the audit log.

---

## 7. Email & Notifications

**Provider:** Resend (simple, good deliverability, React Email support), behind a `Mailer` interface in `server/src/integrations` so it can be swapped for AWS SES.
**Local development:** a console mailer prints each email's subject and links to the terminal and saves the HTML to `server/.mail/`. `npm run email:dev -w server` previews every template in the browser. Staging sends real emails through Resend, so the client can receive them.

**Flow:** service event → `notify(userId, 'BOOKING_CONFIRMED', data)` → `notifications` document (shown in the in-app notification centre and pushed over Socket.IO) → `email.send` / `sms.send` job (section 4) → provider API → delivery status stored on the notification, updated by the providers' status webhooks (Resend and Twilio). Failed sends retry up to 5 times with exponential backoff, and bounced emails are shown to support staff on the user's record.

**Channels (spec §19)**
- **Email:** every template below.
- **In-app:** notification centre with an unread count, for bookings, messages, payments, payouts, verification and incidents.
- **SMS:** phone verification codes, pickup and return reminders, new booking requests for Hosts, and (opt-in) new messages.
- **Push (mobile apps / PWA):** not in the 30-day scope (MILESTONES.md). `notify()` sends through one channel adapter per delivery method, so a push channel is added later without changing the events that trigger notifications (section 10.4).
- Users choose which non-essential emails and SMS they receive in their notification preferences.
- **Build order:** `notify()` with its email, SMS and in-app channels is built with the booking flow (Days 11–13), so a Host hears about a booking request straight away. Until Socket.IO arrives with messaging (Days 17–19), the notification bell in the header refreshes every minute. The full notification centre and preferences follow on Days 21–22.
- **SMS quiet hours:** a non-urgent SMS that would arrive between 9 pm and 7 am NZ time waits until 7 am (hours set in `platformSettings`). Verification codes, a pickup reminder for a trip starting within 3 hours, and missing check-in and late-return alerts are sent straight away.
- **Marketing messages (NZ Unsolicited Electronic Messages Act 2007):** transactional messages about a user's account and bookings are always sent. Promotional emails and SMS go only to users who opted in, identify the sender, and include an unsubscribe link or reply keyword that is honoured within 5 working days.

All the example notifications in spec §19 are covered: booking received, booking confirmed, payment successful, pickup reminder, return reminder, new message, verification required, payout processed, cancellation and incident update.

**Email templates** (`server/src/emails`)

| Group | Templates |
|---|---|
| Account | Welcome, Verify email, Reset password, Password changed, Verification required, Verification approved or rejected, Account suspended, Account closed |
| Host | Host application received, Host application approved or rejected, Listing submitted, Listing approved, changes requested or rejected, Listing changes (new photos, documents) approved or rejected, Payout setup needed before a listing goes live, Vehicle suspended, WOF/CoF/rego/insurance expiring (30 / 7 days), RUC licence running out, Maintenance reminder |
| Booking | Booking request (Host), Booking waiting for verification (Guest), Booking confirmed (Guest + Host), Booking declined/expired, Cancellation (both; a Host cancellation tells the Guest about the full refund and tells the Host about any cancellation fee), Pickup reminder (24 h / 2 h), Check-in not done, Return reminder, Late return (Guest + Host), Trip completed + review request |
| Payments | Payment receipt (GST receipt), Payment failed (with a link to retry), Extra charge (extra kilometres, or a charge from a resolved incident), Refund issued, Payout processed (Host, showing any deductions) |
| Support and safety | New message (only if unread for 10 min), Incident created and updated, Support ticket received and replied, Admin alerts |

**Deliverability:** a dedicated sending subdomain (`mail.<domain>`) with SPF, DKIM and DMARC records, plus a List-Unsubscribe header on non-transactional mail.

---

## 8. Payments, Refunds & Payouts (Stripe)

### 8.1 Payments, refunds and payouts

1. The Guest checks out, and the API creates a PaymentIntent in NZD with `capture_method: manual` for request-to-book (and for an Instant Book checkout whose verification is in review, section 8.2), or automatic for Instant Book.
2. The Stripe Payment Element handles cards, Apple Pay and Google Pay. The Guest can choose a saved card or add a new one. Card data never reaches our servers.
3. The card is saved for later charges (`setup_future_usage: off_session`), with the Guest's consent shown at checkout, so extra kilometre charges and other post-trip charges (item 11) can be taken after the trip.
4. A `payment_intent.succeeded` webhook confirms the booking, blocks the calendar (the `HOLD` block becomes `BOOKED`), and sends the emails. For a request-to-book, the authorisation (`payment_intent.amount_capturable_updated`) sends the request to the Host, and the capture after the Host accepts confirms it.
5. A request-to-book that the Host does not accept within 24 h expires and the payment authorisation is released (`booking.expireRequest` job).
6. **Failed payments:** at checkout, the Guest sees a clear reason and can retry or choose another method while the dates stay held for 30 minutes. A failed extra charge sends the Guest a link to pay, is retried, and alerts support. Every failure is recorded on the payment and shown in the admin dashboard.
7. **Saved payment methods and history:** each Guest has a Stripe Customer. The Guest dashboard lists saved cards (add or remove) and a payment history with receipts and refunds.
8. The Host onboards to **Stripe Connect Express** (bank account and identity) from the Host dashboard.
9. **Payouts:** the `payout.transfer` job creates a Transfer to the Host 24 h after the trip starts. The payout is held if an incident or card dispute is open, check-in is missing, or payout setup is unfinished (section 4.3). Hosts see upcoming and paid payouts.
10. **Cancellations:** the policy engine calculates the refund and the cancellation fee, and the API issues a full or partial refund and records both on the booking and payment. Admins, and support staff with the `REFUNDS` permission, can also issue refunds from the admin dashboard.
    - **Host cancellations:** the Guest always gets a full refund. Any Host cancellation fee (set in `platformSettings`; $0 until the client decides, section 16 item 3) is added to the Host's `feesOwedCents` and deducted from their next payout, shown as a line on that payout. Admins can waive the fee, and the waiver is written to the audit log. Repeated Host cancellations raise a risk flag for admins.
11. **Other post-trip charges:** when an incident is resolved against the Guest (fuel not returned as the listing's fuel policy requires, cleaning, late return, or a damage amount allowed by the Guest Agreement), an admin adds an extra charge to the booking, linked to the incident. It uses the same saved-card, pay-link and retry flow as extra kilometres, and the Host's share is added to their payout.
12. **Card disputes (chargebacks):** a `charge.dispute.created` webhook alerts admins, links the dispute to the booking, and holds any unpaid payout. Admins answer it in Stripe with the evidence from the booking (inspection photos, messages, agreement acceptance).
13. All webhooks are verified by signature and are idempotent: each Stripe event ID is saved in `stripeEvents` (unique index), so a repeated event is skipped.
14. **Request-to-book payments:** the card is authorised at checkout while the dates are held. When the Host accepts, the payment is captured and the booking is confirmed. A decline, the 24 h expiry or a rejected verification releases the authorisation. Card authorisations last about 7 days, so the 24 h request window is always within that limit.
15. **Refunds and the Host's share:** every refund records who pays for it: the platform (a goodwill refund, or its own fees) or the Host (a policy refund of rental the Host would otherwise receive). A Host-funded refund made before the payout reduces it. One made after the payout is deducted from the Host's next payout and shown as a line on it, or, if an admin chooses, the Stripe transfer is reversed. Both are written to the audit log.
16. **Guest cancellation fees:** the part of a cancelled booking that is kept is split as described in section 5, and the Host's share appears in their earnings and next payout.
17. **Apple Pay and Google Pay setup:** the staging and production domains are registered with Stripe as payment method domains, so the wallet buttons appear. They show only on devices and browsers that support them, with cards as the fallback.
18. **Receipts and GST (spec §17, §23):** each receipt shows the booking reference, every line item, the GST included and the GST details that NZ rules require, in the form the client's accountant confirms (section 16, item 7). It is available as a page and a PDF. Overseas cards are charged in NZD, and the card issuer converts.
19. **Payout timing shown to Hosts:** the transfer moves the Host's share to their Stripe balance 24 h after the trip starts (item 9), and Stripe then pays their bank on its payout schedule, usually within a few business days. The Host dashboard shows both dates. Until the Host finishes payout setup, payouts wait with the reason "payout setup needed".
20. **Host payout account problems:** a Stripe `account.updated` webhook showing that payouts are disabled (for example, Stripe needs more details from the Host) holds the Host's payouts with the reason "payout setup needed", tells the Host what Stripe needs, and releases the payouts once it is fixed.
21. **Failed refunds:** a refund that fails (for example, to a closed card) is recorded on the payment and alerts support in the admin dashboard, so they can return the money another way.
22. **Commission invoices for GST-registered Hosts (spec §23):** each payout statement lists the platform commission and its GST, so a GST-registered Host can use it as a tax invoice, in the form the accountant confirms (section 16, item 7).

### 8.2 Booking lifecycle and edge cases

Every status change goes through the booking service, is recorded in `statusHistory`, and sends the notifications in section 7.

```
PAYMENT_PENDING ──paid (Instant Book, Guest verified)────────────────────────▶ CONFIRMED
PAYMENT_PENDING ──authorised (request-to-book, or verification in review)───▶ PENDING
PAYMENT_PENDING ──30 min without payment──▶ EXPIRED                  (HOLD block removed)
PENDING ──Host accepts (and verification approved)──▶ CONFIRMED      (payment captured)
PENDING ──Host declines──▶ DECLINED
PENDING ──24 h without an answer, or verification rejected──▶ EXPIRED (authorisation released)
CONFIRMED ──check-in done──▶ ACTIVE ──check-out done (or completed by support)──▶ COMPLETED
PENDING or CONFIRMED ──cancelled by the Guest, the Host or an admin (incl. no-shows)──▶ CANCELLED
```

- **Dates held during checkout and requests:** when checkout starts, a `HOLD` block is written through the availability service (section 3). It lasts 30 minutes for payment, and is kept while a request waits for the Host (up to 24 h), so nobody else can book those dates. It becomes a `BOOKED` block on confirmation and is removed on expiry, decline or cancellation. The Host's calendar shows held dates as "Request pending".
- **Verification at checkout (spec §7 step 7):** a Guest who must verify (rules in settings) does it inside checkout. Stripe Identity usually answers within minutes, and checkout waits for the result. If the check needs manual review, the booking becomes a request, even for an Instant Book car: the card is authorised, support staff are alerted, and the booking is confirmed when support approves it within 24 h. Otherwise it expires and the authorisation is released. The Guest is told what is happening at every step.
- **Dashboard grouping (spec §8, §9):** *Upcoming* shows `CONFIRMED` trips plus `PENDING` requests (labelled "Waiting for the Host" or "Verification in review"). *Current* shows `ACTIVE` trips, plus `CONFIRMED` trips whose start time has passed (with a prompt to complete check-in). *Completed* shows `COMPLETED` trips. *Cancelled* shows `CANCELLED`, `DECLINED` and `EXPIRED` bookings, each labelled. `PAYMENT_PENDING` bookings are not shown.
- **Check-in not done:** 1 h after the start time both parties are reminded, and after 2 h support is alerted (`trip.startCheck`). If the Host is not there (for example a delivery or a key-box handover), the Guest can take the check-in photos and readings, and the Host confirms them later; the report records who took each photo. A trip's payout waits until check-in is done (a kept cancellation fee is paid without one, section 4.3).
- **No-shows:** a Guest who does not turn up is treated as a Guest cancellation at the start time, and the booking's cancellation tier applies. A Host who does not turn up, or whose car is not available, is treated as a Host cancellation once support confirms it: the Guest gets a full refund, the Host cancellation fee applies, and support helps the Guest find another car. Either party reports a no-show as an incident (type `NO_SHOW`).
- **Late returns and missing check-out:** at the return time plus the grace period in settings, the Guest is reminded and the Host is told (`trip.returnCheck`); the Host can report a late return, which can lead to a `LATE_RETURN` charge. If check-out is still missing 24 h after the return time, support completes the trip using the Host's odometer and fuel reading and photos. Reviews and extra-kilometre charges then follow as normal.
- **Check-out (spec §14):** the Guest takes the return photos and readings when handing the car back, and the Host reviews and confirms them. Either party can flag new damage during check-out, and the Host can still flag it until the damage-report window in settings closes (for cars returned when the Host isn't there).
- **Withdrawing a request:** a Guest can withdraw a `PENDING` request before the Host answers. The authorisation is released and no fee applies. A Host answers a request by declining it, not by cancelling, so a decline carries no Host cancellation fee.
- **Early return:** when the Guest returns the car early, check-out completes the trip at that time. Unused days are not refunded automatically (launch default, section 16 item 3). Support can make a partial refund when the Host agrees or the car was at fault, recorded with who funds it (section 8.1, item 15).
- **Documents expiring before a booked trip:** the daily Host reminder job also checks confirmed bookings. If the car's WOF, CoF, rego or RUC licence would expire before a booked trip ends, the Host is asked to upload the renewal. If it is still missing 72 hours before the trip, support is alerted, contacts both parties, and can cancel the booking as a Host cancellation.
- **Trip changes and extensions:** changing or extending a confirmed trip is not in the spec. At launch, support handles it: they cancel and rebook, or adjust the booking with an extra charge or a partial refund. A self-service change request is on the roadmap (section 10.5, R13).
- **Admin status edits (spec §18):** admins change a booking's status only along the transitions above, plus "mark completed" and "cancel" overrides. Each change asks for a reason and runs the same side effects as the normal path (refund preview and choice, payout, calendar and notifications). It is recorded in the booking's status history and the audit log.
- **Vehicle deactivated or suspended:** when a Host deactivates a car, it is hidden from search and cannot take new bookings. Its existing bookings stay, and the Host must cancel them explicitly, under the Host cancellation rules. When an admin suspends a vehicle, it is hidden at once and the admin sees its upcoming bookings. The admin keeps each one, or cancels it as a platform cancellation: the Guest gets a full refund, and the Host fee applies only when the Host is at fault.
- **User suspended:** the user's sessions are revoked and they cannot sign in. Their listings are hidden, their payouts are held, and messages from them stop. Their upcoming bookings are listed for the admin, who cancels each one (the other party is refunded in full or told) or lets it go ahead. Lifting the suspension restores the listings and releases the payouts.
- **Host payout setup:** a Host can submit a listing before setting up payouts, but the listing goes live only when Stripe Connect onboarding is finished. This rule is enforced from Days 17–18, when Connect onboarding is built. Until then, the Host dashboard shows a to-do list.
- **Tolls and fines (NZ):** unpaid NZ toll-road charges and traffic or parking infringement notices are sent to the Host as the registered owner. The Host reports them as an incident (type `TOLL` or `FINE`) with the notice attached. Support then gives the issuing authority the driver details it needs, or charges the Guest where the Guest Agreement allows. The process is confirmed by the client's legal adviser (section 16, item 14).
- **Account closure (NZ Privacy Act 2020):** a user asks to close their account from their personal details. Closure is refused while they have an upcoming or active trip, an open incident, an unpaid charge or a payout still due. Otherwise, the account is anonymised and its listings are removed, keeping only the records the law requires (section 14).

---

## 9. Feature Implementation Checklist: 30-Day Schedule

**Assumptions for 30 days**
- **Team:** 1 senior UI/motion designer (full time Days 1–8, then part time for design QA and animation review) and 2–3 full-stack developers working in parallel on frontend and backend. At least one developer has strong frontend animation experience.
- **UI:** built on shadcn/ui, restyled with our luxury design tokens. High-fidelity Figma designs are made for the key screens, linked into a clickable prototype that includes the signature animations. The other flows get mid-fidelity designs, and their screens reuse the same components and motion building blocks.
- **Client:** decisions and feedback arrive within 24 hours.

**Status key:** ✅ Done · 🟡 Partly done (the note says what is left) · ⬜ Not started · 👤 Needs the client or the design team (not a coding task)

**Progress:** the checklist was reset on 24/09/2026 when the stack was simplified (no Redis, no migration tool, no monorepo tooling). No tasks have started yet. The same day, hosting was moved to AWS (section 13), and the plan was cross-checked requirement by requirement against the specification ([project_requirements.md](project_requirements.md)). The gaps found were added to the relevant sections and rows, and every requirement is traced in Appendix A. A second, deeper cross-check on 25/09/2026 fixed the build order of in-app and SMS notifications, and added the rest of the task dependencies with phase gates, Instant Book and cancellation-tier choices in onboarding, missing API endpoints, indexes, validation rules and booking edge cases, GST handling for GST-registered Hosts, NZ marketing-message and fair-trading rules, a load test, and new client inputs (section 16, items 18–19).

### Phase 1: Design & Foundation (Days 1–5)
**Delivers:** design style guide, key page designs and the animation prototype; a staging link where you can create an account and receive a verification email.

| Day | Tasks | Status |
|---|---|---|
| 1 | Kick-off workshop: fees, cancellation tiers (including how a kept Guest cancellation fee is shared with the Host, and no-shows), security deposit, protection, driver eligibility (age, licence classes, overseas licences and IDP), when verification is required, GST treatment, the final brand name (section 16, item 1), items the spec does not define (item 13) and the optional NZ licence-check and plate-lookup services (item 15). Confirm that Stripe Identity is available for the client's NZ Stripe account (section 17). Client creates the AWS, Stripe, Resend, Twilio, Cloudinary, Google Cloud and MongoDB Atlas accounts and confirms the hosting region (by Day 2, section 16 item 6). Until decisions arrive, launch defaults live in `shared/src/policies.ts` and are editable in `platformSettings`. | 👤 Client |
| 1–2 | Project setup (section 2): npm workspaces (`client`, `server`, `shared`), TypeScript, ESLint/Prettier with the client/server import rule, env validation, docker-compose MongoDB replica set. GitHub Actions CI (lint, typecheck, test, build, Lighthouse and bundle size budgets). | ⬜ |
| 2–3 | **AWS staging** (section 13): Dockerfile, CDK stack (VPC with NAT gateway, load balancer, ECS Fargate, ECR, S3 + CloudFront + WAF, Secrets Manager, CloudWatch alarms), GitHub Actions deploy through OIDC with the index sync as a one-off task, and the app settings AWS needs (`/healthz`, graceful shutdown, `trust proxy`, Socket.IO transports). Atlas staging database that accepts only the NAT gateway's IP. Sentry error monitoring on staging from the start. Staging live on `staging.<domain>` with HTTPS. | ⬜ |
| 1–4 | Original luxury art direction: moodboard, photography and video selection, colours and fonts, leading to the **design style guide** and a **Figma component library** that matches the coded components in 12.3 (spec §30). High-fidelity Figma designs for Home, Search, Vehicle listing, Checkout and Host dashboard, for mobile and desktop, with notes on tablet and responsive behaviour. Design tokens in `shared/src/tokens.ts` → Tailwind theme, with a WCAG contrast test. | ⬜ Tokens · 👤 Designer |
| 3–5 | Motion system (durations, easing curves, springs) and a **clickable prototype** (spec §30) that links the high-fidelity key screens into the main Guest flow (Home → Search → Vehicle listing → Checkout) plus the Host dashboard, on mobile and desktop, with the key animations: homepage hero, opening a car (card to listing), search filters, checkout. Motion building blocks in code (`MotionProvider`, `Reveal`, `Stagger`, `CountUp`, `Sheet`, sliding `Tabs`, View Transitions). | ⬜ Code · 👤 Designer |
| 4–8 | Mid-fidelity flow designs (spec §30), for mobile and desktop (the Admin dashboard for desktop and tablet): Guest dashboard including the active trip view, Admin dashboard, Host application and vehicle onboarding, booking and payment flow (including 3-D Secure, payment failed, request waiting for the Host, verification in review and the extra-charge pay link), messaging, vehicle inspection (including an upload waiting for signal). A **states board** covering loading, empty, error and success states and marketplace states (dates unavailable, payment failed, verification pending, listing under review, vehicle suspended). Email template design. Developer-ready specs in Figma Dev Mode. | 👤 Designer |
| 2–3 | Mongoose models for all collections and their indexes, `db:indexes` script, seed data (NZ cities, suburbs and airports, the 5 launch destinations and other popular places, 20 demo vehicles, test Hosts, Guests, a support user and an admin, FAQs, help articles). The development and staging seed also adds completed demo trips with reviews, so screens that show reviews, ratings and trip history can be built before the review feature (Days 21–22). Model tests against an in-memory replica set. | ⬜ |
| 3–5 | Auth: sign-up, login, logout, email verification, password reset, mobile OTP (NZ and overseas numbers), agreement acceptance with versions. Password change, and email and phone changes that are verified again (section 6.1). Rotating refresh tokens, account lockout, rate limits, role and permission middleware, audit-log middleware (section 6). | ⬜ |
| 3–5 | Mailer (Resend + console) + MongoDB job queue and runner (section 4) + templates: welcome, verify email, reset password, password changed. Client: React Router with lazy-loaded routes and role guards. Layout shell: header, footer (legal, support and social links), mobile nav, 404 and error pages, favicons and a web app manifest (the site can be added to a phone's home screen, the base for PWA push later). | ⬜ |
| 5 | Staging check: create an account and receive the verification email. **Design sign-off** (look and feel, key screens, animation prototype). | 👤 Client |

### Phase 2: Core Marketplace (Days 6–15)
**Delivers:** a working staging site where a test Guest can find a car and book it, and a test Host can list a car and receive the booking.

| Day | Tasks | Status |
|---|---|---|
| 6–7 | Component library in `client/src/components/ui` (DateTimeRangePicker, LocationAutocomplete, Gallery + lightbox, Stepper, Calendar, Slider, Badge, Sheet, Toast, Skeleton) with hover, press and focus states. Marketplace components (VehicleCard, PriceBreakdown, StickyBookingBar) in `client/src/features`. | ⬜ |
| 6–8 | Search API: `$geoNear` search with **all 16 filters from spec §5**: price range, location and radius, vehicle type, make and model, year, automatic/manual, seats, fuel type, hybrid/EV, airport delivery, delivery available, Instant Book, minimum rating, unlimited kilometres, pet friendly, child seat. Distance from the searched location, availability exclusion when dates are given, sort (recommended, price, rating, distance, newest), pagination. Location autocomplete (NZ only) combining Google Places with our own cities, suburbs, destinations and airports. **Airport searches** also return cars that deliver to that airport, with the airport delivery fee in the estimated total (section 3). Estimated totals come from the shared pricing function, so its core (trip days in NZ time, discounts, fees and GST, with unit tests) is written first, on Days 6–7. Search Results URLs are `noindex` (section 1.4). | ⬜ |
| 7–9 | **Homepage:** headline "Rent a car from local owners across New Zealand.", search module (Where are you going? with autocomplete, pick-up date and time, return date and time, **Search Cars**), secondary call to action "Have a car? Earn money by sharing it." with a **Become a Host** button, featured vehicles, popular NZ destinations, how it works, Host earnings, safety and trust, customer reviews, FAQs, footer. **Browse Cars** (all vehicles, no dates needed) and **Search Results** (with dates and estimated totals): cards with photo, make and model, year, location and distance, rating and completed trips, daily price, estimated total, delivery and Instant Book badges, key features. Filter sheet on phones, sidebar on desktop, filters in the URL, animated result changes, skeletons. The customer reviews section uses real published reviews only and stays hidden until the threshold in settings is reached. Searches and city pages with no cars show a helpful empty state with a Become a Host prompt. Cars without reviews yet show a **New** label instead of stars, and the minimum-rating filter leaves them out. A search with no place covers all of NZ, and a search with no results suggests removing filters or widening the radius. A signed-in Guest's last search is saved for the Saved cars totals. Search inputs follow the validation rules in section 3. | ⬜ |
| 8–10 | **Vehicle listing page:** swipeable gallery (front, rear, driver side, passenger side, interior, dashboard/odometer, boot, tyres, existing damage) with full-screen view; make, model, year and variant; location; Host card with rating and trip history; price per day; transmission, fuel type, engine/powertrain, seats and doors; **registration and WOF information** (rego and WOF or CoF status with the expiry month, and the RUC status for diesel, EV and PHEV cars, without the number plate, section 3); fuel policy; kilometre allowance; delivery and pickup options; cancellation policy (the listing's tier); reviews; location map (suburb and approximate area only, drawn with the Maps Static API; the exact address after confirmation); **sticky Book button on mobile**. Shared-photo transition from card to listing. | ⬜ |
| 8–11 | **Host application** (Host profile + Host Agreement) and **vehicle onboarding in 6 steps:** (1) rego, make, model, year, variant, **VIN or chassis number** (imports without a VIN), CoF and RUC details where they apply, **body type** (for the vehicle type filter), fuel type, **engine/powertrain** (engine size and cylinders, or EV range and battery), transmission, seats, doors, **key features** and extras (pet friendly, child seat available); (2) registration, WOF, insurance and other documents, plus the registered owner's written consent when the Host is not the owner; (3) photos: the **minimum required set** (front, rear, driver side, passenger side, interior, dashboard/odometer, boot, tyres, plus existing damage where there is any) with instructions, example shots and camera capture on phones, with **low-quality and missing photos flagged for review** (automatic checks in the browser for low resolution, darkness and blur; the flags are shown to support staff in the listing review queue); (4) daily price, weekly and monthly discounts, minimum and maximum rental, kilometre allowance or unlimited kilometres, extra-km price, **fuel policy**, **cancellation tier** (when the client lets Hosts choose, section 3); (5) availability calendar, blocked dates, minimum notice, preparation time, **Instant Book on or off** (off means every booking is a request the Host accepts; this drives the Instant Book badge and filter); (6) Host pickup location, delivery, airport delivery (with meeting instructions), custom delivery locations, delivery fees. Auto-saved drafts, resume where you left off, uploads to Cloudinary with progress, a missing-items check before submit (required documents and photo angles come from `platformSettings`, and fields follow the validation rules in section 3). A basic admin queue approves or rejects Host applications and approves, rejects or requests changes to listings, so a test listing can go live on staging. **Edits to live listings** follow the moderation rules in section 3: new photos and documents wait in the same queue, and rego, VIN or chassis number, make, model or year changes send the listing back to Under Review. An approved listing goes live once the Host's payout setup is finished (enforced from Days 17–18, section 8.2). | ⬜ |
| 10–11 | **Availability calendar:** month and week views, booked dates blocked automatically, manual blocks, **recurring availability**, minimum notice, buffer time, admin override. The single transactional write path that **prevents double-booking** + the 20-simultaneous-bookings test. `HOLD` blocks for checkout and requests waiting for the Host (section 8.2), shown on the Host calendar as "Request pending". | ⬜ |
| 10–12 | Pricing engine completed (section 5: protection plans, delivery fees, GST and the booked-terms copy, building on the Days 6–7 core) + `POST /vehicles/:id/quote` with trip rules (notice, min/max days, delivery options, protection plan). **Price breakdown in NZD:** rental, delivery, service fee, protection, GST, total, with mandatory and optional charges shown separately. | ⬜ |
| 11–13 | **Booking flow (all 11 steps of spec §7):** pick-up and return location, date and time (Host location, delivery address or airport), availability check, price calculation, trip details and policies, sign in or sign up without leaving checkout, verification step (licence details now; the identity check is connected on Days 19–20), choose a saved card or add a new one with Stripe Payment Element (**cards, Apple Pay, Google Pay**), confirm. Verified idempotent webhooks, Instant Book and request-to-book with the 24 h expiry and 30 min payment hold (jobs), failed-payment handling. A Guest can withdraw a request before the Host answers (section 8.2). **`notify()` with its email, SMS (`sms.send` job) and in-app channels, plus a basic notification bell** (section 7, build order): the Host receives the booking by email, SMS and in-app, and accepts or declines it from a basic bookings page (the full dashboard follows in Phase 3). Confirmation sent to both parties. The booking lifecycle and date holds in section 8.2, including a booking that becomes a request while verification is in review. Apple Pay and Google Pay enabled by registering the staging domain with Stripe (section 8.1, item 17). | ⬜ |
| 13–14 | Cancellation policy engine + refund and cancellation fee calculation for **Guest and Host cancellations** (a Host cancellation refunds the Guest in full and records any Host cancellation fee, section 8), **no-shows**, request withdrawals, early returns and the Host's share of a kept Guest cancellation fee (sections 5 and 8.2). A refund and fee preview before the user confirms a cancellation (`/bookings/:id/cancellation-preview`). **Emails:** booking request, booking confirmed, **payment receipt** (GST), **cancellation**, declined/expired, payment failed, refund issued. | ⬜ |
| 12–14 | **Public pages:** How It Works, Become a Host (earnings estimator using the client's assumptions, labelled as an estimate, section 16 item 18), Safety (including what to do after an accident, theft or breakdown: 111 in an emergency, roadside assistance, and reporting an incident), Insurance / Protection, FAQs (from the database, with FAQPage JSON-LD), About Us, Contact Us (form creates a support ticket), and the legal pages from CMS Markdown: **Terms & Conditions, Privacy Policy, Cancellation Policy, Host Agreement, Guest Agreement**. The Cancellation Policy page shows the live tiers from `platformSettings` next to its legal text, so it always matches what the system charges. Legal text is a placeholder until the client supplies it (section 16, item 11), and page copy is approved by the client (item 18). | ⬜ |
| 6–14 | Signature animations built with each screen and checked against the prototype: cinematic homepage, page transitions, photo gallery, search filters, booking steps (section 12.4). | ⬜ |
| 15 | QA pass (lint, typecheck, tests, Playwright for sign-up, login and a full Guest booking; screenshots on phone, tablet and desktop) + **end-to-end booking sign-off** on staging. | ⬜ QA · 👤 Client |

### Phase 3: Dashboards, Trust & Operations (Days 16–24)
**Delivers:** all three dashboards working on staging, with the complete trip lifecycle: book, pick up, return, review and payout.

| Day | Tasks | Status |
|---|---|---|
| 16–18 | **Guest dashboard (all 13 items in spec §8):** upcoming, current, completed and cancelled trips (grouped as in section 8.2); booking details with **receipts** (GST receipt page, printable and downloadable as PDF); **messages**; **saved cars** (with estimated totals for the last searched dates, for comparing); **payment methods** and payment history; **personal details** (including privacy requests and account closure, section 8.2); **driver licence and identity verification**; **reviews** (written and received); **notifications**; **help and support** (help articles, contact support about a booking, my support tickets). Trip detail with cancel and refund preview. The **active trip** view for the spec §28 "use vehicle" step (section 12.6). | ⬜ |
| 16–19 | **Host dashboard (all 10 items in spec §9):** my vehicles with status (Active, Inactive, Under Review, Suspended, plus Draft and Changes requested with the admin's notes) and activate/deactivate (existing bookings stay, section 8.2); bookings (requests with a countdown and the Guest's verification status, rating and trips; upcoming, current, completed and cancelled, grouped as in section 8.2); calendar and availability; a to-do list (payout setup, expiring documents, check-ins due); **earnings dashboard** (today, this week, this month, current vs previous month, lifetime, upcoming payouts, platform fees, per-booking breakdown, monthly chart; earnings are counted on the trip's start date in NZ time, weeks run Monday to Sunday, and amounts are net of Host-funded refunds and deductions; and a **GST-ready earnings statement** as a CSV download by month or NZ tax year (1 April–31 March), with rental, delivery, extra charges, platform fees, deductions and GST shown separately for the Host's records and tax return, spec §23); upcoming and paid payouts; messages; reviews; **vehicle maintenance and document reminders**; Host profile and settings (bio, GST registration and number, notification preferences, payout account). For GST-registered Hosts, the statement also shows the commission with its GST (section 8.1, item 22). | ⬜ |
| 17–18 | Stripe Connect Express onboarding + `payout.transfer` job (24 h after trip start, only once check-in is done, held while an incident or dispute is open or payout setup is unfinished). **Host payouts, refunds and cancellation fees** shown to both parties, including Host cancellation fees and Host-funded refunds deducted from the next payout, and the transfer and bank-arrival dates (section 8.1, items 15 and 19). Extra kilometre charges after check-out, admin extra charges from resolved incidents, failed-payment follow-up, and card dispute alerts (section 8). Payouts held when Stripe disables a Host's payouts (`account.updated`), failed-refund alerts, and commission with GST on payout statements (section 8.1, items 20–22). The extra-charge job and payment flow are built here against sample condition reports and incidents; they run end to end once check-out (Days 19–21) and incidents (Days 20–21) exist. | ⬜ |
| 17–19 | **Messaging (spec §13):** a chat thread for each booking, live through Socket.IO (MongoDB adapter), text and **photo attachments**, automated booking messages including the **pickup and return reminders**, email and opt-in SMS for unread messages, **report and block** (a report goes to support; blocking stops messages from that user, while booking-critical system messages still arrive). Phone numbers are shared once the booking is confirmed, contact details typed into messages are masked until then, threads become read-only 30 days after the trip (section 3), and support staff open a thread only from a report, incident or ticket, with each opening logged (section 6.2). In-app notifications now arrive live over Socket.IO (section 7, build order). | ⬜ |
| 19–20 | **Trust and verification (spec §22):** Guest identity check (Stripe Identity: ID + selfie) and **driver licence** details (NZ licence number, version and class, or an overseas licence with an IDP or approved English translation when it is not in English) checked against the eligibility rules in settings. Guests are asked to use their driver licence as the ID document where Stripe Identity accepts it, so one check covers identity and licence, and the licence number read from the document is matched with the one entered. A licence number already on another account raises a risk flag (section 3). **Host identity verification** as part of the Host application; vehicle document verification by support staff; the checkout verification step connected; manual review queue for support staff. The licence must be valid until the trip ends, and a check that needs manual review turns the booking into a request (section 8.2). The NZ licence-check service is connected here if the client chose one (section 16, item 15). The **data retention job** (`daily.dataRetention`: redacts Stripe Identity sessions 90 days after verification and applies the other periods in section 14). | ⬜ |
| 19–21 | **Digital vehicle handover (spec §14):** check-in by the Host before the trip with guided, **timestamped photos** of the front, rear, both sides, wheels, windscreen, interior and dashboard; **odometer**; **fuel or battery %**; **existing damage** pinned on a car diagram; the Guest reviews and confirms. Check-out repeats the inspection, shows each check-in photo next to the new one, lets either party **flag new damage** (which can open an incident in one tap), and creates the condition record linked to the booking. Moves the trip from Confirmed to Active to Completed. If the Host is not there at the start, the Guest takes the check-in photos and the Host confirms later. At return, the Guest takes the check-out photos and the Host confirms, and can flag new damage until the damage-report window closes. Photos are taken inside the flow with their capture time shown, and are kept in the browser until they upload, so a weak signal loses nothing. The `trip.startCheck` and `trip.returnCheck` jobs cover a missing check-in, a late return and a missing check-out (section 8.2). | ⬜ |
| 20–21 | **Damage and incident reporting (spec §15):** Guest or Host reports, incident type, description, photo and document upload, **case number**, status workflow, assignment to support staff, admin updates visible to both parties, one party or internal only, and a **complete audit trail** of every event. Resolving a case can add an extra charge to the Guest (fuel, cleaning, late return, damage, tolls or fines where the Guest Agreement allows), linked to the case (section 8). No-show, toll and infringement-notice types (section 8.2). The accident, theft and breakdown forms start with emergency guidance (111 in an emergency, then the roadside assistance number from the protection plan). | ⬜ |
| 20–22 | **Help and support:** help centre (help articles for Guests and Hosts), support ticket inbox for support staff (from the contact form, the dashboards and bookings). **Suspicious activity monitoring:** risk flags (many failed payments, booking velocity, card country different from the account, Stripe Radar warnings, repeated reports, repeated Host cancellations, the same licence on more than one account) shown to admins for review. | ⬜ |
| 21–22 | **Two-way reviews and ratings (spec §16)** after a completed trip: overall, communication, cleanliness/condition, pickup/return, vehicle care/Guest behaviour, written review (the categories for each direction are listed in section 3). A review window from settings, with both reviews revealed together once both are in or the window closes (`reviews.reveal`), so neither side can retaliate. **Moderation under defined rules:** reviews with contact details, links or abusive language are held for review, anyone can report a review, and admins hide reviews with a recorded reason. **Notifications:** in-app notification centre, email, **SMS pickup and return reminders**, notification preferences, SMS quiet hours, and marketing opt-in with unsubscribe (section 7). | ⬜ |
| 19–23 | **Admin dashboard (spec §18).** Overview: total users, active Hosts, active vehicles, upcoming bookings, booking revenue, platform fees, Host payouts, cancellations, incident cases, pending verifications, suspended users and vehicles, with a date-range filter. Capabilities: search and manage users, **approve or reject Host applications**, **approve or reject vehicle listings** (including new photos, new documents and key-detail changes on live listings), view and edit booking status (with calendar override), **suspend users**, **suspend vehicles**, manage disputes and incidents, **issue refunds where authorised**, **payments and payouts** (all payments, failed payments and unpaid extra charges; adding an extra charge from a resolved incident; scheduled, held, paid and failed Host payouts; waiving Host cancellation fees), manage fees and platform settings, manage **homepage content** and destination landing pages, manage **FAQs and help articles**, **platform reports** with CSV export (bookings, revenue, fees, payouts, cancellations, GST summary, built with MongoDB aggregations), **audit log**, staff roles and permissions, failed jobs with retry. Two-factor sign-in for all staff (section 6.1). Booking status edits follow the allowed transitions and run the same side effects, and suspensions follow the rules for bookings, listings and payouts (section 8.2). | ⬜ |
| 22–23 | **Design polish** pass across all screens with real content, on real phones and tablets. Design QA against Figma by the designer. | ⬜ · 👤 Designer |
| 24 | QA pass on the full trip lifecycle (book, pick up, return, review, payout) + **dashboards and operational workflows sign-off**. | ⬜ QA · 👤 Client |

### Phase 4: Testing, Launch & Handover (Days 25–30)
**Delivers:** a live website on the client's domain, the admin guide and technical documentation, and access to the source code and all hosting accounts.

| Day | Tasks | Status |
|---|---|---|
| 25 | Playwright E2E following the **Guest and Host journeys in spec §28** from start to finish (search → book → pay → pickup → return → review; Host account → verification → add vehicle → publish → booking → handover → payout → review), plus cancel and refund, incident report and admin approval, and the edge cases in section 8.2 (verification in review, no-show, late return, missing check-out). | ⬜ |
| 25–26 | **Testing on mobile, tablet and desktop**, on **Chrome, Safari (iOS and macOS) and Edge**, plus Android Chrome. Bug fixes. | ⬜ |
| 26 | **Load test** (k6) on a temporary staging database seeded with 10,000 synthetic vehicles: search stays within the 300 ms budget (section 12.5), and bursts of simultaneous bookings, messages and jobs cause no double-bookings or lost jobs (spec §25, §31). **Security review** (OWASP checklist, NoSQL injection checks, CSRF protection (section 14), headers, rate limits, permission checks for support staff, staff two-factor sign-in, private files served only through signed URLs, what each party can see (section 6.2), `npm audit`; AWS: IAM least privilege, security groups, WAF rules, S3 public access blocked, CloudTrail, root account MFA, ECR image scan results). **Speed optimisation** against section 12.5: **Lighthouse 90+ on mobile**, Core Web Vitals, bundle size, image optimisation, MongoDB index review with `explain()`. 60 fps animation check on a mid-range Android phone, reduced-motion check. | ⬜ |
| 26–27 | **SEO:** unique page titles, meta descriptions, structured data (JSON-LD) for every public route, social sharing previews (OG images), sitemap, robots.txt, **landing pages for Auckland, Wellington, Christchurch, Queenstown and Rotorua**, **Google Search Console**. **Analytics and conversion tracking:** GA4 events for search, listing view, checkout started, booking paid, Host sign-up and listing submitted, sent through one small event helper so ad-platform conversion tags (for example Google Ads) can be added later. Search Results pages `noindex` (section 1.4). Analytics disclosed in the Privacy Policy, with a cookie consent banner if the legal adviser requires one (section 14). | ⬜ |
| 27 | **Email domain setup** (SPF, DKIM, DMARC on the sending subdomain). **Production:** the CDK production stack in the client's AWS account, Route 53 and ACM certificates for the client's domain (**HTTPS**), WAF, CloudWatch alarms and a budget alert, an Atlas cluster in the same region that accepts only the NAT gateway's IP, then the first production deploy and a rollback test. **Automated backups** of the database and of uploaded files (Cloudinary backup) verified with a test restore, **Sentry error monitoring** live. Stripe live mode with production webhooks, and the production domain registered with Stripe for Apple Pay and Google Pay. | ⬜ |
| 27–30 | **Founding Hosts:** the client's first Hosts create accounts and list their cars on production, and support staff approve them through the listing queue, so the site launches with real cars in its featured vehicles, city pages and search (production has no demo cars, section 16 item 16). | 👤 Client · ⬜ Support |
| 28–29 | **User acceptance testing** with the client's team, then bug fixes. | 👤 Client · ⬜ Fixes |
| 29 | **Admin training** session (admins and support staff) + **documentation**: admin guide, technical docs (README with setup, folder map and commands; OpenAPI docs; runbook for deploys and rollbacks, the AWS infrastructure (CDK), backups and failed jobs; data retention). | ⬜ |
| 30 | **Go-live approval** + handover of source code and all hosting accounts. The 30 days of post-launch bug-fix support start. | 👤 Client |

### Task dependencies and critical path
The day ranges above follow these dependencies. A task starts when what it needs is ready, or is built against test data where the table says so.

| Task (days) | Needs first | Unblocks |
|---|---|---|
| Client accounts and decisions (Days 1–2) | – | Staging (AWS, Atlas), email, SMS, uploads, payments and identity checks. Until decisions arrive, launch defaults stay in `platformSettings`. |
| Project setup, CI and AWS staging (Days 1–3) | AWS and Atlas accounts | Every other task; every change is deployed to staging |
| Models, indexes and seed (Days 2–3) | Project setup | All API work. The seed's completed demo trips and reviews let review, rating and trip-history screens be built before Days 21–22. |
| Auth, roles and audit-log middleware; mailer and job queue (Days 3–5) | Models; Resend and Twilio accounts | Host application, sign-in inside checkout, dashboards and admin portal; every email, reminder, expiry and payout job |
| Design sign-off and component library (Days 1–7) | Brand inputs (section 16, item 1) | Every screen from Day 6 |
| Pricing core (Days 6–7) | `shared` package | Estimated totals in search (Days 6–9), quotes and checkout (Days 10–13), earnings (Days 16–19) |
| Search API and autocomplete (Days 6–8) | Models, seed, Google Cloud account, pricing core | Homepage search, Browse Cars and Search Results (Days 7–9) |
| Homepage, Browse Cars, Search Results and listing page (Days 7–10) | Component library, search API, seed data, design sign-off | The entry to checkout, comparing cars, SEO pages (Days 26–27) |
| Vehicle onboarding and approval queue (Days 8–11) | Auth, Cloudinary uploads, components | Real listings to search and book on staging |
| Availability service and holds (Days 10–11) | Models | Quotes, booking holds, the Host calendar, admin overrides |
| Booking flow and payments (Days 11–13) | Pricing, availability, auth, job queue, Stripe account | Cancellations (Days 13–14), dashboards (Days 16–19), booking chat (Days 17–19), handover (Days 19–21) |
| Notifications: `notify()`, email, SMS and in-app (Days 11–13), live over Socket.IO (Days 17–19), centre and preferences (Days 21–22) | Mailer and job queue (Days 3–5), Twilio account | Booking requests reaching Hosts (Day 13 onward), reminders, every later notification |
| Cancellations, refunds and cancellation fees (Days 13–14) | Booking flow and payments | Cancel actions in the dashboards (Days 16–19), Host fee and refund deductions in payouts (Days 17–18) |
| Public pages (Days 12–14) | Layout shell, CMS, FAQs and help articles in the seed, cancellation tiers in settings | Links to the full legal pages from sign-up, checkout and the Host application (acceptance is recorded from Days 3–5 against placeholder versions; the final legal text arrives Day 24 and is accepted again as a new version), SEO (Days 26–27) |
| Guest and Host dashboards (Days 16–19) | Bookings, cancellations, pricing (earnings), availability (Host calendar) | Day 24 sign-off. Payouts, messages, verification and reviews plug in as they are built. |
| Stripe Connect and payouts (Days 17–18) | Bookings | Payouts; the rule that listings go live only after payout setup |
| Messaging and realtime (Days 17–19) | Bookings (one thread per booking), uploads, job queue | Live notifications, reminders in the chat, incident communication (Days 20–21) |
| Identity checks (Days 19–20) | Stripe Identity (or the chosen provider), checkout | The checkout verification step and the Host application identity check |
| Handover (Days 19–21) | Bookings, uploads | Trip completion, extra-kilometre charges (the job is written on Days 17–18 against sample condition reports), reviews, the payout check-in rule |
| Incidents (Days 20–21) | Handover | Extra charges from resolved incidents (using the payment flow from Days 17–18), payout holds |
| Reviews (Days 21–22) | Trip completion | Ratings on cards, listings and profiles; the homepage reviews section |
| Help, support tickets and risk flags (Days 20–22) | Auth, bookings, payments (risk signals), contact form (Days 12–14), verification (licence hashes) | The support inbox in the admin dashboard, fraud monitoring |
| Admin dashboard (Days 19–23) | All the features above | Day 24 sign-off, admin training |
| Design polish (Days 22–23) | Screens built with real content | Day 24 sign-off |
| E2E, security, speed and SEO (Days 25–27) | Features complete on Day 24; final public page content | Production launch |
| Production and go-live (Days 27–30) | Client domain and DNS, legal text (Day 24), insurance details (Day 15), founding Hosts (Days 27–30) | Launch |

**Phase gates**

| Phase | Starts when | Ends with |
|---|---|---|
| 1: Design & Foundation (Days 1–5) | Kick-off; client accounts (Days 1–2) | Design sign-off; staging with sign-up and a working verification email (Day 5) |
| 2: Core Marketplace (Days 6–15) | Design sign-off (Day 5). Mid-fidelity flow designs continue until Day 8, ahead of the screens that use them. | End-to-end test booking signed off on staging (Day 15) |
| 3: Dashboards, Trust & Operations (Days 16–24) | Booking flow and payments working on staging (Day 15) | Dashboards and operational workflows signed off (Day 24) |
| 4: Testing, Launch & Handover (Days 25–30) | Features complete and signed off (Day 24); reviewed legal text and approved page copy (Day 24) | Go-live approval (Day 30) |

**Client inputs during the build:** the GST decision (Day 10) is needed to finish pricing (Days 10–12) and receipts (Days 13–14). The Become a Host estimator assumptions (Day 12) are needed for the Become a Host page (Days 12–14). The insurance details (Day 15) replace the placeholder protection plans and Insurance page text. The reviewed legal text (Day 24) replaces the placeholder legal pages before production (Day 27).

**Critical path:** client accounts → staging → models → auth → pricing core and search → onboarding → availability → booking and payments (Day 15 sign-off) → handover → incidents and reviews → admin (Day 24 sign-off) → testing and production → go-live. A delay on this path moves the launch date. Tasks off the path can move within their phase without moving the launch.

### Not included (future phases)
From MILESTONES.md:
- **Search and notifications:** map-based search and mobile push notifications.
- **Apps:** native iOS and Android apps.
- **Growth features:** dynamic pricing, and referral and loyalty programmes.
- **Business tools:** corporate accounts, fleet tools and advanced fraud detection.

Also from the spec's launch phases (§27): airport automation, third-party API integrations, and advanced analytics beyond the essential reports. Section 10.4 shows how the launch build is ready for each of these, and section 10.5 is the roadmap for building them after launch.

---

## 10. Requirements Coverage

### 10.1 Milestones
Every item in [MILESTONES.md](MILESTONES.md), with the days it is built in (section 9) and the plan section that describes how.

**Phase 1: Design & Foundation (sign-off Day 5)**

| Milestone item | Days | Plan section |
|---|---|---|
| Confirm scope, business rules (fees, cancellation policy, deposits) and third-party accounts | 1–2 | 16 |
| Luxurious brand look and feel: colours, fonts, photography, animation style | 1–4 | 12.1, 12.2, 12.4 |
| Key page designs for mobile and desktop: Home, Search, Vehicle listing, Checkout, Host dashboard | 1–4 | 12.6 |
| Clickable prototype of the key animations (homepage, opening a car, search filters, checkout) | 3–5 | 12.4 |
| Code, database and staging environment | 1–3 | 2, 3, 13 |
| Sign-up, login, email verification and password reset, with emails sent | 3–5 | 6, 7 |
| **You receive:** style guide, page designs, animation prototype; staging link with a working verification email | 5 | |

**Phase 2: Core Marketplace (sign-off Day 15)**

| Milestone item | Days | Plan section |
|---|---|---|
| Homepage with NZ location autocomplete, dates and featured vehicles | 6–9 | 12.6 |
| Search results with vehicle cards and all filters (price, type, seats, EV/hybrid, delivery, instant booking and more) | 6–9 | 3 (location search), 11 |
| Vehicle listing page: photo gallery, specs, WOF and registration info, pricing, sticky mobile Book button | 8–10 | 12.6 |
| Host onboarding in 6 steps: vehicle details, documents, photos, pricing, availability, pickup/delivery | 8–11 | 12.6 |
| Availability calendar that prevents double-booking | 10–11 | 3 (double-booking prevention) |
| Booking flow with a price breakdown in NZD (rental, delivery, service fee, protection, GST) | 10–13 | 5 |
| Secure online payments: cards, Apple Pay, Google Pay | 11–13 | 8 |
| Emails: booking request, booking confirmed, payment receipt, cancellation | 11–14 | 7 |
| Public pages: How It Works, Become a Host, Safety, Insurance, FAQs, About, Contact, legal pages | 12–14 | 1.4 (SEO) |
| Animations: cinematic homepage, page transitions, photo gallery, search filters, booking steps | 6–14 | 12.4 |
| **You receive:** staging site where a test Guest books a car and a test Host lists a car and receives the booking | 15 | |

**Phase 3: Dashboards, Trust & Operations (sign-off Day 24)**

| Milestone item | Days | Plan section |
|---|---|---|
| Guest dashboard: trips (upcoming, current, completed, cancelled), receipts, saved cars, payment methods, licence verification, reviews, notifications | 16–18 | 8 (payment methods), 12.6 |
| Host dashboard: vehicles, bookings, calendar, earnings (today, week, month, lifetime), payouts, document reminders | 16–19 | 4.3, 12.6 |
| Admin dashboard: users, Host and vehicle approvals, bookings, refunds, incidents, fee settings, FAQ and homepage content, basic reports, audit log | 19–23 | 6.2, 11, 12.6 |
| Messaging: chat for each booking, photo attachments, automated pickup and return reminders | 17–19 | 4.3, 4.4 |
| Identity and driver licence verification | 19–20 | 1.2, 6, 14 |
| Digital vehicle handover: timestamped check-in/out photos, odometer and fuel readings, damage notes | 19–21 | 3 (conditionReports), 12.6 |
| Damage and incident reporting with case numbers and full history | 20–21 | 3 (incidents) |
| Two-way reviews and ratings with moderation | 21–22 | 3 (reviews) |
| Design polish on real phones | 22–23 | 12 |
| Host payouts, refunds and cancellation fees | 13–14, 17–18 | 8 |
| Notifications by email and in-app, plus SMS for pickup and return reminders | 21–22 | 7 |
| **You receive:** all three dashboards on staging with the full trip lifecycle (book, pick up, return, review, payout) | 24 | |

**Phase 4: Testing, Launch & Handover (sign-off Day 30)**

| Milestone item | Days | Plan section |
|---|---|---|
| Testing on mobile and desktop, on Chrome, Safari and Edge | 25–26 | 15 |
| Security review | 26 | 14 |
| Speed: Lighthouse 90+ on mobile, smooth animations on mid-range phones | 26 | 12.5 |
| SEO: titles, meta descriptions, structured data, sitemap, 5 destination landing pages, Search Console | 26–27 | 1.4 |
| Analytics and conversion tracking | 26–27 | 1.2 |
| Email domain setup so emails reach inboxes | 27 | 7 |
| Production on the client's domain with HTTPS, automated backups and error monitoring | 27 | 13 |
| UAT with the client's team, then bug fixes | 28–29 | |
| Admin training and written documentation | 29 | |
| **You receive:** live website, admin guide and technical docs, source code and all hosting accounts | 30 | |
| After launch: 30 days of bug-fix support | 31–60 | |

**Client inputs and scope notes**
- **Client inputs** are listed in section 16 with their dates (MILESTONES.md names the main ones, due by Day 1, Day 2, Day 15 and Day 24). Feedback within 24 hours at each review point; any delay moves the launch date by the same number of days.
- **Design scope:** high-fidelity Figma designs cover the key pages. The other flows get mid-fidelity designs and are built directly from the design system (section 12.3).
- **Reports scope:** the admin dashboard has essential reports with CSV export. Advanced analytics come later.
- **Not included:** see the end of section 9 and section 10.4.

### 10.2 Client specification
Every section of the client's website specification, with where it is covered in this plan and when it is built. [Appendix A](#appendix-a-requirement-traceability) breaks this down to every individual requirement.

| Spec section | Where it is covered | Days |
|---|---|---|
| §1 Project overview: two-sided NZ marketplace, original branding, NZD, NZ users and operating requirements | Whole plan; 5, 12.1; 16 (item 1: original brand name, item 14: NZ operating obligations) | 1–30 |
| §2 User types: Guest, Host, Administrator, Support/Operations staff | 6.2 roles and permissions | 3–5, 19–23 |
| §3 Public pages (all 18, including Browse Cars, Search Results, Login, Sign Up and the 5 legal pages) | 1.4, 2.2, section 9 Phase 2 | 3–5, 7–14 |
| §4 Homepage: headline, search module with dates and times, Become a Host call to action, all 8 content sections, footer | Section 9 (Days 7–9), 12.6 | 7–9 |
| §5 Search results and Browse Cars: all card fields and all 16 filters, a "New" label for cars without reviews, search validation and empty and no-results states | 3 (location search, search validation), 5 (estimated totals), section 9 (Days 6–9), 12.6 | 6–9 |
| §6 Vehicle listing: all 9 photo types and all vehicle information, incl. engine/powertrain, rego and WOF (plus CoF and RUC where they apply) | 3 (vehicles; location and plate privacy), section 9 (Days 8–10), 12.6 | 8–10 |
| §7 Booking flow (all 11 steps, with the price and policies reviewed before sign-in) and price breakdown with mandatory and optional charges | 5, 6.1, 8.1, 8.2 (lifecycle, date holds, verification in review), section 9 (Days 11–13), 12.6 | 10–13, 19–20 |
| §8 Guest dashboard (all 13 items, incl. messages and help and support) | 8.2 (trip grouping), section 9 (Days 16–18), 12.6 (incl. the active trip view) | 16–18 |
| §9 Host dashboard (all 10 items, incl. maintenance reminders and Host profile) | 8.2 (trip grouping, deactivation), section 9 (Days 16–19), 4.3, 12.6 | 16–19 |
| §10 Host earnings dashboard (all 8 figures, plus a downloadable GST-ready earnings statement) | Section 9 (Days 16–19), 8.1 (payout timing), 12.6 | 16–19 |
| §11 Host onboarding: all 6 steps and their fields (incl. VIN or chassis number), the minimum photo set, photo quality flags sent for review, custom delivery locations, plus the details the filters and listing page need (body type, powertrain, features, extras, fuel policy, Instant Book, cancellation tier) and the registered owner's consent where needed | 3 (vehicles, validation rules), section 9 (Days 8–11), 12.6 | 8–11 |
| §12 Calendar and availability: month/week views, automatic and manual blocks, recurring availability, notice, buffer, no double-booking, admin override | 3 (key rules), 4.3, 8.2 (date holds) | 10–11 |
| §13 Messaging: threads, photos, automated messages, reminders, email/SMS alerts, report and block, contact details masked until confirmation | 3 (contact details in messages), 4.3, 4.4, 6.2 (who sees what), 7, section 9 (Days 17–19) | 17–19 |
| §13 Push alerts for messages | Not in the 30 days (10.4, roadmap 10.5 R1) | Later |
| §14 Digital handover: check-in and check-out with all listed angles, readings, damage, timestamps, confirmation, new-damage flag | 3 (conditionReports), 8.2 (Host not there, missing check-in or check-out), section 9 (Days 19–21), 12.6 | 19–21 |
| §15 Damage and incident reporting: types, evidence, case number, admin communication with both parties, audit trail | 3 (incidents), section 9 (Days 20–21) | 20–21 |
| §16 Reviews and ratings: all 5 rating categories, written review, moderation rules | 3 (reviews, categories for each direction), 4.3 (`reviews.reveal`), section 9 (Days 21–22) | 21–22 |
| §17 Payments and payouts: cards, Apple Pay, Google Pay, NZD, fees, payouts, full and partial refunds (and who funds them), cancellation fees for Guest and Host cancellations and no-shows, post-trip charges (extra km, charges from resolved incidents), receipts and history, failed payments and failed refunds, card processing fees, Host payout-account problems, admin payments and payouts view | 5, 8.1, 8.2, section 9 (Days 13–14, 17–18, 19–23) | 11–14, 17–18, 19–23 |
| §18 Admin dashboard: all 11 overview figures and all 13 capabilities | 6.2, 8.2 (status edits, suspensions), section 9 (Days 19–23), 12.6 | 19–23 |
| §19 Notifications: email, SMS, in-app, all example notifications, SMS quiet hours, marketing consent | 7 (incl. build order) | 3–5, 11–14, 17–19, 21–22 |
| §19 Push notifications (mobile apps / PWA) | Not in the 30 days (10.4, roadmap 10.5 R1); architecture ready, and the web app manifest ships at launch (Days 3–5) | Later |
| §20 Mobile-first design: responsive phone/tablet/desktop, touch targets, simple navigation, fast photos, sticky mobile CTA, camera upload, fast checkout | 12.2, 12.5, 12.6 | Throughout, 25–26 |
| §21 Search and location: autocomplete, cities, suburbs, destinations, distance, airport search (including cars that deliver to the airport) and delivery options, NZ addresses | 1.2, 3 (location and airport search), section 9 (Days 6–8) | 6–8 |
| §21 Map-based browsing ("future or optional" in the spec) | Not in the 30 days (10.4, roadmap 10.5 R2) | Later |
| §22 Trust, safety and verification: email, mobile, identity, licence, Host identity, vehicle documents, listing moderation (new listings and changes to live listings), secure payments, suspicious-activity monitoring, incident reporting, safety pages | 3 (changes to live listings, location and contact privacy), 6, 8, 14, section 9 (Days 8–11, 19–22) | 3–5, 8–11, 19–22 |
| §23 NZ-specific: NZD, NZ dates and times, NZ addresses, rego and WOF (plus CoF and RUC), NZ licence workflow, GST-ready reports (GST receipts, admin GST summary, Host earnings statements by month or NZ tax year), NZ legal documents, NZ protection info, airport rentals (airport search and delivery) and tourist destinations, local and international visitors (overseas phone numbers, licences and cards), tolls and infringement notices, GST-registered Hosts, NZ marketing-message, fair-trading and privacy rules | 3, 5 (trip days in NZ time, GST and Hosts), 6.1, 7 (marketing messages), 8.1 (receipts, commission invoices), 8.2 (tolls and fines), 12.7, 14, 16, section 9 (Days 6–8, 16–19, 19–23) | Throughout |
| §24 SEO and marketing: indexable vehicle, city and destination pages, unique titles and descriptions, structured data, social previews, analytics, conversion tracking, Search Console, city landing pages | 1.4, section 9 (Days 26–27) | 26–27 |
| §25 Technical and performance: HTTPS, scalable backend (load-tested at 10,000 vehicles), role-based access, secure auth (incl. staff two-factor sign-in and CSRF protection), encryption, private files through signed URLs, automated backups (database and uploaded photos and documents), append-only audit logs, image CDN, speed, error monitoring and uptime checks, API-ready | 3, 6, 12.5, 13, 14, section 9 (Day 26) | Throughout |
| §26 Future mobile apps: shared accounts, bookings, push, Host calendar, photo uploads, messaging, check-in/out | 11.1, roadmap 10.5 R4 | Architecture from Day 1 |
| §27 Suggested launch phases | 10.3, 10.5 | – |
| §28 Guest and Host journeys, including comparing vehicles (result cards and Saved cars) and using the vehicle (active trip view) | Section 9 (Day 25 E2E tests follow both journeys), 12.6 (comparing cars, checkout, active trip) | 6–9, 16–18, 25 |
| §29 Design direction: premium but approachable, original NZ identity, visible trust, photography, transparent pricing, one primary action, mobile first | 12.1, 5 (estimated totals include every mandatory charge) | 1–5, 22–23 |
| §30 Designer/developer deliverables: desktop, mobile and tablet designs, clickable prototype of the key screens and animations, design system, all states, dashboard and flow designs, email templates, developer-ready files | Section 9 (Days 1–8), 12 | 1–8 |
| §31 Questions for the development team | 18 | – |
| §32 Final product goal: low-friction booking for Guests, simple dashboard for Hosts | Whole plan | – |

### 10.3 The spec's launch phases (§27) and this plan

| Spec phase | In the 30-day build | Later |
|---|---|---|
| Phase 1 – MVP | All of it: homepage, search and listings, vehicle pages, Guest and Host accounts, onboarding, calendar, bookings, payments, messaging, reviews, admin dashboard, notifications | – |
| Phase 2 – Trust & Operations | Driver/identity verification, digital inspection, damage reporting, advanced Host earnings, advanced admin tools, refund and dispute workflows | Map search, advanced analytics |
| Phase 3 – Growth | – | Native apps, dynamic pricing, airport automation, fleet tools, referrals, loyalty, corporate accounts, API integrations, advanced fraud detection |

The spec names "advanced Host earnings" and "advanced admin tools" without defining them. This plan reads them as the full earnings dashboard (period comparisons, per-booking breakdown, monthly chart and the GST-ready statement) and the admin tools built on Days 19–23 (payments and payouts view, risk flags, moderation queues, staff roles, audit log, failed-job retry and CSV reports). The client confirms this reading (section 16, item 17). The Later items are planned in section 10.5.

### 10.4 Spec items outside the 30 days, and how the build is ready for them
These are excluded in MILESTONES.md. Adding any of them to the 30 days needs a change request. Section 10.5 describes how each one will be built.

| Item (spec §) | Ready in the launch build |
|---|---|
| Push notifications for mobile apps / PWA (§13, §19, §26) | `notify()` sends through one adapter per channel; a push adapter and device-token storage are added without changing any notification events. The web app manifest ships at launch, so PWA web push then only needs a service worker and the push adapter. |
| Map-based vehicle browsing (§21, §27) | Search already returns coordinates and distances from a `2dsphere` index, so a map view only needs the frontend and a map key |
| Advanced analytics (§27) | All events are in MongoDB and GA4; reports are MongoDB aggregations that can be extended |
| Native iOS and Android apps (§26, §27) | REST API with bearer tokens, shared Zod contracts, Socket.IO (section 11.1) |
| Dynamic pricing, airport automation, fleet tools, referrals, loyalty, corporate accounts, API integrations, advanced fraud detection (§27) | Pricing is one shared function, airports are data, roles and permissions are extensible, and risk flags are already recorded |

### 10.5 Post-launch roadmap
How each spec item outside the 30 days will be built, in a suggested order. Each is quoted and scheduled as a change request after launch. The order follows the spec's launch phases (§27) and the dependencies between items. None of them needs the launch data model or API to be rebuilt: they add collections, fields and endpoints.

| # | Item (spec §) | How it will be built | Needs first |
|---|---|---|---|
| R1 | Push notifications for the PWA (§13, §19) | A service worker with Web Push (VAPID keys, the `web-push` library); a `pushSubscriptions` collection; a `PUSH` channel adapter in `notify()`; an opt-in prompt after the first booking, and push choices in notification preferences. Works in Chrome, Edge, Firefox and on Android, and on iPhone once the site is added to the home screen (iOS 16.4 or later). | The launch notification system (7) and web app manifest (Days 3–5) |
| R2 | Map-based browsing (§21, §27 Phase 2) | A list/map switch on Search Results using the Google Maps JavaScript API, with marker clustering and price pins, and search by the visible map area (a `$geoWithin` box query) in the search API. Cars are pinned at their approximate location only (section 3). | Search API (Days 6–8) |
| R3 | Advanced analytics (§27 Phase 2) | GA4's BigQuery export plus reporting on a read-only Atlas analytics node, in a BI tool such as Looker Studio or Metabase: the booking funnel, conversion by city and channel, Host supply and utilisation, cancellation and incident rates, cohort retention, and scheduled email reports. | GA4 events (Days 26–27), admin reports (Days 19–23) |
| R4 | Native iOS and Android apps (§26, §27 Phase 3) | React Native (Expo), reusing `shared` (Zod contracts, pricing, NZ formatters) and the same REST API with bearer tokens; the Socket.IO client for chat; the native camera with an offline queue for inspections; push through APNs and Firebase Cloud Messaging via the R1 channel adapter; deep links from emails. Published to the App Store and Google Play. | R1, OpenAPI docs (Day 29) |
| R5 | Dynamic pricing (§27 Phase 3) | A per-date price calendar for each car, with suggested prices from demand (searches and occupancy by city), seasons, NZ public and school holidays, events and lead time. Hosts opt in and set a minimum and maximum. The shared pricing function reads the per-date prices, so search, quotes and checkout stay consistent. | R3 (demand data) |
| R6 | Airport automation (§27 Phase 3; the spec does not define it) | Proposed: a flight number at checkout, with arrival tracking from a flight-status API that moves pickup reminders when a flight is late; standard handover instructions and photos for each airport; airport access and parking fees as line items; contactless key handover (key-box codes released at check-in). Confirmed with the client before it is quoted (section 16, item 17). | Airport data (section 3), handover (Days 19–21) |
| R7 | Fleet tools for professional Hosts (§27 Phase 3) | Business Host accounts with team members and roles; bulk edits to prices, availability and trip rules; CSV import of vehicles; one calendar across the fleet; earnings and utilisation for each car. | Roles and permissions (6.2), earnings statements (Days 16–19) |
| R8 | API integrations (§25, §27 Phase 3) | A partner API (API keys, scopes, rate limits and webhooks) from the existing OpenAPI contracts; iCal import and export for Hosts who also list elsewhere; an insurer integration for protection policies (spec §31); an accounting export (for example Xero) for GST reporting; the NZ licence-check and plate-lookup services if they were not connected at launch (section 16, item 15). | OpenAPI docs, availability service (section 3) |
| R9 | Referral programme (§27 Phase 3) | Referral links and codes; credit for both people after the referred person's first completed trip, stored in a credit ledger and applied as a discount line by the pricing function; checks against self-referral (same card, device or address); admin controls. | Pricing line items (5), risk flags (14) |
| R10 | Loyalty programme (§27 Phase 3) | Points or tiers earned from completed trips, redeemed as discounts or lower service fees, using the R9 credit ledger. | R9 |
| R11 | Corporate accounts (§27 Phase 3) | Company profiles with company admins, approved drivers (each verified), central billing by card or monthly invoice, trip purpose and cost codes, and company reports. | Roles (6.2), payments (8) |
| R12 | Advanced fraud detection (§27 Phase 3) | Stripe Radar custom rules, device fingerprinting, a risk score built from the risk flags recorded since launch, automatic holds on high-risk bookings for manual review, and triggers to verify again. | Risk flags and their history (Days 20–22) |
| R13 | Trip changes and extensions (not in the spec; recommended) | The Guest asks for new dates or a later return, the Host approves (automatic for Instant Book when the dates are free), and the price difference is charged or refunded through the same pricing, availability and payment services. Until then, support handles changes (section 8.2). | Booking lifecycle (8.2) |

**Suggested order:** R1–R3 first (the rest of the spec's Phase 2, plus the push channel), then R4 (the apps need R1) and R13, then R5–R8, then R9–R12.

---

## 11. API Overview (REST, `/api/v1`)

```
POST   /auth/signup | /auth/login | /auth/logout | /auth/refresh
POST   /auth/verify-email | /auth/forgot-password | /auth/reset-password | /auth/phone/otp | /auth/phone/verify
GET    /me   PATCH /me   PATCH /me/notification-prefs   POST /me/agreements
POST   /me/password                      (signs out other sessions)   POST /me/mfa/setup | /me/mfa/verify (staff)
POST   /me/privacy-requests              (access or correction request, or account closure: a PRIVACY ticket)
GET    /me/favourites   PUT|DELETE /me/favourites/:vehicleId
GET    /me/payment-methods   POST /me/payment-methods/setup   DELETE /me/payment-methods/:id
GET    /me/payments                      (payment history, receipts, refunds)
POST   /payments/:id/pay                 (pay link: retry a failed payment or pay an extra charge)
POST   /me/verification                  (starts Stripe Identity, saves licence details)
POST   /me/host-application              (Host profile + Host Agreement)
PATCH  /me/host-profile                  (bio, GST registration and number, Host settings)
PUT    /me/last-search                   (place and dates, for estimated totals in Saved cars)
GET    /me/reviews                       (written, received, and trips waiting for a review)
GET    /notifications   POST /notifications/read
GET    /search?lat&lng&radius&start&end&filters...   (dates optional: Browse Cars)
GET    /places/suggest?q=                (our destinations and airports + Google Places)
GET    /places/:placeId                  (coordinates and NZ address for the chosen suggestion, same session token)
GET    /vehicles/featured       (homepage featured vehicles)
GET    /vehicles/:slug          GET /vehicles/:id/availability?from&to   GET /vehicles/:id/reviews
POST   /vehicles/:id/quote      → price breakdown
GET    /host/vehicles           (My Vehicles, with status)
POST   /host/vehicles           PATCH /host/vehicles/:id
POST   /host/vehicles/:id/photos|documents|submit|activate|deactivate
POST   /host/vehicles/:id/blocks          DELETE /host/vehicles/:id/blocks/:blockId
PUT    /host/vehicles/:id/recurring-rules   PUT /host/vehicles/:id/maintenance-reminders
GET    /host/earnings?period=   GET /host/earnings/statement?from&to   (CSV)
GET    /host/payouts            POST /host/connect/onboarding-link
POST   /uploads/signature       (signed Cloudinary upload: vehicle photos and documents, message attachments,
                                 inspection photos, incident evidence, support ticket attachments)
POST   /bookings                GET /bookings?role=guest|host&status=   GET /bookings/:id   GET /bookings/:id/receipt
POST   /bookings/:id/accept|decline|cancel   (cancel also withdraws a PENDING request, section 8.2)
GET    /bookings/:id/cancellation-preview    (refund and fee shown before the user confirms)
GET    /bookings/:id/inspections          POST /bookings/:id/inspections   (check-in / check-out)
POST   /bookings/:id/inspections/:stage/confirm   (the other party reviews and confirms the condition report)
GET    /threads  GET /threads/:id/messages  POST /threads/:id/messages  POST /threads/:id/read
POST   /reports                 (report a user, message, review or vehicle)
POST   /users/:id/block         DELETE /users/:id/block
POST   /reviews                 GET /users/:id/reviews
GET    /users/:id/profile       (public profile: only the fields in section 6.2)
POST   /incidents               GET /incidents   GET /incidents/:ref   POST /incidents/:ref/events
GET    /help/articles           GET /help/articles/:slug
POST   /support/tickets         GET /support/tickets   GET /support/tickets/:ref   (contact form works signed out)
POST   /support/tickets/:ref/messages     (replies from the user or support staff)
GET    /destinations            GET /destinations/:slug   GET /faqs   GET /cms/:key
POST   /payments/webhook        (Stripe payments, Connect, Identity and dispute events)
POST   /webhooks/resend | /webhooks/twilio   (email and SMS delivery status, signature-checked)
/admin/*                        overview, users, host-applications, vehicles (incl. pending listing changes),
                                verifications, bookings, payments, extra-charges, payouts, refunds, incidents,
                                disputes, support, moderation, reviews, risk, settings, cms, destinations, faqs,
                                help, reports, exports (CSV), staff, audit, jobs
```

The OpenAPI spec is generated from the Zod schemas in `shared`, which keeps the API ready for native apps. `sitemap.xml`, `robots.txt` and page meta tags are served by the same Express server outside `/api/v1`.

### 11.1 Ready for native apps (spec §26)

| App feature | How the launch build supports it |
|---|---|
| Shared user accounts | The same auth endpoints; apps send the access token in the `Authorization` header |
| Shared bookings | The same `/bookings` endpoints and business rules |
| Push notifications | `notify()` channel adapters: a push adapter plugs in next to email, SMS and in-app (10.4) |
| Host calendar | `/vehicles/:id/availability`, `/host/vehicles/:id/blocks` and `recurring-rules` |
| Vehicle photo uploads | Signed Cloudinary uploads that work from any client |
| Messaging | REST for history + Socket.IO, which has native iOS and Android clients |
| Digital check-in/check-out | `/bookings/:id/inspections`, with photo timestamps and location from the device |

---

## 12. Design Implementation

### 12.1 Design principles (spec §29)
The site should feel **luxurious, calm and confident**, like a premium travel brand, while staying approachable and easy to use for everyone.

- **Original brand:** completely original branding, wording and UI, inspired by the convenience of established peer-to-peer platforms but not copying any of them (spec §1, §29).
- **Clean NZ identity:** full-bleed NZ landscapes and professionally shot hero cars with consistent colour grading. Host photos appear in consistent 4:3 frames. Photography is a major part of the design.
- **Editorial layout:** generous whitespace, large serif headlines, a strong grid, few elements per screen, and no clutter or excessive text.
- **Restrained colour:** ivory, ink and deep pounamu green, with champagne gold used sparingly for highlights.
- **Quiet, purposeful motion:** every animation helps orientation, gives feedback or adds a moment of delight. Animations are smooth and short and never make the user wait.
- **Speed is part of luxury:** pages appear almost instantly, and every tap responds within 100 ms.
- **Trust visible throughout the booking journey:** verified badges, ratings and trip counts, clear protection information, secure-payment signals and the cancellation policy at each decision point.
- One obvious primary action per screen · pricing always transparent · mobile first, then tablet and desktop.

### 12.2 Design tokens

| Token | Value | Use |
|---|---|---|
| `--color-primary` | `#0E3B32` (deep pounamu green) | Primary buttons, links, brand (white text 12:1 contrast) |
| `--color-primary-hover` | `#0A2C25` | |
| `--color-gold` | `#C8A96A` (champagne gold) | Thin rules, icons, Instant Book badge, highlights on dark or green backgrounds |
| `--color-gold-text` | `#8A6A2E` | Gold text on light backgrounds (passes 4.5:1) |
| `--color-ink` | `#0B1210` | Headings, body text, dark sections (hero overlay, footer) |
| `--color-muted` | `#5E6662` | Secondary text |
| `--color-bg` / `--color-surface` | `#FAF8F4` (ivory) / `#FFFFFF` | Page background / cards |
| `--color-border` | `#E7E2D9` (warm grey) | Hairline borders and dividers |
| `--color-success` / `warning` / `danger` | `#1E8E5A` / `#C98A12` / `#C8372D` | States |
| Font: display | **Fraunces** (variable serif) 400–600, tracking −0.02em | Page and section headlines (32 px and larger) |
| Font: UI and body | **Inter** (variable) 400/500/600, tabular numbers for prices | Everything else. Small uppercase labels use 0.08em tracking. |
| Type scale | 12 · 14 · 16 · 18 · 20 · 24 · 32 · 40 · 56 · 72 (fluid with `clamp()`) | Body text is 16 px minimum on mobile |
| Spacing | 4 px base (4, 8, 12, 16, 24, 32, 48, 64, 96, 128) | Sections: 64 px padding on mobile, 96–128 px on desktop |
| Radius | 10 px inputs and buttons · 16 px cards · 24 px sheets and modals · 999 px chips | |
| Shadows | `card`: `0 1px 2px rgb(11 18 16 / .04), 0 8px 24px rgb(11 18 16 / .06)` · `lift` (hover): `0 2px 4px rgb(11 18 16 / .05), 0 16px 40px rgb(11 18 16 / .10)` | Soft, layered depth instead of heavy borders |
| Glass | ivory at 80% + `backdrop-filter: blur(16px)` | Sticky header and sticky booking bar only (blur is costly on budget phones) |
| Touch target | min 44 × 44 px | Spec §20 |
| Breakpoints | `sm 640` · `md 768` (tablet) · `lg 1024` · `xl 1280` | |

The tokens are defined once in `shared/src/tokens.ts`, which generates these CSS variables and the Tailwind theme. The email templates use the same values (section 2.3).

### 12.3 Component library (built in `client/src/components/ui`)
Button (primary, secondary, ghost, danger; sizes) · Input · Select · DateTimeRangePicker (NZ format, dates and times) · LocationAutocomplete · Checkbox/Switch · Slider (price range) · Badge (Instant Book, Delivery, EV, Airport, Verified) · Rating stars · PhotoGallery/Lightbox · Stepper (onboarding) · FileUpload/CameraCapture · Calendar (month/week) · StatCard · Chart · DataTable · Tabs · Modal/Drawer (mobile bottom sheet) · Toast · EmptyState · Skeleton loaders · Avatar + Verified tick · Timeline (incidents, bookings, tickets) · ReportDialog.

Marketplace-specific components (VehicleCard, PriceBreakdown, StickyBookingBar, ChatBubble, InspectionCamera, DamageDiagram) live in `client/src/features` and are built from the generic components above.

Every component needs the following states: default, hover, focus (visible ring), disabled, loading, error and empty, matching the states board from the design phase. Hover, press, open and close states use the motion building blocks in 12.4.

### 12.4 Motion & animation

**Libraries**
- **Motion** (`motion/react`), loaded through `LazyMotion` with the `domAnimation` feature set to keep the first download small. Used for entrances, gestures (swipe gallery, drag-to-close sheets), layout animations (filter chips, result lists) and `AnimatePresence` for modals, toasts and checkout steps.
- **View Transitions API**, through React Router's `viewTransition` option, for route changes: a soft cross-fade between pages, and the VehicleCard photo morphing into the listing gallery. Browsers without support navigate instantly without the effect.
- **CSS transitions** for simple hover and focus states, with no JavaScript.
- One animation library only: no GSAP and no scroll-hijacking smooth-scroll library. This keeps the bundle small and scrolling native.

**Motion tokens**

| Token | Value | Use |
|---|---|---|
| `--dur-micro` | 120 ms | Hover, press, toggles |
| `--dur-short` | 200 ms | Dropdowns, tooltips, toasts |
| `--dur-medium` | 320 ms | Modals, bottom sheets, page transitions |
| `--dur-long` | 700 ms | Hero and section entrances |
| `--ease-out` | `cubic-bezier(0.22, 1, 0.36, 1)` | Entrances: fast start, soft landing |
| `--ease-in-out` | `cubic-bezier(0.65, 0, 0.35, 1)` | Page transitions, elements moving on screen |
| Spring | stiffness 300, damping 30 | Drag, swipe, sheets, layout changes |
| Stagger | 60 ms between items, first 6 items only | Lists and grids |
| Travel | 16–24 px | Fade-up distance for scroll reveals |

**Signature animations**

| Where | Animation |
|---|---|
| Home hero | Slow zoom on a looping NZ road video (still image on slow connections or with Save-Data on). Headline lines rise in one after another, then the search panel glides up. |
| Scroll sections | Content fades up once as it enters the screen. Destination tiles have a subtle parallax on desktop. |
| VehicleCard | Photo zooms 4% on hover and the card lifts with a deeper shadow. The heart pops with a spring when saved. |
| Card → listing | The card photo morphs into the listing gallery (shared-element view transition). |
| Search | Results re-order smoothly when filters change. Skeletons shimmer while loading. The filter sheet drags down to close. |
| Date picker | The selected range fills across the days, and the total price rolls to the new amount. |
| Vehicle listing | Swipeable gallery with momentum. The full-screen lightbox opens from the tapped photo. The sticky booking bar slides in after the hero. |
| Checkout | Sections expand smoothly. **Confirm and pay** turns into a progress indicator, then an animated tick with a soft gold shimmer. |
| Dashboards | Stat numbers count up, charts draw in, and the active tab indicator slides between tabs. |
| Everywhere | Buttons press to 98% scale, toasts slide in, modals scale up from 96%, and link underlines grow from left to right. |

**Rules**
- Animate only `transform` and `opacity`, which the GPU handles smoothly. Never animate width, height, top or left.
- Motion never blocks the user: content is usable straight away, and entrance animations play once per page per session, not every time the user goes back.
- `prefers-reduced-motion` is respected: movement becomes a simple fade (`<MotionConfig reducedMotion="user">`) and the hero shows a still image.
- Target 60 fps on a mid-range Android phone (e.g. Samsung Galaxy A series), checked in QA.

### 12.5 Speed budget

| Metric (mobile, 75th percentile of real users) | Our target | Google "good" threshold |
|---|---|---|
| Largest Contentful Paint (main content visible) | ≤ 2.0 s | 2.5 s |
| Interaction to Next Paint (response to taps) | ≤ 150 ms | 200 ms |
| Cumulative Layout Shift (content jumping) | ≤ 0.05 | 0.1 |
| Lighthouse performance score (public pages, mobile) | ≥ 90 | – |
| JavaScript on first load (gzipped) | ≤ 170 KB | – |
| Search API response (95th percentile) | ≤ 300 ms | – |

**How we meet it**
- **Code:** route-level code splitting with Vite, with the Host and admin areas loaded only when needed. Motion loads through `LazyMotion`. No heavy libraries on public pages.
- **Instant navigation:** vehicle data is prefetched with TanStack Query when a card comes into view or is hovered, so listings open instantly. Favourites, messages and calendar blocks update optimistically.
- **Images:** Cloudinary serves AVIF/WebP at the right size (`srcset`), with blurred placeholders while loading, `fetchpriority="high"` on the main image, and lazy loading below the fold. The hero video is 2 MB or less, with a poster image. Phone photos are resized in the browser before upload, so uploads are fast on mobile data.
- **Fonts:** two self-hosted variable fonts, subset to Latin, preloaded, with size-matched fallbacks so text does not jump.
- **Layout stability:** skeletons match the final layout, and every image has a fixed aspect ratio.
- **Backend:** MongoDB indexes checked with `explain()`, a 60 s in-memory cache per instance for homepage content, featured vehicles and popular searches, and short CloudFront caching of public GET responses. Availability is always re-checked at quote and booking time. CloudFront serves the hashed JS and CSS from S3 with a 1-year cache, and adds Brotli compression and HTTP/3 (section 13).
- **Enforced in CI:** Lighthouse CI and a bundle size check run on every pull request and fail the build when a budget is broken. Real-user Core Web Vitals are tracked in Sentry.

### 12.6 Key screen layouts (mobile first)

**Navigation** (spec §20, simple navigation): the header holds Browse Cars, How It Works, Become a Host, Help, Log in and Sign up (or the account menu when signed in). On mobile the ☰ menu holds the same links, so the header shows only the logo and Log in; Sign up is in the menu and on the Log in page. Dashboards use the tab bars described below.

**Home (mobile)**
```
┌──────────────────────────┐
│ ☰  Logo          Log in  │
├──────────────────────────┤
│ [ Hero video: NZ road ]  │
│ Rent a car from local    │
│ owners across New Zealand│
│ ┌──────────────────────┐ │
│ │📍 Where are you going?│ │
│ │📅 Pick-up: date, time │ │
│ │📅 Return:  date, time │ │
│ │ [    Search Cars    ] │ │
│ └──────────────────────┘ │
│ Have a car? Earn money   │
│ by sharing it.           │
│ [   Become a Host    ]   │
├──────────────────────────┤
│ Featured vehicles ◀ ▶    │
│ Popular NZ destinations  │
│ How it works (3 steps)   │
│ Host earnings            │
│ Safety & trust           │
│ Customer reviews         │
│ FAQs                     │
│ Footer: legal, support,  │
│ social links             │
└──────────────────────────┘
```

The Become a Host call to action sits directly under the search panel, as in spec §4. Hero text, featured vehicles, destination tiles, the customer reviews shown (picked from published reviews) and footer links are editable by admins (`cmsBlocks`, `destinations`). Only real, published reviews are shown, and the section stays hidden until the threshold in settings is reached. When admins have not picked featured vehicles, the newest highly rated cars are shown.

**Browse Cars and Search Results**: Browse Cars (`/cars`) shows all active vehicles with the same filters and no dates needed; cards show the daily price. Search Results (`/search`) adds the dates, excludes unavailable cars and shows the estimated total. Mobile shows a single column of cards, a sticky summary bar ("Auckland · 12–15 Oct · Filters (3)"), and filters in a bottom sheet. Tablet shows two columns with the filter sheet. Desktop uses a 280 px filter sidebar and a 3-column card grid. The map view is not included (section 10.4).

**Comparing cars** (Guest journey, spec §28): every card shows the same facts in the same place, including the estimated total for the chosen dates, so results can be compared at a glance. The heart adds a car to Saved cars, which shows each saved car's estimated total for the Guest's last searched dates, so the shortlist can be compared before choosing.

**VehicleCard**
```
┌────────────────────────┐
│ [photo 4:3]      ♡     │
│ ⚡Instant  🚚Delivery   │
├────────────────────────┤
│ Toyota RAV4 Hybrid 2022│
│ ★ 4.9 (38 trips)       │
│ Mt Eden · 4.2 km away  │
│ Auto · 5 seats · Hybrid│
│ $89/day   $267 total   │
└────────────────────────┘
```
A car with no reviews yet shows **New** in place of the stars, and its trip count once it has one.

**Vehicle listing**: full-width swipeable gallery → make, model, year and variant, rating and location → Host card with a verified badge, rating and trips → specs grid (transmission, fuel type, engine/powertrain, seats, doors) → features → rego and WOF → policies (fuel, km allowance, cancellation) → pickup/delivery options → protection → reviews → location map. On desktop, a sticky right-hand booking panel. On mobile, a **sticky bottom bar** showing "$89/day · $267 total [Book]".

**Checkout** (single page, collapsible sections, following spec §7). A price summary is on screen from the first step (a right-hand panel on desktop, a collapsible bar above the button on mobile) and updates as the Guest changes options. The Guest therefore reviews the price and policies before signing in, as in the Guest journey (spec §28).
1. Trip: pick-up and return location, date and time (editable), with the availability check (spec §7 steps 1–3)
2. Protection plan: radio cards with a clear summary of cover
3. Trip details, policies and price: fuel, kilometres, cancellation, and the price breakdown calculated by the API (spec §7 steps 4–5)
4. Sign in or create an account (skipped when already signed in)
5. Verification: licence details and identity check, where required (skipped when already verified). If the check needs manual review, the booking continues as a request (section 8.2).
6. Payment: saved cards or a new card with the Stripe Payment Element, Apple Pay and Google Pay buttons
7. Final price breakdown (spec §7): **Mandatory** (rental, with any weekly or monthly discount on its own line, service fee, and protection when the plan is mandatory) and **Optional** (delivery or airport delivery fee, and protection when the Guest chooses a plan), then the GST included in the total on its own line, with the **Total NZD** in bold
8. Guest Agreement checkbox, then **Confirm and pay** (or **Request to book** for a car without Instant Book)

**Host onboarding**: the Host application first (profile, Host Agreement, identity check), then a 6-step progress stepper (vehicle details, documents, photos, pricing, availability, pickup and delivery), one topic per screen, Save & exit on every step, and example photos for each required angle. After submitting, the Host sees what is left before the listing can go live (approval, payout setup; section 8.2).

**Guest and Host dashboards**: sidebar navigation on desktop and tablet, bottom tab bar on mobile (Trips · Messages · Saved · Account for Guests; Today · Vehicles · Calendar · Earnings · Inbox for Hosts). The Guest account area holds receipts, payment methods and history, personal details, licence verification, reviews, notifications and help and support. The Host area holds bookings by status, reviews, reminders, and profile and settings. The Host earnings page shows stat cards (today, week, month with % change vs last month, lifetime), a bar chart of monthly earnings, platform fees, upcoming payouts, a per-booking table and a **Download statement** button (CSV by month or NZ tax year).

**Active trip** (Guest and Host; the spec §28 "use vehicle" step): from check-in to check-out, the trip sits at the top of Trips (Today for Hosts). It shows the return time and place with directions, message and call buttons, the protection summary and excess, **Report an incident**, and emergency help (111, then the roadside assistance number from the protection plan). Near the return time it shows a **Start check-out** button.

**Admin dashboard**: desktop-first sidebar (Overview, Users, Host applications, Vehicles, Verifications, Bookings, Payments & payouts, Refunds, Incidents & disputes, Support, Moderation, Risk, Content, FAQs & help, Settings, Reports, Staff, Audit log, Jobs). Overview shows the KPI stat cards from spec §18. Each list is a filterable data table with a detail drawer, and reports export to CSV. Support staff see only the sections their role allows. The portal is designed for desktop and tablet, and still works on a phone for urgent tasks (tables scroll sideways and details open full screen), so staff can handle a verification or incident away from a desk.

**Inspection (check-in/out)**: a full-screen camera guide per angle (front, rear, driver side, passenger side, wheels, windscreen, interior, dashboard) with an overlay silhouette and a progress count ("Front 1/8"). Then odometer and fuel/battery inputs, damage pins on a car diagram, a review screen, and Guest confirmation. At check-out, each angle shows the check-in photo next to the new one, with a "Flag new damage" button. Each photo is taken with the camera inside the flow and shows its capture time. On a weak signal, photos wait in the browser and upload when the connection returns (shown as "Uploading when back online"), so an inspection is never lost.

**Incident**: case number and status at the top, then a timeline of every event with evidence photos, and a reply box for updates.

### 12.7 Accessibility & content
- WCAG 2.2 AA: contrast of 4.5:1 or more, keyboard navigation, focus rings, labelled inputs, alt text.
- Champagne gold is decorative on light backgrounds because its contrast is too low for text there. Gold text uses `--color-gold-text` or sits on ink or green.
- Reduced-motion preferences are respected (see 12.4). Videos have no sound and can be paused.
- NZ English copy ("kilometres", "licence", "tyres"). Prices shown as `$89` with "NZD" on totals. NZ date and time formats.
- Plain language suitable for local customers and international visitors, including clear guidance on overseas licences, IDPs and approved translations, and phone verification that works with overseas numbers.
- Times are always NZ time, labelled "NZ time" for visitors whose device is set to another time zone. Overseas cards are charged in NZD, and the card issuer converts.

---

## 13. Deployment Architecture (AWS)

The site runs on **AWS in Sydney (`ap-southeast-2`)**, in an AWS account owned by the client, in the same region as the MongoDB Atlas cluster (Atlas on AWS). The whole setup is written as code with **AWS CDK (TypeScript)** in `infra/`, so staging and production are built the same way and either can be rebuilt from scratch.

```
Route 53 (DNS) → CloudFront (HTTPS with ACM certificates, HTTP/3, Brotli) + AWS WAF
                  ├─ /assets/*   → S3 bucket: hashed JS, CSS and fonts from the Vite build, cached for 1 year
                  └─ everything else: pages, /api/v1, /socket.io, sitemap.xml, robots.txt, /healthz
                       → Application Load Balancer (public subnets, accepts traffic from CloudFront only)
                          → ECS Fargate service (private subnets, 2 tasks in 2 Availability Zones)
                             one Docker image: REST API + Socket.IO + page HTML with SEO tags + job runner
                              ├─ outbound traffic through a NAT gateway with a fixed Elastic IP
                              ├─ MongoDB Atlas M10+, AWS Sydney (accepts only the NAT gateway's IP)
                              └─ Stripe, Resend, Twilio, Cloudinary, Google Places, Sentry

Supporting services: ECR (images) · Secrets Manager · CloudWatch (logs, metrics, alarms) · AWS Budgets · CloudTrail
```

**Region:** Sydney is the closest AWS region to NZ that offers every service used here and every Atlas cluster tier. AWS opened an Auckland region (`ap-southeast-6`) in September 2025, and Atlas supports dedicated M10+ clusters there. If the client wants data kept in NZ, production moves to Auckland by changing one CDK setting and creating the Atlas cluster there. Staging would stay in Sydney, because Atlas's smaller Flex tier is not offered in Auckland. The region is confirmed by Day 2 (section 16, item 6).

### 13.1 AWS services

| Service | Setup | Why |
|---|---|---|
| **ECS on Fargate** | Production: 2 tasks (0.5 vCPU, 1 GB each) in 2 Availability Zones, auto scaling on CPU up to 6 tasks. Staging: 1 task. | Runs the container with no servers to manage or patch. 2 tasks give zero-downtime deploys and keep the site up if one Availability Zone fails. |
| **Application Load Balancer** | One load balancer shared by staging and production (host-based rules, one target group each). WebSockets, health check on `/healthz`, cookie stickiness, 30 s deregistration delay. | Spreads traffic over the tasks, holds long-lived Socket.IO connections, and removes unhealthy tasks |
| **CloudFront + AWS WAF** | One distribution per environment. WAF uses the AWS managed common and known-bad-input rules plus a per-IP rate limit. | HTTPS, HTTP/3 and Brotli from edge locations close to NZ users, caching, and a first filter against attacks before traffic reaches the app |
| **S3** | Private bucket for the hashed build assets, readable only by CloudFront (origin access control). Files from older releases are kept for 30 days. | Assets load fast from the edge, and pages opened before a deploy keep working (13.3) |
| **VPC** | 2 Availability Zones: public subnets for the load balancer and NAT gateway, private subnets for the tasks. One NAT gateway at launch. | The tasks cannot be reached from the internet, and all their outbound traffic leaves from one fixed IP that Atlas allows |
| **Route 53 + ACM** | DNS for the client's domain. Free TLS certificates that renew automatically (CloudFront's in `us-east-1`, the load balancer's in Sydney). | HTTPS everywhere, with nothing to renew by hand |
| **ECR** | One image repository. Images are tagged with the git commit, scanned on push, and the last 20 are kept. | Every deploy and rollback uses an exact, known image |
| **Secrets Manager** | One secret per environment: Atlas connection string, JWT and encryption keys, and the Stripe, Resend, Twilio, Cloudinary, Google and Sentry keys | Secrets never sit in git or in the image. ECS passes them to the container as environment variables when it starts. |
| **CloudWatch** | Container logs (Pino JSON, kept 30 days). Alarms for 5xx errors, unhealthy tasks, high CPU or memory, and failed jobs (from a log metric filter), emailed to the team through SNS. | Logs and alerts alongside Sentry's error tracking |
| **IAM** | Task roles with only the permissions they need. GitHub Actions deploys through OpenID Connect (OIDC) with a short-lived role, so no AWS keys are stored in GitHub. | Least privilege (section 14) |
| **AWS Budgets + CloudTrail** | A monthly budget alert. CloudTrail records every change made to the AWS account. | Cost control and an audit trail for the infrastructure |

### 13.2 Container image
- A multi-stage `Dockerfile` at the repo root. The build stage (`node:22-bookworm-slim`) runs `npm ci` and `npm run build`: Vite builds `client/dist`, and tsup builds the server and its `seed` and `sync-indexes` scripts into `server/dist`. The runtime stage contains only `server/dist`, `client/dist` and the server's production dependencies, runs as the non-root `node` user, and starts with `node server/dist/server.js`.
- `.dockerignore` keeps `node_modules`, `.env` files, `e2e`, `infra` and `.git` out of the build.
- One image runs everywhere. Staging and production differ only in their environment variables and secrets, and `docker build` reproduces production on a developer's machine. Because the app is a standard container, it can move to another container host later without code changes.

### 13.3 Deploys (GitHub Actions)
- **Pipeline:** `npm ci` (npm cache) → lint, typecheck, test (API and services against an in-memory MongoDB replica set) → build → Lighthouse CI and bundle size check → Playwright E2E → build the Docker image and push it to ECR → upload the new hashed assets to S3 (nothing is deleted) → run the **index sync as a one-off ECS task** with the new image → update the ECS service to the new version → wait until the service is stable → smoke test (`/healthz`, the homepage and a search request).
- **The pipeline never connects to the database.** The index sync runs inside the private subnets, so it reaches Atlas through the NAT gateway's allowed IP. The pipeline waits for it and stops if it fails.
- **Environments:** `local` → `staging` (auto-deploy from `develop`, on `staging.<domain>`) → `production` (deploy from `main` after a manual approval in GitHub). At launch both run in the same AWS account, with separate ECS services, target groups, secrets, log groups and IAM roles. Each environment has its own Atlas project, database and credentials.
- **Rolling deploys:** ECS starts the new tasks, waits until they pass the load balancer health check, then drains the old ones (minimum healthy 100%). The ECS deployment circuit breaker rolls back automatically if the new version never becomes healthy. A manual rollback redeploys the previous image tag.
- **Pages opened before a deploy keep working:** old hashed assets stay in S3, so a visitor who loaded the site before a release can still open lazy-loaded pages. If a file is missing anyway, the client catches Vite's `vite:preloadError` event and reloads the page once.
- **Seed data** is loaded the same way, as a one-off task. Staging gets the full seed. Production is seeded once with reference data only (NZ cities, airports, destinations, FAQs, help articles, default settings), without demo users or vehicles.
- **Infrastructure changes** are reviewed with `cdk diff` in the pull request and applied with `cdk deploy`.

### 13.4 App settings that AWS needs
- **Health check:** `GET /healthz` returns 200 when the app is running and connected to MongoDB. It returns 503 when it is not, or while the app is shutting down. The load balancer, ECS and the smoke test all use it.
- **Graceful shutdown:** ECS sends `SIGTERM` when it replaces or removes a task. The app then fails its health check so no new traffic arrives, stops the job runner from claiming jobs and puts any unfinished job back in the queue, closes Socket.IO connections (browsers reconnect to the other task on their own), finishes the requests in progress, and disconnects from MongoDB. The ECS stop timeout is 60 s.
- **Real visitor IP:** Express runs behind two proxies (CloudFront, then the load balancer), so `trust proxy` is set to `2`. Rate limits, risk flags, audit logs and agreement records then see the visitor's IP, not the load balancer's.
- **Socket.IO over 2 tasks:** the client connects with `transports: ['websocket', 'polling']`, so it uses a WebSocket and falls back to HTTP long-polling only on networks that block WebSockets. Load balancer stickiness keeps a long-polling client on one task. CloudFront forwards all cookies and headers on `/socket.io/*` without caching, with a 60 s origin timeout. The MongoDB adapter (section 4.4) delivers events across tasks.
- **Only CloudFront reaches the app:** the load balancer's security group accepts CloudFront's IP ranges only, and its listener also requires a secret header that CloudFront adds, so nobody can get around WAF by calling the load balancer directly. The tasks accept traffic from the load balancer only.
- **Caching rules:** hashed assets are cached for a year. Public page HTML and public GET API responses that are the same for every visitor (featured vehicles, destinations, FAQs, CMS content) send `Cache-Control: public, s-maxage=60`. Anything tied to a user (account, bookings, messages, notifications) sends `private, no-store`. Cookies are not part of CloudFront's cache key, so only responses that do not depend on the user may be cached.
- **WAF exceptions:** the managed rule that blocks request bodies over 8 KB is set to count-only, and the webhook paths (Stripe, Resend and Twilio) are excluded from the rate limit, so large Stripe events and bursts of delivery reports are never blocked. Photo and document uploads go directly to Cloudinary, not through AWS.

### 13.5 Backups and monitoring
- **Backups:** Atlas daily snapshots retained for 30 days, plus continuous backup for point-in-time recovery. Uploaded files (vehicle photos, vehicle documents, inspection photos, incident evidence, message attachments) are not in the database, so Cloudinary's automatic backup is turned on to keep a backup copy of every upload; a file deleted or overwritten by mistake can be restored. A restore of both the database and a sample of files is tested before launch. The AWS setup itself needs no backup: it is rebuilt from the CDK code, and the images are in ECR.
- **Monitoring:** Sentry for errors and real-user Core Web Vitals, and CloudWatch for logs, metrics and alarms (13.1). A Route 53 health check calls the homepage and `/healthz` from several locations and alerts the team through SNS if the site cannot be reached from outside AWS.

### 13.6 Scale path from 100 to 10,000+ vehicles
- ECS auto scaling adds app tasks under load.
- The job runner moves to its own ECS service from the same image (`RUN_JOBS=true` on it, `false` on the web tasks).
- A second NAT gateway in the other Availability Zone, and Atlas PrivateLink so database traffic stays inside AWS.
- Raise the Atlas cluster tier (auto-scaling), send search and reporting reads to secondary nodes, and add Atlas Search for heavy filtering.
- Separate AWS accounts for staging and production under AWS Organizations.
- Before launch, a k6 load test with 10,000 synthetic vehicles checks the search budget and concurrent bookings at that size (section 9, Day 26), so the first of these steps is taken on evidence, not guesswork.

### 13.7 Estimated monthly running cost at launch

| Item | NZD a month (approx.) |
|---|---|
| ECS Fargate: 2 production tasks + 1 staging task (0.5 vCPU, 1 GB each) | 110 |
| Application Load Balancer (shared by staging and production) | 40 |
| NAT gateway, public IP addresses and data transfer | 100 |
| CloudFront, WAF, S3, Route 53, ECR, Secrets Manager, CloudWatch | 40–70 |
| MongoDB Atlas: M10 production cluster with continuous backup + Flex staging cluster | 150–200 |
| Resend (email) + Sentry (monitoring) | 80 |
| **Total** | **about 520–600** |

Rough figures at about NZD 1.70 per USD, Sydney prices, before GST, plus usage-based Stripe, SMS, Maps and Cloudinary fees. Confirm them with the AWS Pricing Calculator before quoting them to the client. AWS costs more than a simple hosting platform would, mainly for the load balancer and NAT gateway. In exchange, the app runs in the same region as the database, with WAF, private networking and automatic rollback. To save money, the staging task can be stopped outside working hours, and a Compute Savings Plan lowers the Fargate cost once usage is steady.

---

## 14. Security & Compliance Checklist
- HTTPS everywhere (ACM certificates on CloudFront and the load balancer), HSTS, Helmet security headers, strict CORS. AWS WAF on CloudFront with managed rule sets and a per-IP rate limit (section 13.4).
- Zod validation on every input. Mongoose `sanitizeFilter` and `strictQuery` block NoSQL operator injection (for example, a login body of `{ "email": { "$ne": null } }`).
- **Cross-site request forgery:** auth cookies are `SameSite=Lax`, and every request that changes data must carry the site's own `Origin` header. The mobile apps use bearer tokens, not cookies, so the check does not affect them.
- **Dependencies:** Dependabot security updates and `npm audit` in CI; ECR scans every image (section 13.1).
- Role and permission checks on every API route; support staff get only the permissions they need, and each party sees only the other's details it needs (section 6.2).
- **Staff two-factor sign-in:** every admin and support account uses an authenticator app (section 6.1).
- **Private files:** vehicle documents, inspection photos, incident evidence, and message and ticket attachments are private and served only through short-lived signed URLs (section 3).
- MongoDB Atlas accepts connections only from the NAT gateway's fixed IP (the address all app traffic leaves AWS from), with a separate database user per environment and only the permissions it needs. CI never connects to the database directly (section 13.3).
- **AWS account:** the root user is protected with MFA and not used day to day; the team signs in through IAM Identity Center; GitHub Actions deploys with a short-lived OIDC role, so no AWS keys are stored anywhere; ECS task roles have least privilege; the tasks sit in private subnets and accept traffic only from the load balancer, which accepts only CloudFront; S3 public access is blocked; CloudTrail is on; ECR scans every image.
- Licence and ID documents stored privately and accessed through signed URLs only. Sensitive fields such as licence numbers are encrypted in the application (AES-256-GCM) before they are saved. Atlas also encrypts all data at rest.
- PCI scope is kept minimal: card data never reaches our servers (Stripe Elements). 3-D Secure where the card requires it.
- The client cannot import server code (section 2.3), and only `VITE_` variables reach the browser, so secrets cannot end up in the browser bundle. Server secrets live in AWS Secrets Manager, never in git or the Docker image.
- **Fraud and suspicious activity:** email, mobile, identity and licence verification; Stripe Radar; risk flags for booking velocity, repeated failed payments, card country different from the account, repeated user reports and repeated Host cancellations, reviewed by admins. Advanced fraud detection comes later (spec §27).
- **Messaging safety:** report and block, rate limits on messages, and all messages kept on the platform as evidence for incidents.
- **Legal records:** acceptance of each version of the Terms, Privacy Policy and Host and Guest Agreements is stored with time and IP.
- **NZ Privacy Act 2020:** privacy policy; requests to access or correct personal information, handled as PRIVACY support tickets and answered within 20 working days; account closure with anonymisation (section 8.2); and a breach process that notifies the Privacy Commissioner and affected people of notifiable privacy breaches. The Privacy Policy names the service providers that hold personal information (AWS and MongoDB Atlas in Sydney, Stripe, Resend, Twilio, Cloudinary, Google, Sentry) and the countries where they store it.
- **Marketing messages (NZ Unsolicited Electronic Messages Act 2007):** promotional email and SMS only with consent, with the sender identified and a working unsubscribe (section 7).
- **Prices and earnings claims (NZ Fair Trading Act 1986):** every advertised price and estimated total includes all mandatory charges (section 5). The Become a Host estimator is labelled as an estimate and states its assumptions, which the client approves (section 16, item 18).
- **Audit trails:** audit logs and incident events are append-only (section 3) and kept for the retention period below.
- **Analytics and cookies:** GA4 and any conversion tags are disclosed in the Privacy Policy. A cookie consent banner (with Google Consent Mode) is added if the client's legal adviser requires one, for example for visitors from overseas.
- **Data retention (proposed, to be confirmed by the client's legal adviser):** bookings and financial records for 7 years (NZ tax requirement); ID images deleted 90 days after verification, including their backup copies, keeping only the result; inspection photos, messages and incident records for 2 years after the trip, or longer while a case is open; audit logs for 7 years, like the financial records; deleted accounts anonymised except for records the law requires. The periods are stored in `platformSettings` and applied by the `daily.dataRetention` job.
- Audit log for all admin and support actions.
- Legal, insurance and eligibility rules must be confirmed by the client's advisers before launch (spec §22).

---

## 15. Definition of Done (per feature)
1. It matches the approved design on mobile, tablet and desktop.
2. Loading, empty and error states are implemented.
3. It has API validation and role and permission checks.
4. It has unit or API tests, with an E2E test for critical paths.
5. It is accessible: keyboard and screen reader checked.
6. Emails and notifications are triggered where relevant.
7. Its animations follow the motion system, respect reduced motion, and run smoothly on a mid-range phone.
8. It stays within the speed budget (section 12.5).
9. Its code sits in the right folder (`client`, `server` or `shared`) and passes lint, typecheck and tests in CI.
10. It works in Chrome, Safari and Edge, is deployed to staging and is verified by QA.
11. It handles the validation rules and edge cases that apply to it (section 3 and section 8.2), with tests for them.

---

## 16. Open Decisions & Client Inputs
| # | Decision or input | Needed by |
|---|---|---|
| 1 | Business name, logo, domain name, and access to the domain's DNS settings (to point it at Route 53). The name must be original (spec §1, §29): the working name "DriveShare" is already used by a US peer-to-peer car rental service, so it is a placeholder only. Check the chosen name with IPONZ (trade marks), the Companies Office and domain availability. | Day 1 |
| 2 | Guest service fee % and Host commission %, and who absorbs Stripe's card processing fees (the platform by default, section 5) | Day 1 |
| 3 | Cancellation policy tiers, and whether Hosts choose a tier for each listing or one policy applies to every car; the fee (if any) when a Host cancels a confirmed booking; how a kept Guest cancellation fee is shared with the Host (section 5); how Guest and Host no-shows are treated; and whether unused days are refunded on an early return (default: no, section 8.2) | Day 1 |
| 4 | Security deposit (yes/no, amount) | Day 1 |
| 5 | Driver eligibility: minimum age, licence classes accepted (NZ full/restricted, overseas, with an IDP or approved translation when not in English), years held, which Guests must complete identity verification before booking, and when email and mobile verification are required (default in section 6.1) | Day 1 |
| 6 | Hosting and third-party accounts: an AWS account owned by the client (the team gets access through IAM Identity Center) and the hosting region (Sydney by default, or Auckland to keep data in NZ, section 13); Stripe (payments), Resend (email), Twilio (SMS), plus Cloudinary, Google Cloud and MongoDB Atlas | Day 2 |
| 7 | GST treatment of the rental amount, platform fees and Host payouts (client's accountant), including how GST-registered and unregistered Hosts are treated, and the form of the commission tax invoice for GST-registered Hosts (sections 5 and 8.1) | Day 10 |
| 8 | Support contact details and social media links for the footer and Contact page | Day 12 |
| 9 | Insurance and protection details from the insurance partner, including the roadside assistance number shown on the Safety page, the active trip view and the incident forms | Day 15 |
| 10 | Review moderation rules and prohibited content (what gets a review held or hidden); the review window and whether both reviews are revealed together; the number of published reviews before the homepage shows them; the damage-report window after check-out and the late-return grace period | Day 19 |
| 11 | Legal text (Terms & Conditions, Privacy Policy, Cancellation Policy, Host and Guest Agreements), reviewed by the client's legal adviser, plus data retention periods | Day 24 |
| 12 | Confirmation from the client's legal, insurance and compliance advisers of the final verification, insurance/protection and eligibility rules (spec §22). Any changes are applied in `platformSettings` without code changes. | Day 28 |
| 13 | Items the spec does not define or leaves open: messaging before a booking request (enquiries), changes and extensions to confirmed trips, additional drivers, how a security deposit hold would work (item 4), and when a VIN or chassis number is required (spec §11 says "where required"). Launch defaults: messaging opens with a booking or request, support handles trip changes (section 8.2), no additional drivers, and one of VIN or chassis number is required for every car (a setting). | Day 1 |
| 14 | NZ operating and compliance obligations (spec §1, §22), confirmed by the client's legal and compliance advisers: whether the platform or its Hosts need a rental service (transport service) licence; the WOF or CoF inspection rules for cars rented to the public; Road User Charges for diesel, EV and PHEV cars; and how tolls and traffic or parking infringement notices are passed to the Guest (section 8.2) | Day 10 |
| 15 | Optional NZ data services: automatic NZ driver licence checks (for example NZTA's Driver Licence Verification Service through an approved provider) and number-plate lookup (for example CarJam) to fill in vehicle details and flag stolen cars. Both are paid, and sign-up can take time; until they are connected, support staff check licences and documents by hand. | Day 1 (decide) |
| 16 | Founding Hosts and their cars, listed on production before go-live (production has no demo cars), so the featured vehicles, city pages and search have real content | Days 27–30 |
| 17 | Confirmation of how the plan reads the spec's undefined items: "advanced Host earnings" and "advanced admin tools" (section 10.3), and "airport automation" (roadmap 10.5, R6) | Day 24 |
| 18 | Page copy and figures: About Us, How It Works, Safety, Become a Host and the homepage sections, FAQs and help articles (drafted by the team from the spec, approved by the client), and the assumptions behind the Become a Host earnings estimator (typical daily prices and booked days by city) | Day 12 (estimator assumptions); Day 24 (approved copy) |
| 19 | Languages for international visitors (spec §23): English only at launch, written in plain language (section 12.7). Other languages would be a change request. | Day 1 |

## 17. Risks & Mitigations
| Risk | Mitigation |
|---|---|
| Insurance details arrive on Day 15, after the booking flow is built (Days 10–13) | Protection plans are config-driven: placeholder plans are used until then and replaced from the admin settings without code changes |
| Scope creep from items outside the 30 days (push notifications, map search, advanced analytics, growth features) | They are listed in section 10.4 with how the build is ready for them. Changes go through a change request. |
| 30-day timeline slips | Parallel frontend and backend work, a ready-made UI kit, a simple stack with one deployable service, 24 h client feedback, and a daily stand-up with a scope check |
| AWS takes more setup than a one-click hosting platform | The whole setup is AWS CDK code, built and proven on staging on Days 2–3. Production is the same stack with different settings. |
| A deploy breaks pages already open in a visitor's browser | Rolling deploys with health checks and automatic rollback, old assets kept in S3, and one page reload if a file is missing (section 13.3) |
| AWS costs higher than planned | Budget alert from day one, small tasks that scale up only under load, staging stopped outside working hours, and a Compute Savings Plan once usage is steady (section 13.7) |
| No Redis: background jobs and realtime depend on MongoDB | Atomic job claims, unique job keys, crash recovery for stuck jobs, retries with backoff, a failed-jobs view with retry in the admin dashboard, and tests for the job runner. The runner can move to its own process when volume grows. |
| No migration tool: schema changes on live data | Additive changes only, with defaults in the Mongoose schemas; no renamed fields; each release is checked on staging with production-like data before it goes live |
| Email going to spam | Dedicated subdomain, SPF/DKIM/DMARC, warm-up period, monitored bounce rate |
| Double-bookings (MongoDB has no range-overlap constraint) | One calendar write path, run in a transaction that locks the vehicle document, plus an automated concurrent-booking test |
| Search engines index a client-rendered React app less reliably | Server-injected meta tags and JSON-LD on every public route, sitemap, Search Console monitoring from launch. A prerendering service (e.g. Prerender.io) as a fallback. |
| Payment disputes and damage claims | Timestamped inspections with side-by-side comparison, incident history, messages kept on the platform, agreement records, payouts held while incidents or disputes are open |
| Guests unable to pay extra charges after the trip | Card saved with consent at checkout, pay link and retries on failure, support alerted, unpaid charges visible in the admin dashboard |
| Rich visuals and animations slow the site on budget phones | Speed budget enforced in CI, `transform`/`opacity`-only animations, one animation library, still-image fallback for the hero video, testing on a mid-range Android phone |
| Luxury-level polish is hard to fit into 30 days | A fixed list of signature animations agreed at design sign-off. Everything else reuses shared motion building blocks. Senior designer on the team and a dedicated polish pass on Days 22–23. |
| Host-uploaded photos vary in quality and weaken the premium look | Consistent 4:3 frames, a photo guide with example shots in onboarding, automatic quality flags, admin approval of listing photos. Cinematic brand imagery carries the marketing pages. |
| Stripe Identity may not be offered for the client's NZ Stripe account, or may not read every overseas licence | Checked on Days 1–2. Verification sits behind one interface, so an NZ identity provider or licence-check service (section 16, item 15) can replace it, and the manual review queue covers anything the automatic check cannot decide. |
| The marketplace launches with too few cars | Founding Hosts list on production before go-live (section 16, item 16). City pages and searches with no cars show a helpful empty state with a Become a Host prompt, and the homepage reviews section stays hidden until there are enough real reviews. |
| Earnings or price claims seen as misleading (NZ Fair Trading Act) | Every total includes all mandatory charges; the Become a Host estimator states its assumptions, is labelled as an estimate, and uses figures the client approves (section 16, item 18) |
| Search or booking slows down as listings grow | Indexed location search, a 300 ms search budget, and a load test at 10,000 vehicles before launch (section 9, Day 26) |
| Weak mobile signal where cars are handed over | Inspection photos wait in the browser, keep their capture time and upload when the signal returns (section 12.6); the check-in and return jobs alert support if a handover is not recorded (section 8.2) |

---

## 18. Answers to the Client's Questions (spec §31)

| Question | Answer | Details |
|---|---|---|
| What technology stack do you recommend and why? | React + Vite (TypeScript) for the website, Node.js + Express for the API, MongoDB Atlas for data. One language across the whole product, a fast and animated UI, flexible documents with built-in location search and transactions, one service to deploy on AWS in Sydney, and an API that native apps can reuse. | 1, 2, 13 |
| What payment provider will be used? | Stripe: NZD, cards, Apple Pay and Google Pay, Stripe Connect for Host payouts, Stripe Identity for verification, Radar for fraud screening. | 8 |
| How will driver and identity verification be implemented? | Stripe Identity checks an ID document and a selfie. Guests are asked to use their driver licence as the ID document where Stripe Identity accepts it, so one check covers both, and the licence number read from it is matched with the one entered. The Guest enters licence number, version, class and expiry (or an overseas licence, with an IDP or approved translation if it is not in English), checked against the eligibility rules. Support staff review anything the automatic check cannot decide. Hosts complete identity verification in their Host application. If the client chooses, NZ licences are also checked automatically through an approved NZ licence-check provider (section 16, item 15). If a check needs manual review during checkout, the booking waits as a request until support decides (section 8.2). | 6, 8.2, 9 (Days 19–20), 16 |
| How will vehicle data be verified? | Hosts enter rego, VIN (or the chassis number for imports that have no VIN), WOF and rego expiry, and upload registration, WOF and insurance documents. Support staff check the documents against the details before approving the listing, including that the Host is the registered owner or has the owner's written consent. New photos and documents on a live listing are checked before they appear, and a change to the rego, VIN, chassis number or vehicle details sends the listing back for review. Listings are hidden from search when the WOF or rego expires, and Hosts get reminders before that. CoF and Road User Charges details are recorded where they apply. Optionally, a plate-lookup service fills in the make, model, year and WOF and rego expiry from the plate and flags stolen cars (section 16, item 15). | 3, 4.3, 16 |
| How will insurance/protection information be integrated? | Protection plans (price, cover summary, excess, mandatory or optional) are set in the admin settings, shown at checkout and on the Insurance / Protection page, and saved on each booking. Details come from the client's insurance partner by Day 15. If the partner offers an API, it can be connected later; until then, bookings can be exported for the insurer. | 5, 16 |
| How will host payouts work? | Hosts connect their NZ bank account through Stripe Connect Express. 24 hours after each trip starts, once check-in shows the trip went ahead, the Host's share (rental and delivery less commission) is transferred automatically, and Stripe pays it into their bank on its payout schedule. Payouts are held while an incident or card dispute is open. A listing goes live only after the Host's payout setup is complete. | 8 |
| How will cancellations and refunds work? | Cancellation tiers set by the client (e.g. Flexible, Moderate, Strict) with a short grace period. The system calculates the refund and cancellation fee and refunds the card through Stripe, fully or partly. If the Host cancels, the Guest is refunded in full, and any Host cancellation fee is deducted from the Host's next payout. Admins, and support staff with permission, can issue manual refunds. Every refund is logged, with who funds it. No-shows follow the same rules, and the Host receives their share of a kept Guest cancellation fee. | 5, 8 |
| How will fraud prevention be handled? | Email, mobile, identity and licence verification; Stripe Radar and 3-D Secure; risk flags (booking velocity, failed payments, card country mismatch, repeated reports) reviewed by admins; listing moderation; two-factor sign-in for staff; audit logs. Advanced fraud detection is a later phase (roadmap 10.5, R12). | 14 |
| What data will be stored and for how long? | Accounts, verification results, vehicles and documents, bookings, payments, messages, inspection photos, incidents, reviews and audit logs, all in MongoDB Atlas (Sydney) and Cloudinary; application logs in AWS CloudWatch (Sydney) for 30 days. Proposed retention: financial records and audit logs 7 years, ID images 90 days after verification, inspection photos, messages and incidents 2 years after the trip. To be confirmed by the client's legal adviser. Private files are served only through short-lived signed URLs. The Privacy Policy lists these providers and where they store data. | 3, 14 |
| How will the platform scale from 100 to 10,000+ vehicles? | Indexed location search, a larger Atlas tier, more app tasks through ECS auto scaling, the job runner as its own service, read replicas for search and reports, and Atlas Search for heavy filtering, all without rebuilding. A load test with 10,000 synthetic vehicles is run before launch to confirm it. | 13, 9 (Day 26) |
| What is included in the initial MVP? | All of the spec's Phase 1 (MVP) and most of Phase 2 (verification, inspections, damage reporting, advanced earnings and admin tools, refund and dispute workflows) in 30 days. The later items have a roadmap in section 10.5. | 10.3, 10.5 |
| What third-party integrations are required? | Stripe (payments, Connect, Identity, Radar), Resend (email), Twilio (SMS), Cloudinary (images), Google Places (location search) and the Maps Static API (listing map), MongoDB Atlas (database), Sentry (errors), GA4 and Search Console (analytics and SEO), AWS (hosting: ECS Fargate, load balancer, CloudFront CDN and WAF, Route 53 DNS). Optional at launch: an NZ licence-check service and a plate-lookup service (section 16, item 15). Later: Web Push, the Google Maps JavaScript API for map search, and APNs and Firebase for app push (section 10.5). | 1.2, 10.5, 13 |
| What ongoing hosting, maintenance and support costs should be expected? | About NZD $520–600 a month at launch for AWS hosting, MongoDB Atlas, email and monitoring (section 13.7), plus usage-based Stripe, SMS, Maps and Cloudinary fees. 30 days of bug-fix support are included after launch; ongoing maintenance and support after that is quoted separately. | 13 |

---

## Appendix A: Requirement Traceability

Every requirement in [project_requirements.md](project_requirements.md), bullet by bullet and step by step, with where this plan covers it and when it is built. IDs follow the spec's section numbers (for example, 5.F3 is the third search filter in spec §5). Requirements with the same coverage share a row, and each one is still named. "Later" marks items scheduled after the 30-day build, with their roadmap entry in section 10.5. This table is the basis of the requirement-by-requirement cross-check recorded in section 9 (Progress).

| ID | Requirement (spec) | Covered in (plan section) | When |
|---|---|---|---|
| **§1** | **Project overview** | | |
| 1.1 | Two-sided marketplace connecting Hosts and Guests across NZ | Whole plan; 6.2 | Days 1–30 |
| 1.2 | Simple, modern, mobile-first experience | 12.1, 12.4, 12.6 | Days 1–5, throughout |
| 1.3 | Inspired by established peer-to-peer platforms, with completely original branding, wording and visual design | 12.1; 16 item 1 | Days 1–5 |
| 1.4 | Hosts list vehicles and set pricing, availability and pickup/delivery options | 3 (`vehicles`); 9 Days 8–11 | Days 8–11 |
| 1.5 | Guests search, compare, book and pay online | 9 Days 6–13; 12.6 (Comparing cars, Checkout) | Days 6–13 |
| 1.6 | Platform manages verification, bookings, payments, messaging, handover records, reviews and support | 6, 8, 9 (Phases 2–3) | Days 8–23 |
| 1.7 | Consumer prices default to NZD; designed around NZ users and operating requirements | 3 (Key rules), 5, 12.7; 16 item 14 | Throughout |
| **§2** | **Primary user types** | | |
| 2.1 | Guest / Renter: searches for and books a vehicle | 6.2 | Days 3–5 |
| 2.2 | Host / Vehicle Owner: lists and manages one or more vehicles | 6.2; 9 Days 8–11, 16–19 (My Vehicles) | Days 8–19 |
| 2.3 | Administrator: marketplace, users, vehicles, bookings, payments, disputes and platform settings | 6.2; 9 Days 19–23 | Days 19–23 |
| 2.4 | Support / Operations staff: verification, customer support, incidents and booking issues | 6.2 (limited permissions); 9 Days 19–23 | Days 19–23 |
| **§3** | **Public website pages** | | |
| 3.1–3.4 | Home; Browse Cars; Search Results; Individual Vehicle Listing | 1.4, 2.2, 12.6; 9 Days 7–10 | Days 7–10 |
| 3.5–3.11 | How It Works; Become a Host; Safety; Insurance / Protection; FAQs; About Us; Contact Us | 1.4, 2.2; 9 Days 12–14 | Days 12–14 |
| 3.12–3.13 | Login; Sign Up | 2.2 (auth routes), 6.1, 12.6 (Navigation); 9 Days 3–5 | Days 3–5 |
| 3.14–3.18 | Terms & Conditions; Privacy Policy; Cancellation Policy; Host Agreement; Guest Agreement | 1.4, 2.2, 6.1 (versioned acceptance); 9 Days 12–14; 16 item 11 | Days 12–14; final text Day 24 |
| **§4** | **Homepage** | | |
| 4.1 | Proposition: "Rent a car from local owners across New Zealand." | 9 Days 7–9; 12.6 (Home) | Days 7–9 |
| 4.2–4.5 | Search module: "Where are you going?" with autocomplete; pick-up date and time; return date and time; Search Cars button | 9 Days 6–9; 12.3 (LocationAutocomplete, DateTimeRangePicker); 12.6 | Days 6–9 |
| 4.6 | "Have a car? Earn money by sharing it." with a Become a Host button | 9 Days 7–9; 12.6 | Days 7–9 |
| 4.7–4.13 | Featured vehicles; popular NZ destinations; how the marketplace works; Host earning proposition; safety and trust features; customer reviews; FAQs | 9 Days 7–9; 3 (`cmsBlocks`, `destinations`, `reviews`); 12.6 | Days 7–9 (real reviews from Days 21–22) |
| 4.14 | Footer with legal, support and social links | 9 Days 3–5 (layout shell); 3 (`cmsBlocks`); 16 item 8 | Days 3–5 |
| **§5** | **Search results / Browse Cars** | | |
| 5.1 | Responsive vehicle cards on desktop and mobile | 12.6 (Browse Cars and Search Results, VehicleCard) | Days 7–9 |
| 5.2–5.11 | Card: main image; make and model; year; location; star rating and completed trips; daily price; estimated total where applicable; delivery availability; instant-booking indicator; key features | 12.6 (VehicleCard, "New" label for cars without reviews); 3 (distance); 5 (estimated totals) | Days 6–9 |
| 5.F1–5.F16 | Filters: price range; location/radius; vehicle type; make and model; year; automatic/manual; number of seats; fuel type; hybrid/EV; airport delivery; delivery available; instant booking; minimum rating; unlimited kilometres; pet friendly; child seat available | 3 (vehicle fields, location and airport search, search validation); 9 Days 6–8, 8–11 (Instant Book set in onboarding step 5) | Days 6–8 |
| **§6** | **Individual vehicle listing** | | |
| 6.1 | One of the highest-priority pages, with enough information for an informed booking | 9 Days 1–4 (high-fidelity design), Days 8–10; 12.6 | Days 1–4, 8–10 |
| 6.2–6.10 | Gallery: front; rear; driver side; passenger side; interior; dashboard/odometer; boot; tyres; existing damage if applicable | 3 (`vehicles.photos`); 9 Days 8–10 (gallery), 8–11 (required set) | Days 8–11 |
| 6.11–6.18 | Make, model, year and variant; location; Host rating and trip history; price per day; transmission; fuel type; engine/powertrain; seats and doors | 3 (`vehicles`, location privacy); 1.2 (Maps Static API for the approximate-area map); 9 Days 8–10; 12.6 | Days 8–10 |
| 6.19–6.20 | Registration information and WOF/compliance information where appropriate | 3 (vehicles: WOF, CoF, rego and RUC; plate privacy); 9 Days 8–10 | Days 8–10 |
| 6.21–6.23 | Fuel policy; kilometre allowance; delivery/pickup options | 3 (`vehicles`); 9 Days 8–10; 12.6 | Days 8–10 |
| **§7** | **Booking flow** | | |
| 7.1–7.2 | Guest selects the pickup, then the return, location, date and time | 12.6 (Checkout step 1); 3 (`pickupOptionId`, `returnOptionId`) | Days 11–13 |
| 7.3 | System checks vehicle availability | 3 (double-booking prevention); 8.2 (date holds) | Days 10–13 |
| 7.4 | System calculates the booking price | 5; 11 (`/quote`) | Days 6–7, 10–12 |
| 7.5 | Guest reviews trip details and policies | 12.6 (Checkout step 3; price summary before sign-in) | Days 11–13 |
| 7.6 | Guest signs in or creates an account | 6.1 (sign in during checkout) | Days 11–13 |
| 7.7 | Guest completes identity/driver verification where required | 6.1; 8.2 (verification in review); 9 Days 19–20 | Days 11–13, 19–20 |
| 7.8 | Guest enters or selects a payment method | 8.1 items 2, 7; 12.6 (Checkout step 6) | Days 11–13 |
| 7.9 | Guest confirms the booking | 12.6 (Checkout step 8); 8.1 items 4, 14 | Days 11–13 |
| 7.10 | Host receives a booking notification | 7 (Booking request: email, SMS, in-app; build order); 9 Days 11–13 | Days 11–13 |
| 7.11 | Booking confirmation issued to both parties | 7 (Booking confirmed, Guest + Host) | Days 11–14 |
| 7.12–7.17 | Price breakdown: vehicle rental; delivery fee if applicable; platform/service fee; protection/insurance charge where applicable; taxes/GST where applicable; total payable in NZD | 5; 12.6 (Checkout step 7) | Days 10–13 |
| 7.18 | Final checkout screen clearly separates mandatory and optional charges | 5; 12.6 (Checkout step 7) | Days 10–13 |
| **§8** | **Guest account / dashboard** | | |
| 8.1–8.4 | Upcoming, current, completed and cancelled trips | 8.2 (dashboard grouping); 9 Days 16–18 | Days 16–18 |
| 8.5 | Booking details and receipts | 8.1 item 18; 11 (`/bookings/:id/receipt`); 9 Days 16–18 | Days 13–18 |
| 8.6 | Messages | 9 Days 17–19 | Days 17–19 |
| 8.7 | Saved / favourite vehicles | 3 (`favouriteVehicleIds`); 12.6 (Comparing cars) | Days 16–18 |
| 8.8 | Payment methods | 8.1 item 7 | Days 16–18 |
| 8.9 | Personal details | 6.1 (account changes); 8.2 (account closure); 14 | Days 16–18 |
| 8.10 | Driver licence / identity verification | 6.1; 9 Days 19–20 | Days 16–20 |
| 8.11 | Reviews | 9 Days 16–18 (written and received), 21–22 | Days 16–22 |
| 8.12 | Notifications | 7 | Days 21–22 |
| 8.13 | Help and support | 9 Days 16–18, 20–22 | Days 16–22 |
| **§9** | **Host dashboard** | | |
| 9.1–9.2 | My Vehicles; vehicle status Active / Inactive / Under Review / Suspended | 3 (`vehicles.status`); 8.2 (deactivation, suspension); 9 Days 16–19 | Days 16–19 |
| 9.3 | Bookings: upcoming, current, completed, cancelled | 8.2 (dashboard grouping); 9 Days 16–19 | Days 16–19 |
| 9.4 | Calendar and vehicle availability | 3 (availability rules); 9 Days 10–11, 16–19 | Days 10–11, 16–19 |
| 9.5–9.6 | Earnings dashboard; upcoming payouts | 9 Days 16–19; 8.1 items 9, 19 | Days 16–19 |
| 9.7–9.8 | Messages; reviews | 9 Days 17–19, 21–22 | Days 17–22 |
| 9.9 | Vehicle maintenance / document reminders | 3 (`maintenanceReminders`, `documents`); 4.3 (`daily.hostReminders`) | Days 16–19 |
| 9.10 | Host profile and settings | 3 (`hostProfile`); 9 Days 16–19 | Days 16–19 |
| **§10** | **Host earnings dashboard** | | |
| 10.1–10.5 | Today's earnings; this week; this month; current month vs previous month; total lifetime earnings | 9 Days 16–19 (counted by trip start date in NZ time); 12.6 (earnings page) | Days 16–19 |
| 10.6–10.8 | Upcoming payouts; platform fees; booking-level earnings breakdown | 9 Days 16–19; 8.1 items 9, 19 | Days 16–19 |
| **§11** | **Add a vehicle / Host onboarding** | | |
| 11.1 | Guided multi-step form instead of one long page | 12.6 (Host onboarding); 9 Days 8–11 | Days 8–11 |
| 11.2–11.11 | Step 1: registration number; make; model; year; variant; VIN/chassis where required; fuel type; transmission; seats; doors | 3 (`vehicles`, validation rules); 9 Days 8–11 | Days 8–11 |
| 11.12–11.15 | Step 2: registration/vehicle documents; WOF information; insurance information; other documents the platform requires | 3 (`documents`, incl. the owner's consent; required documents in `platformSettings`); 9 Days 8–11 | Days 8–11 |
| 11.16–11.18 | Step 3: minimum set of photos; clear instructions and example photos; low-quality or missing photos flagged for review | 3 (`photos.qualityFlag`); 9 Days 8–11; 12.6 | Days 8–11 |
| 11.19–11.24 | Step 4: daily price; weekly discount; monthly discount; minimum rental period; maximum rental period; optional kilometre pricing | 3 (`pricing`, `rules`); 5; 9 Days 8–11 | Days 8–11 |
| 11.25–11.28 | Step 5: availability calendar; blocked dates; minimum notice before booking; preparation time between bookings | 3 (`rules`, `availabilityBlocks`); 9 Days 8–11 | Days 8–11 |
| 11.29–11.33 | Step 6: Host pickup location; delivery option; airport delivery; custom delivery locations; delivery fees | 3 (`deliveryOptions`); 9 Days 8–11 | Days 8–11 |
| **§12** | **Vehicle calendar and availability** | | |
| 12.1 | Monthly and weekly calendar views | 12.3 (Calendar); 9 Days 10–11 | Days 10–11 |
| 12.2–12.3 | Booked dates blocked automatically; Hosts block dates manually | 3 (`availabilityBlocks`); 9 Days 10–11 | Days 10–11 |
| 12.4 | Recurring availability | 3 (Recurring availability); 4.3 (`availability.expandRecurring`) | Days 10–11 |
| 12.5–12.6 | Minimum booking notice; buffer time between bookings | 3 (`rules`, validation rules) | Days 10–11 |
| 12.7 | Prevent double-booking | 3 (double-booking prevention); 8.2 (date holds) | Days 10–11 |
| 12.8 | Admin override | 3 (same write path, audit log); 9 Days 19–23 | Days 10–11, 19–23 |
| **§13** | **Messaging** | | |
| 13.1 | Secure in-platform messaging between Guest and Host | 4.4; 3 (contact details masked until confirmation); 6.2 (who sees what); 14 (messaging safety) | Days 17–19 |
| 13.2–13.3 | Text messages; booking-specific threads | 3 (`threads`, `messages`) | Days 17–19 |
| 13.4 | Photo attachments | 3 (private files); 11 (`/uploads/signature`) | Days 17–19 |
| 13.5–13.7 | Automated booking messages; pickup reminders; return reminders | 4.3 (`reminder.pickup`, `reminder.return`) | Days 17–19 |
| 13.8 | Push/email/SMS notifications | Email and SMS: 4.3, 7. Push: 10.4, 10.5 R1 | Email and SMS Days 17–22; push later |
| 13.9 | Report/block | 3 (`reports`, `blockedUserIds`); 9 Days 17–19 | Days 17–19 |
| **§14** | **Digital vehicle handover / condition report** | | |
| 14.1 | Digital check-in and check-out that records the condition of every vehicle | 3 (`conditionReports`); 9 Days 19–21 | Days 19–21 |
| 14.2–14.8 | Check-in: Host photographs the car before the trip; front, rear, both sides, wheels, windscreen, interior and dashboard; odometer; fuel/battery level; existing damage; timestamped photos; Guest reviews and confirms | 3 (incl. inspection validation); 12.6 (Inspection); 8.2 (Host not there) | Days 19–21 |
| 14.9–14.13 | Check-out: repeat the inspection; odometer and fuel/battery level; new photos; flag potential new damage; condition record linked to the booking | 3 (incl. inspection validation, one report per stage); 12.6 (Inspection); 8.2 (check-out, missing check-out, early return) | Days 19–21 |
| **§15** | **Damage and incident reporting** | | |
| 15.1–15.4 | Guest or Host reports an incident; photos and supporting information; incident type; description | 3 (`incidents`, incident validation); 9 Days 20–21 | Days 20–21 |
| 15.5 | System creates a case / reference number | 3 (`caseRef`) | Days 20–21 |
| 15.6 | Admin reviews evidence and communicates with both parties | 3 (`events.visibility`); 9 Days 20–21 | Days 20–21 |
| 15.7 | Complete audit trail of the case | 3 (append-only `events`, Key rules: audit trails); 6.2 (audit log) | Days 20–21 |
| **§16** | **Reviews and ratings** | | |
| 16.1 | Both sides review each other after a completed booking | 3 (`reviews`, review validation, one review each way by unique index); 4.3 (`trip.reviewRequest`, `reviews.reveal`) | Days 21–22 |
| 16.2–16.7 | Overall star rating; communication; vehicle cleanliness/condition; pickup/return experience; vehicle care/guest behaviour; written review | 3 (`reviews`, categories for each direction); 9 Days 21–22 | Days 21–22 |
| 16.8 | Platform moderates reviews under defined rules | 9 Days 21–22; 16 item 10 | Days 21–22 |
| **§17** | **Payments and payouts** | | |
| 17.1–17.2 | Major card payments; Apple Pay and Google Pay | 8.1 items 2, 17 | Days 11–13 |
| 17.3 | All consumer-facing prices in NZD | 3 (Money), 5, 12.7 | Throughout |
| 17.4 | Secure payment processing through a suitable provider | 1.2 (Stripe); 14 (PCI scope, 3-D Secure) | Days 11–13 |
| 17.5–17.6 | Automatic platform fee calculation; Host payout calculation | 5 (incl. card processing fees, GST and Hosts); 8.1 items 20, 22 | Days 6–7, 10–12, 17–18 |
| 17.7–17.8 | Refund handling; partial refunds | 8.1 items 10, 15, 21 (failed refunds); 11 (`cancellation-preview`) | Days 13–14, 17–18 |
| 17.9 | Cancellation fee handling | 5 (Guest cancellation fees); 8.1 items 10, 16; 8.2 (no-shows, request withdrawal, early return) | Days 13–14 |
| 17.10 | Receipts and payment history | 8.1 items 7, 18 | Days 13–18 |
| 17.11 | Failed payment handling | 8.1 item 6 | Days 11–13, 17–18 |
| 17.12 | Payment provider and fee structure chosen during implementation | 1.2 (Stripe recommended); 16 items 2, 6 | Days 1–2 |
| **§18** | **Admin dashboard** | | |
| 18.1 | Complete marketplace oversight | 12.6 (Admin dashboard); 9 Days 19–23 | Days 19–23 |
| 18.2–18.12 | Overview: total registered users; active Hosts; active vehicles; upcoming bookings; booking revenue; platform fees; Host payouts; cancellations; damage/incident cases; pending verification; suspended users or vehicles | 9 Days 19–23 | Days 19–23 |
| 18.13–18.15 | Search and manage users; approve/reject Host applications; approve/reject vehicle listings | 9 Days 8–11 (basic queue), 19–23 | Days 8–11, 19–23 |
| 18.16 | View and edit booking status | 8.2 (admin status edits); 9 Days 19–23 | Days 19–23 |
| 18.17–18.18 | Suspend users; suspend vehicles | 8.2 (suspension effects); 9 Days 19–23 | Days 19–23 |
| 18.19–18.20 | Manage disputes and incidents; issue refunds where authorised | 6.2 (`REFUNDS` permission); 8.1 items 10, 12, 15 | Days 19–23 |
| 18.21–18.23 | Manage fees and platform settings; homepage content; FAQs and help articles | 3 (`platformSettings`, `cmsBlocks`, `faqs`, `helpArticles`); 9 Days 19–23 | Days 19–23 |
| 18.24–18.25 | View platform reports; maintain audit logs | 9 Days 19–23; 3 (`auditLogs`); 6.2 | Days 19–23 |
| **§19** | **Notifications** | | |
| 19.1 | Central notification system | 7 (`notify()`, build order) | Days 3–5 (mailer), 11–13 (`notify()`), 21–22 (centre) |
| 19.2–19.3 | Email; SMS where appropriate | 7 (incl. SMS quiet hours, marketing consent) | Days 3–5, 11–14, 21–22 |
| 19.4 | Push notifications for mobile apps / PWA | 10.4 (ready at launch); 10.5 R1 | Later |
| 19.5 | In-platform notifications | 7 (build order); 4.4 | Days 11–13 (bell), 17–19 (live), 21–22 (centre) |
| 19.6–19.15 | Examples: booking received; booking confirmed; payment successful; pickup reminder; return reminder; new message; verification required; payout processed; cancellation; incident update | 7 (email templates and in-app notifications) | Days 11–22 |
| **§20** | **Mobile-first design** | | |
| 20.1 | Works exceptionally well on mobile, for people travelling or standing beside a car | 12.1, 12.5, 12.6; 15 | Throughout |
| 20.2 | Responsive design for mobile, tablet and desktop | 12.2 (breakpoints), 12.6 | Throughout |
| 20.3 | Large touch targets | 12.2 (44 px minimum) | Throughout |
| 20.4 | Simple navigation | 12.6 (Navigation, dashboard tab bars) | Days 3–5 |
| 20.5 | Fast photo loading | 12.5 (Images) | Throughout |
| 20.6 | Sticky booking CTA on mobile listing pages | 12.6 (Vehicle listing) | Days 8–10 |
| 20.7 | Easy camera/photo upload from phones | 12.3 (FileUpload/CameraCapture), 12.5, 12.6 (Inspection, uploads that wait for signal) | Days 8–11, 19–21 |
| 20.8 | Fast checkout | 12.6 (Checkout), 6.1, 8.1 | Days 11–13 |
| **§21** | **Search and location** | | |
| 21.1–21.2 | Location autocomplete; city, suburb and popular destination search | 1.2 (Google Places and Place Details, and our own places); 3; 11 (`/places`); 9 Days 6–8 | Days 6–8 |
| 21.3 | Map-based vehicle browsing (future or optional) | 10.4, 10.5 R2 | Later |
| 21.4 | Distance from the selected location | 3 (`$geoNear`) | Days 6–8 |
| 21.5 | Airport search and delivery options | 3 (Airport search, `deliveryOptions`) | Days 6–11 |
| 21.6 | NZ-focused address formatting | 3 (Addresses) | Days 2–3 |
| **§22** | **Trust, safety and verification** | | |
| 22.1–22.2 | Email verification; mobile verification | 6.1 | Days 3–5 |
| 22.3–22.5 | Identity verification; driver licence verification; Host identity verification | 1.2 (Stripe Identity); 9 Days 19–20; 16 item 15 | Days 19–20 |
| 22.6 | Vehicle documentation verification | 3 (`documents`); 9 Days 8–11, 19–20 | Days 8–11, 19–20 |
| 22.7 | Listing moderation | 3 (Changes to live listings); 9 Days 8–11 | Days 8–11 |
| 22.8 | Secure payments | 8.1, 14 | Days 11–13 |
| 22.9 | Fraud / suspicious activity monitoring | 14; 3 (same licence on several accounts); 9 Days 20–22 | Days 19–22 |
| 22.10 | Clear incident reporting | 3 (`incidents`); 9 Days 20–21; 12.6 (Active trip) | Days 16–21 |
| 22.11 | Clear safety and protection information | 9 Days 12–14 (Safety and Insurance pages); 12.6 (Checkout, Active trip) | Days 12–18 |
| 22.12 | Final verification, insurance, protection and eligibility rules confirmed with the client's advisers before launch | 16 items 12, 14 | Day 28 |
| **§23** | **NZ-specific requirements** | | |
| 23.1 | NZD throughout the customer experience | 3 (Money), 5, 12.7 | Throughout |
| 23.2 | NZ date and time conventions | 3 (Times), 5 (Trip days), 12.7 | Throughout |
| 23.3 | NZ locations and addresses | 3 (Addresses); seed data (9 Days 2–3) | Days 2–3 |
| 23.4 | NZ vehicle registration and WOF information | 3 (vehicles, Expired documents); 4.3 (reminders) | Days 8–11, 16–19 |
| 23.5 | NZ driver licence workflow | 3 (`driverLicence`); 9 Days 19–20; 16 item 15 | Days 19–20 |
| 23.6 | GST-ready financial reporting | 5 (incl. GST and Hosts); 8.1 items 18, 22; 9 Days 16–19 (Host statements), 19–23 (GST summary); 16 item 7 | Days 13–23 |
| 23.7 | NZ-specific Terms & Conditions and privacy documentation | 9 Days 12–14; 14 (Privacy Act, Unsolicited Electronic Messages Act, Fair Trading Act); 16 item 11 | Days 12–14; final text Day 24 |
| 23.8 | NZ-specific insurance/protection information | 16 item 9; 17 (config-driven protection plans) | Days 12–15 |
| 23.9 | Airport rentals and major tourist destinations | 3 (Airport search, `destinations`); 1.4 | Days 6–11, 26–27 |
| 23.10 | Designed for local customers and international visitors | 6.1 (overseas phone numbers), 12.7; 16 item 19 (languages) | Throughout |
| **§24** | **SEO and marketing** | | |
| 24.1 | Search-engine-friendly vehicle and location pages | 1.4 | Days 26–27 |
| 24.2 | Unique page titles and meta descriptions | 1.4 | Days 26–27 |
| 24.3 | Structured data | 1.4 (JSON-LD) | Days 26–27 |
| 24.4 | Indexable destination pages | 1.4 (`/rental/:city`, sitemap) | Days 26–27 |
| 24.5 | Fast loading performance | 12.5 | Throughout, Day 26 |
| 24.6 | Social sharing previews | 1.4 (Open Graph) | Days 26–27 |
| 24.7–24.8 | Analytics integration; conversion tracking | 1.2 (GA4); 9 Days 26–27 | Days 26–27 |
| 24.9 | Google Search Console or equivalent tooling | 1.2; 9 Days 26–27 | Days 26–27 |
| 24.10 | Landing pages for major NZ cities and destinations | 1.4; 9 Days 26–27 (5 at launch, more added by admins) | Days 26–27 |
| **§25** | **Technical and performance** | | |
| 25.1 | Responsive web application | 1.1, 12 | Throughout |
| 25.2 | Secure HTTPS | 13 (ACM, CloudFront), 14 | Days 2–3, 27 |
| 25.3 | Scalable backend architecture | 13.1, 13.6; 9 Day 26 (load test at 10,000 vehicles) | Days 2–3, 26 |
| 25.4 | Role-based access control | 6.2 | Days 3–5 |
| 25.5 | Secure authentication | 6.1 (incl. staff two-factor sign-in); 14 (CSRF protection) | Days 3–5, 19–23 |
| 25.6 | Encrypted sensitive data | 14 (application encryption, Atlas encryption at rest, HTTPS) | Throughout |
| 25.7 | Automated backups | 13.5 | Day 27 |
| 25.8 | Audit logging for important admin actions | 6.2; 3 (`auditLogs`, append-only) | Days 3–5, 19–23 |
| 25.9 | Image optimisation and CDN support | 1.2 (Cloudinary), 12.5, 13 | Throughout |
| 25.10 | Fast page load times | 12.5 | Throughout |
| 25.11 | Error monitoring and logging | 1.2 (Sentry, Pino, CloudWatch), 13.5 (incl. the external uptime check) | Days 2–3, 27 |
| 25.12 | API-ready architecture for future mobile apps and integrations | 11, 11.1 | Throughout |
| **§26** | **Future mobile app** | | |
| 26.1 | Native iOS and Android apps later, without rebuilding the marketplace backend | 11.1; 10.5 R4 | Architecture from Day 1; apps later |
| 26.2–26.8 | Shared user accounts; shared bookings; push notifications; Host calendar; vehicle photo uploads; messaging; digital check-in/check-out | 11.1 | Architecture from Day 1; apps later |
| **§27** | **Suggested launch phases** | | |
| 27.1–27.13 | Phase 1 (MVP): homepage; search and vehicle listings; vehicle detail pages; Guest accounts; Host accounts; vehicle onboarding; availability calendar; booking system; payments; messaging; basic reviews; admin dashboard; basic notifications | 10.3 | In the 30 days |
| 27.14–27.19 | Phase 2: driver/identity verification; digital vehicle inspection; damage reporting; advanced Host earnings; advanced admin tools; refund/dispute workflows | 10.3 (the reading of "advanced" is confirmed in 16 item 17) | In the 30 days |
| 27.20–27.21 | Phase 2: map search; advanced analytics | 10.4; 10.5 R2, R3 | Later |
| 27.22–27.30 | Phase 3: native mobile apps; dynamic pricing; airport automation; fleet tools for professional Hosts; referral programme; loyalty programme; corporate accounts; API integrations; advanced fraud detection | 10.4; 10.5 R4–R12 | Later |
| **§28** | **Key user journeys** | | |
| 28.1–28.13 | Guest: visit the website; search Auckland or a destination; select dates; compare vehicles; open the listing; review price and policies; create an account / verify identity; pay; receive confirmation; complete digital pickup; use the vehicle; complete the return; review the Host | 9 Day 25 (E2E); 12.6 (Comparing cars, Checkout, Active trip, Inspection) | Days 6–22; tested Day 25 |
| 28.14–28.24 | Host: create a Host account; complete verification; add a vehicle; upload documents and photos; set the price; set availability; publish the listing; receive a booking; complete the handover; receive the payout; review the Guest | 9 Day 25 (E2E); 8.2 (payout setup before going live) | Days 8–22; tested Day 25 |
| **§29** | **Design direction** | | |
| 29.1–29.9 | Premium but approachable; clean NZ-focused identity; trust visible throughout the booking journey; photography a major part of the design; no clutter or excessive text; transparent pricing; obvious primary action on every important page; mobile first, then desktop; original branding and UI | 12.1, 12.2; 5 (estimated totals); 16 item 1 | Days 1–5, 22–23 |
| **§30** | **Designer / developer deliverables** | | |
| 30.1–30.3 | Desktop designs; mobile designs; tablet/responsive behaviour | 9 Days 1–8 | Days 1–8 |
| 30.4 | Clickable prototype | 9 Days 3–5 | Days 3–5 |
| 30.5 | Design system / component library | 9 Days 1–4, 6–7; 12.2, 12.3 | Days 1–7 |
| 30.6 | All key marketplace states and error states | 9 Days 4–8 (states board); 12.3 | Days 4–8 |
| 30.7–30.14 | Guest dashboard; Host dashboard; Admin dashboard; booking flow; payment flow; vehicle onboarding flow; messaging flow; vehicle inspection flow | 9 Days 1–4 (high fidelity: Host dashboard and checkout), Days 4–8 (mid fidelity: the other dashboards and flows) | Days 1–8 |
| 30.15 | Email notification templates | 9 Days 4–8 (design); 7 (coded templates) | Days 4–8 |
| 30.16 | Developer-ready design files and specifications | 9 Days 4–8 (Figma Dev Mode) | Days 4–8 |
| **§31** | **Questions for the development team** | | |
| 31.1–31.13 | Technology stack; payment provider; driver and identity verification; vehicle data verification; insurance/protection integration; Host payouts; cancellations and refunds; fraud prevention; data stored and for how long; scaling from 100 to 10,000+ vehicles; MVP contents; third-party integrations; ongoing hosting, maintenance and support costs | 18 (one answer to each) | Answered |
| **§32** | **Final product goal** | | |
| 32.1 | A Guest goes from searching to a completed booking with minimal friction | 12.5, 12.6 (Checkout); 9 Day 25 | Throughout |
| 32.2 | A Host lists, manages, rents and earns from their vehicle through a simple dashboard | 9 Days 8–11, 16–19; 12.6 | Throughout |

**Result of the cross-check:** every requirement above has a place in the plan. All are built in the 30 days except push notifications (13.8 in part, 19.4), map-based browsing (21.3, 27.20), advanced analytics (27.21), the spec's Phase 3 items (27.22–27.30) and the native apps themselves (26.1–26.8, whose architecture ships at launch). MILESTONES.md excludes these, and section 10.5 plans them. Items that need a client or adviser decision before they are final are listed in section 16.

**Second cross-check (25/09/2026):** each spec section was checked again, this time against the depth of the plan (data model, indexes, validation rules, API, jobs, notifications, payments, edge cases, compliance, schedule and dependencies), not just against its headings. Every bullet was already traced. The gaps found were below the bullet level, and each is now covered in the section named:
- **Build order:** the booking request reached Hosts in-app and by SMS on Days 11–13, but those channels were only built on Days 21–22 (section 7 build order; section 9 Days 11–13).
- **Dependencies:** homepage and listing pages, public pages, notifications, cancellations, dashboards, messaging, help and risk flags, and design polish had no dependency rows, and there were no phase gates (section 9).
- **Onboarding:** there was no step for the Host to turn Instant Book on or off (it drives a spec §5 badge and filter), and nowhere to store a listing's cancellation tier, although each booking copies one (section 3; section 9 Days 8–11).
- **Data and API:** indexes for reviews, payments, payouts, inspections, incidents, threads and audit logs, with one review each way and one condition report per stage enforced; GST registration for Hosts; last search; homepage FAQ and destination flags. Endpoints for the cancellation preview, place details, public profiles, Host profile edits, "my reviews" and the last search (sections 3 and 11).
- **Validation and edge cases:** search, inspection, review, incident and extra-charge rules, and the registered-owner check; withdrawing a request, early returns, documents that expire before a booked trip, Host payout accounts that Stripe disables, and failed refunds (sections 3, 8.1 and 8.2).
- **NZ and security:** GST-registered Hosts and commission invoices, the Unsolicited Electronic Messages Act, the Fair Trading Act, where the Privacy Policy says data is stored, SMS quiet hours, masked contact details in messages, CSRF protection, append-only audit trails, duplicate-licence risk flags, an external uptime check, and a load test at 10,000 vehicles (sections 5, 7, 8.1, 13, 14; section 9 Day 26).
- **Client inputs:** processing fees, per-listing cancellation tiers, early-return refunds, VIN or chassis "where required", the roadside assistance number, page copy and estimator assumptions, and languages (section 16, items 2, 3, 9, 13, 18 and 19).
