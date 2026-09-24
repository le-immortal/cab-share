# Cab Share: Product & Technical Specification (v1)

## 1. Overview

Cab Share lets verified employees of pre-approved partner companies post and
find shared cab rides (e.g. "heading to the airport Friday 6pm", "landing
Tuesday morning, need a ride into the city"), so they can split fares with
someone from a trusted, work-verified pool instead of a stranger off the
street.

v1 targets **one city**, supports **general point-to-point trips** (not just
airport runs, though airport trips are expected to be the majority use case),
and ships as **both** a website and a Teams bot from day one, sharing one
backend.

## 2. Goals (v1)

- Let a verified employee post a trip: origin, destination, date, time, seats.
- Let anyone verified search/browse trips by origin, destination, date, time.
- Let a user save a search so they're notified when a matching trip appears.
- Let a user request to join a trip, and the poster accept/decline.
- Give matched users an in-app chat to coordinate, with no forced contact-info
  sharing.
- Verify every user is a real employee of a pre-approved company via a
  work-email OTP check.
- Notify users over email and Teams.

## 3. Non-Goals (explicitly out of scope for v1)

- Payment processing or in-app fare splitting: riders settle the fare
  between themselves offline.
- Ratings/reviews, gender-based matching/filtering: v1 trust & safety is
  limited to report/block.
- Multi-city support: single city launch only.
- Recurring/daily-commute trip templates: every trip is posted one-off in
  v1 (flagged as a likely fast-follow, see §11).
- SMS or mobile push notifications.
- Native mobile apps.

## 4. User Roles

| Role | Description |
|---|---|
| **Member** | Verified employee of a partner company. Can post, search, request to join, chat, report. |
| **Admin** | Manages the partner-company allowlist, reviews reports, can suspend users/trips. |

## 5. Verification & Onboarding

- Signup requires a **work email address**.
- The email's domain must match an entry in a **curated allowlist of partner
  company domains** maintained by admins (e.g. `contoso.com`,
  `fabrikam.com`). Unlisted domains are rejected with a message pointing to a
  "request your company be added" form.
- On match, the system sends a one-time code (OTP) to that work email. The
  user enters the code (on the website) or replies with it (in Teams chat
  with the bot) to complete verification.
- A verified **User** record is created, keyed to the work email + company.
- **Teams linking:** since partner companies are separate Entra/Teams
  tenants, the bot cannot rely on Teams SSO alone. The first time a user
  messages the bot, it runs the same work-email + OTP flow and links the
  resulting Teams conversation reference to their User record. This is what
  allows proactive Teams notifications later.

## 6. Core Features

### 6.1 Trip posting
A trip has: origin (free-text + geocoded point), destination (free-text +
geocoded point), date, time (single time; "flexible ±30 min" toggle),
seats available, free-text notes, and status (`open` / `full` / `cancelled`
/ `expired`). Trips auto-expire shortly after their date/time passes.

### 6.2 Search & browse
Filter the trip board by origin (within a radius), destination (within a
radius), date (single day or range), and time window. Results show as a
list (map view is a nice-to-have, not required for v1).

### 6.3 Saved search alerts (opt-in auto-match)
A user can save a search (origin, destination, date range) as an **alert**.
Whenever a new trip is posted that matches an active alert, the alert owner
is notified (email + Teams). This sits alongside manual browsing; it does
not replace it.

### 6.4 Join requests & in-app chat
- A member requests to join an open trip.
- The poster sees the request and **accepts or declines** it.
- Sending a request opens an **in-app chat thread** between the two users
  (scoped to that request), so they can ask questions before/after the
  poster decides.
- Real contact info (phone number, personal email) is never auto-revealed by
  the platform; anything beyond in-app chat is the users' own choice.
- Accepting a request decrements `seatsAvailable`; the trip flips to `full`
  when it hits zero.

### 6.5 Reporting / blocking
Any user or trip can be reported (reason + free-text detail), landing in an
admin queue. Admins can suspend a user or remove a trip. No automated
enforcement in v1.

### 6.6 Notifications
Delivered over **email and Teams** for: OTP codes, new join request, request
accepted/declined, new chat message, saved-search alert match.

## 7. Admin Capabilities

- Add/remove/approve partner company domains.
- View and act on reports (suspend user, remove trip).
- Basic usage view (active users, trips posted/closed).

## 8. Data Model (entities)

- **User**: id, workEmail, companyDomain, displayName, teamsConversationRef?, status, createdAt
- **CompanyDomain**: id, domain, companyName, status, addedBy, createdAt
- **Trip**: id, postedByUserId, origin, originGeo, destination, destinationGeo, date, time, isFlexibleTime, seatsAvailable, notes, status, createdAt
- **JoinRequest**: id, tripId, requestedByUserId, status(pending/accepted/declined/cancelled), createdAt, respondedAt
- **ChatThread**: id, joinRequestId, createdAt
- **ChatMessage**: id, threadId, senderUserId, body, sentAt
- **SavedSearchAlert**: id, userId, originQuery, destinationQuery, dateRangeStart, dateRangeEnd, active, createdAt
- **Report**: id, reportedByUserId, targetType(user/trip/message), targetId, reason, details, status, createdAt

## 9. High-Level API Surface (shared by web + bot)

```
POST   /auth/otp/request          { workEmail }
POST   /auth/otp/confirm          { workEmail, otp }
GET    /companies                 (allowlist lookup during signup)
POST   /admin/companies           (admin: add/approve domain)
GET    /trips?origin&destination&date&radius
POST   /trips
GET    /trips/:id
PATCH  /trips/:id                 (edit/cancel)
POST   /trips/:id/join-requests
PATCH  /join-requests/:id         (accept/decline/cancel)
GET    /chat-threads/:id/messages
POST   /chat-threads/:id/messages
POST   /saved-searches
GET    /saved-searches            (mine)
DELETE /saved-searches/:id
POST   /reports
GET    /admin/reports
```

## 10. Architecture & Tech Stack

- **apps/web**: React (Vite) SPA. OTP-based login, session via JWT/cookie.
- **apps/bot**: Bot Framework SDK (Node/TypeScript), Adaptive Cards for
  trip cards, join requests, and chat; registered as an Azure Bot Service
  resource.
- **apps/api**: Node.js/TypeScript (Express or Fastify) REST API, the
  single source of truth consumed by both the web app and the bot.
- **packages/shared**: shared TypeScript types/DTOs across all three apps.
- **Database**: PostgreSQL (PostGIS extension for radius/geo search),
  accessed via Prisma.
- **Email**: SendGrid or Azure Communication Services Email, for OTP and
  notifications.
- **Hosting**: Azure App Service/Container Apps for `api`/`web`, Azure Bot
  Service for the Teams channel registration.

## 11. Non-Functional Requirements

- **Privacy**: store only what's needed (work email, display name, company,
  trip data). No phone numbers or personal emails are collected by the
  platform itself. Location data is limited to the origin/destination points
  a user explicitly enters.
- **Security**: rate-limit OTP requests/attempts; short OTP expiry; signed
  session tokens; admin actions gated behind an admin role check.
- **Scale target**: single city, low-thousands of users, with no need for
  heavy horizontal scaling in v1.

## 12. Known Platform Risk: Teams distribution across companies

A Teams app normally needs either (a) tenant-admin approval to install
org-wide, or (b) publication to the public Teams App Store so any
individual can add it personally. Since partner companies are separate
tenants and we don't want to chase admin approval per company, **v1 assumes
route (b): publish as a personally-installable Teams app**, relying on the
work-email OTP check (§5), not tenant membership, as the actual trust
boundary. Revisit if a partner company blocks personal app installs
tenant-wide.

## 13. Roadmap: Likely Fast-Follows (post-v1)

- Recurring/daily-commute trip templates.
- Ratings/reviews after a completed trip.
- Optional gender-match filter.
- Multi-city support.
- In-app fare-split calculator (still no real payments).

## 14. Open Assumptions to Confirm During Build

- Single timezone assumed (single-city launch).
- "Flexible ±30 min" is the only time-fuzziness modeled; no recurring
  weekly patterns yet.
- Public Teams App Store distribution (§12) is acceptable to start.
