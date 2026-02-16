# Venue & Event Intelligence App — Research Briefing

**Compiled: February 2026**
**Status: Pre-planning research — do NOT begin implementation until plan is approved**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Voice-to-Data Pipeline — Core Feature Deep Dive](#2-voice-to-data-pipeline)
3. [Platform & Framework Recommendation](#3-platform--framework)
4. [Database & Backend Architecture](#4-database--backend)
5. [Offline Strategy & iOS Limitations](#5-offline-strategy)
6. [CRM Integration — HoneyBook](#6-crm-integration)
7. [Data Model Design](#7-data-model)
8. [Security, Privacy & Legal](#8-security-privacy)
9. [Existing Solutions & Competitive Landscape](#9-competitive-landscape)
10. [Risks, Blind Spots & Unknowns](#10-risks-and-blind-spots)
11. [Cost Estimates](#11-cost-estimates)
12. [Key Decisions Before Planning](#12-key-decisions)
13. [Sources](#13-sources)

---

## 1. Executive Summary

### What We're Building

A **voice-first, mobile-optimized Progressive Web App** that serves as a business intelligence capture system for a performing musician working weddings, corporate events, and parties in San Diego. The app's core innovation is an AI-powered pipeline that converts stream-of-consciousness voice memos into structured, searchable database records.

### The Core Problem

The user already captures rich intelligence at every gig — venue details, contact info, vendor networks, performance observations, competitive intelligence, and action items — but does it via voice memos that sit unprocessed. An estimated **80% of captured business intelligence is lost** because there's no systematic pipeline from voice → structured, searchable, actionable data.

### This App Is One Node in a Larger Ecosystem

```
┌──────────────────────────────────────────────────────────┐
│                    YOUR PHONE (in the field)              │
│                                                          │
│  ┌─────────────────────────────────────┐                 │
│  │  THIS APP (PWA)                     │                 │
│  │  - Voice recording                  │                 │
│  │  - Manual forms                     │                 │
│  │  - Photo capture                    │                 │
│  │  - Offline queue                    │                 │
│  └──────────────┬──────────────────────┘                 │
└─────────────────┼────────────────────────────────────────┘
                  │ sync when online
                  ▼
         ┌────────────────┐
         │  Backend API   │
         │  (Node/TS)     │
         │  + AI Layer    │◄──── Voice-to-Data Processing
         │  (transcribe   │      (Whisper/Deepgram → Claude)
         │   + parse)     │
         └───────┬────────┘
                 │
    ┌────────────┼────────────┬──────────────────┐
    │            │            │                  │
    ▼            ▼            ▼                  ▼
┌────────┐ ┌─────────┐ ┌──────────┐    ┌──────────────┐
│Supabase│ │HoneyBook│ │ Research │    │ Google Drive  │
│  (DB)  │ │  (CRM)  │ │  Agent   │    │ (docs/files)  │
│        │ │via MCP  │ │ (Python) │    │              │
└────────┘ └─────────┘ └──────────┘    └──────────────┘
                            │
                  Future: Sales Agent, Google Workspace,
                  Social Media, Email Marketing (via APIs
                  or connector services)
```

### Key Constraints

- **User is a coding beginner** — architecture must be clean, well-documented, and maintainable via Claude Code direction
- **Solo user today**, but code should be structured for eventual productization (open-source or SaaS)
- **API-first architecture** — the API layer must be rock-solid before the UI gets polished
- **Voice capture UI needed from day one** — can't do a car debrief through a test API
- **Next gig is within 7 days** — bare-bones voice capture should be testable ASAP

---

## 2. Voice-to-Data Pipeline

This is the **single most important feature** and the hardest technical challenge. Everything else is standard CRUD.

### 2.1 Architecture: Record → Transcribe → Parse → Store

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   RECORD     │───►│  TRANSCRIBE  │───►│   AI PARSE   │───►│    STORE     │
│  (browser    │    │  (speech →   │    │  (text →     │    │  (structured │
│   MediaRec   │    │   text)      │    │   JSON)      │    │   DB records)│
│   API)       │    │              │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
     Phone               Server              Server              Server
   webm/opus          Whisper API         Claude API          Supabase/PG
                     or Deepgram
```

### 2.2 Speech-to-Text Options

| Service | Cost (5 min avg) | Latency | Accuracy | Key Features |
|---------|-----------------|---------|----------|-------------|
| **OpenAI Whisper API** | ~$0.03/recording | 10-30s batch | Excellent | Cheapest, proven, batch only |
| **Deepgram** | ~$0.04/recording | Real-time capable | Excellent | Streaming + batch, entity detection |
| **AssemblyAI** | ~$0.06/recording | 15-45s | Excellent | Built-in entity extraction, action items |
| **Google Cloud STT** | ~$0.04/recording | 10-30s | Very good | Good integration ecosystem |
| **Web Speech API** | Free | Real-time | Variable | Browser-native, no server cost, unreliable |

**Recommendation: OpenAI Whisper API for v1.**
- At 8-10 events/month, cost is under $1/month for transcription
- Proven accuracy with natural speech, background noise
- Simple API, well-documented, good TypeScript SDK
- Can switch to Deepgram later if real-time streaming is needed

**Estimated monthly cost at full volume (10 events, ~5 min each):**
- Whisper: ~$0.30/month
- Claude parsing: ~$0.50-1.00/month (depending on prompt size)
- **Total voice pipeline: under $2/month**

### 2.3 AI Parsing Strategy — Two-Pass System

This is the critical insight from discovery: **contacts and action items are urgent; venue intelligence is not.**

#### Pass 1: Quick Parse (runs immediately after transcription, <30 seconds)

**Purpose:** Extract time-sensitive items the user needs to act on within 24-48 hours.

**Extracts:**
- New contact names, companies, phone numbers, emails
- Action items ("need to follow up with Jake", "ask photographer for video")
- Referral potential flags ("she does 3-4 weddings a month")
- Content capture notes ("got great video of flamenco set")

**Prompt strategy:** Focused, short prompt asking Claude to extract ONLY contacts + action items as JSON. Fast, cheap, high-priority.

```
Output example (Pass 1):
{
  "contacts": [
    {
      "name": "Jake",
      "company": "Bright Room Photography",
      "role": "photographer",
      "phone": null,
      "email": null,
      "context": "works Torrey Pines twice a month",
      "referral_potential": "high",
      "action": "follow up"
    }
  ],
  "action_items": [
    {
      "action": "Follow up with Jake from Bright Room Photography",
      "urgency": "within_48_hours",
      "category": "networking"
    },
    {
      "action": "Request flamenco set video from photographer",
      "urgency": "within_48_hours",
      "category": "content"
    }
  ]
}
```

#### Pass 2: Deep Parse (runs in background, can take minutes)

**Purpose:** Extract venue intelligence, performance observations, competitive intel, relationship assessments, and update/create full records.

**Extracts:**
- Venue details (stage dimensions, power, sound limits, load-in, parking)
- Performance observations (what worked, what didn't, crowd reactions)
- Competitive intelligence (other performers, pricing intel)
- Relationship assessments (coordinator quality, planner reliability)
- Comparisons to previous events at the same venue
- Lessons learned / "next time" notes

**Critical capability:** The deep parse must determine **whether to UPDATE an existing record or CREATE a new one.** This requires passing existing venue/contact names to the AI so it can match mentions like "Torrey Pines" to "The Lodge at Torrey Pines" in the database.

```
Output example (Pass 2):
{
  "venue_updates": {
    "match": "The Lodge at Torrey Pines",
    "confidence": 0.95,
    "updates": {
      "outdoor_stage_dimensions": "15x12 feet",
      "power_notes": "Outdoor setup requires 50-foot extension from behind bar",
      "sound_limit_db": 85,
      "lesson_learned": "Bring own 100-foot extension cord for outdoor events"
    }
  },
  "event_record": {
    "venue": "The Lodge at Torrey Pines",
    "event_type": "wedding",
    "crowd_size": 180,
    "crowd_demographics": "skewed older",
    "performance_highlights": "Flamenco set was standout — mother of bride emotional response",
    "musical_adjustments": "Monitor dB during dinner set, cocktail set was fine at 85dB",
    "client_satisfaction": "high"
  },
  "competitive_intel": {
    "competitor": "SD Music Entertainment",
    "role": "DJ",
    "relationship": "complementary, not competing — handles reception DJ handoff",
    "notes": "Good fit for referral partnership"
  }
}
```

### 2.4 Voice Pipeline Risks & Gotchas

| Risk | Severity | Mitigation |
|------|----------|------------|
| **Background noise at venues** | Medium | Whisper handles noise well; user records in car (quiet) for main debrief |
| **Names/companies misheard** | High | Show parsed results for user review before committing; fuzzy matching against existing DB records |
| **"Update vs. create" confusion** | High | Always show user a confirmation screen: "Did you mean this existing venue/contact?" with match confidence |
| **Phone numbers in speech** | Medium | Whisper often garbles phone numbers; may need manual correction step |
| **Accidental personal info** | Medium | User may mention things not meant for the database; review step is essential |
| **Long recordings exceed context** | Low | 15 min = ~3,000 words of transcript; well within Claude's context window |
| **Transcription API downtime** | Low | Queue recordings locally, process when API returns; recordings are never lost |

### 2.5 Recording in the Browser — Technical Notes

**MediaRecorder API** is the standard for in-browser audio recording. Key findings:

- **Format:** Use `audio/webm;codecs=opus` — now cross-browser including Safari
- **Always feature-detect** with `MediaRecorder.isTypeSupported()` — never hardcode formats
- **iPhone Safari quirks:** Works but has different frame duration (2.5ms vs 20ms); file extension mapping must handle this
- **Chrome bug:** WebM files lack duration metadata — not seekable. Not a problem for transcription, only for playback
- **Long recordings:** Use chunked recording (`recorder.start(timeSlice)`) to avoid memory issues on mobile
- **Audio compression:** Opus codec produces small files (~50KB/minute); a 15-min recording is under 1MB
- **Microphone permission:** User must grant once; PWAs added to home screen remember the permission

**Recommendation:** Record as webm/opus, upload in chunks if >5 minutes, compress on device. A 10-minute debrief will be ~500KB — trivially uploadable even on cellular.

---

## 3. Platform & Framework

### 3.1 Recommendation: React + Vite + Workbox

| Choice | Reasoning |
|--------|-----------|
| **React** | Largest ecosystem, most learning resources (critical for beginner), best Claude Code support for generating/modifying React code, first-class TypeScript support |
| **Vite** | Fast build tool, excellent PWA plugin (`vite-plugin-pwa` uses Workbox under the hood), modern DX |
| **Workbox** | Google's service worker library, handles caching strategies, offline fallback, background sync polyfill |
| **TypeScript** | Strict typing catches errors early, better for learning, better Claude Code assistance |

**Why not the others?**
- **Angular:** Overkill for solo dev, steep learning curve, verbose
- **Vue/Nuxt:** Excellent but smaller ecosystem means fewer learning resources and examples
- **SvelteKit:** Great performance but smallest ecosystem; harder for Claude Code to generate
- **Ionic + Capacitor:** Worth considering later for native camera/GPS access, but adds complexity for v1

### 3.2 Project Structure (Separate from WedMix)

This is a **new project** — completely separate repository and codebase. No relation to the existing WedMix Spotify app.

```
gig-intel/                    # working name
├── apps/
│   └── web/                  # React PWA (Vite)
│       ├── src/
│       │   ├── components/   # React components
│       │   ├── pages/        # Route pages
│       │   ├── hooks/        # Custom React hooks
│       │   ├── services/     # API client, audio recorder
│       │   ├── stores/       # State management
│       │   ├── types/        # TypeScript interfaces
│       │   └── workers/      # Service worker, sync logic
│       ├── public/
│       │   └── manifest.json # PWA manifest
│       └── vite.config.ts
├── packages/
│   └── api/                  # Backend API (Express/TS)
│       ├── src/
│       │   ├── routes/       # API route definitions
│       │   ├── controllers/  # Business logic
│       │   ├── services/     # External API integrations
│       │   │   ├── transcription.ts  # Whisper API
│       │   │   ├── ai-parser.ts      # Claude API (voice→data)
│       │   │   ├── honeybook.ts      # HoneyBook integration
│       │   │   └── storage.ts        # File/photo storage
│       │   ├── middleware/   # Auth, error handling
│       │   ├── models/       # Database models/schemas
│       │   └── types/        # Shared TypeScript types
│       └── tests/
├── supabase/                 # Database migrations, seed data
├── .env.example
├── package.json              # Monorepo root
└── CLAUDE.md                 # AI assistant guide
```

---

## 4. Database & Backend

### 4.1 Recommendation: Supabase (Hosted PostgreSQL)

| Criteria | Supabase | PocketBase | Firebase |
|----------|----------|------------|---------|
| **SQL/Relational** | Full PostgreSQL | SQLite | NoSQL (Firestore) |
| **TypeScript SDK** | First-class, type-generated | Community SDK | Good but NoSQL |
| **Free tier** | 500MB DB, 1GB storage | Self-host (free) | Generous but pay-per-read |
| **Auth built-in** | Yes (OAuth, JWT, magic links) | Yes | Yes |
| **File storage** | Yes (S3-compatible) | Yes | Yes |
| **Real-time** | Yes (Postgres changes) | Yes (SSE) | Yes (best-in-class) |
| **Offline sync** | Via PowerSync or DIY | Not native | Built-in |
| **AI/MCP integration** | Official MCP server exists | None | None |
| **Scalability** | Excellent | Limited | Excellent |
| **Self-host option** | Yes (Docker) | Yes (single binary) | No |
| **Learning value** | Teaches SQL, real DB skills | Teaches SQLite | Teaches NoSQL |

**Why Supabase wins:**
1. **PostgreSQL** — relational data model is perfect for venue/event/contact relationships. This data is deeply relational (many-to-many between vendors, venues, events, planners). NoSQL would fight you.
2. **Official MCP server** — Claude Code can interact with Supabase directly, which is a huge advantage for a user who directs development through Claude Code.
3. **Type generation** — Supabase generates TypeScript types from your database schema. This means the database IS the type system. Change a column, regenerate types, and TypeScript catches every place in your code that needs updating.
4. **Auth** — Built-in auth means you don't have to build login/registration from scratch.
5. **File storage** — Photos can go directly into Supabase Storage (S3-compatible). No separate service needed.
6. **Free tier is enough for v1** — 500MB database, 1GB file storage. At 20-30 contacts/month + venue photos, this lasts a long time.

### 4.2 Offline Sync Strategy with Supabase

Since Supabase has no built-in offline support, we need a sync layer. Given that offline is "nice to have" (not critical), the recommendation is:

**DIY IndexedDB + Sync Queue** (not PowerSync, which adds complexity and cost for a nice-to-have feature)

```
User records voice memo or fills form
         │
         ▼
┌─────────────────────┐
│  Write to IndexedDB  │◄── Always write locally first
│  (Dexie.js)          │
└──────────┬──────────┘
           │
           ▼
    ┌──────────────┐     YES    ┌──────────────────┐
    │   Online?    │───────────►│ Sync to Supabase  │
    └──────┬───────┘            │ (API call)        │
           │ NO                 └──────────────────┘
           ▼
┌──────────────────────┐
│ Queue in IndexedDB   │
│ (sync when online)   │
└──────────────────────┘
```

**Library choice: Dexie.js** for IndexedDB
- Clean Promise-based API
- TypeScript support
- 200KB gzipped
- Well-documented, actively maintained
- Dexie Cloud (optional) adds sync if we want managed sync later

**Conflict resolution: Last-write-wins** with timestamps
- Sufficient for single-user app
- Each record has `updated_at` timestamp
- If server record is newer than local, server wins (someone updated via another device/system)
- If local record is newer, local wins (user made changes offline)

---

## 5. Offline Strategy & iOS Limitations

### 5.1 Critical iOS PWA Constraints

These are **non-negotiable realities** that affect architecture decisions:

| Constraint | Impact | Workaround |
|-----------|--------|------------|
| **No Background Sync API** | Can't auto-sync when connectivity returns if app is closed | Sync on app open; show "X items pending sync" badge |
| **50MB cache storage limit** | Can't cache unlimited photos/audio offline | Compress aggressively; prioritize syncing media first |
| **7-day storage eviction** | If PWA unused for 7 days, Safari MAY clear IndexedDB | User likely opens weekly (8-10 gigs/month); request `navigator.storage.persist()` |
| **IndexedDB instability history** | Rare data loss/corruption on iOS | Always have server as source of truth; local is just a cache/queue |
| **No home screen install prompt** | iOS doesn't show automatic "Add to Home Screen" | Manual instructions; works fine once added |
| **Service worker cache bugs** | Old assets sometimes fail to update | Version your service worker; force refresh strategy |

### 5.2 Practical Impact for This App

The iOS limitations are **manageable** for this use case because:

1. **User opens the app at every gig** (8-10x/month) — so the 7-day eviction window is rarely hit
2. **Voice recordings are small** (~500KB for 10 minutes of opus audio) — well under the 50MB cache limit
3. **Critical data syncs quickly** — contacts and action items are small JSON payloads
4. **Photos are the biggest storage concern** — but the user said photos are secondary to data
5. **Background Sync isn't critical** — the user can tap "sync now" when they open the app after a gig

### 5.3 Recommended Offline Architecture

```
┌─────────────────────────────────────────────────────┐
│                    SERVICE WORKER                     │
│                                                      │
│  Cache Strategy:                                     │
│  ├── App Shell (HTML/CSS/JS) → Cache-First           │
│  ├── API data → Network-First, Cache Fallback        │
│  ├── Images/Photos → Cache-First, Background Update  │
│  └── Audio recordings → Store in IndexedDB           │
│                                                      │
│  Offline Indicators:                                 │
│  ├── Banner: "You're offline — data saved locally"   │
│  ├── Badge: "3 items pending sync"                   │
│  └── Auto-sync on reconnection (when app is open)    │
└─────────────────────────────────────────────────────┘
```

---

## 6. CRM Integration — HoneyBook

### 6.1 HoneyBook Overview

HoneyBook is a CRM specifically built for creative entrepreneurs (photographers, musicians, planners, etc.). It handles:
- **Contacts** — client and vendor contact management
- **Projects** — one project per event/lead
- **Invoices & Contracts** — payment tracking and e-signatures
- **Workflows** — automated email sequences and task reminders
- **Pipeline** — lead stages (inquiry → booked → completed)

### 6.2 HoneyBook API & MCP Server

**Key finding: HoneyBook's public API is limited.**

- HoneyBook does not have a fully open public REST API like Salesforce or HubSpot
- The HoneyBook MCP server exists but its capabilities need to be tested by the user in their Claude environment
- HoneyBook does support Zapier integrations with triggers and actions for contacts, projects, and invoices

**Action required before planning:**
- The user needs to activate and test the HoneyBook MCP server in their Claude environment
- Document exactly what read/write operations it supports
- This determines whether we build direct API integration or use Zapier as middleware

### 6.3 Integration Strategy (Tiered)

**Tier 1 (v1 — Guaranteed to work):**
- Export contacts/venues as structured CSV for manual HoneyBook import
- Include a "HoneyBook Project ID" field in event records for manual cross-referencing
- Generate formatted contact cards the user can reference when entering data into HoneyBook

**Tier 2 (v1.5 — If MCP server supports it):**
- Direct read/write via HoneyBook MCP server from Claude Code
- Auto-create contacts in HoneyBook when new contacts are captured
- Pull existing HoneyBook project data to avoid duplication

**Tier 3 (v2 — Zapier middleware):**
- Zapier webhook: when new contact created in this app → create in HoneyBook
- Zapier webhook: when new lead in HoneyBook → enrich with venue data from this app
- Bidirectional sync via Zapier as middleware

### 6.4 Data Formatting for HoneyBook Compatibility

HoneyBook contacts have these standard fields:
- First name, Last name
- Email, Phone
- Company name
- Contact type (Client, Vendor, Planner, etc.)
- Source (how they found you)
- Notes

**Our app should capture data in HoneyBook-compatible field structure from the start** — even if sync isn't automated in v1. This means:
- Always separate first/last name (not a single "name" field)
- Always capture email AND phone (HoneyBook needs at least one)
- Map our "vendor type" categories to HoneyBook's contact types
- Include a "source" field (e.g., "Met at Lodge at Torrey Pines, Johnson Wedding, 2/15/2026")

---

## 7. Data Model Design

### 7.1 Entity Relationship Diagram

```
┌──────────────┐
│    VENUE     │
│──────────────│
│ id           │
│ name         │       ┌──────────────────┐
│ address      │       │  VENUE_CONTACT   │
│ neighborhood │       │──────────────────│
│ category     │◄──────│ venue_id (FK)    │
│ capacity     │       │ contact_id (FK)  │
│ indoor_outdoor│      │ role             │
│ stage_dims   │       │ department       │
│ power_notes  │       └───────┬──────────┘
│ sound_limit  │               │
│ load_in_notes│       ┌───────▼──────────┐
│ coi_required │       │    CONTACT       │
│ parking_notes│       │──────────────────│
│ wifi         │       │ id               │
│ green_room   │       │ first_name       │
│ curfew_time  │       │ last_name        │
│ pref_vendor  │       │ email            │
│ relationship │       │ phone            │
│ rating       │       │ company          │
│ notes        │       │ contact_type     │──── enum: client, planner,
│ created_at   │       │ referral_source  │     vendor, coordinator,
│ updated_at   │       │ referral_potential│    venue_staff, other
└──────┬───────┘       │ relationship_status│── enum: active, nurturing,
       │               │ honeybook_id     │     cold, new
       │               │ notes            │
       │               │ tags[]           │
       │               │ created_at       │
       │               │ updated_at       │
       │               └───────┬──────────┘
       │                       │
       │               ┌───────▼──────────┐
┌──────▼───────┐       │  EVENT_VENDOR    │
│    EVENT     │       │──────────────────│
│──────────────│       │ event_id (FK)    │
│ id           │◄──────│ contact_id (FK)  │
│ venue_id (FK)│       │ vendor_type      │
│ event_type   │──     │ quality_rating   │
│ event_date   │  │    │ notes            │
│ client_id(FK)│  │    └──────────────────┘
│ planner_id   │  │
│ guest_count  │  │    enum: wedding, corporate,
│ crowd_energy │  │    party, private, fundraiser
│ set_list     │  │
│ music_notes  │  │    ┌──────────────────┐
│ highlights   │       │  EVENT_DEBRIEF   │
│ adjustments  │       │──────────────────│
│ content_notes│◄──────│ event_id (FK)    │
│ satisfaction │       │ phase            │──── enum: pre_brief, setup,
│ follow_ups[] │       │ recording_url    │     during_event, break,
│ honeybook_id │       │ transcript       │     post_event, car_debrief
│ created_at   │       │ parsed_data (JSON)│
│ updated_at   │       │ processed        │
└──────────────┘       │ created_at       │
                       └──────────────────┘

┌──────────────────┐
│  COMPETITOR      │
│──────────────────│
│ id               │
│ name             │
│ company          │
│ performer_type   │
│ venues[]         │
│ pricing_intel    │
│ relationship     │──── enum: complementary,
│ notes            │     competing, referral_partner
│ created_at       │
│ updated_at       │
└──────────────────┘

┌──────────────────┐
│  ACTION_ITEM     │
│──────────────────│
│ id               │
│ event_id (FK)    │
│ description      │
│ urgency          │──── enum: immediate, within_48h,
│ category         │     this_week, someday
│ status           │──── enum: pending, done, skipped
│ due_date         │
│ related_contact  │
│ created_at       │
│ completed_at     │
└──────────────────┘

┌──────────────────┐
│  PHOTO           │
│──────────────────│
│ id               │
│ venue_id (FK)    │
│ event_id (FK)    │
│ storage_url      │
│ caption          │
│ photo_type       │──── enum: stage, power, load_in,
│ created_at       │     setup, selfie, vendor, other
└──────────────────┘
```

### 7.2 Key Design Decisions

**Contacts are unified.** A single `contacts` table with a `contact_type` enum. A person can appear in multiple roles via junction tables (a coordinator can also be a planner at a different event). This keeps the data model simple and avoids duplicate records.

**Events are the central entity.** Everything connects through events — venues host events, clients book events, planners coordinate events, vendors work events. The event record is where all the intelligence converges.

**Debriefs are separate from events.** A single event can have multiple debrief recordings (pre-brief, during, car debrief). Each debrief stores the raw audio URL, the transcript, and the parsed JSON output. This preserves the raw data even if we improve parsing later.

**Competitors are their own entity.** Not just contacts — they track competitive pricing intelligence, venue presence, and whether they're complementary or competing. This feeds the research agent.

**Action items are first-class.** Not buried in event notes — they're extracted, tracked, and have statuses. This is the "urgent" output from the quick parse.

**Soft delete everywhere.** `deleted_at` timestamp instead of hard delete. Contacts, venues, and events are never truly deleted — they're archived. This prevents accidental data loss and supports audit trail.

**Tags are arrays.** PostgreSQL supports array columns natively. Tags like `["luxury", "outdoor", "torrey-pines-area", "preferred-vendor"]` enable flexible filtering without a separate tags table.

### 7.3 San Diego Geographic Context

For the "massive San Diego database" vision:

- **Neighborhood field** on venues — not just city. San Diego neighborhoods matter (La Jolla, Del Mar, Gaslamp, Coronado, Rancho Santa Fe, etc.)
- **Region grouping** — North County, Central, South Bay, East County, Coastal
- **Coordinate storage** — latitude/longitude for future map features
- **Don't over-engineer geography for v1** — a simple `neighborhood` text field + `region` enum is enough. Full geo-tagging can come later.

---

## 8. Security, Privacy & Legal

### 8.1 CCPA (California Consumer Privacy Act)

**Key finding: CCPA likely does NOT apply to this app in its current form.**

CCPA applies to for-profit businesses that meet ANY of these thresholds:
- Annual gross revenue over $25 million
- Buy, sell, or share personal information of 100,000+ California residents
- Derive 50%+ of annual revenue from selling personal information

A solo musician collecting ~300 contacts/year for their own business use almost certainly falls below all thresholds. **However:**

- If the app is later productized (SaaS for other performers), CCPA becomes relevant
- If contact data is ever sold or shared with third parties, CCPA applies
- Best practice: **build with CCPA-compatible patterns from the start** even if not legally required. It's much harder to retrofit privacy compliance than to bake it in.

### 8.2 Practical Privacy Recommendations

| Area | Recommendation |
|------|---------------|
| **Consent** | For v1 (personal tool), no formal consent needed — you're collecting business contacts in a professional context. If productized, add consent tracking. |
| **Data minimization** | Only collect what you'll actually use. Don't capture data "just in case." |
| **Encryption in transit** | HTTPS everywhere (Supabase handles this). Never transmit data over HTTP. |
| **Encryption at rest** | Supabase encrypts data at rest by default. Audio files in Supabase Storage are also encrypted. |
| **Authentication** | Use Supabase Auth — supports email/password, magic links, or OAuth (Google). Don't build custom auth. |
| **Voice recordings** | These contain the most sensitive data (names, numbers spoken aloud). Store server-side only (not in browser cache long-term). Delete from local storage after successful upload. |
| **Device loss** | If phone is lost, the PWA's local data is at risk. Mitigate by: syncing frequently, minimizing what's stored locally, relying on Supabase as source of truth. |
| **No secrets in client code** | API keys for Whisper/Claude go in the backend `.env`, never in the frontend. The PWA calls YOUR API, which calls external APIs. |
| **Photo metadata** | Strip EXIF data from photos before storage (EXIF can contain GPS coordinates, device info). |
| **Backup** | Supabase includes automated daily backups on paid plans. On free tier, set up a periodic `pg_dump` export. |

### 8.3 CAN-SPAM / TCPA (If Contacts Are Used for Marketing Later)

- **CAN-SPAM:** If you email contacts collected through this app, you must include unsubscribe option and physical address. Not relevant for v1 (personal use) but critical if contacts feed into email marketing.
- **TCPA:** If you text contacts, you need prior express consent. Document how/where consent was given.
- **Recommendation:** Add an optional `marketing_consent` boolean field to contacts now. It costs nothing and prevents legal issues later.

---

## 9. Existing Solutions & Competitive Landscape

### 9.1 What Exists Today

| Tool | What It Does | Why It's Not Enough |
|------|-------------|-------------------|
| **HoneyBook** | CRM for creatives | No voice input, no venue intelligence, no field data capture |
| **Airtable** | Flexible database | No voice-to-data, no offline, requires manual entry |
| **Google Forms** | Data collection | No offline, no AI, not mobile-optimized for field use |
| **Notion** | All-in-one workspace | No voice parsing, no offline PWA, not designed for field capture |
| **Fieldwire / Fulcrum** | Field data collection (construction) | Wrong industry, no AI parsing, no CRM integration |
| **Otter.ai** | Voice transcription | Transcribes but doesn't parse into structured records |
| **Generic CRM apps** | Contact management | No venue intelligence, no event context, no voice input |

### 9.2 The Gap

**No existing tool combines:**
1. Voice-first data capture
2. AI parsing into structured records
3. Event/venue-specific data model
4. Wedding/events industry context
5. CRM integration
6. Offline capability
7. API-first architecture for ecosystem integration

This is genuinely a novel combination. The closest analog is a vertical CRM + voice transcription + AI parsing — which nobody has built specifically for performing musicians.

### 9.3 Open-Source Patterns Worth Studying

- **Twenty CRM** (GitHub, 5k+ stars) — Open-source CRM built with TypeScript, good data model reference for contact/company/activity patterns
- **Cal.com** (GitHub, 30k+ stars) — Open-source scheduling, good patterns for event management and API-first architecture
- **Huly** (GitHub, 15k+ stars) — Project management platform, good patterns for activity tracking and real-time sync
- **PocketBase** examples — Simple self-hosted backends with offline patterns

---

## 10. Risks, Blind Spots & Unknowns

### 10.1 HIGH RISK — Must Address Before Building

| # | Risk | Details | Mitigation |
|---|------|---------|------------|
| 1 | **Voice parsing accuracy** | AI may misparse names, numbers, company names from natural speech. Garbled phone numbers are especially common. | Always show parsed results for human review before committing to DB. Never auto-commit without confirmation. Build a "correction" UI that's fast to use. |
| 2 | **HoneyBook integration uncertainty** | We don't know what the MCP server actually supports. If it's read-only or limited, the integration strategy changes completely. | **User must test the HoneyBook MCP server BEFORE we plan the integration layer.** |
| 3 | **Scope creep** | The vision is massive (research agent, sales agent, Google Workspace, social media, email marketing). V1 must be ruthlessly scoped. | V1 = voice capture + AI parsing + manual forms + basic search. Period. Everything else is v2+. |
| 4 | **Beginner codebase maintenance** | User directs via Claude Code but doesn't write code. If architecture is overly complex, future modifications become difficult. | Keep module count low, avoid deep abstraction layers, write extensive CLAUDE.md documentation for each module. |

### 10.2 MEDIUM RISK — Plan For But Don't Block On

| # | Risk | Details | Mitigation |
|---|------|---------|------------|
| 5 | **iOS audio recording edge cases** | MediaRecorder on iOS Safari has quirks (different opus frame duration, no duration metadata). Some older iPhones may behave differently. | Test on user's actual iPhone early. Use format detection, not hardcoded formats. |
| 6 | **Transcription cost scaling** | At current volume (~$2/month), costs are trivial. If productized (100 users × 10 events), costs scale linearly. | Whisper is cheapest option. Can self-host Whisper later if costs matter. |
| 7 | **Database schema evolution** | The data model will change as the user discovers what fields matter and which don't. | Use Supabase migrations for all schema changes. Never modify schema manually. |
| 8 | **Research agent integration (Python ↔ TypeScript)** | The research agent is Python; this app is TypeScript. They need to talk via REST API. | Design the API with the research agent as a first-class consumer from day one. Document API endpoints for the Python agent. |
| 9 | **Photo storage costs** | Photos at venues could add up. Full-res photos from a modern phone are 3-8MB each. | Compress/resize before upload (max 1920px wide). Supabase free tier has 1GB storage. |

### 10.3 LOW RISK — Be Aware Of

| # | Risk | Details | Mitigation |
|---|------|---------|------------|
| 10 | **Offline data loss on iOS** | Safari may evict IndexedDB after 7 days of non-use. | User opens app at every gig (weekly). Request persistent storage. Always sync when online. |
| 11 | **Multiple device sync** | User may want to access data from laptop and phone. | Supabase handles this — it's a server-side database. PWA is just a client. |
| 12 | **Eventual productization** | Moving from personal tool to multi-tenant SaaS requires significant architecture changes (auth, row-level security, billing). | Build with single-user auth now, but use Supabase RLS (Row Level Security) from day one so the pattern is in place. |

### 10.4 BLIND SPOTS — Things We Haven't Thought About Yet

| # | Blind Spot | Why It Matters |
|---|-----------|---------------|
| 1 | **What happens when you return to a venue?** | Second visit should show all previous data, notes, and contacts for that venue. The UI needs a strong "venue briefing" view, not just data entry. This is a READ experience, not just WRITE. |
| 2 | **Debrief review workflow** | After AI parsing, the user needs to review/correct/approve parsed data. This UI is critical — if it's slow or clunky, the user won't use it. It needs to be faster than manual entry, or the voice pipeline loses its value. |
| 3 | **Search and recall** | "Who was that photographer I met at Torrey Pines last March?" — full-text search across contacts, events, and debrief transcripts is essential. Supabase supports full-text search natively. |
| 4 | **Duplicate detection** | User mentions "Sarah from LVL" in three different debriefs. The system needs to recognize this is the same person and update one record, not create three. Fuzzy name matching is needed. |
| 5 | **Relationship graph** | The real value isn't individual records — it's the CONNECTIONS. "This planner works with these 3 venues and books these 2 photographers." Graph-style queries matter more than table views. |
| 6 | **Export for research agent** | The Python research agent will want to query this app. What format? REST API returning JSON? CSV export? Direct database access? Define the contract early. |
| 7 | **Gig prep mode** | Before a gig, the user wants a briefing: venue details, previous events there, known contacts, planner history, things to remember ("bring 100-foot extension cord"). This is the inverse of data capture — it's data retrieval. |
| 8 | **Voice memo archival** | Should raw audio recordings be kept forever? They're the source of truth but they cost storage. Policy: keep for 90 days, then delete audio but keep transcript? |
| 9 | **What about non-gig contacts?** | User might meet vendors at networking events, industry mixers, or online. Not every contact comes from a gig. The app should support adding contacts outside of the event flow. |
| 10 | **Time zone handling** | Events have dates and times. San Diego is Pacific Time, but some clients/planners might be elsewhere. Use UTC internally, display in local time. |

---

## 11. Cost Estimates

### 11.1 Monthly Operating Costs (v1, Single User)

| Service | Free Tier | Estimated Monthly Cost |
|---------|-----------|----------------------|
| **Supabase** (database + auth + storage) | 500MB DB, 1GB storage, 50K auth users | **$0** (free tier sufficient for years) |
| **OpenAI Whisper API** (transcription) | None | **$0.30** (~10 recordings × 5 min) |
| **Claude API** (voice parsing) | None | **$0.50-1.00** (~10 parse operations) |
| **Vercel/Netlify** (PWA hosting) | Generous free tiers | **$0** |
| **Domain name** (optional) | N/A | **$12/year** |
| **Total** | | **~$1-2/month** |

### 11.2 Development Costs

| Item | Cost |
|------|------|
| **Claude Code** (development tool) | Existing subscription |
| **OpenAI API key** (for Whisper) | Pay-as-you-go, ~$0.006/min |
| **Anthropic API key** (for parsing) | Pay-as-you-go, minimal at this volume |
| **Supabase** | Free tier |
| **Total upfront** | **~$0** (aside from existing subscriptions) |

---

## 12. Key Decisions Before Planning

These must be answered before creating the implementation plan:

### Decision 1: HoneyBook MCP Server Capabilities
**Status: UNKNOWN — User must test**
- Activate the HoneyBook MCP server in Claude environment
- Document what operations it supports (read contacts? create contacts? read projects?)
- This determines whether Tier 2 integration (direct sync) is possible for v1

### Decision 2: New Repository or Subdirectory?
**Recommendation: New repository** (`gig-intel` or similar name)
- Completely separate from WedMix
- Clean CLAUDE.md tailored to this project
- Independent package.json and dependencies
- Confirm with user

### Decision 3: App Name
Working name needed for the repository, PWA manifest, and branding. Suggestions:
- `gig-intel` — short, descriptive
- `gig-vault` — implies secure data storage
- `venue-book` — venue-focused
- `set-brief` — references the debrief workflow
- User should pick or suggest

### Decision 4: Hosting Platform for Backend API
**Options:**
- **Supabase Edge Functions** — serverless, TypeScript, colocated with DB (simplest)
- **Railway/Render** — simple Node.js hosting, free tier available
- **Vercel Serverless Functions** — if using Next.js for the PWA
- **Self-hosted VPS** — most control, most operational burden

**Recommendation: Supabase Edge Functions** for v1. Everything in one platform. Simpler mental model for a beginner.

### Decision 5: Claude API vs. OpenAI for Voice Parsing
**Options:**
- **Claude API** — better at nuanced parsing, user already knows Claude's capabilities
- **OpenAI GPT-4** — alternative, similar quality
- **Both** — Whisper for transcription (OpenAI), Claude for parsing (Anthropic)

**Recommendation: Whisper (OpenAI) for transcription + Claude API for parsing.** Best-in-class for each task, and the user already trusts Claude's intelligence.

---

## 13. Sources

### PWA & Frameworks
- [Best PWA Frameworks 2026 — WebOsmotic](https://webosmotic.com/blog/pwa-frameworks/)
- [10 Best PWA Frameworks 2026 — Zegocloud](https://www.zegocloud.com/blog/progressive-web-app-framework)
- [PWA Frameworks with Key Features — TestMuAI](https://www.testmuai.com/blog/progressive-web-app-frameworks/)
- [Top PWA Frameworks 2026 — Alphabold](https://www.alphabold.com/top-frameworks-and-tools-to-build-progressive-web-apps/)

### iOS PWA Limitations
- [PWAs on iOS 2025 — Medium](https://ravi6997.medium.com/pwas-on-ios-in-2025-why-your-web-app-might-beat-native-0b1c35acf845)
- [PWA iOS Limitations — MagicBell](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide)
- [PWA on iOS Status & Limitations — Brainhub](https://brainhub.eu/library/pwa-on-ios)
- [Do PWAs Work on iOS? 2026 Guide — MobiLoud](https://www.mobiloud.com/blog/progressive-web-apps-ios)
- [Safari PWA Limitations — Vinova](https://vinova.sg/navigating-safari-ios-pwa-limitations/)

### Database & Backend
- [Supabase vs Firebase vs PocketBase 2025 — Supadex](https://www.supadex.app/blog/supabase-vs-firebase-vs-pocketbase-which-one-should-you-choose-in-2025)
- [Supabase vs PocketBase — Leanware](https://www.leanware.co/insights/supabase-vs-pocketbase)
- [Supabase vs Firebase — Supabase Official](https://supabase.com/alternatives/supabase-vs-firebase)

### Offline Sync
- [PowerSync + Supabase — PowerSync Blog](https://www.powersync.com/blog/bringing-offline-first-to-supabase)
- [Offline-First PWA with IndexedDB + Supabase — Medium (Jan 2026)](https://medium.com/@oluwadaprof/building-an-offline-first-pwa-notes-app-with-next-js-indexeddb-and-supabase-f861aa3a06f9)
- [RxDB Supabase Replication Plugin](https://rxdb.info/replication-supabase.html)
- [WatermelonDB + Supabase — Supabase Blog](https://supabase.com/blog/react-native-offline-first-watermelon-db)

### Audio Recording
- [MediaRecorder API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder)
- [iPhone Safari MediaRecorder — BuildWithMatija](https://www.buildwithmatija.com/blog/iphone-safari-mediarecorder-audio-recording-transcription)
- [Cross-Browser Recording — Media Codings](https://media-codings.com/articles/recording-cross-browser-compatible-media)
- [opus-media-recorder Polyfill — GitHub](https://github.com/kbumsik/opus-media-recorder)

---

*This document is a research briefing only. No implementation plan has been created. All findings should be reviewed and key decisions (Section 12) resolved before planning begins.*
