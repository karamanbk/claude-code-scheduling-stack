# 03 — Data Model

This guide defines the Prisma schema for a **single-tenant** deployment. We deliberately omit organizations, memberships, invitations, billing, plans, trials, and seat counting.

## The core entities

```
User ───────< AvailabilitySchedule
  │
  ├──< CalendarConnection
  ├──< VideoConnection
  └──< Booking (as host)

EventType ──< EventTypeHost >── User      (who can host this event type)
  │
  ├──< Booking
  └──> RoutingRule (optional)

Booking ──< BookingInvitee >── Contact

RoutingRule ──< (config: rules / AI prompt)

Workflow ──< WorkflowEventType >── EventType
  └──< ScheduledWorkflowJob

Settings (single row: branding, integrations config, defaults)
```

## Users

In single-tenant mode, a `User` is simply a team member who can log in and host meetings.

```prisma
model User {
  id            String    @id @default(uuid())
  email         String    @unique
  name          String
  avatarUrl     String?
  passwordHash  String?
  emailVerified DateTime?
  timezone      String    @default("UTC")
  role          UserRole  @default(MEMBER) // ADMIN can edit settings; MEMBER hosts meetings
  dismissedUiItems Json?                   // for onboarding/UI hints
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  calendarConnections   CalendarConnection[]
  videoConnections      VideoConnection[]
  availabilitySchedules AvailabilitySchedule[]
  bookingsAsHost        Booking[]            @relation("host")
  accounts              Account[]
  sessions              Session[]
}

enum UserRole {
  ADMIN
  MEMBER
}
```

(`Account` and `Session` are the standard NextAuth tables — see the auth guide.)

## Settings (the single tenant's config)

Instead of an `Organization` table, keep one `Settings` row for global configuration.

```prisma
model Settings {
  id              String  @id @default(uuid())
  companyName     String  @default("My Company")
  logo            String?
  brandColor      String  @default("#5046e5")
  timezone        String  @default("UTC")

  // Integration config
  llmApiKey       String? @db.Text  // encrypted at rest
  smtpHost        String?
  smtpPort        Int?
  smtpUser        String?
  smtpPass        String? @db.Text  // encrypted at rest
  smtpFrom        String?

  // Enrichment / AI defaults
  enrichmentFields Json?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}
```

Always keep exactly one row. A helper like `getSettings()` that creates the row on first access keeps call sites simple.

## Event types

A bookable meeting template.

```prisma
model EventType {
  id                  String   @id @default(uuid())
  title               String
  slug                String   @unique
  description         String?  @db.Text
  duration            Int      // minutes
  color               String   @default("#5046e5")
  isActive            Boolean  @default(true)
  bookingMode         BookingMode @default(DIRECT) // DIRECT = calendar-first; ROUTER = form/routing first
  schedulingType      SchedulingType @default(SINGLE) // SINGLE host or ROUND_ROBIN
  locationType        LocationType   @default(GOOGLE_MEET)
  customLocation      String?
  bufferBefore        Int      @default(0)
  bufferAfter         Int      @default(0)
  minimumNotice       Int      @default(60)   // minutes
  schedulingWindow    Int      @default(60)   // days into the future
  dailyLimit          Int?
  formFields          Json                    // the booking form definition
  thankYouRedirectUrl String?
  passParamsToThankYou Boolean @default(false)
  smartFormEnabled    Boolean  @default(false)
  smartFormConfig     Json?
  internalNote        String?                 // team-only note, not shown to invitees
  routingRuleId       String?
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  routingRule       RoutingRule?    @relation(fields: [routingRuleId], references: [id])
  hosts             EventTypeHost[]
  bookings          Booking[]
}

enum BookingMode { DIRECT ROUTER }
enum SchedulingType { SINGLE ROUND_ROBIN }
enum LocationType { GOOGLE_MEET ZOOM TEAMS PHONE IN_PERSON CUSTOM }
```

`EventTypeHost` is the join table that says which users can host a given event type (and supports round-robin):

```prisma
model EventTypeHost {
  id          String  @id @default(uuid())
  eventTypeId String
  userId      String
  weight      Int     @default(1)   // round-robin weighting

  eventType EventType @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id])

  @@unique([eventTypeId, userId])
}
```

## Availability

