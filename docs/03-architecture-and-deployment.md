# Architecture Diagram & Cloud Deployment

> Architecture overview, system diagram, and cloud deployment details for presentation.

---

## 1. High-Level Architecture Diagram (Text)

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         PROJECT ATLAS                                   ║
║              AI-Native Animated Edutainment Video Platform              ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║   ┌──────────────────────────────────────────────────────────────────┐  ║
║   │                    PRESENTATION LAYER                             │  ║
║   │                                                                    │  ║
║   │   ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │  ║
║   │   │   Next.js    │  │  Zustand     │  │    React Query         │ │  ║
║   │   │   App Router │  │  State Mgmt  │  │    (8s polling)        │ │  ║
║   │   │   + SSR      │  │              │  │                        │ │  ║
║   │   └──────────────┘  └──────────────┘  └────────────────────────┘ │  ║
║   │                                                                    │  ║
║   │   UI: Radix UI + Tailwind CSS + ReactFlow + Phosphor Icons       │  ║
║   │   Features: Timeline Editor, Scene Graph, Voice Preview,          │  ║
║   │             Character Chat, Video Preview, Cost Dashboard         │  ║
║   └──────────────────────────────────┬───────────────────────────────┘  ║
║                                      │ HTTPS (Nginx)                    ║
║   ┌──────────────────────────────────▼───────────────────────────────┐  ║
║   │                      API LAYER                                    │  ║
║   │                                                                    │  ║
║   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐ │  ║
║   │   │ Express  │ │ Better   │ │ Helmet   │ │  14 Route Modules  │ │  ║
║   │   │ Server   │ │ Auth     │ │ + CORS   │ │  + Zod Validation  │ │  ║
║   │   │ Port 3001│ │          │ │ + Rate   │ │                    │ │  ║
║   │   └──────────┘ └──────────┘ └──────────┘ └────────────────────┘ │  ║
║   │                                                                    │  ║
║   │   Routes: projects, topics, research, scripts, voice, scenes,    │  ║
║   │           frames, characters, captions, renders, exports, costs, │  ║
║   │           discover, preview                                       │  ║
║   └──────────────────────────────────┬───────────────────────────────┘  ║
║                                      │ BullMQ Jobs                      ║
║   ┌──────────────────────────────────▼───────────────────────────────┐  ║
║   │                    WORKER LAYER                                   │  ║
║   │                                                                    │  ║
║   │   ┌──────────────────────────────────────────────────────────┐   │  ║
║   │   │              12 BullMQ Workers                            │   │  ║
║   │   │                                                            │   │  ║
║   │   │  Topic Discovery → Channel Analysis → Scene Planning     │   │  ║
║   │   │  Frame Generation (c=1) → Video Generation (c=2)         │   │  ║
║   │   │  TTS Generation → Character Generation (c=1)             │   │  ║
║   │   │  Render/Compose (c=1) → Export → Captions                │   │  ║
║   │   │  Transition Planning                                      │   │  ║
║   │   └──────────────────────────────────────────────────────────┘   │  ║
║   │                                                                    │  ║
║   │   Tools: FFmpeg (video), Sharp (images), ffprobe (duration)      │  ║
║   └──────────┬──────────────────────────────────┬────────────────────┘  ║
║              │                                  │                        ║
║   ┌──────────▼──────────┐          ┌───────────▼───────────┐            ║
║   │    PostgreSQL 16     │          │      Redis 7          │            ║
║   │    (Prisma ORM)      │          │    (BullMQ Queue)     │            ║
║   │                      │          │    (Voice Cache)      │            ║
║   │  Tables:             │          │                       │            ║
║   │  - User/Auth (4)     │          └───────────────────────┘            ║
║   │  - Project/Content(8)│                                               ║
║   │  - Cost Tracking (6) │                                               ║
║   │  - Renders/Export(3) │                                               ║
║   └──────────────────────┘                                               ║
║                                                                          ║
╠══════════════════════════════════════════════════════════════════════════╣
║                     EXTERNAL AI SERVICES                                 ║
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐   ║
║   │                    GOOGLE CLOUD                                  │   ║
║   │                                                                   │   ║
║   │   ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐  │   ║
║   │   │  Gemini LLM    │  │  Gemini Image  │  │    Veo 3.1      │  │   ║
║   │   │  (ADK Agents)  │  │  Generation    │  │  Video Gen      │  │   ║
║   │   │                │  │                │  │                  │  │   ║
║   │   │  • 2.5 Flash   │  │  • 3 Pro Image │  │  • Direct API   │  │   ║
║   │   │  • 3.1 Pro     │  │    Preview     │  │  • 4-8s clips   │  │   ║
║   │   │                │  │                │  │  • 720p          │  │   ║
║   │   │  8 Agent Types │  │  Frame gen +   │  │                  │  │   ║
║   │   │  via @google/  │  │  validation    │  │  via @google/    │  │   ║
║   │   │  adk           │  │                │  │  genai           │  │   ║
║   │   └────────────────┘  └────────────────┘  └─────────────────┘  │   ║
║   └─────────────────────────────────────────────────────────────────┘   ║
║                                                                          ║
║   ┌────────────────┐  ┌────────────────┐  ┌─────────────────────────┐  ║
║   │   ElevenLabs   │  │    fal.ai      │  │     Replicate           │  ║
║   │                │  │                │  │                          │  ║
║   │  • v3 TTS      │  │  • Kling O3    │  │  • google/veo-3.1       │  ║
║   │  • SFX Gen     │  │  • SeDance 1.5 │  │  • kwaivgi/kling-v2.1   │  ║
║   │  • Voice API   │  │    Pro         │  │  • bytedance/seedance   │  ║
║   │                │  │                │  │  • bytedance/seed.-lite │  ║
║   └────────────────┘  └────────────────┘  └─────────────────────────┘  ║
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐   ║
║   │                   RESEARCH APIs                                  │   ║
║   │  Brave Search · OpenAlex · Semantic Scholar · Wikipedia ·       │   ║
║   │  CrossRef · Reddit · HackerNews · Google Trends · YouTube v3   │   ║
║   └─────────────────────────────────────────────────────────────────┘   ║
║                                                                          ║
║   ┌─────────────────┐                                                   ║
║   │  Object Storage  │  AWS S3 / DigitalOcean Spaces                   ║
║   │  (Media Assets)  │  Images, Videos, Audio, Renders                  ║
║   └─────────────────┘                                                   ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## 2. Video Production Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VIDEO PRODUCTION PIPELINE                             │
│                    (End-to-End Flow)                                     │
└─────────────────────────────────────────────────────────────────────────┘

  USER INPUT                                                     FINAL OUTPUT
  ─────────                                                     ────────────
  • Topic idea          ═══════════════════════════════════▶     • HD MP4 Video
  • Platform (YT/TT)                                             • Subtitles
  • Video type                                                   • Multiple exports
  • Tone preference                                              • Cost breakdown

  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  TOPIC   │───▶│ RESEARCH │───▶│  SCRIPT  │───▶│  VOICE   │
  │ DISCOVER │    │ SYNTHESIS│    │ WRITING  │    │ NARRATION│
  │          │    │          │    │          │    │          │
  │ Gemini   │    │ Gemini + │    │ Gemini   │    │ElevenLabs│
  │ 2.5 Flash│    │ 8 Search │    │ 3.1 Pro  │    │ v3 TTS   │
  │          │    │ APIs     │    │          │    │          │
  │ 10 topics│    │ Brief +  │    │ 9-part   │    │ MP3 +    │
  │ scored   │    │ claims + │    │ script   │    │ word     │
  │          │    │ sources  │    │          │    │ timings  │
  └──────┬──┘    └──────┬───┘    └─────┬────┘    └────┬─────┘
         │              │              │               │
    User selects   Auto-run      User approves    Auto-generates
         │              │              │               │
         ▼              ▼              ▼               ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐
  │ CHARACTER│   │  SCENE   │   │  FRAME   │   │    VIDEO     │
  │ DESIGN   │   │ PLANNING │   │ GENERATE │   │  GENERATION  │
  │          │   │          │   │          │   │              │
  │ Gemini   │   │ Gemini   │   │ Gemini   │   │ Veo / Kling  │
  │ 3 Pro    │   │ 3.1 Pro  │   │ 3 Pro    │   │ SeDance /    │
  │ Image    │   │          │   │ Image    │   │ Replicate    │
  │          │   │ 5-14s    │   │          │   │              │
  │ Portrait │   │ scenes   │   │ Start +  │   │ 5-15s clips  │
  │ + desc   │   │ + prompts│   │ End frame│   │ per scene    │
  │          │   │ + motion │   │ per scene│   │              │
  └──────────┘   └──────────┘   └─────┬────┘   └──────┬───────┘
                                      │                │
                              Style reference    ┌─────▼───────┐
                              chain (N→N+1)      │ KEN BURNS   │
                                                 │ FALLBACK    │
                                                 │ (if fails)  │
                                                 │ FFmpeg pan/ │
                                                 │ zoom — $0   │
                                                 └─────┬───────┘
                                                       │
                                                       ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                      COMPOSITION (RENDER)                         │
  │                                                                    │
  │  1. Speed-adjust clips to match narration timing                 │
  │  2. Crossfade transitions (FFmpeg xfade filter)                  │
  │  3. AI Sound Design:                                              │
  │     • Gemini designs ambient sound descriptions                  │
  │     • ElevenLabs generates ambient SFX (1 per scene)             │
  │     • ElevenLabs generates transition SFX (at boundaries)        │
  │  4. Audio mixing:                                                 │
  │     • Sidechain ducking (bed audio ducks under narration)        │
  │     • Mix: voiceover + bed + ambient + transitions               │
  │  5. Subtitle generation:                                          │
  │     • Word-level timestamps from TTS                             │
  │     • ASS format with custom fonts (Google Fonts)                │
  │     • Style: colors, font size, position per caption settings    │
  │  6. Final encode: H.264 MP4, CRF 18, AAC 192kbps, 24fps        │
  │                                                                    │
  └───────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                         EXPORT                                    │
  │                                                                    │
  │  Formats: MP4, MOV, WebM                                         │
  │  Resolutions: 720p, 1080p, 4K                                    │
  │  Quality: Standard, High, Ultra                                   │
  │  Storage: S3 / DigitalOcean Spaces                               │
  │  Download: Signed URLs with Content-Disposition                  │
  └───────────────────────────────────────────────────────────────────┘
