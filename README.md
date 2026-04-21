<div align="center">

<img src="public/img_logo.png" alt="EchoForge" width="220"/>

# EchoForge

**AI-Powered Text-to-Speech & Voice Cloning Platform**

[![Next.js](https://img.shields.io/badge/Next.js-16.1.6-black?style=flat-square&logo=nextdotjs)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-19.2.3-61DAFB?style=flat-square&logo=react)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square&logo=typescript)](https://www.typescriptlang.org/)
[![Prisma](https://img.shields.io/badge/Prisma-7.6-2D3748?style=flat-square&logo=prisma)](https://www.prisma.io/)
[![tRPC](https://img.shields.io/badge/tRPC-11.x-398CCB?style=flat-square&logo=trpc)](https://trpc.io/)
[![Clerk](https://img.shields.io/badge/Auth-Clerk-6C47FF?style=flat-square)](https://clerk.com/)
[![Polar](https://img.shields.io/badge/Billing-Polar.sh-0EA5E9?style=flat-square)](https://polar.sh/)
[![Sentry](https://img.shields.io/badge/Monitoring-Sentry-362D59?style=flat-square&logo=sentry)](https://sentry.io/)

EchoForge is a production-grade, full-stack AI voice platform that enables organizations to clone voices and generate high-quality speech from text using the [Chatterbox TTS](https://github.com/resemble-ai/chatterbox) model — deployed on GPU-accelerated infrastructure via [Modal](https://modal.com/). It ships with multi-tenant authentication, usage-metered billing, an interactive audio player, in-browser recording, and a rich component design system.

Deployed on **Railway** — the live production app is running now.

[**🚀 Live Demo**](https://echoforge-production-0143.up.railway.app/) · [**🐛 Report Bug**](https://github.com/Rahul9214/EchoForge/issues/new?template=bug_report.md&labels=bug) · [**✨ Request Feature**](https://github.com/Rahul9214/EchoForge/issues/new?template=feature_request.md&labels=enhancement)

</div>

---

## Table of Contents

1. [Overview](#overview)
2. [Key Features](#key-features)
3. [Architecture Overview](#architecture-overview)
4. [Tech Stack](#tech-stack)
5. [Project Structure](#project-structure)
6. [Core Modules Deep Dive](#core-modules-deep-dive)
   - [Authentication & Multi-Tenancy](#authentication--multi-tenancy)
   - [AI Backend — Chatterbox on Modal](#ai-backend--chatterbox-on-modal)
   - [API Layer — tRPC Routers](#api-layer--trpc-routers)
   - [Storage — Cloudflare R2](#storage--cloudflare-r2)
   - [Billing — Polar.sh Usage Metering](#billing--polarsh-usage-metering)
   - [Database — PostgreSQL + Prisma](#database--postgresql--prisma)
   - [Frontend Features](#frontend-features)
7. [Data Models](#data-models)
8. [Environment Variables](#environment-variables)
9. [Getting Started](#getting-started)
10. [Scripts & Tooling](#scripts--tooling)
11. [UI Component Library](#ui-component-library)
12. [Voice System](#voice-system)
13. [Audio Pipeline](#audio-pipeline)
14. [Monitoring & Observability](#monitoring--observability)
15. [Security Model](#security-model)
16. [Performance Considerations](#performance-considerations)
17. [Deployment](#deployment)
18. [Contributing](#contributing)

---

## Overview

EchoForge solves a real problem: turning text into expressive, natural-sounding speech at scale — using an organization's own cloned voices. The platform is built around three pillars:

| Pillar | Description |
|---|---|
| **Voice Cloning** | Upload or record a voice sample; the AI learns it and uses it for generation |
| **Text-to-Speech** | Generate WAV audio files from arbitrary text with fine-grained model controls |
| **Billing & Metering** | Pay-as-you-go at **$0.30 per 1,000 characters** via Polar.sh usage events |

Every component is designed with **production readiness** in mind: typed end-to-end with TypeScript, multi-tenant by default, error-tracked by Sentry, and backed by presigned short-lived audio URLs that protect generated content.

---

## Key Features

### 🎤 Voice Cloning
- **Upload audio files** (any format, up to 20 MB) or **record directly in the browser** using the microphone
- Rich voice metadata: name, description, 12 voice categories, multi-locale language support (sourced from the `locale-codes` library)
- Custom voices are scoped to the **organization** — fully isolated from other tenants
- Voice samples are stored in **Cloudflare R2** and referenced by the AI model via a pre-keyed path

### 🗣️ Text-to-Speech
- Generate high-quality WAV audio from up to **5,000 characters** of text
- **20 pre-built system voices** covering categories from Audiobook to Advertising to Meditation
- Fine-tune generation with **4 AI parameter sliders**:
  - **Creativity** (Temperature 0–2)
  - **Voice Variety** (Top-P 0–1)
  - **Expression Range** (Top-K 1–10,000)
  - **Natural Flow** (Repetition Penalty 1–2)
- Interactive audio player powered by **WaveSurfer.js** with waveform visualization, seek ±10s, and one-click WAV download
- **Generation History** — every generation is persisted per organization, browsable from the Settings panel

### 🖥️ Dashboard
- **Hero text input panel** with a beautiful gradient-bordered textarea
- Real-time **cost estimator** that previews the charge as you type
- **6 Quick-Action cards** pre-loaded with creative prompts (narration, ads, movie scenes, game characters, podcasts, meditation guides)
- Responsive collapsible sidebar with active route detection

### 🧩 Voice Explorer
- Browse all voices (system + custom) in a paginated card layout
- **Live audio preview** — click play on any voice card to hear a sample
- Country flag emoji derived from locale tags using Unicode regional indicator characters
- Directly launch TTS with any voice via "Use this voice" deep-link
- Safe delete with confirmation dialog (custom voices only)

### 💳 Billing
- **Pay-as-you-go** via [Polar.sh](https://polar.sh) — subscription required to generate audio
- Estimated cost shown in the sidebar, updated each period
- One-click checkout redirect and customer portal session via tRPC mutations
- Polar usage events fired asynchronously (fire-and-forget) so they never block the response

### 🔐 Authentication
- Powered by **Clerk** — enterprise-grade auth with SSO, magic links, and social logins
- **Organization-first** design: users must belong to an org before accessing the dashboard
- Middleware enforces `/org-selection` redirect for authenticated-but-unorganized users
- `orgProcedure` tRPC middleware ensures every mutation/query is tenant-scoped

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         Browser                             │
│   Next.js App Router  ·  React 19  ·  TanStack Query       │
│   tRPC Client  ·  Clerk UI  ·  WaveSurfer  ·  shadcn/ui    │
└──────────────┬───────────────────────────────┬──────────────┘
               │ HTTP / tRPC                   │ Auth tokens
               ▼                               ▼
┌─────────────────────────┐     ┌──────────────────────────┐
│   Next.js Server        │     │       Clerk              │
│   (App Router + tRPC)   │     │  Multi-tenant Auth       │
│                         │     └──────────────────────────┘
│  ┌───────────────────┐  │
│  │  tRPC Routers     │  │     ┌──────────────────────────┐
│  │  · voices         │  │────▶│   PostgreSQL Database    │
│  │  · generations    │  │     │   (Prisma ORM)           │
│  │  · billing        │  │     └──────────────────────────┘
│  └───────────────────┘  │
│           │             │     ┌──────────────────────────┐
│  ┌────────┴──────────┐  │────▶│   Cloudflare R2          │
│  │  REST API Routes  │  │     │   (voice + audio files)  │
│  │  · /api/audio/    │  │     └──────────────────────────┘
│  │  · /api/voices/   │  │
│  │  · /api/trpc/     │  │     ┌──────────────────────────┐
│  └───────────────────┘  │────▶│   Polar.sh               │
└─────────────────────────┘     │   (billing & metering)   │
               │                └──────────────────────────┘
               │ HTTP POST /generate (API key)
               ▼
┌─────────────────────────────────────────────────────────────┐
│              Modal GPU Deployment (Python)                  │
│  chatterbox_tts.py · ChatterboxTurboTTS · FastAPI           │
│  GPU: NVIDIA A10G · Concurrent inputs: 10                   │
│  R2 CloudBucketMount (read-only for voice samples)          │
└─────────────────────────────────────────────────────────────┘
```

**Key design principles:**
- **End-to-end type safety** — types flow from Prisma schema → tRPC router output types → React component props via `inferRouterOutputs`
- **Tenant isolation** — every database query is filtered by `orgId` extracted from Clerk's auth token in `orgProcedure`
- **Serverless-first** — Next.js API routes + Modal for GPU — no always-on servers to manage
- **Storage as source of truth** — R2 object keys are stored in the DB; audio is served via authenticated Route Handlers, never exposed directly

---

## Tech Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| Next.js | 16.1.6 | React framework, App Router |
| React | 19.2.3 | UI rendering |
| TypeScript | 5.x | Type safety |
| Tailwind CSS | 4.x | Utility-first styling |
| shadcn/ui + Radix UI | Latest | Accessible component library |
| TanStack Query | 5.x | Server state & caching |
| TanStack Form | 1.x | Form state management |
| tRPC client | 11.x | Type-safe API calls |
| WaveSurfer.js | 7.x | Waveform audio player |
| RecordRTC | 5.x | In-browser audio recording |
| nuqs | 2.x | URL query state management |
| Sonner | 2.x | Toast notifications |
| Recharts | 3.x | Data visualization |
| date-fns | 4.x | Date formatting |
| clsx + tailwind-merge | Latest | Conditional class utilities |
| Lucide React | 1.x | Icon library |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| tRPC server | 11.x | Typesafe API router |
| Prisma | 7.x | ORM & query builder |
| Prisma Client (pg adapter) | 7.x | PostgreSQL driver (Edge-compatible) |
| AWS SDK v3 (S3-compatible) | 3.x | Cloudflare R2 file storage |
| Clerk Next.js | 7.x | Authentication & session |
| Polar SDK | 0.47 | Billing, subscriptions, usage metering |
| openapi-fetch | 0.17 | Typed HTTP client for Chatterbox API |
| superjson | 2.x | Serialization for Date/Map/Set in tRPC |
| Sentry Next.js | 10.x | Error tracking & monitoring |
| Zod | 4.x | Runtime schema validation |
| t3-oss/env-nextjs | 0.13 | Type-safe environment variables |

### AI Infrastructure
| Technology | Purpose |
|---|---|
| Modal | Serverless GPU deployment platform |
| ChatterboxTurboTTS | AI TTS model (voice cloning capability) |
| FastAPI | Python HTTP server wrapping the model |
| NVIDIA A10G GPU | Inference hardware |
| PEFT | Parameter-efficient fine-tuning adapters |
| torchaudio | Audio I/O in Python |

---

## Project Structure

```
echoforge/
├── chatterbox_tts.py          # Python Modal deployment — AI TTS backend
├── prisma/
│   ├── schema.prisma          # Database schema (Voice + Generation models)
│   └── migrations/            # DB migration history
├── scripts/
│   ├── seed-system-voices.ts  # Seeds 20 system voices into DB + R2
│   ├── sync-api.ts            # Syncs OpenAPI types from Chatterbox API
│   └── system-voices/         # WAV files for system voice samples
├── public/
│   └── img_logo.png           # EchoForge branding asset
└── src/
    ├── app/                   # Next.js App Router
    │   ├── layout.tsx         # Root layout (Clerk + tRPC + Toaster + nuqs)
    │   ├── globals.css        # Global design tokens
    │   ├── (dashboard)/       # Route group — protected dashboard
    │   │   ├── layout.tsx     # Sidebar + SidebarInset layout
    │   │   ├── page.tsx       # / → DashboardView
    │   │   ├── text-to-speech/
    │   │   │   ├── page.tsx             # /text-to-speech (TTS studio)
    │   │   │   └── [generationId]/page.tsx  # /text-to-speech/:id (generation detail)
    │   │   └── voices/
    │   │       ├── layout.tsx  # /voices layout (Suspense boundary)
    │   │       └── page.tsx    # /voices (voice explorer)
    │   ├── api/
    │   │   ├── trpc/[trpc]/route.ts     # tRPC HTTP handler
    │   │   ├── audio/[generationId]/route.ts  # Serve generated audio via R2 presigned URL
    │   │   └── voices/
    │   │       ├── create/route.ts      # POST — create custom voice (upload to R2 + DB)
    │   │       └── [voiceId]/route.ts   # GET — stream voice sample from R2
    │   ├── org-selection/     # Clerk org picker page
    │   ├── sign-in/           # Clerk sign-in page
    │   └── sign-up/           # Clerk sign-up page
    ├── features/              # Feature-sliced architecture
    │   ├── dashboard/
    │   │   ├── components/    # Sidebar, Header, HeroPattern, TextInputPanel, QuickActionCard
    │   │   ├── data/          # quickActions.ts (6 pre-built prompts)
    │   │   └── views/         # DashboardView
    │   ├── text-to-speech/
    │   │   ├── components/    # 14 components (see below)
    │   │   ├── contexts/      # TTSVoicesContext — passes voice list to nested components
    │   │   ├── data/          # constants.ts (TEXT_MAX_LENGTH, COST_PER_UNIT), sliders.ts
    │   │   ├── hooks/         # useWaveSurfer — WaveSurfer.js integration
    │   │   └── views/         # TextToSpeechView, TextToSpeechDetailView, Layout
    │   ├── voices/
    │   │   ├── components/    # VoiceCard, VoiceCreateDialog, VoiceCreateForm, VoiceRecorder, VoicesList, VoicesToolbar
    │   │   ├── data/          # voice-categories.ts, voice-scoping.ts (canonical names)
    │   │   ├── hooks/         # useAudioRecorder — RecordRTC integration
    │   │   ├── lib/           # params.ts (nuqs search param helpers)
    │   │   └── views/         # VoicesView, VoicesLayout
    │   └── billing/
    │       ├── components/    # UsageContainer — upgrade card + usage card
    │       └── hooks/         # useCheckout — tRPC mutation helper
    ├── components/
    │   ├── page-header.tsx    # Shared mobile page header
    │   ├── voice-avatar/      # DiceBear-based deterministic avatar
    │   └── ui/                # 57 shadcn/ui components
    ├── hooks/
    │   ├── use-app-form.ts    # Typed TanStack Form context helper
    │   ├── use-audio-playback.ts  # Simple HTML5 Audio play/pause hook
    │   └── use-mobile.ts      # Media query hook (< 768px)
    ├── lib/
    │   ├── db.ts              # PrismaClient singleton (pg adapter)
    │   ├── env.ts             # t3-oss typed env (server-side only)
    │   ├── r2.ts              # S3Client for R2: upload, delete, presign
    │   ├── chatterbox-client.ts  # openapi-fetch typed client
    │   └── polar.ts           # Polar SDK client
    ├── trpc/
    │   ├── init.ts            # tRPC setup, baseProcedure, authProcedure, orgProcedure
    │   ├── client.tsx         # TRPCReactProvider + useTRPC hook
    │   ├── server.tsx         # Caller factory for RSC usage
    │   ├── query-client.ts    # TanStack QueryClient config
    │   └── routers/
    │       ├── _app.ts        # Root AppRouter combining all sub-routers
    │       ├── voices.ts      # voices.getAll, voices.delete
    │       ├── generations.ts # generations.create, generations.getAll, generations.getById
    │       └── billing.ts     # billing.getStatus, billing.createCheckout, billing.createPortalSession
    ├── types/
    │   ├── chatterbox-api.d.ts  # Auto-generated OpenAPI types for Chatterbox API
    │   └── global.d.ts
    ├── proxy.ts               # Clerk middleware (route protection + org enforcement)
    └── instrumentation.ts     # Sentry server-side SDK init
```

---

## Core Modules Deep Dive

### Authentication & Multi-Tenancy

EchoForge uses **Clerk** for authentication with an **organization-first** model. Every user must belong to an organization before accessing the dashboard.

**Middleware (`src/proxy.ts`)**:
```typescript
export default clerkMiddleware(async (auth, req) => {
  const { userId, orgId } = await auth();

  if (!userId) await auth.protect();          // redirect to sign-in
  if (!orgId) redirect("/org-selection");     // force org selection
});
```

**tRPC Procedures** (`src/trpc/init.ts`):
- `baseProcedure` — any unauthenticated call; augmented with Sentry tracing middleware
- `authProcedure` — requires `userId`; throws `UNAUTHORIZED` otherwise
- `orgProcedure` — requires both `userId` **and** `orgId`; throws `FORBIDDEN` if no org; the `orgId` is forwarded to all router handlers, ensuring complete data isolation

All database queries in `voices` and `generations` routers filter by `ctx.orgId`, so it is architecturally impossible for one organization to access another's data.

---

### AI Backend — Chatterbox on Modal

The AI inference layer lives in `chatterbox_tts.py` — a Python file that deploys a serverless GPU application to [Modal](https://modal.com/).

**Key design decisions:**

| Decision | Detail |
|---|---|
| **GPU** | NVIDIA A10G — specified via `gpu="a10g"` in `chatterbox_tts.py` (line 84); Modal provisions this hardware automatically for each container |
| **Concurrency** | `@modal.concurrent(max_inputs=10)` — up to 10 simultaneous requests per container |
| **Cold start mitigation** | `scaledown_window=60*5` — containers stay warm for 5 minutes after last request |
| **Model loading** | `@modal.enter()` — model is loaded once per container lifecycle, not per request |
| **Voice storage** | R2 bucket is **mounted read-only** inside Modal as a FUSE filesystem — no download overhead per-request |
| **Security** | `X-Api-Key` header required; validated against `CHATTERBOX_API_KEY` env secret |
| **Auth** | API key injected via `modal.Secret.from_name("chatterbox-api-key")` — never in source code |

**Generation parameters the model accepts:**
| Parameter | Range | Default | UX Label |
|---|---|---|---|
| `temperature` | 0.0–2.0 | 0.8 | Creativity |
| `top_p` | 0.0–1.0 | 0.95 | Voice Variety |
| `top_k` | 1–10,000 | 1,000 | Expression Range |
| `repetition_penalty` | 1.0–2.0 | 1.2 | Natural Flow |
| `norm_loudness` | bool | true | Loudness normalization |

**Testing the Modal deployment locally:**
```bash
# CLI test
modal run chatterbox_tts.py \
  --prompt "Hello from Chatterbox [chuckle]." \
  --voice-key "voices/system/<voice-id>"

# HTTP test
curl -X POST "https://<your-modal-endpoint>/generate" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: <your-api-key>" \
  -d '{"prompt": "Hello!", "voice_key": "voices/system/<id>"}' \
  --output output.wav
```

---

### API Layer — tRPC Routers

All client–server communication goes through tRPC v11 with SuperJSON serialization (supports `Date`, `Map`, `Set` natively across the wire).

#### `voices` router
| Procedure | Type | Input | Description |
|---|---|---|---|
| `getAll` | Query | `{ query?: string }` | Returns `{ custom, system }` voice arrays. Custom voices are filtered by `orgId`; system voices are global. Supports case-insensitive full-text search on name + description. |
| `delete` | Mutation | `{ id: string }` | Deletes a custom voice owned by the org. Also deletes the R2 object (best-effort, ignores failure). |

#### `generations` router
| Procedure | Type | Input | Description |
|---|---|---|---|
| `getAll` | Query | — | All generations for the org, ordered by `createdAt DESC` |
| `getById` | Query | `{ id: string }` | Single generation with `audioUrl` pointing to `/api/audio/:id` |
| `create` | Mutation | `{ text, voiceId, temperature, topP, topK, repetitionPenalty }` | Full generation pipeline (see below) |

**`generations.create` pipeline:**
1. Check for active Polar subscription — throw `FORBIDDEN: SUBSCRIPTION_REQUIRED` if none
2. Resolve voice from DB (validates ownership for custom voices)
3. Verify `r2ObjectKey` exists — voice must have an uploaded sample
4. POST to Chatterbox API — receive raw WAV ArrayBuffer
5. Create `Generation` record in DB (without `r2ObjectKey`)
6. Upload WAV to R2 at key `generations/orgs/{orgId}/{generationId}`
7. Update `Generation` record with `r2ObjectKey`
8. Fire Polar usage event `tts_generation` with `{ characters: text.length }` *(fire-and-forget)*
9. Return `{ id: generationId }`

If step 6 or 7 fails, the orphaned DB record is cleaned up. Sentry logs every significant event in this pipeline.

#### `billing` router
| Procedure | Type | Description |
|---|---|---|
| `getStatus` | Query | Returns subscription status + estimated cost in cents from Polar |
| `createCheckout` | Mutation | Creates Polar checkout session and returns redirect URL |
| `createPortalSession` | Mutation | Creates Polar customer portal session for subscription management |

---

### Storage — Cloudflare R2

All audio assets (voices + generated speech) are stored in **Cloudflare R2** using an S3-compatible API (`@aws-sdk/client-s3`).

**Object key schema:**
```
voices/system/{voiceId}           # Pre-seeded system voice WAV samples
voices/custom/{orgId}/{voiceId}   # User-uploaded custom voice samples
generations/orgs/{orgId}/{generationId}  # Generated speech WAV files
```

**Audio serving strategy:**
- Audio is **never served directly from R2** (no public bucket)
- Next.js Route Handlers (`/api/audio/[generationId]` and `/api/voices/[voiceId]`) act as authenticated proxies — they fetch a 1-hour presigned URL from R2 and redirect to it
- This ensures only authenticated organization members can access audio files

**R2 utility functions** (`src/lib/r2.ts`):
```typescript
uploadAudio({ buffer, key, contentType? })  // PutObjectCommand
deleteAudio(key)                             // DeleteObjectCommand
getSignedAudioUrl(key)                       // GetObjectCommand + presigner (1hr TTL)
```

---

### Billing — Polar.sh Usage Metering

EchoForge uses **Polar.sh** as its billing infrastructure with a **pay-as-you-go** model.

**Pricing:** `$0.30 per 1,000 characters` (i.e., `COST_PER_UNIT = 0.0003` per character)

**Billing flow:**
1. User subscribes via the Polar checkout (Stripe-backed)
2. Every successful generation fires a Polar usage event: `{ name: "tts_generation", metadata: { characters: N } }`
3. Polar aggregates events and bills the subscription per its meter configuration
4. The sidebar shows real-time estimated cost pulled from `customerState.activeSubscriptions[].meters[].amount`

**Security:** The Polar access token (`POLAR_ACCESS_TOKEN`) is server-only — never exposed to the browser. All billing operations go through `orgProcedure`, guaranteeing the customer external ID is always the trusted `orgId` from Clerk.

---

### Database — PostgreSQL + Prisma

**Schema** (`prisma/schema.prisma`):

#### `Voice` model
```prisma
model Voice {
  id          String        @id @default(cuid())
  orgId       String?                           # null = system voice (global)
  name        String
  description String?
  category    VoiceCategory @default(GENERAL)
  language    String        @default("en-US")
  variant     VoiceVariant                      # SYSTEM | CUSTOM
  r2ObjectKey String?                           # R2 object key for the WAV sample
  generations Generation[]
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt
  @@index([variant])
  @@index([orgId])
}
```

#### `Generation` model
```prisma
model Generation {
  id                String   @id @default(cuid())
  orgId             String
  voiceId           String?                     # nullable (SetNull on voice delete)
  voice             Voice?   @relation(...)
  text              String
  voiceName         String                      # denormalized snapshot of voice name
  r2ObjectKey       String?
  temperature       Float
  topP              Float
  topK              Int
  repetitionPenalty Float
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  @@index([orgId])
  @@index([voiceId])
}
```

**Key design decisions:**
- `voiceName` is **denormalized** into `Generation` — so the history always shows the correct voice name even if the voice is later renamed or deleted
- `voiceId` is nullable with `onDelete: SetNull` — deleting a voice doesn't cascade-delete its generations
- The Prisma client is generated into `src/generated/prisma` (not `node_modules`) for better Next.js edge compatibility using `@prisma/adapter-pg`

**12 Voice Categories:**
`AUDIOBOOK` · `CONVERSATIONAL` · `CUSTOMER_SERVICE` · `GENERAL` · `NARRATIVE` · `CHARACTERS` · `MEDITATION` · `MOTIVATIONAL` · `PODCAST` · `ADVERTISING` · `VOICEOVER` · `CORPORATE`

---

### Frontend Features

#### Dashboard (`/`)
The dashboard home serves as the product's hero entry point:
- A **gradient hero input** (`text-input-panel.tsx`) — a visually striking multi-layer gradient-bordered textarea with live cost estimation
- **6 Quick Action cards** arranged in a responsive grid, each pre-seeded with a creative scenario that deep-links into the TTS studio with a pre-filled prompt
- A **`HeroPattern` background** with a wavy animated SVG grid using `simplex-noise`

#### Text-to-Speech Studio (`/text-to-speech`)

The primary workspace with three panes:

**Left pane — Text Input** (`text-input-panel.tsx`):
- Textarea with up to 5,000 character limit
- Character counter + live cost badge
- Prompt suggestions button (opens drawer with curated writing prompt ideas)
- Generate button disabled until text is non-empty + subscription check

**Right pane — Settings** (`settings-panel.tsx`):
Two tabs — **Settings** and **History**:
- *Settings tab*: Voice selector dropdown + 4 AI parameter sliders
- *History tab*: All past generations listed with voice avatar, name, relative timestamp; click to open the detail view

**Far-right pane — Voice Preview** (`voice-preview-panel.tsx`, desktop only):
- WaveSurfer.js waveform visualization
- `mm:ss / mm:ss` time display
- Play / Pause / Seek ±10s controls
- One-click WAV download with a slugified filename derived from the text content
- Mobile version (`voice-preview-mobile.tsx`) uses a compact bottom-mounted drawer

**`TextToSpeechForm`** is built with **TanStack Form** and wraps the entire studio, providing:
- A typed form context accessible from any nested component via `useTypedAppFormContext`
- Form submission triggers `generations.create` mutation
- On success: navigates to `/text-to-speech/{generationId}` (the detail view)
- Subscription gate: catches `SUBSCRIPTION_REQUIRED` error and redirects to checkout

**Voice Selector** (`voice-selector.tsx`):
- Groups voices into "Your voices" and "System voices" sections
- Searchable combobox with `cmdk`
- Animated popover built on Radix UI primitives
- Reads from `TTSVoicesContext` to avoid prop drilling

#### Voice Explorer (`/voices`)
- Search bar (debounced, URL-synced via `nuqs`)
- `VoicesToolbar` with a "Clone a new voice" button that opens `VoiceCreateDialog`
- Separate sections for custom and system voices
- `VoicesView` uses `useSuspenseQuery` for data fetching with a Suspense/ErrorBoundary wrapper in the layout

#### Voice Creation Flow
`VoiceCreateDialog` → `VoiceCreateForm`:

1. **Audio tab selection**: *Upload* (drag-and-drop dropzone, any audio format, 20 MB max) or *Record* (browser microphone via RecordRTC)
2. **Voice name** input with `Tag` icon prefix
3. **Category** select from 12 options
4. **Language** searchable combobox built with `locale-codes` library (hundreds of locales with region display names)
5. **Description** textarea
6. Validation with Zod schema via TanStack Form's `validators.onSubmit`
7. On submit: `POST /api/voices/create?name=...&category=...&language=...` with raw audio binary as body

The `VoiceCreateForm` uses a `footer` render prop pattern, allowing the `VoiceCreateDialog` to place the submit button inside the dialog footer.

---

## Data Models

### VoiceVariant Enum
| Value | Description |
|---|---|
| `SYSTEM` | Curated by EchoForge; available to all organizations |
| `CUSTOM` | Created by an organization; scoped to that org only |

### Voice Categories

| Key | Label | Example Use |
|---|---|---|
| `AUDIOBOOK` | Audiobook | Long-form narration, chapter reading |
| `CONVERSATIONAL` | Conversational | Chat agents, casual dialogue |
| `CUSTOMER_SERVICE` | Customer Service | IVR, support bots |
| `GENERAL` | General | All-purpose narration |
| `NARRATIVE` | Narrative | Storytelling, documentary |
| `CHARACTERS` | Characters | Game NPCs, animation |
| `MEDITATION` | Meditation | Guided relaxation, mindfulness |
| `MOTIVATIONAL` | Motivational | Coaching, fitness content |
| `PODCAST` | Podcast | Episode intros, hosting |
| `ADVERTISING` | Advertising | Commercials, promos |
| `VOICEOVER` | Voiceover | Corporate videos, explainers |
| `CORPORATE` | Corporate | Training, presentations |

### System Voices (20 pre-built)

| Name | Category | Locale | Description |
|---|---|---|---|
| Aaron | Audiobook | en-US | Soothing and calm, like a self-help audiobook narrator |
| Abigail | Conversational | en-GB | Friendly and warm, approachable tone |
| Anaya | Customer Service | en-IN | Polite and professional |
| Andy | General | en-US | Versatile and clear, reliable all-purpose narrator |
| Archer | Narrative | en-US | Laid-back and reflective, storytelling pace |
| Brian | Customer Service | en-US | Professional and helpful |
| Chloe | Corporate | en-AU | Bright and bubbly, cheerful personality |
| Dylan | General | en-US | Thoughtful and intimate |
| Emmanuel | Characters | en-US | Nasally and distinctive, cartoon-like quality |
| Ethan | Voiceover | en-US | Polished and warm, studio-quality delivery |
| Evelyn | Conversational | en-US | Warm Southern charm |
| Gavin | Meditation | en-US | Calm and reassuring, smooth natural flow |
| Gordon | Motivational | en-US | Warm and encouraging, uplifting tone |
| Ivan | Characters | ru-RU | Deep and cinematic, dramatic movie-character presence |
| Laura | Conversational | en-US | Authentic and warm, conversational Midwestern tone |
| Lucy | Customer Service | en-US | Direct and composed, professional phone manner |
| Madison | Podcast | en-US | Energetic and unfiltered, casual chatty vibe |
| Marisol | Advertising | en-US | Confident and polished, persuasive ad-ready delivery |
| Meera | Customer Service | en-IN | Friendly and helpful, service-oriented tone |
| Walter | Narrative | en-US | Old and raspy with deep gravitas |

---

## Environment Variables

All server-side environment variables are validated at startup using `@t3-oss/env-nextjs` with Zod schemas. The app will throw a descriptive error at build/start time if any variable is missing.

```bash
# .env

# ── Application ──────────────────────────────────────────────
APP_URL=https://your-domain.com

# ── Database ─────────────────────────────────────────────────
DATABASE_URL=postgresql://user:password@host:5432/echoforge

# ── Clerk Authentication ──────────────────────────────────────
# Obtain from https://dashboard.clerk.com
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
CLERK_SECRET_KEY=sk_live_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/org-selection

# ── Cloudflare R2 ─────────────────────────────────────────────
# Obtain from https://dash.cloudflare.com → R2 → Manage API Tokens
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key_id
R2_SECRET_ACCESS_KEY=your_secret_access_key
R2_BUCKET_NAME=echoforge-app

# ── Chatterbox TTS API (Modal deployment) ────────────────────
CHATTERBOX_API_URL=https://your-org--chatterbox-tts-chatterbox-serve.modal.run
CHATTERBOX_API_KEY=your_api_key_here

# ── Polar.sh Billing ─────────────────────────────────────────
# Obtain from https://polar.sh → Settings → API
POLAR_ACCESS_TOKEN=polar_...
POLAR_SERVER=sandbox          # or "production"
POLAR_PRODUCT_ID=prod_...

# ── Optional ─────────────────────────────────────────────────
SENTRY_AUTH_TOKEN=...         # For source map uploads
SKIP_ENV_VALIDATION=true      # Skip env validation (CI/testing only)
```

> **Note:** `NEXT_PUBLIC_*` variables are exposed to the browser. All others are server-only, enforced by `server-only` imports and `t3-oss/env-nextjs`.

---

## Getting Started

### Prerequisites
- Node.js 20+
- pnpm / npm / yarn
- PostgreSQL database (local or hosted)
- Cloudflare R2 bucket
- Clerk account
- Polar.sh account
- Modal account + deployed `chatterbox_tts.py`

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/Rahul9214/EchoForge.git
cd echoforge

# 2. Install dependencies (Prisma client is auto-generated via postinstall)
npm install

# 3. Set up environment variables
cp .env.example .env
# Edit .env with your credentials

# 4. Run database migrations
npx prisma migrate deploy

# 5. Seed system voices (requires WAV files in scripts/system-voices/)
npx tsx scripts/seed-system-voices.ts

# 6. Start the development server
npm run dev
```

The app will be available at `http://localhost:3000`.

### Setting Up the AI Backend

```bash
# 1. Install Modal CLI and Python dependencies
pip install modal chatterbox-tts fastapi

# 2. Authenticate with Modal
modal setup

# 3. Create Modal secrets
modal secret create hf-token HF_TOKEN=your_huggingface_token
modal secret create chatterbox-api-key CHATTERBOX_API_KEY=your_chosen_key
modal secret create cloudflare-r2 \
  AWS_ACCESS_KEY_ID=your_r2_key_id \
  AWS_SECRET_ACCESS_KEY=your_r2_secret

# 4. Deploy the TTS API
modal deploy chatterbox_tts.py

# Copy the deployment URL and set it as CHATTERBOX_API_URL in .env
```

---

## Scripts & Tooling

| Script | Command | Description |
|---|---|---|
| Dev server | `npm run dev` | Start Next.js in development mode with HMR |
| Build | `npm run build` | Production build with Sentry source map upload |
| Start | `npm run start` | Start production server |
| Lint | `npm run lint` | ESLint with Next.js config |
| Seed voices | `npx tsx scripts/seed-system-voices.ts` | Uploads 20 system voice WAVs to R2 and seeds DB |
| Sync API types | `npm run sync-api` | Fetches OpenAPI spec from Chatterbox and regenerates `types/chatterbox-api.d.ts` |
| Prisma generate | `npx prisma generate` | Regenerates Prisma client (also runs automatically `postinstall`) |
| Prisma migrate | `npx prisma migrate dev` | Create and apply new migrations in development |
| Prisma studio | `npx prisma studio` | Visual database browser at `localhost:5555` |

---

## UI Component Library

EchoForge is built on top of **shadcn/ui** with extensive customization. The `src/components/ui/` directory contains **57 components**:

### Core UI Primitives
`Button` · `Input` · `Textarea` · `Select` (native + Radix) · `Checkbox` · `Switch` · `Radio Group` · `Slider` · `Label` · `Badge` · `Skeleton` · `Spinner` · `Progress` · `Separator`

### Layout & Navigation
`Sidebar` (21 KB — the full collapsible sidebar system) · `Navigation Menu` · `Tabs` · `Resizable` (panels) · `Scroll Area` · `Pagination` · `Breadcrumb`

### Overlay Components
`Dialog` · `Alert Dialog` · `Sheet` · `Drawer` (Vaul) · `Popover` · `Tooltip` · `Hover Card` · `Dropdown Menu` · `Context Menu` · `Menubar`

### Data Display
`Table` · `Chart` (Recharts wrapper) · `Accordion` · `Carousel` (Embla) · `Calendar` · `Avatar` · `Card`

### Form Components
`Form` (react-hook-form wrapper) · `Field` (TanStack Form wrapper) · `Combobox` · `Command` · `Input OTP` · `Input Group` · `Button Group` · `Toggle` · `Toggle Group`

### Feedback
`Alert` · `Sonner` (toast) · `Empty` (empty states)

### Custom Components
`WavyBackground` — animated SVG background using simplex noise for the dashboard hero  
`VoiceAvatar` — DiceBear deterministic avatar generated from a seed (voice ID or name)  
`PageHeader` — mobile-only header with sidebar trigger

### Design System Tokens
Defined in `globals.css` with CSS custom properties following the shadcn/ui convention:
- `--background`, `--foreground`, `--muted`, `--muted-foreground`
- `--primary`, `--secondary`, `--destructive`, `--border`, `--ring`
- `--chart-1` through `--chart-5` for data visualization
- `--font-inter`, `--font-geist-mono` as CSS font variables

---

## Voice System

### Voice Avatar Generation

Every voice is represented by a unique, deterministic avatar generated by **DiceBear** (`@dicebear/core` + `@dicebear/collection`).

```typescript
// src/components/voice-avatar/use-voice-avatar.ts
// Creates an SVG avatar from a seed string (voice ID or name)
// Ensures consistent visual identity across the app
```

### Language Display

Voice cards display a **country flag emoji** and **region name** derived from the voice's locale tag (BCP 47 format, e.g., `en-US`, `ru-RU`):

```typescript
const flag = [...country.toUpperCase()]
  .map((c) => String.fromCodePoint(0x1f1e6 + c.charCodeAt(0) - 65))
  .join(""); // "US" → 🇺🇸

const region = new Intl.DisplayNames(["en"], { type: "region" }).of(country);
```

This uses the Unicode Regional Indicator Symbol block — no emoji library required.

### In-Browser Recording

The `VoiceRecorder` component (`src/features/voices/components/voice-recorder.tsx`) uses `useAudioRecorder` hook powered by **RecordRTC**:

- Requests microphone permission
- Records in WAV format
- Shows an **animated live waveform** while recording (WaveSurfer in real-time mode)
- Displays elapsed time in `HH:MM:SS` format
- On stop: creates a `File` object from the Blob, injected directly into the voice creation form
- Supports re-record and remove

---

## Audio Pipeline

### Generation Request Flow

```
Browser                          Next.js Server              Modal (Python)
   │                                    │                         │
   ├─ tRPC generations.create ─────────▶│                         │
   │                                    ├─ Verify subscription    │
   │                                    ├─ Resolve voice in DB    │
   │                                    ├─ POST /generate ────────▶│
   │                                    │                         ├─ Load voice from R2 mount
   │                                    │                         ├─ ChatterboxTurboTTS.generate()
   │                                    │◀── WAV ArrayBuffer ──────┤
   │                                    ├─ Create Generation in DB │
   │                                    ├─ Upload WAV to R2       │
   │                                    ├─ Update Generation.r2ObjectKey
   │                                    ├─ Polar usage event (async)
   │◀── { id: generationId } ──────────┤                         │
   │                                    │                         │
   ├─ navigate /text-to-speech/{id}     │                         │
   ├─ tRPC generations.getById ────────▶│                         │
   │◀── { ...generation, audioUrl } ───┤                         │
   │                                    │                         │
   ├─ fetch audioUrl (/api/audio/{id}) ▶│                         │
   │                                    ├─ Lookup generation.r2ObjectKey
   │                                    ├─ Generate presigned URL (1hr TTL)
   │◀── 302 redirect to presigned URL ──┤                         │
   │                                    │                         │
   ├─ WaveSurfer loads audio from R2 URL│                         │
```

### Audio Playback — WaveSurfer Hook

`useWaveSurfer` (`src/features/text-to-speech/hooks/use-wavesurfer.ts`) manages WaveSurfer.js lifecycle:

- **Creates** a WaveSurfer instance on mount with a custom style (bar waveform, teal progress color `#4A8A9A`)
- **Autoplay** support with graceful handling of browser autoplay policies (`NotAllowedError` is silently caught)
- **Proper cleanup**: `ws.destroy()` called on unmount with a `destroyed` flag to suppress in-flight callbacks
- **Seek operations**: `seekForward(seconds)` / `seekBackward(seconds)` clamp to valid `[0, duration]` range
- **Time tracking**: `timeupdate` event drives `currentTime` state, displayed as `mm:ss`

### Simple Audio Playback Hook

`useAudioPlayback` (`src/hooks/use-audio-playback.ts`) handles simpler use cases (voice card previews):

- Accepts a `File | string` (file for local preview, string for API URL)
- Creates an `HTMLAudioElement` and tracks play/pause/loading state
- Used in `VoiceCard`, `FileDropzone`, and `VoiceRecorder` for quick play/pause

---

## Monitoring & Observability

EchoForge uses **Sentry** for full-stack error tracking and performance monitoring:

**Server-side** (`sentry.server.config.ts`):
- Sentry Node SDK initialized in `src/trpc/init.ts`
- `sentryMiddleware` wraps all tRPC procedures — automatically captures RPC input in error reports
- Structured logging with `Sentry.logger.info / .error` at key generation pipeline steps

**Client-side** (`src/instrumentation-client.ts`):
- Browser Sentry SDK captures unhandled errors and React error boundaries
- `global-error.tsx` — Next.js root error boundary reports to Sentry

**Next.js integration** (`next.config.ts`):
- `withSentryConfig` wraps the entire Next.js config — integrates source map uploads at build time for human-readable production stack traces
- `tunnelRoute: "/monitoring"` routes Sentry requests through the Next.js app, bypassing ad-blockers that block `sentry.io` directly
- `removeDebugLogging: true` tree-shakes Sentry debug statements from the production bundle for smaller output

---

## Security Model

| Concern | Mitigation |
|---|---|
| **Authentication** | Clerk JWT verified on every request; server-only SDK never trusts client-supplied user IDs |
| **Authorization** | `orgProcedure` middleware extracts `orgId` from verified JWT — not from request body |
| **Tenant isolation** | All DB queries filtered by `ctx.orgId`; `orgId` is never accepted as user input |
| **Audio access control** | R2 bucket is private; audio served via short-lived (1-hour) presigned URLs through authenticated Route Handlers |
| **API key** | Chatterbox API key stored in Modal secrets, injected via env — never in source code or browser |
| **Environment validation** | All secrets validated at startup by Zod; app refuses to start if any are missing |
| **Input validation** | All tRPC inputs validated by Zod schemas; text max 5,000 characters enforced at both UI and server |
| **File upload** | Voice uploads streamed directly to R2 via Route Handler; 20 MB limit enforced by Next.js proxy config |
| **CSRF** | Not applicable — tRPC uses per-request Clerk auth cookies; no traditional form submissions |

---

## Performance Considerations

- **`useSuspenseQuery`** — TTS views and voice lists use Suspense queries, enabling streaming SSR and concurrent rendering
- **Query invalidation** — After voice creation/deletion or generation, only the relevant query cache is invalidated via `queryClient.invalidateQueries`, not the entire cache
- **`Promise.all`** in `voices.getAll` — custom and system voices are fetched in parallel, halving latency
- **Sidebar state** — The collapsed/expanded state is persisted in a `sidebar_state` cookie, read server-side in the layout, preventing layout shift on page load
- **`proxyClientMaxBodySize: "20mb"`** — Next.js proxy body size increased to handle large voice file uploads
- **Sentry `removeDebugLogging`** — Debug log statements are tree-shaken from the production bundle
- **Route-level Suspense** — Each dashboard page wraps its view in a Suspense boundary so navigation feels instant with streaming
- **`simplex-noise`** for hero background — WebGL-free, lightweight noise-based animation
- **`nuqs`** for URL state — Voice search query is synced to the URL with debouncing, enabling shareable filtered views

---

## Deployment

EchoForge is deployed to [**Railway**](https://railway.app/) — the live production instance is available at:

> **[https://echoforge-production-0143.up.railway.app/](https://echoforge-production-0143.up.railway.app/)**

Railway runs the Next.js server as a long-lived container (not serverless functions), which makes it a great fit for the streaming audio proxy routes and tRPC handler. The PostgreSQL database is also hosted on Railway in the same project, keeping latency between the app server and DB minimal.

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and link your project
railway login
railway link

# Deploy
railway up

# Or connect your GitHub repo in the Railway dashboard for
# automatic deployments on every push to main.
```

**Environment variables** are set directly in the Railway dashboard under your service → Variables. The Sentry auth token for source map uploads goes in `.env.sentry-build-plugin` locally or as a build-time env var in Railway.

**Infrastructure checklist (what is deployed and running):**
- [x] **PostgreSQL** — hosted on Railway (same project, private network)
- [x] **Cloudflare R2** — private bucket storing all voice samples and generated audio
- [x] **Clerk** — production authentication instance
- [x] **Polar.sh** — billing and usage metering
- [x] **Modal** — `chatterbox_tts.py` deployed as a serverless GPU endpoint (`modal deploy chatterbox_tts.py`)
- [x] **Sentry** — error tracking enabled for both server and client
- [x] **Prisma migrations** — run against the Railway PostgreSQL instance
- [x] **System voices** — 20 voices seeded into DB + R2 via `seed-system-voices.ts`

---

## Contributing

1. Fork the repository
2. Create your feature branch: `git checkout -b feature/amazing-feature`
3. Commit your changes: `git commit -m 'feat: add amazing feature'`
4. Push to the branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

**Coding standards:**
- TypeScript strict mode — no `any` unless absolutely necessary
- All tRPC procedures must use at minimum `authProcedure`; org-scoped data must use `orgProcedure`
- New features should follow the feature-sliced directory structure: `components/`, `views/`, `hooks/`, `data/`
- Environment variables must be added to `src/lib/env.ts` with a Zod validation type

---

<div align="center">

**Built with ❤️ using Next.js, tRPC, Prisma, Clerk, Polar.sh, Modal, Chatterbox TTS, Cloudflare R2, and Railway**

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Railway-0B0D0E?style=for-the-badge&logo=railway)](https://echoforge-production-0143.up.railway.app/)
[![GitHub](https://img.shields.io/badge/Source-GitHub-181717?style=for-the-badge&logo=github)](https://github.com/Rahul9214/EchoForge)

</div>
