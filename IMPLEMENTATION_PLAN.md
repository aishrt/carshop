# NZ Car Rental Marketplace: Implementation & Design Plan

Working name: **DriveShare NZ** (placeholder until the client confirms the brand)
Source: *NZ Peer-to-Peer Car Rental Marketplace – Website Specification (Sept 2026)*, referred to as "spec §N" below
Client-facing timeline: [MILESTONES.md](MILESTONES.md) (30-day delivery). Every milestone item and every section of the spec (§1–§32) is mapped to this plan in [section 10](#10-requirements-coverage). The spec's questions for the development team (§31) are answered in [section 18](#18-answers-to-the-clients-questions-spec-31).
Stack: **React + Vite** (TypeScript) for the website, **MongoDB** for all data, and a small **Node.js + Express** API between them (see [section 1](#1-technology-stack)). No Redis, no migration tool, no monorepo tooling.
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
| Maps / places | Google Places Autocomplete (restricted to `country: nz`), combined with our own destinations and airports | City, suburb, destination and airport search, NZ address formatting (spec §21) |
| Help and support | Support tickets and help articles stored in MongoDB | Help centre, contact form and the support staff inbox, with no third-party helpdesk needed at launch |
| Monitoring | Sentry + structured logs (Pino) | Error monitoring and logging (spec §25) |
| Performance | Lighthouse CI + bundle size check in CI, real-user Core Web Vitals in Sentry | The build fails if a page breaks the speed budget (section 12.5) |
| Analytics | GA4 + Google Search Console | Analytics, conversion tracking and search monitoring (spec §24) |
| Testing | Vitest, Supertest, mongodb-memory-server, Playwright | Unit, API (against an in-memory MongoDB replica set) and end-to-end tests |
| CI/CD | GitHub Actions | Lint, typecheck, test, build and deploy on every push |

### 1.3 Deliberately left out
- **No Redis.** MongoDB covers everything Redis would have done: background jobs, rate limits, Socket.IO sync between instances, and sessions (section 4).
- **No migration tool.** Mongoose schemas define the structure, indexes are synced from the schemas on deploy, and schema changes are additive (section 3).
- **No monorepo tooling** (Turborepo, pnpm, Nx). Plain **npm workspaces**, built into npm, share code between the client and the server (section 2).
- **No separate worker or web server.** Background jobs run inside the API process, and Express serves the website. There is one service to deploy (section 13).

### 1.4 SEO with a plain React app
A Vite React app renders in the browser. Vehicle, city and destination pages must still be indexable and show correct link previews (spec §24), so the Express server does the following:

1. **Server-injected metadata for every public route.** Before sending `index.html`, Express writes a unique page title, meta description, canonical URL, Open Graph tags (social sharing previews) and JSON-LD structured data into it. Static pages use a fixed table in `server/src/seo/pages.ts`: Home, Browse Cars, How It Works, Become a Host, Safety, Insurance / Protection, FAQs, Help, About Us, Contact Us, and the legal pages (Terms & Conditions, Privacy Policy, Cancellation Policy, Host Agreement, Guest Agreement). Vehicle pages (`/cars/:slug`) use the vehicle record. City and destination landing pages (`/rental/:city`) use the `destinations` collection: the 5 launch cities are seeded, and admins can add more destinations without code changes.
2. **Sitemap and robots.** `sitemap.xml` is generated from the static pages, active vehicles and destinations. `robots.txt` and `noindex` tags keep private areas (account, host, admin, checkout, login, sign-up) out of search results.

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
| Server dev and build | `tsx watch` in development, **tsup** for production | tsup bundles the server and the `shared` code it uses into `server/dist` (`noExternal: ['@driveshare/shared']`) |
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
│   │   └── server.ts              # /api/v1, Socket.IO, and the client build in production
│   ├── scripts/                   # seed.ts, sync-indexes.ts
│   └── test/
├── shared/                        # @driveshare/shared: Zod schemas (API contracts), types, enums, pricing,
│                                  # cancellation policy, design tokens, NZD, date and NZ address formatters.
│                                  # Runs in the browser and in Node.
├── e2e/                           # Playwright tests against the client + API
├── .github/workflows/ci.yml
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
| `npm install stripe -w server` | Adds a dependency to one workspace |

### 2.5 Environment variables

- `server/.env.example` and `client/.env.example` are committed; `.env` files are ignored. At startup the server checks its variables against a Zod schema in `src/env.ts` and exits with a clear message if one is missing or invalid.
- Only `VITE_`-prefixed variables reach the browser bundle. Secrets are never given that prefix.

---

## 3. Data Model (MongoDB collections)

Each collection's Mongoose model lives in its module, for example `server/src/modules/vehicles/vehicle.model.ts`.

**Embed or reference:** data that is always read with its parent and stays small (photos, documents, delivery options, line items, refunds, inspection photos, incident events, ticket messages) is embedded. Data that grows without limit or is queried on its own (bookings, messages, availability blocks, reviews, notifications, audit logs, jobs) gets its own collection.

```
users              _id, email (unique, lowercase), phone (E.164), passwordHash, firstName, lastName, dob, avatarUrl,
                   roles[] (GUEST|HOST|ADMIN|SUPPORT), permissions[] (e.g. REFUNDS for authorised support staff),
                   emailVerifiedAt, phoneVerifiedAt, status (ACTIVE|SUSPENDED), suspendedReason,
                   stripeCustomerId, favouriteVehicleIds[], blockedUserIds[], notificationPrefs,
                   riskFlags[] { code, detail, createdAt, clearedBy }, createdAt
                   ├─ identityVerification { status (NONE|PENDING|APPROVED|REJECTED), provider, providerRef,
                   │                         verifiedAt, reviewedBy }                    (Guests and Hosts)
                   ├─ driverLicence { number (encrypted), version, country, class (NZ_FULL|NZ_RESTRICTED|
                   │                  NZ_LEARNER|OVERSEAS), englishProof? (IDP|APPROVED_TRANSLATION),
                   │                  issuedAt, expiry, status, reviewedBy }
                   ├─ agreements[] { type (TERMS|PRIVACY|GUEST|HOST), version, acceptedAt, ip }
                   └─ hostProfile { status (APPLIED|APPROVED|REJECTED|SUSPENDED), appliedAt, reviewedBy,
                                    reviewNotes, stripeAccountId, payoutsEnabled, bio, responseRate,
                                    tripCount, rating {avg, count}, feesOwedCents (Host cancellation fees
                                    not yet deducted from a payout) }
sessions           userId, refreshTokenHash, userAgent, ip, expiresAt (TTL index: deleted on expiry)
vehicles           _id, hostId, slug (unique), regoPlate, vin?, chassisNo? (NZ imports often have a chassis
                   number instead of a 17-character VIN; one of the two is required), make, model, year, variant,
                   transmission, bodyType (HATCHBACK|SEDAN|WAGON|SUV|UTE|VAN|PEOPLE_MOVER|COUPE|CONVERTIBLE),
                   fuelType (PETROL|DIESEL|HYBRID|PHEV|EV), seats, doors, features[], wofExpiry, regoExpiry,
                   powertrain { engineCc, cylinders, description, evRangeKm, batteryKwh },
                   fuelPolicy, kmAllowancePerDay, unlimitedKm, petFriendly, childSeat,
                   pricing { dailyCents, weeklyDiscountPct, monthlyDiscountPct, extraKmCents },
                   rules { minDays, maxDays, minNoticeHours, bufferHours, instantBook },
                   status (DRAFT|UNDER_REVIEW|CHANGES_REQUESTED|REJECTED|ACTIVE|INACTIVE|SUSPENDED),
                   reviewNotes, onboardingStep,
                   location { type: "Point", coordinates: [lng, lat] }, suburb, city, region,
                   rating { avg, count }, tripCount, bookingSeq (used to serialise bookings, see below)
                   ├─ photos[]               { type (FRONT|REAR|DRIVER|PASSENGER|INTERIOR|DASH|BOOT|TYRES|DAMAGE),
                   │                           url, order, qualityFlag (OK|LOW_RES|DARK|BLURRY|ADMIN_FLAGGED),
                   │                           status (PENDING|APPROVED|REJECTED) }
                   ├─ documents[]            { type (REGO|WOF|INSURANCE|OTHER), url, expiry,
                   │                           status (PENDING|VERIFIED|REJECTED), reviewedBy }
                   ├─ deliveryOptions[]      { _id, type (PICKUP|DELIVERY|AIRPORT|CUSTOM), label, address,
                   │                           airportCode, feeCents, radiusKm }
                   ├─ recurringRules[]       { daysOfWeek[], startTime, endTime }   (e.g. unavailable weekdays 8am–6pm)
                   └─ maintenanceReminders[] { title, dueAt?, dueOdometer?, notes, doneAt }
availabilityBlocks vehicleId, startAt, endAt, reason (BOOKED|HOST_BLOCK|RECURRING|BUFFER|ADMIN), bookingId?
bookings           _id, ref (DS-XXXXXX, unique), vehicleId, guestId, hostId, startAt, endAt,
                   pickupOptionId, pickupAddress?, returnOptionId, returnAddress?, protectionPlan,
                   status (PENDING|CONFIRMED|ACTIVE|COMPLETED|CANCELLED|DECLINED|EXPIRED),
                   vehicleSnapshot { title, photoUrl, regoPlate },
                   price { subtotalCents, deliveryCents, serviceFeeCents, protectionCents, gstCents, totalCents,
                           hostPayoutCents, platformFeeCents },
                   cancellationPolicy, cancelledBy, cancelledAt, cancellationFeeCents
                   ├─ lineItems[]    { code, label, amountCents, mandatory }
                   └─ extraCharges[] { type (EXTRA_KM|FUEL|CLEANING|LATE_RETURN|DAMAGE|OTHER), description,
                                       amountCents, incidentId?, addedBy, paymentId, status }
payments           bookingId, type (BOOKING|EXTRA_CHARGE), stripePaymentIntentId (unique), amountCents,
                   status (PENDING|AUTHORISED|SUCCEEDED|FAILED|REFUNDED|PARTIALLY_REFUNDED), method, failureReason
                   ├─ refunds[] { amountCents, reason, issuedBy, stripeRefundId, createdAt }
                   └─ dispute?  { stripeDisputeId, reason, status, dueBy }       (card chargebacks)
payouts            hostId, bookingId, amountCents, stripeTransferId, status (SCHEDULED|HELD|PAID|FAILED),
                   scheduledFor, paidAt
                   └─ deductions[] { type (HOST_CANCELLATION_FEE|OTHER), bookingId, amountCents }
threads            bookingId (unique), participantIds[], lastMessageAt
messages           threadId, senderId, body, attachments[], systemGenerated, readAt, createdAt
reviews            bookingId, authorId, subjectId, direction (GUEST_TO_HOST|HOST_TO_GUEST), overall,
                   communication, cleanliness, pickupReturn, care, body, status (PUBLISHED|PENDING|HIDDEN),
                   moderation { reason, by, at }
conditionReports   bookingId, stage (CHECK_IN|CHECK_OUT), odometer, fuelOrBatteryPct, notes,
                   damagePins[] { x, y, note, isNew }, confirmedByGuestAt, confirmedByHostAt
                   └─ photos[] { angle (FRONT|REAR|DRIVER_SIDE|PASSENGER_SIDE|WHEELS|WINDSCREEN|INTERIOR|
                                 DASHBOARD|DAMAGE), url, takenAt, exifTakenAt, lat, lng }
incidents          caseRef (unique), bookingId, reporterId, type (DAMAGE|ACCIDENT|THEFT|BREAKDOWN|CLEANING|
                   FUEL|LATE_RETURN|DISPUTE|OTHER), description, status (OPEN|INVESTIGATING|AWAITING_RESPONSE|
                   RESOLVED|CLOSED), assignedTo
                   └─ events[] { actorId, action, note, attachments[], visibility (BOTH|GUEST|HOST|INTERNAL),
                                 createdAt }   (full audit trail, append only)
reports            reporterId, targetType (USER|MESSAGE|REVIEW|VEHICLE), targetId, reason, note,
                   status (OPEN|ACTIONED|DISMISSED), handledBy
supportTickets     ref (unique), userId? (or name + email from the contact form), bookingId?, category, subject,
                   status (OPEN|PENDING|RESOLVED), assignedTo
                   └─ messages[] { authorId, body, attachments[], internal, createdAt }
helpArticles       slug, title, body (Markdown), category, audience (GUEST|HOST|ALL), published, order
notifications      userId, type, channel (EMAIL|SMS|IN_APP), payload, sentAt, readAt
jobs               type, payload, runAt, status (QUEUED|RUNNING|DONE|FAILED|CANCELLED), attempts, maxAttempts,
                   uniqueKey?, refId? (e.g. a bookingId), lockedAt, lockedBy, lastError, finishedAt   (section 4)
auditLogs          actorId, action, entity, entityId, before, after, ip, createdAt
stripeEvents       eventId (unique), type, processedAt   (webhook idempotency)
faqs               question, answer, category, order
cmsBlocks          key (unique), content, version   (homepage hero and sections, featured vehicles, footer links
                   incl. social links, legal pages in Markdown)
destinations       slug, city, region, intro, heroImage, location, airports[]   (landing pages, homepage tiles)
platformSettings   one document: fees, GST rate, cancellation tiers, Host cancellation fee, protection plans,
                   driver eligibility rules, verification requirements, review rules, risk-flag thresholds
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

**Indexes and schema changes, without migrations**
- Indexes are declared in the Mongoose schemas. `autoIndex` is on in development. In staging and production it is off, and `npm run db:indexes` (`syncIndexes()` for every model) runs in the deploy step.
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
- **Expired documents:** search and booking skip vehicles whose WOF or rego expires before the trip ends, and Hosts are reminded 30 and 7 days before expiry (section 4.3).
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
  When the second booking is retried, it sees the first booking's block and returns `409 Conflict`. Transactions need a replica set: Atlas provides one, and docker-compose runs MongoDB locally as a single-node replica set. An automated test sends 20 simultaneous bookings for one car and checks that exactly one succeeds. An admin override (block or unblock dates) uses the same function and is written to the audit log.

---

## 4. Background Jobs & Realtime (MongoDB, no Redis)

### 4.1 What MongoDB handles

| Need | How it works |
|---|---|
| Background jobs (emails, SMS, reminders, expiries, payouts, extra charges) | `jobs` collection + a job runner inside the API process (4.2) |
| Live messaging and notifications across several server instances | Socket.IO with `@socket.io/mongo-adapter` (4.4) |
| Rate limiting (login, sign-up, OTP, password reset, contact form, messages) | `express-rate-limit` with a MongoDB store; counters expire through a TTL index |
| Login sessions | `sessions` collection with a TTL index (section 6) |
| Caching | A short in-memory cache per instance (60 s) for homepage content, featured vehicles and popular searches, and Cloudflare caching of public GET responses. Availability is never cached: it is always re-checked at quote and booking time. |

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
- **Crash recovery:** a job stuck in `RUNNING` for more than 10 minutes (its instance crashed) is put back in the queue.
- **Cancelling:** when a booking is cancelled, all its queued jobs are cancelled by `refId`.
- **Handlers are idempotent:** each checks the current state before acting (for example, a payout handler skips a booking that already has a transfer), so a retried job never does the work twice.
- **Recurring jobs:** after a daily job runs, it schedules its next run with a dated `uniqueKey` (e.g. `daily.hostReminders:2026-10-01`). The unique index means several instances create it only once.
- **Cleanup:** finished jobs are deleted after 30 days by the TTL index on `finishedAt`.
- **Setting:** `RUN_JOBS=true|false` per instance. At launch every instance runs jobs. At larger scale the runner can run as its own process from the same code (section 13).

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
| `payout.transfer` | 24 h after the trip starts | Creates the Stripe transfer to the Host. If an incident or card dispute is open, the payout is held and re-checked daily. |
| `messages.unreadEmail` | 10 min after a message | Emails the recipient if the message is still unread (and SMS if they opted in) |
| `availability.expandRecurring` | When recurring rules change, and monthly | Rebuilds `RECURRING` blocks for the next 12 months through the availability service |
| `daily.hostReminders` | Daily at 9:00 am NZ time | WOF, rego and insurance reminders at 30 and 7 days before expiry, plus Host-set maintenance reminders when due |
| `daily.dataRetention` | Daily | Deletes data that has passed its retention period, such as ID images 90 days after verification (section 14) |

### 4.4 Realtime
- Socket.IO runs on the same Express server. Users join a room for their own notifications and one for each booking chat they are part of.
- `@socket.io/mongo-adapter` passes events between API instances through a MongoDB collection, so a message sent to one instance reaches a user connected to another.
- Job handlers run in the same process, so they emit notifications through the same Socket.IO server.
- If the socket disconnects, TanStack Query refetches messages and notifications when the connection returns, so nothing is lost.

---

## 5. Pricing Engine (`shared/src/pricing.ts`)

One function, imported as `@driveshare/shared/pricing`, is shared by the client (for display) and the API (the source of truth):

```
days            = ceil((end - start) / 24h)
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

---

## 6. Authentication, Roles & Permissions

### 6.1 Sign-in and sessions
- Email and password (bcrypt), with an email verification link and a password reset link sent by email.
- Mobile verification by SMS one-time code (Twilio Verify). Phone numbers are stored in E.164 format. The input defaults to +64 but accepts overseas numbers, so international visitors can verify too (spec §23).
- Short-lived JWT access token (15 min) and a rotating refresh token (30 days), both in `httpOnly`, `Secure`, `SameSite=Lax` cookies. Refresh tokens are stored hashed in the `sessions` collection, and a TTL index removes them when they expire. Mobile apps can use the same tokens via the `Authorization` header.
- **Sign in during checkout:** a Guest who is not logged in can sign in or create an account inside the booking flow without losing their selected car, dates and options (spec §7 step 6).
- **Agreements:** Terms and Privacy are accepted at sign-up, the Guest Agreement at checkout and the Host Agreement in the Host application. Each acceptance is saved with the document version, time and IP address. When a legal document changes, users accept the new version at their next sign-in.
- Rate limiting on auth routes (MongoDB store, section 4.1), and account lockout after repeated failures.

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

---

## 7. Email & Notifications

**Provider:** Resend (simple, good deliverability, React Email support), behind a `Mailer` interface in `server/src/integrations` so it can be swapped for AWS SES.
**Local development:** a console mailer prints each email's subject and links to the terminal and saves the HTML to `server/.mail/`. `npm run email:dev -w server` previews every template in the browser. Staging sends real emails through Resend, so the client can receive them.

**Flow:** service event → `notify(userId, 'BOOKING_CONFIRMED', data)` → `notifications` document (shown in the in-app notification centre and pushed over Socket.IO) → `email.send` / `sms.send` job (section 4) → provider API → delivery status stored. Failed sends retry up to 5 times with exponential backoff.

**Channels (spec §19)**
- **Email:** every template below.
- **In-app:** notification centre with an unread count, for bookings, messages, payments, payouts, verification and incidents.
- **SMS:** phone verification codes, pickup and return reminders, new booking requests for Hosts, and (opt-in) new messages.
- **Push (mobile apps / PWA):** not in the 30-day scope (MILESTONES.md). `notify()` sends through one channel adapter per delivery method, so a push channel is added later without changing the events that trigger notifications (section 10.4).
- Users choose which non-essential emails and SMS they receive in their notification preferences.

All the example notifications in spec §19 are covered: booking received, booking confirmed, payment successful, pickup reminder, return reminder, new message, verification required, payout processed, cancellation and incident update.

**Email templates** (`server/src/emails`)

| Group | Templates |
|---|---|
| Account | Welcome, Verify email, Reset password, Password changed, Verification required, Verification approved or rejected, Account suspended |
| Host | Host application received, Host application approved or rejected, Listing submitted, Listing approved, changes requested or rejected, Listing changes (new photos, documents) approved or rejected, Vehicle suspended, WOF/rego/insurance expiring (30 / 7 days), Maintenance reminder |
| Booking | Booking request (Host), Booking confirmed (Guest + Host), Booking declined/expired, Cancellation (both; a Host cancellation tells the Guest about the full refund and tells the Host about any cancellation fee), Pickup reminder (24 h / 2 h), Return reminder, Trip completed + review request |
| Payments | Payment receipt (GST receipt), Payment failed (with a link to retry), Extra charge (extra kilometres, or a charge from a resolved incident), Refund issued, Payout processed (Host, showing any deductions) |
| Support and safety | New message (only if unread for 10 min), Incident created and updated, Support ticket received and replied, Admin alerts |

**Deliverability:** a dedicated sending subdomain (`mail.<domain>`) with SPF, DKIM and DMARC records, plus a List-Unsubscribe header on non-transactional mail.

---

## 8. Payments, Refunds & Payouts (Stripe)

1. The Guest checks out, and the API creates a PaymentIntent in NZD with `capture_method: manual` for request-to-book, or automatic for Instant Book.
2. The Stripe Payment Element handles cards, Apple Pay and Google Pay. The Guest can choose a saved card or add a new one. Card data never reaches our servers.
3. The card is saved for later charges (`setup_future_usage: off_session`), with the Guest's consent shown at checkout, so extra kilometre charges and other post-trip charges (item 11) can be taken after the trip.
4. A `payment_intent.succeeded` webhook confirms the booking, blocks the calendar, and sends the emails.
5. A request-to-book that the Host does not accept within 24 h expires and the payment authorisation is released (`booking.expireRequest` job).
6. **Failed payments:** at checkout, the Guest sees a clear reason and can retry or choose another method while the dates stay held for 30 minutes. A failed extra charge sends the Guest a link to pay, is retried, and alerts support. Every failure is recorded on the payment and shown in the admin dashboard.
7. **Saved payment methods and history:** each Guest has a Stripe Customer. The Guest dashboard lists saved cards (add or remove) and a payment history with receipts and refunds.
8. The Host onboards to **Stripe Connect Express** (bank account and identity) from the Host dashboard.
9. **Payouts:** the `payout.transfer` job creates a Transfer to the Host 24 h after the trip starts. The payout is held if an incident or card dispute is open. Hosts see upcoming and paid payouts.
10. **Cancellations:** the policy engine calculates the refund and the cancellation fee, and the API issues a full or partial refund and records both on the booking and payment. Admins, and support staff with the `REFUNDS` permission, can also issue refunds from the admin dashboard.
    - **Host cancellations:** the Guest always gets a full refund. Any Host cancellation fee (set in `platformSettings`; $0 until the client decides, section 16 item 3) is added to the Host's `feesOwedCents` and deducted from their next payout, shown as a line on that payout. Admins can waive the fee, and the waiver is written to the audit log. Repeated Host cancellations raise a risk flag for admins.
11. **Other post-trip charges:** when an incident is resolved against the Guest (fuel not returned as the listing's fuel policy requires, cleaning, late return, or a damage amount allowed by the Guest Agreement), an admin adds an extra charge to the booking, linked to the incident. It uses the same saved-card, pay-link and retry flow as extra kilometres, and the Host's share is added to their payout.
12. **Card disputes (chargebacks):** a `charge.dispute.created` webhook alerts admins, links the dispute to the booking, and holds any unpaid payout. Admins answer it in Stripe with the evidence from the booking (inspection photos, messages, agreement acceptance).
13. All webhooks are verified by signature and are idempotent: each Stripe event ID is saved in `stripeEvents` (unique index), so a repeated event is skipped.

---

## 9. Feature Implementation Checklist: 30-Day Schedule

**Assumptions for 30 days**
- **Team:** 1 senior UI/motion designer (full time Days 1–8, then part time for design QA and animation review) and 2–3 full-stack developers working in parallel on frontend and backend. At least one developer has strong frontend animation experience.
- **UI:** built on shadcn/ui, restyled with our luxury design tokens. High-fidelity Figma designs are made for the key screens, linked into a clickable prototype that includes the signature animations. The other flows get mid-fidelity designs, and their screens reuse the same components and motion building blocks.
- **Client:** decisions and feedback arrive within 24 hours.

**Status key:** ✅ Done · 🟡 Partly done (the note says what is left) · ⬜ Not started · 👤 Needs the client or the design team (not a coding task)

**Progress:** the checklist was reset on 24/09/2026 when the stack was simplified (no Redis, no migration tool, no monorepo tooling). No tasks have started yet.

### Phase 1: Design & Foundation (Days 1–5)
**Delivers:** design style guide, key page designs and the animation prototype; a staging link where you can create an account and receive a verification email.

| Day | Tasks | Status |
|---|---|---|
| 1 | Kick-off workshop: fees, cancellation tiers, security deposit, protection, driver eligibility (age, licence classes, overseas licences and IDP), when verification is required, GST treatment. Client creates the Stripe, Resend, Twilio, Cloudinary, Google Cloud and MongoDB Atlas accounts (by Day 2). Until decisions arrive, launch defaults live in `shared/src/policies.ts` and are editable in `platformSettings`. | 👤 Client |
| 1–2 | Project setup (section 2): npm workspaces (`client`, `server`, `shared`), TypeScript, ESLint/Prettier with the client/server import rule, env validation, docker-compose MongoDB replica set. GitHub Actions CI (lint, typecheck, test, build, Lighthouse and bundle size budgets). Staging deploy: one Node service + an Atlas staging database. | ⬜ |
| 1–4 | Original luxury art direction: moodboard, photography and video selection, colours and fonts, leading to the **design style guide** and a **Figma component library** that matches the coded components in 12.3 (spec §30). High-fidelity Figma designs for Home, Search, Vehicle listing, Checkout and Host dashboard, for mobile and desktop, with notes on tablet and responsive behaviour. Design tokens in `shared/src/tokens.ts` → Tailwind theme, with a WCAG contrast test. | ⬜ Tokens · 👤 Designer |
| 3–5 | Motion system (durations, easing curves, springs) and a **clickable prototype** (spec §30) that links the high-fidelity key screens into the main Guest flow (Home → Search → Vehicle listing → Checkout) plus the Host dashboard, on mobile and desktop, with the key animations: homepage hero, opening a car (card to listing), search filters, checkout. Motion building blocks in code (`MotionProvider`, `Reveal`, `Stagger`, `CountUp`, `Sheet`, sliding `Tabs`, View Transitions). | ⬜ Code · 👤 Designer |
| 4–8 | Mid-fidelity flow designs (spec §30): Guest dashboard, Admin dashboard, Host application and vehicle onboarding, booking and payment flow, messaging, vehicle inspection. A **states board** covering loading, empty, error and success states and marketplace states (dates unavailable, payment failed, verification pending, listing under review, vehicle suspended). Email template design. Developer-ready specs in Figma Dev Mode. | 👤 Designer |
| 2–3 | Mongoose models for all collections and their indexes, `db:indexes` script, seed data (NZ cities, suburbs and airports, the 5 launch destinations and other popular places, 20 demo vehicles, test Hosts, Guests, a support user and an admin, FAQs, help articles). Model tests against an in-memory replica set. | ⬜ |
| 3–5 | Auth: sign-up, login, logout, email verification, password reset, mobile OTP (NZ and overseas numbers), agreement acceptance with versions. Rotating refresh tokens, account lockout, rate limits, role and permission middleware (section 6). | ⬜ |
| 3–5 | Mailer (Resend + console) + MongoDB job queue and runner (section 4) + templates: welcome, verify email, reset password, password changed. Client: React Router with lazy-loaded routes and role guards. Layout shell: header, footer (legal, support and social links), mobile nav, 404 and error pages, favicons and a web app manifest (the site can be added to a phone's home screen, the base for PWA push later). | ⬜ |
| 5 | Staging check: create an account and receive the verification email. **Design sign-off** (look and feel, key screens, animation prototype). | 👤 Client |

### Phase 2: Core Marketplace (Days 6–15)
**Delivers:** a working staging site where a test Guest can find a car and book it, and a test Host can list a car and receive the booking.

| Day | Tasks | Status |
|---|---|---|
| 6–7 | Component library in `client/src/components/ui` (DateTimeRangePicker, LocationAutocomplete, Gallery + lightbox, Stepper, Calendar, Slider, Badge, Sheet, Toast, Skeleton) with hover, press and focus states. Marketplace components (VehicleCard, PriceBreakdown, StickyBookingBar) in `client/src/features`. | ⬜ |
| 6–8 | Search API: `$geoNear` search with **all 16 filters from spec §5**: price range, location and radius, vehicle type, make and model, year, automatic/manual, seats, fuel type, hybrid/EV, airport delivery, delivery available, Instant Book, minimum rating, unlimited kilometres, pet friendly, child seat. Distance from the searched location, availability exclusion when dates are given, sort (recommended, price, rating, distance, newest), pagination. Location autocomplete (NZ only) combining Google Places with our own cities, suburbs, destinations and airports. **Airport searches** also return cars that deliver to that airport, with the airport delivery fee in the estimated total (section 3). | ⬜ |
| 7–9 | **Homepage:** headline "Rent a car from local owners across New Zealand.", search module (Where are you going? with autocomplete, pick-up date and time, return date and time, **Search Cars**), secondary call to action "Have a car? Earn money by sharing it." with a **Become a Host** button, featured vehicles, popular NZ destinations, how it works, Host earnings, safety and trust, customer reviews, FAQs, footer. **Browse Cars** (all vehicles, no dates needed) and **Search Results** (with dates and estimated totals): cards with photo, make and model, year, location and distance, rating and completed trips, daily price, estimated total, delivery and Instant Book badges, key features. Filter sheet on phones, sidebar on desktop, filters in the URL, animated result changes, skeletons. | ⬜ |
| 8–10 | **Vehicle listing page:** swipeable gallery (front, rear, driver side, passenger side, interior, dashboard/odometer, boot, tyres, existing damage) with full-screen view; make, model, year and variant; location; Host card with rating and trip history; price per day; transmission, fuel type, engine/powertrain, seats and doors; **registration and WOF information**; fuel policy; kilometre allowance; delivery and pickup options; cancellation policy; reviews; location map; **sticky Book button on mobile**. Shared-photo transition from card to listing. | ⬜ |
| 8–11 | **Host application** (Host profile + Host Agreement) and **vehicle onboarding in 6 steps:** (1) rego, make, model, year, variant, **VIN or chassis number** (imports without a VIN), **body type** (for the vehicle type filter), fuel type, **engine/powertrain** (engine size and cylinders, or EV range and battery), transmission, seats, doors, **key features** and extras (pet friendly, child seat available); (2) registration, WOF, insurance and other documents; (3) photos: the **minimum required set** (front, rear, driver side, passenger side, interior, dashboard/odometer, boot, tyres, plus existing damage where there is any) with instructions, example shots and camera capture on phones, with **low-quality and missing photos flagged for review** (automatic checks in the browser for low resolution, darkness and blur; the flags are shown to support staff in the listing review queue); (4) daily price, weekly and monthly discounts, minimum and maximum rental, kilometre allowance or unlimited kilometres, extra-km price, **fuel policy**; (5) availability calendar, blocked dates, minimum notice, preparation time; (6) Host pickup location, delivery, airport delivery, custom delivery locations, delivery fees. Auto-saved drafts, resume where you left off, uploads to Cloudinary with progress, a missing-items check before submit. A basic admin queue approves or rejects Host applications and approves, rejects or requests changes to listings, so a test listing can go live on staging. **Edits to live listings** follow the moderation rules in section 3: new photos and documents wait in the same queue, and rego, VIN or chassis number, make, model or year changes send the listing back to Under Review. | ⬜ |
| 10–11 | **Availability calendar:** month and week views, booked dates blocked automatically, manual blocks, **recurring availability**, minimum notice, buffer time, admin override. The single transactional write path that **prevents double-booking** + the 20-simultaneous-bookings test. | ⬜ |
| 10–12 | Pricing engine (section 5) + `POST /vehicles/:id/quote` with trip rules (notice, min/max days, delivery options, protection plan). **Price breakdown in NZD:** rental, delivery, service fee, protection, GST, total, with mandatory and optional charges shown separately. | ⬜ |
| 11–13 | **Booking flow (all 11 steps of spec §7):** pick-up and return location, date and time (Host location, delivery address or airport), availability check, price calculation, trip details and policies, sign in or sign up without leaving checkout, verification step (licence details now; the identity check is connected on Days 19–20), choose a saved card or add a new one with Stripe Payment Element (**cards, Apple Pay, Google Pay**), confirm. Verified idempotent webhooks, Instant Book and request-to-book with the 24 h expiry and 30 min payment hold (jobs), failed-payment handling. The Host receives the booking by email, SMS and in-app, and accepts or declines it from a basic bookings page (the full dashboard follows in Phase 3). Confirmation sent to both parties. | ⬜ |
| 13–14 | Cancellation policy engine + refund and cancellation fee calculation for **Guest and Host cancellations** (a Host cancellation refunds the Guest in full and records any Host cancellation fee, section 8). **Emails:** booking request, booking confirmed, **payment receipt** (GST), **cancellation**, declined/expired, payment failed, refund issued. | ⬜ |
| 12–14 | **Public pages:** How It Works, Become a Host (earnings estimator), Safety, Insurance / Protection, FAQs (from the database, with FAQPage JSON-LD), About Us, Contact Us (form creates a support ticket), and the legal pages from CMS Markdown: **Terms & Conditions, Privacy Policy, Cancellation Policy, Host Agreement, Guest Agreement**. Legal text is a placeholder until the client supplies it (section 16, item 8). | ⬜ |
| 6–14 | Signature animations built with each screen and checked against the prototype: cinematic homepage, page transitions, photo gallery, search filters, booking steps (section 12.4). | ⬜ |
| 15 | QA pass (lint, typecheck, tests, Playwright for sign-up, login and a full Guest booking; screenshots on phone, tablet and desktop) + **end-to-end booking sign-off** on staging. | ⬜ QA · 👤 Client |

### Phase 3: Dashboards, Trust & Operations (Days 16–24)
**Delivers:** all three dashboards working on staging, with the complete trip lifecycle: book, pick up, return, review and payout.

| Day | Tasks | Status |
|---|---|---|
| 16–18 | **Guest dashboard (all 13 items in spec §8):** upcoming, current, completed and cancelled trips; booking details with **receipts** (GST receipt page, printable and downloadable as PDF); **messages**; **saved cars** (with estimated totals for the last searched dates, for comparing); **payment methods** and payment history; **personal details**; **driver licence and identity verification**; **reviews**; **notifications**; **help and support** (help articles, contact support about a booking, my support tickets). Trip detail with cancel and refund preview. | ⬜ |
| 16–19 | **Host dashboard (all 10 items in spec §9):** my vehicles with status (Active, Inactive, Under Review, Suspended) and activate/deactivate; bookings (requests with a countdown, upcoming, current, completed, cancelled); calendar and availability; **earnings dashboard** (today, this week, this month, current vs previous month, lifetime, upcoming payouts, platform fees, per-booking breakdown, monthly chart, and a **GST-ready earnings statement** as a CSV download by month or NZ tax year (1 April–31 March), with rental, delivery, extra charges, platform fees, deductions and GST shown separately for the Host's records and tax return, spec §23); upcoming and paid payouts; messages; reviews; **vehicle maintenance and document reminders**; Host profile and settings. | ⬜ |
| 17–18 | Stripe Connect Express onboarding + `payout.transfer` job (24 h after trip start, held while an incident or dispute is open). **Host payouts, refunds and cancellation fees** shown to both parties, including Host cancellation fees deducted from the next payout. Extra kilometre charges after check-out, admin extra charges from resolved incidents, failed-payment follow-up, and card dispute alerts (section 8). | ⬜ |
| 17–19 | **Messaging (spec §13):** a chat thread for each booking, live through Socket.IO (MongoDB adapter), text and **photo attachments**, automated booking messages including the **pickup and return reminders**, email and opt-in SMS for unread messages, **report and block** (a report goes to support; blocking stops messages from that user, while booking-critical system messages still arrive). | ⬜ |
| 19–20 | **Trust and verification (spec §22):** Guest identity check (Stripe Identity: ID + selfie) and **driver licence** details (NZ licence number, version and class, or an overseas licence with an IDP or approved English translation when it is not in English) checked against the eligibility rules in settings; **Host identity verification** as part of the Host application; vehicle document verification by support staff; the checkout verification step connected; manual review queue for support staff. | ⬜ |
| 19–21 | **Digital vehicle handover (spec §14):** check-in by the Host before the trip with guided, **timestamped photos** of the front, rear, both sides, wheels, windscreen, interior and dashboard; **odometer**; **fuel or battery %**; **existing damage** pinned on a car diagram; the Guest reviews and confirms. Check-out repeats the inspection, shows each check-in photo next to the new one, lets either party **flag new damage** (which can open an incident in one tap), and creates the condition record linked to the booking. Moves the trip from Confirmed to Active to Completed. | ⬜ |
| 20–21 | **Damage and incident reporting (spec §15):** Guest or Host reports, incident type, description, photo and document upload, **case number**, status workflow, assignment to support staff, admin updates visible to both parties, one party or internal only, and a **complete audit trail** of every event. Resolving a case can add an extra charge to the Guest (fuel, cleaning, late return, damage), linked to the case (section 8). | ⬜ |
| 20–22 | **Help and support:** help centre (help articles for Guests and Hosts), support ticket inbox for support staff (from the contact form, the dashboards and bookings). **Suspicious activity monitoring:** risk flags (many failed payments, booking velocity, card country different from the account, Stripe Radar warnings, repeated reports, repeated Host cancellations) shown to admins for review. | ⬜ |
| 21–22 | **Two-way reviews and ratings (spec §16)** after a completed trip: overall, communication, cleanliness/condition, pickup/return, vehicle care/Guest behaviour, written review. **Moderation under defined rules:** reviews with contact details, links or abusive language are held for review, anyone can report a review, and admins hide reviews with a recorded reason. **Notifications:** in-app notification centre, email, **SMS pickup and return reminders**, notification preferences. | ⬜ |
| 19–23 | **Admin dashboard (spec §18).** Overview: total users, active Hosts, active vehicles, upcoming bookings, booking revenue, platform fees, Host payouts, cancellations, incident cases, pending verifications, suspended users and vehicles. Capabilities: search and manage users, **approve or reject Host applications**, **approve or reject vehicle listings** (including new photos, new documents and key-detail changes on live listings), view and edit booking status (with calendar override), **suspend users**, **suspend vehicles**, manage disputes and incidents, **issue refunds where authorised**, **payments and payouts** (all payments, failed payments and unpaid extra charges; adding an extra charge from a resolved incident; scheduled, held, paid and failed Host payouts; waiving Host cancellation fees), manage fees and platform settings, manage **homepage content** and destination landing pages, manage **FAQs and help articles**, **platform reports** with CSV export (bookings, revenue, fees, payouts, cancellations, GST summary, built with MongoDB aggregations), **audit log**, staff roles and permissions, failed jobs with retry. | ⬜ |
| 22–23 | **Design polish** pass across all screens with real content, on real phones and tablets. Design QA against Figma by the designer. | ⬜ · 👤 Designer |
| 24 | QA pass on the full trip lifecycle (book, pick up, return, review, payout) + **dashboards and operational workflows sign-off**. | ⬜ QA · 👤 Client |

### Phase 4: Testing, Launch & Handover (Days 25–30)
**Delivers:** a live website on the client's domain, the admin guide and technical documentation, and access to the source code and all hosting accounts.

| Day | Tasks | Status |
|---|---|---|
| 25 | Playwright E2E following the **Guest and Host journeys in spec §28** from start to finish (search → book → pay → pickup → return → review; Host account → verification → add vehicle → publish → booking → handover → payout → review), plus cancel and refund, incident report and admin approval. | ⬜ |
| 25–26 | **Testing on mobile, tablet and desktop**, on **Chrome, Safari (iOS and macOS) and Edge**, plus Android Chrome. Bug fixes. | ⬜ |
| 26 | **Security review** (OWASP checklist, NoSQL injection checks, headers, rate limits, permission checks for support staff, `npm audit`). **Speed optimisation** against section 12.5: **Lighthouse 90+ on mobile**, Core Web Vitals, bundle size, image optimisation, MongoDB index review with `explain()`. 60 fps animation check on a mid-range Android phone, reduced-motion check. | ⬜ |
| 26–27 | **SEO:** unique page titles, meta descriptions, structured data (JSON-LD) for every public route, social sharing previews (OG images), sitemap, robots.txt, **landing pages for Auckland, Wellington, Christchurch, Queenstown and Rotorua**, **Google Search Console**. **Analytics and conversion tracking:** GA4 events for search, listing view, checkout started, booking paid, Host sign-up and listing submitted. | ⬜ |
| 27 | **Email domain setup** (SPF, DKIM, DMARC on the sending subdomain). **Production:** Atlas cluster in Sydney with network access restricted, deploy on the client's domain with **HTTPS**, **automated backups** of the database and of uploaded files (Cloudinary backup) verified with a test restore, **Sentry error monitoring** live. | ⬜ |
| 28–29 | **User acceptance testing** with the client's team, then bug fixes. | 👤 Client · ⬜ Fixes |
| 29 | **Admin training** session (admins and support staff) + **documentation**: admin guide, technical docs (README with setup, folder map and commands; OpenAPI docs; runbook for deploys, backups and failed jobs; data retention). | ⬜ |
| 30 | **Go-live approval** + handover of source code and all hosting accounts. The 30 days of post-launch bug-fix support start. | 👤 Client |

### Not included (future phases)
From MILESTONES.md:
- **Search and notifications:** map-based search and mobile push notifications.
- **Apps:** native iOS and Android apps.
- **Growth features:** dynamic pricing, and referral and loyalty programmes.
- **Business tools:** corporate accounts, fleet tools and advanced fraud detection.

Also from the spec's launch phases (§27): airport automation, third-party API integrations, and advanced analytics beyond the essential reports. Section 10.4 shows how the launch build is ready for each of these.

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
- **Client inputs** (by Day 1, Day 2, Day 15 and Day 24) are listed in section 16. Feedback within 24 hours at each review point; any delay moves the launch date by the same number of days.
- **Design scope:** high-fidelity Figma designs cover the key pages. The other flows get mid-fidelity designs and are built directly from the design system (section 12.3).
- **Reports scope:** the admin dashboard has essential reports with CSV export. Advanced analytics come later.
- **Not included:** see the end of section 9 and section 10.4.

### 10.2 Client specification
Every section of the client's website specification, with where it is covered in this plan and when it is built.

| Spec section | Where it is covered | Days |
|---|---|---|
| §1 Project overview: two-sided NZ marketplace, original branding, NZD | Whole plan; 5, 12.1 | 1–30 |
| §2 User types: Guest, Host, Administrator, Support/Operations staff | 6.2 roles and permissions | 3–5, 19–23 |
| §3 Public pages (all 18, including Browse Cars, Login, Sign Up and the 5 legal pages) | 1.4, 2.2, section 9 Phase 2 | 3–5, 7–14 |
| §4 Homepage: headline, search module with dates and times, Become a Host call to action, all 8 content sections, footer | Section 9 (Days 7–9), 12.6 | 7–9 |
| §5 Search results and Browse Cars: all card fields and all 16 filters | 3 (location search), section 9 (Days 6–9), 12.6 | 6–9 |
| §6 Vehicle listing: all 9 photo types and all vehicle information, incl. engine/powertrain, rego and WOF | 3 (vehicles), section 9 (Days 8–10), 12.6 | 8–10 |
| §7 Booking flow (all 11 steps) and price breakdown with mandatory and optional charges | 5, 6.1, 8, section 9 (Days 11–13), 12.6 | 10–13, 19–20 |
| §8 Guest dashboard (all 13 items, incl. messages and help and support) | Section 9 (Days 16–18), 12.6 | 16–18 |
| §9 Host dashboard (all 10 items, incl. maintenance reminders and Host profile) | Section 9 (Days 16–19), 4.3, 12.6 | 16–19 |
| §10 Host earnings dashboard (all 8 figures, plus a downloadable GST-ready earnings statement) | Section 9 (Days 16–19), 12.6 | 16–19 |
| §11 Host onboarding: all 6 steps and their fields (incl. VIN or chassis number), the minimum photo set, photo quality flags sent for review, custom delivery locations, plus the details the filters and listing page need (body type, powertrain, features, extras, fuel policy) | 3 (vehicles), section 9 (Days 8–11), 12.6 | 8–11 |
| §12 Calendar and availability: month/week views, automatic and manual blocks, recurring availability, notice, buffer, no double-booking, admin override | 3 (key rules), 4.3 | 10–11 |
| §13 Messaging: threads, photos, automated messages, reminders, email/SMS alerts, report and block | 4.3, 4.4, 7, section 9 (Days 17–19) | 17–19 |
| §14 Digital handover: check-in and check-out with all listed angles, readings, damage, timestamps, confirmation, new-damage flag | 3 (conditionReports), section 9 (Days 19–21), 12.6 | 19–21 |
| §15 Damage and incident reporting: types, evidence, case number, admin communication with both parties, audit trail | 3 (incidents), section 9 (Days 20–21) | 20–21 |
| §16 Reviews and ratings: all 5 rating categories, written review, moderation rules | 3 (reviews), section 9 (Days 21–22) | 21–22 |
| §17 Payments and payouts: cards, Apple Pay, Google Pay, NZD, fees, payouts, full and partial refunds, cancellation fees for Guest and Host cancellations, post-trip charges (extra km, charges from resolved incidents), receipts and history, failed payments, admin payments and payouts view | 5, 8, section 9 (Days 13–14, 17–18, 19–23) | 11–14, 17–18, 19–23 |
| §18 Admin dashboard: all 11 overview figures and all 13 capabilities | 6.2, section 9 (Days 19–23), 12.6 | 19–23 |
| §19 Notifications: email, SMS, in-app, all example notifications | 7 | 3–5, 11–14, 21–22 |
| §19 Push notifications (mobile apps / PWA) | Not in the 30 days (10.4); architecture ready, and the web app manifest ships at launch (Days 3–5) | Later |
| §20 Mobile-first design: responsive phone/tablet/desktop, touch targets, simple navigation, fast photos, sticky mobile CTA, camera upload, fast checkout | 12.2, 12.5, 12.6 | Throughout, 25–26 |
| §21 Search and location: autocomplete, cities, suburbs, destinations, distance, airport search (including cars that deliver to the airport) and delivery options, NZ addresses | 1.2, 3 (location and airport search), section 9 (Days 6–8) | 6–8 |
| §21 Map-based browsing ("future or optional" in the spec) | Not in the 30 days (10.4) | Later |
| §22 Trust, safety and verification: email, mobile, identity, licence, Host identity, vehicle documents, listing moderation (new listings and changes to live listings), secure payments, suspicious-activity monitoring, incident reporting, safety pages | 3 (changes to live listings), 6, 8, 14, section 9 (Days 8–11, 19–22) | 3–5, 8–11, 19–22 |
| §23 NZ-specific: NZD, NZ dates and times, NZ addresses, rego and WOF, NZ licence workflow, GST-ready reports (GST receipts, admin GST summary, Host earnings statements by month or NZ tax year), NZ legal documents, NZ protection info, airport rentals (airport search and delivery) and tourist destinations, local and international visitors (overseas phone numbers and licences) | 3, 5, 6.1, 12.7, 16, section 9 (Days 6–8, 16–19, 19–23) | Throughout |
| §24 SEO and marketing: indexable vehicle, city and destination pages, unique titles and descriptions, structured data, social previews, analytics, conversion tracking, Search Console, city landing pages | 1.4, section 9 (Days 26–27) | 26–27 |
| §25 Technical and performance: HTTPS, scalable backend, role-based access, secure auth, encryption, automated backups (database and uploaded photos and documents), audit logs, image CDN, speed, error monitoring, API-ready | 6, 12.5, 13, 14 | Throughout |
| §26 Future mobile apps: shared accounts, bookings, push, Host calendar, photo uploads, messaging, check-in/out | 11.1 | Architecture from Day 1 |
| §27 Suggested launch phases | 10.3 | – |
| §28 Guest and Host journeys, including comparing vehicles (result cards and Saved cars) | Section 9 (Day 25 E2E tests follow both journeys), 12.6 (comparing cars) | 6–9, 16–18, 25 |
| §29 Design direction: premium but approachable, original NZ identity, visible trust, photography, transparent pricing, one primary action, mobile first | 12.1 | 1–5, 22–23 |
| §30 Designer/developer deliverables: desktop, mobile and tablet designs, clickable prototype of the key screens and animations, design system, all states, dashboard and flow designs, email templates, developer-ready files | Section 9 (Days 1–8), 12 | 1–8 |
| §31 Questions for the development team | 18 | – |
| §32 Final product goal: low-friction booking for Guests, simple dashboard for Hosts | Whole plan | – |

### 10.3 The spec's launch phases (§27) and this plan

| Spec phase | In the 30-day build | Later |
|---|---|---|
| Phase 1 – MVP | All of it: homepage, search and listings, vehicle pages, Guest and Host accounts, onboarding, calendar, bookings, payments, messaging, reviews, admin dashboard, notifications | – |
| Phase 2 – Trust & Operations | Driver/identity verification, digital inspection, damage reporting, advanced Host earnings, advanced admin tools, refund and dispute workflows | Map search, advanced analytics |
| Phase 3 – Growth | – | Native apps, dynamic pricing, airport automation, fleet tools, referrals, loyalty, corporate accounts, API integrations, advanced fraud detection |

### 10.4 Spec items outside the 30 days, and how the build is ready for them
These are excluded in MILESTONES.md. Adding any of them to the 30 days needs a change request.

| Item (spec §) | Ready in the launch build |
|---|---|
| Push notifications for mobile apps / PWA (§13, §19, §26) | `notify()` sends through one adapter per channel; a push adapter and device-token storage are added without changing any notification events. The web app manifest ships at launch, so PWA web push then only needs a service worker and the push adapter. |
| Map-based vehicle browsing (§21, §27) | Search already returns coordinates and distances from a `2dsphere` index, so a map view only needs the frontend and a map key |
| Advanced analytics (§27) | All events are in MongoDB and GA4; reports are MongoDB aggregations that can be extended |
| Native iOS and Android apps (§26, §27) | REST API with bearer tokens, shared Zod contracts, Socket.IO (section 11.1) |
| Dynamic pricing, airport automation, fleet tools, referrals, loyalty, corporate accounts, API integrations, advanced fraud detection (§27) | Pricing is one shared function, airports are data, roles and permissions are extensible, and risk flags are already recorded |

---

## 11. API Overview (REST, `/api/v1`)

```
POST   /auth/signup | /auth/login | /auth/logout | /auth/refresh
POST   /auth/verify-email | /auth/forgot-password | /auth/reset-password | /auth/phone/otp
GET    /me   PATCH /me   PATCH /me/notification-prefs   POST /me/agreements
GET    /me/favourites   PUT|DELETE /me/favourites/:vehicleId
GET    /me/payment-methods   POST /me/payment-methods/setup   DELETE /me/payment-methods/:id
GET    /me/payments                      (payment history, receipts, refunds)
POST   /me/verification                  (starts Stripe Identity, saves licence details)
POST   /me/host-application              (Host profile + Host Agreement)
GET    /notifications   POST /notifications/read
GET    /search?lat&lng&radius&start&end&filters...   (dates optional: Browse Cars)
GET    /places/suggest?q=                (our destinations and airports + Google Places)
GET    /vehicles/featured       (homepage featured vehicles)
GET    /vehicles/:slug          GET /vehicles/:id/availability?from&to   GET /vehicles/:id/reviews
POST   /vehicles/:id/quote      → price breakdown
GET    /host/vehicles           (My Vehicles, with status)
POST   /host/vehicles           PATCH /host/vehicles/:id   POST /host/vehicles/:id/photos|documents|submit
POST   /host/vehicles/:id/blocks          DELETE /host/vehicles/:id/blocks/:blockId
PUT    /host/vehicles/:id/recurring-rules   PUT /host/vehicles/:id/maintenance-reminders
GET    /host/earnings?period=   GET /host/earnings/statement?from&to   (CSV)
GET    /host/payouts            POST /host/connect/onboarding-link
POST   /uploads/signature       (signed Cloudinary upload: vehicle photos and documents, message attachments,
                                 inspection photos, incident evidence, support ticket attachments)
POST   /bookings                GET /bookings?role=guest|host&status=   GET /bookings/:id   GET /bookings/:id/receipt
POST   /bookings/:id/accept|decline|cancel
GET    /bookings/:id/inspections          POST /bookings/:id/inspections   (check-in / check-out)
POST   /bookings/:id/inspections/:stage/confirm   (the other party reviews and confirms the condition report)
GET    /threads  GET /threads/:id/messages  POST /threads/:id/messages  POST /threads/:id/read
POST   /reports                 (report a user, message, review or vehicle)
POST   /users/:id/block         DELETE /users/:id/block
POST   /reviews                 GET /users/:id/reviews
POST   /incidents               GET /incidents   GET /incidents/:ref   POST /incidents/:ref/events
GET    /help/articles           GET /help/articles/:slug
POST   /support/tickets         GET /support/tickets   GET /support/tickets/:ref   (contact form works signed out)
GET    /destinations            GET /destinations/:slug   GET /faqs   GET /cms/:key
POST   /payments/webhook        (Stripe payments, Connect, Identity and dispute events)
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
- **Backend:** MongoDB indexes checked with `explain()`, a 60 s in-memory cache per instance for homepage content, featured vehicles and popular searches, and short Cloudflare caching of public GET responses. Availability is always re-checked at quote and booking time. Brotli compression and HTTP/3 come through Cloudflare.
- **Enforced in CI:** Lighthouse CI and a bundle size check run on every pull request and fail the build when a budget is broken. Real-user Core Web Vitals are tracked in Sentry.

### 12.6 Key screen layouts (mobile first)

**Navigation** (spec §20, simple navigation): the header holds Browse Cars, How It Works, Become a Host, Help and Log in (or the account menu when signed in). On mobile the ☰ menu holds the same links, so the header shows only the logo and Log in. Dashboards use the tab bars described below.

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

The Become a Host call to action sits directly under the search panel, as in spec §4. Hero text, featured vehicles, destination tiles, the customer reviews shown (picked from published reviews) and footer links are editable by admins (`cmsBlocks`, `destinations`).

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

**Vehicle listing**: full-width swipeable gallery → make, model, year and variant, rating and location → Host card with a verified badge, rating and trips → specs grid (transmission, fuel type, engine/powertrain, seats, doors) → features → rego and WOF → policies (fuel, km allowance, cancellation) → pickup/delivery options → protection → reviews → location map. On desktop, a sticky right-hand booking panel. On mobile, a **sticky bottom bar** showing "$89/day · $267 total [Book]".

**Checkout** (single page, collapsible sections, following spec §7)
1. Trip: pick-up and return location, date and time (editable), with the availability check
2. Protection plan: radio cards with a clear summary of cover
3. Trip details and policies: fuel, kilometres, cancellation
4. Sign in or create an account (skipped when already signed in)
5. Verification: licence details and identity check, where required (skipped when already verified)
6. Payment: saved cards or a new card with the Stripe Payment Element, Apple Pay and Google Pay buttons
7. Price breakdown (spec §7): **Mandatory** (rental, service fee, and protection when the plan is mandatory) and **Optional** (delivery or airport delivery fee, and protection when the Guest chooses a plan), then the GST included in the total on its own line, with the **Total NZD** in bold
8. Guest Agreement checkbox, then **Confirm and pay**

**Host onboarding**: the Host application first (profile, Host Agreement, identity check), then a 6-step progress stepper (vehicle details, documents, photos, pricing, availability, pickup and delivery), one topic per screen, Save & exit on every step, and example photos for each required angle.

**Guest and Host dashboards**: sidebar navigation on desktop and tablet, bottom tab bar on mobile (Trips · Messages · Saved · Account for Guests; Today · Vehicles · Calendar · Earnings · Inbox for Hosts). The Guest account area holds receipts, payment methods and history, personal details, licence verification, reviews, notifications and help and support. The Host area holds bookings by status, reviews, reminders, and profile and settings. The Host earnings page shows stat cards (today, week, month with % change vs last month, lifetime), a bar chart of monthly earnings, platform fees, upcoming payouts, a per-booking table and a **Download statement** button (CSV by month or NZ tax year).

**Admin dashboard**: desktop-first sidebar (Overview, Users, Host applications, Vehicles, Verifications, Bookings, Payments & payouts, Refunds, Incidents & disputes, Support, Moderation, Risk, Content, FAQs & help, Settings, Reports, Staff, Audit log, Jobs). Overview shows the KPI stat cards from spec §18. Each list is a filterable data table with a detail drawer, and reports export to CSV. Support staff see only the sections their role allows.

**Inspection (check-in/out)**: a full-screen camera guide per angle (front, rear, driver side, passenger side, wheels, windscreen, interior, dashboard) with an overlay silhouette and a progress count ("Front 1/8"). Then odometer and fuel/battery inputs, damage pins on a car diagram, a review screen, and Guest confirmation. At check-out, each angle shows the check-in photo next to the new one, with a "Flag new damage" button.

**Incident**: case number and status at the top, then a timeline of every event with evidence photos, and a reply box for updates.

### 12.7 Accessibility & content
- WCAG 2.2 AA: contrast of 4.5:1 or more, keyboard navigation, focus rings, labelled inputs, alt text.
- Champagne gold is decorative on light backgrounds because its contrast is too low for text there. Gold text uses `--color-gold-text` or sits on ink or green.
- Reduced-motion preferences are respected (see 12.4). Videos have no sound and can be paused.
- NZ English copy ("kilometres", "licence", "tyres"). Prices shown as `$89` with "NZD" on totals. NZ date and time formats.
- Plain language suitable for local customers and international visitors, including clear guidance on overseas licences, IDPs and approved translations, and phone verification that works with overseas numbers.

---

## 13. Deployment Architecture

**Recommended for launch (simple, low cost, scales to thousands of vehicles):**

```
Cloudflare DNS + CDN (HTTPS, Brotli, hashed JS/CSS/image assets cached at the edge)
 └─ App service (server/)   one Node.js service, 2 instances:
                            · REST API (/api/v1) + Socket.IO
                            · serves the React build, with SEO meta tags, sitemap.xml and robots.txt
                            · background job runner (MongoDB jobs collection)

Used by the app service:
 ├─ MongoDB Atlas           M10+ dedicated cluster, AWS Sydney, continuous cloud backup
 ├─ Cloudinary              images and documents
 └─ Stripe, Resend, Twilio, Google Places, Sentry
```

- **Hosting:** one Node web service on Render or Railway, built with `npm ci && npm run build` and started with `npm start`. Two instances for zero-downtime deploys; the Socket.IO MongoDB adapter and the atomic job claims keep them in step.
- **Region:** Sydney (`ap-southeast-2`), the closest to NZ, for low latency. The Atlas cluster and the service are in the same region.
- **Environments:** `local` → `staging` (auto-deploy from `develop`) → `production` (deploy from `main` after approval). Each environment has its own Atlas database and credentials.
- **CI (GitHub Actions):** `npm ci` (npm cache) → lint, typecheck, test (API and services against an in-memory MongoDB replica set) → build → Lighthouse CI and bundle size check → Playwright E2E → `npm run db:indexes` against the target database → deploy → smoke test.
- **Secrets** are kept in the platform environment settings, never in git.
- **Backups:** Atlas daily snapshots retained for 30 days, plus continuous backup for point-in-time recovery. Uploaded files (vehicle photos, vehicle documents, inspection photos, incident evidence, message attachments) are not in the database, so Cloudinary's automatic backup is turned on to keep a backup copy of every upload; a file deleted or overwritten by mistake can be restored. A restore of both the database and a sample of files is tested before launch.
- **Scale path from 100 to 10,000+ vehicles:** raise the Atlas cluster tier (auto-scaling), add more app instances, run the job runner as its own process from the same code (`RUN_JOBS=true` on it, `false` on the web instances), send search and reporting reads to secondary nodes, add Atlas Search for heavy filtering, and serve the React build from a static CDN host.

**Estimated monthly running cost at launch:** about NZD $150–400 (hosting, MongoDB Atlas M10, email, monitoring), plus usage-based Stripe, SMS and Maps fees. The Atlas cluster is the largest single item.

---

## 14. Security & Compliance Checklist
- HTTPS everywhere, HSTS, Helmet security headers, strict CORS.
- Zod validation on every input. Mongoose `sanitizeFilter` and `strictQuery` block NoSQL operator injection (for example, a login body of `{ "email": { "$ne": null } }`).
- Role and permission checks on every API route; support staff get only the permissions they need (section 6.2).
- MongoDB Atlas access limited to the hosting provider's IP addresses, with a separate database user per environment and only the permissions it needs.
- Licence and ID documents stored privately and accessed through signed URLs only. Sensitive fields such as licence numbers are encrypted in the application (AES-256-GCM) before they are saved. Atlas also encrypts all data at rest.
- PCI scope is kept minimal: card data never reaches our servers (Stripe Elements). 3-D Secure where the card requires it.
- The client cannot import server code (section 2.3), and only `VITE_` variables reach the browser, so secrets cannot end up in the browser bundle.
- **Fraud and suspicious activity:** email, mobile, identity and licence verification; Stripe Radar; risk flags for booking velocity, repeated failed payments, card country different from the account, repeated user reports and repeated Host cancellations, reviewed by admins. Advanced fraud detection comes later (spec §27).
- **Messaging safety:** report and block, rate limits on messages, and all messages kept on the platform as evidence for incidents.
- **Legal records:** acceptance of each version of the Terms, Privacy Policy and Host and Guest Agreements is stored with time and IP.
- **NZ Privacy Act 2020:** privacy policy, data access and deletion requests, and a breach notification process.
- **Data retention (proposed, to be confirmed by the client's legal adviser):** bookings and financial records for 7 years (NZ tax requirement); ID images deleted 90 days after verification, including their backup copies, keeping only the result; inspection photos, messages and incident records for 2 years after the trip, or longer while a case is open; deleted accounts anonymised except for records the law requires.
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

---

## 16. Open Decisions & Client Inputs
| # | Decision or input | Needed by |
|---|---|---|
| 1 | Business name, logo, domain name | Day 1 |
| 2 | Guest service fee % and Host commission % | Day 1 |
| 3 | Cancellation policy tiers, and the fee (if any) when a Host cancels a confirmed booking | Day 1 |
| 4 | Security deposit (yes/no, amount) | Day 1 |
| 5 | Driver eligibility: minimum age, licence classes accepted (NZ full/restricted, overseas, with an IDP or approved translation when not in English), years held, and which Guests must complete identity verification before booking | Day 1 |
| 6 | Third-party accounts: Stripe (payments), Resend (email), Twilio (SMS), plus Cloudinary, Google Cloud and MongoDB Atlas | Day 2 |
| 7 | GST treatment of the rental amount, platform fees and Host payouts (client's accountant) | Day 10 |
| 8 | Support contact details and social media links for the footer and Contact page | Day 12 |
| 9 | Insurance and protection details from the insurance partner | Day 15 |
| 10 | Review moderation rules and prohibited content (what gets a review held or hidden) | Day 20 |
| 11 | Legal text (Terms & Conditions, Privacy Policy, Cancellation Policy, Host and Guest Agreements), reviewed by the client's legal adviser, plus data retention periods | Day 24 |
| 12 | Confirmation from the client's legal, insurance and compliance advisers of the final verification, insurance/protection and eligibility rules (spec §22). Any changes are applied in `platformSettings` without code changes. | Day 28 |

## 17. Risks & Mitigations
| Risk | Mitigation |
|---|---|
| Insurance details arrive on Day 15, after the booking flow is built (Days 10–13) | Protection plans are config-driven: placeholder plans are used until then and replaced from the admin settings without code changes |
| Scope creep from items outside the 30 days (push notifications, map search, advanced analytics, growth features) | They are listed in section 10.4 with how the build is ready for them. Changes go through a change request. |
| 30-day timeline slips | Parallel frontend and backend work, a ready-made UI kit, a simple stack with one deployable service, 24 h client feedback, and a daily stand-up with a scope check |
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

---

## 18. Answers to the Client's Questions (spec §31)

| Question | Answer | Details |
|---|---|---|
| What technology stack do you recommend and why? | React + Vite (TypeScript) for the website, Node.js + Express for the API, MongoDB Atlas for data. One language across the whole product, a fast and animated UI, flexible documents with built-in location search and transactions, one service to deploy, and an API that native apps can reuse. | 1, 2 |
| What payment provider will be used? | Stripe: NZD, cards, Apple Pay and Google Pay, Stripe Connect for Host payouts, Stripe Identity for verification, Radar for fraud screening. | 8 |
| How will driver and identity verification be implemented? | Stripe Identity checks an ID document and a selfie. The Guest enters licence number, version, class and expiry (or an overseas licence, with an IDP or approved translation if it is not in English), checked against the eligibility rules. Support staff review anything the automatic check cannot decide. Hosts complete identity verification in their Host application. | 6, 9 (Days 19–20) |
| How will vehicle data be verified? | Hosts enter rego, VIN (or the chassis number for imports that have no VIN), WOF and rego expiry, and upload registration, WOF and insurance documents. Support staff check the documents against the details before approving the listing. New photos and documents on a live listing are checked before they appear, and a change to the rego, VIN, chassis number or vehicle details sends the listing back for review. Listings are hidden from search when the WOF or rego expires, and Hosts get reminders before that. | 3, 4.3 |
| How will insurance/protection information be integrated? | Protection plans (price, cover summary, excess, mandatory or optional) are set in the admin settings, shown at checkout and on the Insurance / Protection page, and saved on each booking. Details come from the client's insurance partner by Day 15. If the partner offers an API, it can be connected later; until then, bookings can be exported for the insurer. | 5, 16 |
| How will host payouts work? | Hosts connect their NZ bank account through Stripe Connect Express. 24 hours after each trip starts, the Host's share (rental and delivery less commission) is transferred automatically. Payouts are held while an incident or card dispute is open. | 8 |
| How will cancellations and refunds work? | Cancellation tiers set by the client (e.g. Flexible, Moderate, Strict) with a short grace period. The system calculates the refund and cancellation fee and refunds the card through Stripe, fully or partly. If the Host cancels, the Guest is refunded in full, and any Host cancellation fee is deducted from the Host's next payout. Admins, and support staff with permission, can issue manual refunds. Every refund is logged. | 5, 8 |
| How will fraud prevention be handled? | Email, mobile, identity and licence verification; Stripe Radar and 3-D Secure; risk flags (booking velocity, failed payments, card country mismatch, repeated reports) reviewed by admins; listing moderation; audit logs. Advanced fraud detection is a later phase. | 14 |
| What data will be stored and for how long? | Accounts, verification results, vehicles and documents, bookings, payments, messages, inspection photos, incidents, reviews and audit logs, all in MongoDB Atlas (Sydney) and Cloudinary. Proposed retention: financial records 7 years, ID images 90 days after verification, inspection photos, messages and incidents 2 years after the trip. To be confirmed by the client's legal adviser. | 3, 14 |
| How will the platform scale from 100 to 10,000+ vehicles? | Indexed location search, a larger Atlas tier, more app instances, the job runner in its own process, read replicas for search and reports, and Atlas Search for heavy filtering, all without rebuilding. | 13 |
| What is included in the initial MVP? | All of the spec's Phase 1 (MVP) and most of Phase 2 (verification, inspections, damage reporting, advanced earnings and admin tools, refund and dispute workflows) in 30 days. | 10.3 |
| What third-party integrations are required? | Stripe (payments, Connect, Identity, Radar), Resend (email), Twilio (SMS), Cloudinary (images), Google Places (location search), MongoDB Atlas (database), Sentry (errors), GA4 and Search Console (analytics and SEO), Cloudflare (DNS and CDN), Render or Railway (hosting). | 1.2, 13 |
| What ongoing hosting, maintenance and support costs should be expected? | About NZD $150–400 a month for hosting and services at launch, plus usage-based Stripe, SMS and Maps fees. 30 days of bug-fix support are included after launch; ongoing maintenance and support after that is quoted separately. | 13 |