```

---

## 3. Cloud Deployment Architecture

### 3.1 Production Infrastructure

```
                    ┌─────────────────────────────┐
                    │         INTERNET              │
                    │  video.example.com (HTTPS)    │
                    └──────────────┬────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │          NGINX                 │
                    │    SSL Termination             │
                    │    Let's Encrypt Certs         │
                    │    Reverse Proxy               │
                    │                                │
                    │  /api/* ──▶ api:3001           │
                    │  /*     ──▶ web:3000           │
                    │  /health ──▶ api:3001          │
                    │                                │
                    │  Timeouts: 300s (video gen)    │
                    │  Max Body: 50MB                │
                    └────┬──────────────┬────────────┘
                         │              │
              ┌──────────▼──┐   ┌──────▼──────────┐
              │  Express API │   │  Next.js Web    │
              │  Container   │   │  Container      │
              │  Port 3001   │   │  Port 3000      │
              │              │   │                  │
              │  Node.js 20  │   │  Node.js 20     │
              │  Alpine      │   │  Standalone      │
              └──────┬───────┘   └─────────────────┘
                     │
              ┌──────▼──────────┐
              │  Workers         │
              │  Container       │
              │                  │
              │  Node.js 20      │
              │  + FFmpeg         │
              │  12 BullMQ       │
              │  workers         │
              └──────┬───────────┘
                     │
         ┌───────────┼───────────┐
         │                       │
  ┌──────▼──────┐    ┌──────────▼──┐
  │ PostgreSQL  │    │    Redis    │
  │ 16 Alpine   │    │   7 Alpine  │
  │             │    │             │
  │ Volume:     │    │ Volume:     │
  │ pgdata      │    │ redisdata   │
  └─────────────┘    └─────────────┘
```

### 3.2 Docker Service Configuration

| Service | Image | Port | Volumes | Dependencies |
|---------|-------|------|---------|-------------|
| **postgres** | postgres:16-alpine | 5432 | pgdata | - |
| **redis** | redis:7-alpine | 6379 | redisdata | - |
| **migrate** | atlas-api (build) | - | - | postgres |
| **api** | atlas-api (build) | 3001 | local-storage | postgres, redis, migrate |
| **workers** | atlas-workers (build) | - | local-storage | postgres, redis |
| **web** | atlas-web (build) | 3000 | - | api |
| **nginx** | nginx:alpine | 80, 443 | certs, nginx.conf | api, web |

### 3.3 Environment Variables (Production)

```
# Database
DATABASE_URL=postgresql://atlas:${DB_PASSWORD}@postgres:5432/atlas_dev
DB_PASSWORD=<secure>

# API
PORT=3001
NODE_ENV=production
CORS_ORIGIN=https://video.example.com
NEXT_PUBLIC_API_URL=https://video.example.com/api

# Google Cloud
GEMINI_API_KEY=<key>

# ElevenLabs
ELEVENLABS_API_KEY=<key>

# Video Providers
VIDEO_PROVIDER=replicate-kling  (default)
FAL_KEY=<key>
REPLICATE_API_TOKEN=<key>

# Search
BRAVE_SEARCH_API_KEY=<key>
YOUTUBE_API_KEY=<key>

# Storage (S3-compatible)
S3_BUCKET=atlas-media
S3_ENDPOINT=<endpoint>
AWS_ACCESS_KEY_ID=<key>
AWS_SECRET_ACCESS_KEY=<key>
AWS_REGION=us-east-1
```

### 3.4 Production Domain
- **Domain**: `video.example.com`
- **SSL**: Let's Encrypt (auto-renewal)
- **Protocol**: HTTPS only (HTTP → 301 redirect)

---

## 4. Data Flow Diagrams

### 4.1 Authentication Flow

```
Browser                    Nginx                Express API           PostgreSQL
  │                          │                     │                      │
  │  POST /api/auth/signup   │                     │                      │
  │─────────────────────────▶│────────────────────▶│                      │
  │                          │                     │  Create User         │
  │                          │                     │─────────────────────▶│
  │                          │                     │  Create Session      │
  │                          │                     │─────────────────────▶│
  │  Set-Cookie: session     │                     │                      │
  │◀─────────────────────────│◀────────────────────│                      │
  │                          │                     │                      │
  │  GET /api/projects       │                     │                      │
  │  Cookie: session         │                     │                      │
  │─────────────────────────▶│────────────────────▶│                      │
  │                          │                     │  requireAuth()       │
  │                          │                     │  Validate session    │
  │                          │                     │─────────────────────▶│
  │                          │                     │  ✓ User attached     │
  │                          │                     │◀─────────────────────│
  │  200 OK + projects       │                     │  Query projects      │
  │◀─────────────────────────│◀────────────────────│─────────────────────▶│
```

### 4.2 Video Generation Flow (Single Scene)

```
API                   Redis/BullMQ          Worker              AI Provider        Storage
 │                       │                    │                     │                 │
 │  POST /generate-video │                    │                     │                 │
 │  ─────────────────────▶                    │                     │                 │
 │  Add job to queue     │                    │                     │                 │
 │  ──────────────────── ▶│                    │                     │                 │
 │  200 { queued }       │                    │                     │                 │
 │◀──────────────────────│                    │                     │                 │
 │                       │  Dequeue job       │                     │                 │
 │                       │───────────────────▶│                     │                 │
 │                       │                    │  Download frames    │                 │
 │                       │                    │────────────────────────────────────── ▶│
 │                       │                    │◀─────────────────────────────────────  │
 │                       │                    │                     │                 │
 │                       │                    │  Enrich motion      │                 │
 │                       │                    │  (Gemini ADK)       │                 │
 │                       │                    │────────────────────▶│                 │
 │                       │                    │◀────────────────────│                 │
 │                       │                    │                     │                 │
 │                       │                    │  Generate video     │                 │
 │                       │                    │  (Veo/Kling/etc)    │                 │
 │                       │                    │────────────────────▶│                 │
 │                       │                    │  Poll/await...      │                 │
 │                       │                    │◀────────────────────│                 │
 │                       │                    │                     │                 │
 │                       │                    │  Validate duration  │                 │
 │                       │                    │  (ffprobe)          │                 │
 │                       │                    │                     │                 │
 │                       │                    │  Upload clip        │                 │
 │                       │                    │─────────────────────────────────────▶ │
 │                       │                    │                     │                 │
 │                       │                    │  Track cost         │                 │
 │                       │                    │  Update DB status   │                 │
 │                       │                    │                     │                 │
 │  React Query poll     │                    │                     │                 │
 │  (every 8s)           │                    │                     │                 │
 │  GET /projects/:id    │                    │                     │                 │
 │  → status: complete   │                    │                     │                 │
```

---

## 5. Technology Stack Summary

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| Next.js | 14.2.3 | SSR/SSG React framework |
| React | 18.3.0 | UI library |
| TypeScript | 5.4.5 | Type safety |
| Tailwind CSS | 3.4.3 | Utility-first styling |
| Zustand | 4.5.7 | Client state management |
| React Query | 5.90.21 | Server state + polling |
| ReactFlow | 12.10.1 | Node-based graph editor |
| Radix UI | Various | Accessible primitives |
| Phosphor Icons | 2.1.10 | Icon library |
| better-auth | 1.5.5 | Auth client |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| Express | 4.19.2 | HTTP server |
| TypeScript | 5.4.5 | Type safety |
| Prisma | 5.16.0 | ORM + migrations |
| BullMQ | 5.8.3 | Job queue |
| Redis | 7 (Docker) | Queue backend + cache |
| PostgreSQL | 16 (Docker) | Primary database |
| better-auth | 1.5.5 | Authentication |
| Zod | 3.23.8 | Validation |
| Helmet | 7.1.0 | Security headers |
| FFmpeg | System | Video/audio processing |

### AI Services
| Service | SDK | Purpose |
|---|---|---|
| Google Gemini | @google/genai 1.45.0 | LLM + Image + Video |
| Google ADK | @google/adk 0.4.0 | Agent framework |
| ElevenLabs | REST API | TTS + SFX |
| fal.ai | @fal-ai/client 1.9.4 | Kling + SeDance video |
| Replicate | replicate 1.0.0 | Multi-model video |

### Infrastructure
| Technology | Purpose |
|---|---|
| Docker (multi-stage) | Containerization |
| Docker Compose | Service orchestration |
| Nginx | Reverse proxy + SSL |
| Let's Encrypt | SSL certificates |
| Turborepo | Monorepo build orchestration |
| AWS S3 / DO Spaces | Media object storage |

---

## 6. Monorepo Package Dependency Graph

```
                    @atlas/shared
                    (types, pricing)
                         │
          ┌──────────────┼──────────────────────────┐
          │              │                          │
     @atlas/db     @atlas/prompts          @atlas/style-system
     (Prisma)      (LLM templates)         (design tokens)
          │              │                          │
          ├──────────────┤                          │
          │              │                          │
  @atlas/cost-      @atlas/integrations             │
  estimation        (AI providers)                  │
          │              │                          │
          │         @atlas/validation                │
          │         (Zod schemas)                    │
          │              │                          │
          │     @atlas/motion-fallback              │
          │     (Ken Burns FFmpeg)                   │
          │              │                          │
          ├──────────────┼──────────────────────────┤
          │              │                          │
     @atlas/api    @atlas/workers            @atlas/web
     (Express)     (BullMQ)                  (Next.js)
```

---

## 7. API Endpoints Summary

| Category | Endpoints | Key Operations |
|----------|-----------|---------------|
| **Auth** | `/api/auth/*` | Signup, login, session |
| **Projects** | 5 endpoints | CRUD + settings |
| **Discovery** | 3 endpoints | Topic generation + selection |
| **Topics** | 3 endpoints | List, approve, reject |
| **Research** | 3 endpoints | Start, get, delete |
| **Scripts** | 5 endpoints | Generate, list, approve, delete, rewrite |
| **Voice** | 4 endpoints | Presets, generate, get, delete |
| **Scenes** | 6 endpoints | Plan, list, regenerate, generate videos |
| **Frames** | 4 endpoints | Generate all, generate single, list, regenerate |
| **Characters** | 6 endpoints | CRUD + generate + select |
| **Captions** | 4 endpoints | Get, update, regenerate, translate |
| **Renders** | 4 endpoints | Start, list, get, download |
| **Exports** | 3 endpoints | Create, list, download |
| **Costs** | 2 endpoints | Breakdown, estimate |
| **Preview** | 1 endpoint | Generate preview |
| **Total** | **~53 endpoints** | Full video production API |

---

## 8. Key Metrics

| Metric | Value |
|--------|-------|
| **Packages** | 11 (shared, db, integrations, prompts, cost-estimation, motion-fallback, style-system, validation, analytics, media, ui) |
| **Applications** | 3 (api, workers, web) |
| **API Routes** | 14 modules, ~53 endpoints |
| **Workers** | 12 BullMQ workers |
| **AI Providers** | 4 platforms (Google, ElevenLabs, fal.ai, Replicate) |
| **Video Models** | 7 provider variants |
| **Research APIs** | 9 data sources |
| **Database Tables** | 16+ (auth, content, cost tracking) |
| **Project States** | 23 lifecycle states |
| **Cost Stages** | 17 tracked cost categories |
| **LLM Agents** | 8 ADK agent types |
| **Docker Services** | 7 production containers |