```prisma
model AvailabilitySchedule {
  id        String   @id @default(uuid())
  userId    String
  name      String   @default("Working Hours")
  timezone  String   @default("UTC")
  rules     Json     // weekly rules: [{ day: 1, start: "09:00", end: "17:00" }, ...]
  overrides Json?    // date-specific overrides

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

## Bookings & contacts

```prisma
model Booking {
  id                 String   @id @default(uuid())
  eventTypeId        String
  hostUserId         String
  startTime          DateTime
  endTime            DateTime
  timezone           String
  status             BookingStatus @default(CONFIRMED)
  meetingUrl         String?
  meetingProvider    LocationType?
  hostCalendarEventId String?
  rescheduleToken    String?  @unique @default(uuid())
  crmContactId       String?  // external CRM id, if synced
  crmMeetingId       String?
  createdAt          DateTime @default(now())

  eventType EventType        @relation(fields: [eventTypeId], references: [id])
  host      User             @relation("host", fields: [hostUserId], references: [id])
  invitees  BookingInvitee[]
}

enum BookingStatus { PENDING CONFIRMED CANCELLED }

model Contact {
  id           String  @id @default(uuid())
  email        String
  firstName    String?
  lastName     String?
  phone        String?
  company      String?
  jobTitle     String?
  customFields Json?
  utmSource    String?
  utmMedium    String?
  utmCampaign  String?

  // Enrichment fields (populated by AI/enrichment step)
  companyWebsite     String?
  companyIndustry    String?
  companySize        String?
  personLinkedinUrl  String?
  enrichmentData     Json?
  crmContactId       String?

  createdAt DateTime @default(now())
  bookings  BookingInvitee[]
}

model BookingInvitee {
  id        String @id @default(uuid())
  bookingId String
  contactId String

  booking Booking @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  contact Contact @relation(fields: [contactId], references: [id])
}
```

## Integrations (per-user OAuth tokens)

```prisma
model CalendarConnection {
  id           String   @id @default(uuid())
  userId       String
  provider     String   // "google" | "microsoft"
  accessToken  String   @db.Text  // encrypted
  refreshToken String   @db.Text  // encrypted
  tokenExpiry  DateTime
  calendarId   String?
  isActive     Boolean  @default(true)

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([userId, provider])
}

model VideoConnection {
  id           String   @id @default(uuid())
  userId       String
  provider     String   // "zoom"
  accessToken  String   @db.Text
  refreshToken String   @db.Text
  tokenExpiry  DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([userId, provider])
}
```

## Routing & workflows

These are detailed in their own guides, but the tables are:

```prisma
model RoutingRule {
  id            String   @id @default(uuid())
  name          String
  description   String?
  isActive      Boolean  @default(true)
  aiPrompt      String   @db.Text @default("")
  routingConfig Json?    // mode, formFields, manualRules, AI condition→action map, defaultAction
  internalNote  String?
  createdAt     DateTime @default(now())

  eventTypes EventType[]
}

model Workflow {
  id            String          @id @default(uuid())
  name          String
  isActive      Boolean         @default(true)
  trigger       WorkflowTrigger
  offsetMinutes Int             // negative = before event, positive = after
  action        WorkflowAction  @default(EMAIL)
  emailSubject  String?
  emailBody     String?         @db.Text
  smsBody       String?         @db.Text
  createdAt     DateTime        @default(now())

  eventTypes WorkflowEventType[]
}

enum WorkflowTrigger { BOOKING_CREATED BEFORE_EVENT AFTER_EVENT FORM_NO_BOOKING }
enum WorkflowAction { EMAIL SMS }

model WorkflowEventType {
  id          String @id @default(uuid())
  workflowId  String
  eventTypeId String
  workflow  Workflow  @relation(fields: [workflowId], references: [id], onDelete: Cascade)
  eventType EventType @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  @@unique([workflowId, eventTypeId])
}

model ScheduledWorkflowJob {
  id         String   @id @default(uuid())
  workflowId String
  bookingId  String
  sendAt     DateTime
  status     String   @default("PENDING") // PENDING | SENT | CANCELLED | FAILED
  attempts   Int      @default(0)
  createdAt  DateTime @default(now())
}
```

## Migrate

```bash
npx prisma migrate dev --name core-model
npx prisma generate
```

## Working with Claude Code

Implement the schema in slices: users + settings + auth tables first (migrate, verify), then event types + availability, then bookings + contacts, then integrations, then routing + workflows. After each slice run `prisma generate` and a `tsc --noEmit` so types stay in sync. If you change a field, regenerate the client **and restart the dev server** — Next.js caches the Prisma client at boot.
