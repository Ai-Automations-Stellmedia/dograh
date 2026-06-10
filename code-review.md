I have collected and synthesized data from the entire codebase. Here is your   
  complete Code Review Graph — a navigable map of what every part does, how data 
  flows, and where to make changes.

  ---
  🧠 Dograh Code Review Graph

  Generated: 2025-06-06 | Version: 1.34.0

  ---
  1. Architecture Overview

  ┌──────────────────────────────────────────────────────────────────┐
  │                        DOGRAH PLATFORM                            │
  │  (Self-hostable Voice AI — alternative to Vapi & Retell)         │
  ├──────────────────────────────────────────────────────────────────┤
  │  Front Door: nginx (80/443) or Next.js dev server (3000/3010)     │
  ├──────────────────────────────────────────────────────────────────┤
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
  │  │   Next.js    │  │   FastAPI    │  │  Redis / ARQ Queue   │    │
  │  │   UI (3010)  │◄─┤  API (8000)  │◄─┤  Background Workers  │    │
  │  │              │  │              │  │                      │    │
  │  │  Zustand     │  │  SQLAlchemy  │  │  process_workflow_   │    │
  │  │  ReactFlow   │  │  Pipecat     │  │  completion          │    │
  │  │  shadcn/ui   │  │  26 Routes   │  │  campaign_batch      │    │
  │  └──────────────┘  └──────────────┘  └──────────────────────┘    │
  │         │                  │                  │                    │
  │         ▼                  ▼                  ▼                    │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
  │  │   MinIO      │  │  PostgreSQL  │  │   Pipecat (submod)   │    │
  │  │  (Audio/S3)  │  │  + pgvector  │  │   Voice Pipeline     │    │
  │  └──────────────┘  └──────────────┘  └──────────────────────┘    │
  │                                                                   │
  │  Auxiliaries: coturn (TURN/STUN), cloudflared (tunnel),           │
  │               dograh-init (runtime config generator)              │
  └──────────────────────────────────────────────────────────────────┘

  ---
  2. Top-Level Orchestration (Repo Root)

  These files control how the system is built, deployed, and wired together.     

  File/Dir: docker-compose.yaml
  Purpose: Production stack: postgres, redis, minio, api (multi-worker), ui,     
    nginx, coturn, cloudflared, dograh-init
  When to Change: Add/remove services or change networking
  ────────────────────────────────────────
  File/Dir: docker-compose-local.yaml
  Purpose: Dev-only: postgres + redis + minio (no nginx/coturn)
  When to Change: Local infra tweaks
  ────────────────────────────────────────
  File/Dir: .env
  Purpose: Deployment secrets (JWT, DB password, registry, telemetry toggle)     
  When to Change: Rotate secrets or change image registry
  ────────────────────────────────────────
  File/Dir: remote_up.sh
  Purpose: Entrypoint for remote OSS deploy. Runs preflight, validates env,      
  starts
    stack with --profile remote
  When to Change: Custom deployment logic
  ────────────────────────────────────────
  File/Dir: scripts/
  Purpose: Bash + PowerShell pairs for cross-platform dev
  When to Change: Add new CLI tooling
  ────────────────────────────────────────
  File/Dir: scripts/lib/setup_common.sh
  Purpose: Shared dograh_* functions sourced by deploy scripts
  When to Change: Shared setup logic
  ────────────────────────────────────────
  File/Dir: scripts/run_dograh_init.sh
  Purpose: Renders nginx & coturn config templates at container start
  When to Change: Reverse proxy or TURN config changes
  ────────────────────────────────────────
  File/Dir: deploy/templates/
  Purpose: Nginx and coturn config templates that get runtime-filled
  When to Change: Load-balancing or TLS rules
  ────────────────────────────────────────
  File/Dir: pipecat/
  Purpose: Git submodule — voice pipeline framework (fork/maintenance)
  When to Change: Upgrading audio pipeline behavior
  ────────────────────────────────────────
  File/Dir: sdk/python/
  Purpose: Published to PyPI as dograh-sdk
  When to Change: External SDK changes
  ────────────────────────────────────────
  File/Dir: sdk/typescript/
  Purpose: Published to npm as @dograh/sdk
  When to Change: External SDK changes

  ---
  3. Backend (api/) — The Engine

  3.1 Entrypoint & Wiring

  api/app.py — FastAPI factory
  - Lifespan manager: starts ARQ pool, pre-loads Langfuse credentials, sets up   
  WorkerSyncManager (Redis pub/sub for cross-worker config sync)
  - Mounts all routers under /api/v1
  - Mounts MCP server at /api/v1/mcp
  - CORS: distinguishes OSS vs SaaS origins
  - Health check: GET /api/v1/health

  api/constants.py — All environment variables live here (DATABASE_URL,
  REDIS_URL, MINIO_ENDPOINT, TURN_HOST, OSS_JWT_SECRET, etc.)

  api/enums.py — Every enum type used across routes and DB models

  3.2 Routes (api/routes/) — API Surface

  All routers are included by api/routes/main.py. Handlers are thin: auth/org    
  injection → Pydantic validation → service call → masking → JSON response.      

  Route File: auth.py
  What It Does: Local auth (JWT login/signup), not used in SaaS Stack mode       
  Key Endpoints: POST /auth/signup, /login, GET /auth/me
  ────────────────────────────────────────
  Route File: workflow.py
  What It Does: CRUD + publish + run workflows, versions, drafts
  Key Endpoints: POST /workflow/create/, /{id}/publish, /{id}/runs,
  /{id}/validate
  ────────────────────────────────────────
  Route File: campaign.py
  What It Does: Campaign creation, CSV upload, start/pause/resume, progress &    
    reports
  Key Endpoints: POST /campaign/create, /{id}/start, /{id}/pause, /{id}/progress 
  ────────────────────────────────────────
  Route File: telephony.py
  What It Does: Initiate test calls, provider webhooks (Twilio, Plivo, etc.),    
    WebSocket signaling
  Key Endpoints: POST /telephony/initiate-call, /callback/*
  ────────────────────────────────────────
  Route File: user.py
  What It Does: Profile + user-level LLM/TTS/STT config
  Key Endpoints: GET /user, /user/configuration
  ────────────────────────────────────────
  Route File: organization.py
  What It Does: Org profile + usage quotas + org-level config
  Key Endpoints: GET /organization, /organization/configuration
  ────────────────────────────────────────
  Route File: credentials.py
  What It Does: Third-party API key storage (encrypted/masked)
  Key Endpoints: GET /credentials, POST /credentials
  ────────────────────────────────────────
  Route File: workflow_text_chat.py
  What It Does: Text-based workflow chat sessions via WebSocket
  Key Endpoints: WS /workflow/{id}/text-chat
  ────────────────────────────────────────
  Route File: workflow_recording.py
  What It Does: Call recording metadata & tagging
  Key Endpoints: GET /workflow/recording/{id}
  ────────────────────────────────────────
  Route File: tool.py
  What It Does: Reusable HTTP API tools library
  Key Endpoints: GET /tool, POST /tool, PUT /tool/{id}
  ────────────────────────────────────────
  Route File: knowledge_base.py
  What It Does: Document upload → embedding pipeline trigger
  Key Endpoints: POST /knowledge-base/{id}/upload, /embed
  ────────────────────────────────────────
  Route File: public_agent.py
  What It Does: Public agent info (no auth) for embeds
  Key Endpoints: GET /public/agent/{uuid}
  ────────────────────────────────────────
  Route File: public_embed.py
  What It Does: Embedded widget WebSocket entry
  Key Endpoints: WS /public/embed/{token}
  ────────────────────────────────────────
  Route File: workflow_embed.py
  What It Does: Generate embed tokens for public use
  Key Endpoints: GET /workflow/{id}/embed-token
  ────────────────────────────────────────
  Route File: folder.py
  What It Does: Organize workflows into folders
  Key Endpoints: GET /folder, POST /folder
  ────────────────────────────────────────
  Route File: reports.py
  What It Does: Aggregate analytics across runs
  Key Endpoints: GET /reports/analytics
  ────────────────────────────────────────
  Route File: node_types.py
  What It Does: Returns available node definitions for the workflow builder      
  Key Endpoints: GET /node-types
  ────────────────────────────────────────
  Route File: agent_stream.py
  What It Does: Server-sent events for live agent streaming
  Key Endpoints: GET /agent/stream
  ────────────────────────────────────────
  Route File: webrtc_signaling.py
  What It Does: WebRTC peer negotiation (SDP exchange, ICE)
  Key Endpoints: WS /webrtc/signal

  3.3 Services (api/services/) — Business Logic

  This is where the heavy lifting happens.

  api/services/workflow/ — Graph Execution

  ┌───────────────────────┬───────────────────────────────────────────────────┐  
  │        Module         │                      Purpose                      │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ workflow_graph.py     │ Parses ReactFlow DTO into an execution graph;     │  
  │                       │ validates edges and node connections              │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ dto.py                │ ReactFlowDTO — validates the JSON shape the       │  
  │                       │ frontend sends                                    │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ pipecat_engine.py     │ Executes the workflow graph by building a Pipecat │  
  │                       │  pipeline                                         │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ node_data.py          │ Discriminated union of all node types (Choice,    │  
  │                       │ Agent, Transfer, End, etc.)                       │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ trigger_paths.py      │ UUID generation for inbound trigger webhooks;     │  
  │                       │ conflict detection                                │  
  ├───────────────────────┼───────────────────────────────────────────────────┤  
  │ disposition_mapper.py │ Maps call outcomes to disposition codes           │  
  └───────────────────────┴───────────────────────────────────────────────────┘  

  api/services/pipecat/ — Live Call Runtime

  ┌────────────────────────────────┬──────────────────────────────────────────┐  
  │             Module             │                 Purpose                  │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ run_pipeline.py                │ Main orchestrator of a call lifecycle    │  
  │                                │ (50+ imports, the heart of the system)   │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ pipeline_builder.py            │ Connects STT → LLM → TTS + transport     │  
  │                                │ layer                                    │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ service_factory.py             │ Instantiates LLM/TTS/STT based on        │  
  │                                │ org/user/workflow config hierarchy       │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ event_handlers.py              │ Processes Pipecat async events           │  
  │                                │ (transcription, bot speaking, etc.)      │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ transport_setup.py             │ WebRTC / SIP / Twilio / Plivo transport  │  
  │                                │ setup                                    │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ worker_runner.py               │ Long-lived async worker that runs the    │  
  │                                │ pipeline loop                            │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ pre_call_fetch.py              │ HTTP prefetch of customer data before    │  
  │                                │ call starts                              │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ turn_context.py                │ Generates TURN server credentials for    │  
  │                                │ NAT traversal                            │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ realtime_feedback_observer.py  │ Streams call events to the frontend via  │  
  │                                │ WebSocket                                │  
  ├────────────────────────────────┼──────────────────────────────────────────┤  
  │ pipeline_metrics_aggregator.py │ Tracks token usage, duration, cost per   │  
  │                                │ run                                      │  
  └────────────────────────────────┴──────────────────────────────────────────┘  

  api/services/campaign/ — Batch Dialing

  ┌───────────────────────────────────┬──────────────────────────────────────┐   
  │              Module               │               Purpose                │   
  ├───────────────────────────────────┼──────────────────────────────────────┤   
  │ runner.py                         │ State machine: created → queued →    │   
  │                                   │ running → completed                  │   
  ├───────────────────────────────────┼──────────────────────────────────────┤   
  │ orchestrator.py                   │ Concurrency control, rate limiting,  │   
  │                                   │ schedule windows                     │   
  ├───────────────────────────────────┼──────────────────────────────────────┤   
  │ campaign_call_dispatcher.py       │ Takes queued runs and fires them via │   
  │                                   │  telephony providers                 │   
  ├───────────────────────────────────┼──────────────────────────────────────┤   
  │ circuit_breaker.py                │ Auto-pauses campaigns if failure     │   
  │                                   │ rate exceeds threshold               │   
  ├───────────────────────────────────┼──────────────────────────────────────┤   
  │ source_sync.py /                  │ CSV ingestion into queued_runs       │   
  │ source_sync_factory.py            │                                      │   
  └───────────────────────────────────┴──────────────────────────────────────┘   

  api/services/telephony/ — Phone Provider Abstraction

  ┌──────────────────────┬────────────────────────────────────┐
  │        Module        │              Purpose               │
  ├──────────────────────┼────────────────────────────────────┤
  │ factory.py           │ Provider factory                   │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/twilio/    │ Twilio outbound + callback parsing │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/plivo/     │ Plivo integration                  │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/vonage/    │ Vonage integration                 │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/telnyx/    │ Telnyx integration                 │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/vobiz/     │ Vobiz integration                  │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/cloudonix/ │ Cloudonix integration              │
  ├──────────────────────┼────────────────────────────────────┤
  │ providers/ari/       │ Asterisk ARI                       │
  └──────────────────────┴────────────────────────────────────┘

  api/services/configuration/ — Config Hierarchy

  ┌───────────────────┬───────────────────────────────────────────────────────┐  
  │      Module       │                        Purpose                        │  
  ├───────────────────┼───────────────────────────────────────────────────────┤  
  │ registry.py       │ Service provider registry (OpenAI, Anthropic, Google, │  
  │                   │  ElevenLabs, etc.)                                    │  
  ├───────────────────┼───────────────────────────────────────────────────────┤  
  │ resolve.py        │ Merges org defaults → user defaults → workflow        │  
  │                   │ overrides                                             │  
  ├───────────────────┼───────────────────────────────────────────────────────┤  
  │ check_validity.py │ Live validation of API keys                           │  
  ├───────────────────┼───────────────────────────────────────────────────────┤  
  │ masking.py        │ Strips secrets from API responses                     │  
  └───────────────────┴───────────────────────────────────────────────────────┘  

  api/services/integrations/ — Post-Call Hooks

  - Zapier: Webhooks on ALL_CALLS or QUALIFIED_CALLS
  - Custom webhooks: User-defined POST callbacks after run completion

  api/services/auth/

  - depends.py — FastAPI dependency get_user() (JWT or API key validation)       
  - stack_auth.py — Stack Auth SaaS provider integration

  api/services/worker_sync/

  - Redis pub/sub to broadcast config changes across uvicorn workers (needed     
  because each worker is a separate process)

  3.4 Database (api/db/)

  api/db/models.py — 20+ SQLAlchemy tables. Key relationships:

  users ──► organizations (selected_organization_id)
          │
          ├─► workflows ──► workflow_definitions (draft / published versions)    
          │                    │
          │                    ▼
          │               workflow_runs (calls/text sessions)
          │                    │
          │                    ▼
          │               workflow_recordings, queued_runs (for campaigns)       
          │
          ├─► campaigns ──► queued_runs
          │
          ├─► telephony_configurations ──► telephony_phone_numbers
          │
          ├─► tools, knowledge_bases, integrations, api_keys, folders

  api/db/db_client.py — Composite class inheriting from ~23 specialized client   
  modules. Every client enforces organization_id scoping (tenant isolation).     

  api/alembic/versions/ — 50+ migration files tracking schema evolution.

  3.5 Schemas (api/schemas/)

  Pydantic models that define the contract between frontend ↔ backend. Key files:  - workflow.py — WorkflowResponse, WorkflowRunResponseSchema,
  CreateWorkflowRequest
  - telephony_config.py — Provider configuration shapes
  - user_configuration.py — LLM/TTS/STT settings
  - tool.py — HTTP tool schemas

  3.6 Background Tasks (api/tasks/)

  ARQ (Redis-backed) workers. Registered in arq.py.

  ┌────────────────────────────────────┬───────────┬─────────────────────────┐   
  │                Task                │ Triggered │      What It Does       │   
  │                                    │     By    │                         │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │                                    │ Pipeline  │ Upload                  │   
  │ process_workflow_completion        │ end       │ transcript/recording to │   
  │                                    │           │  MinIO/S3, update DB    │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │ upload_voicemail_audio_to_s3       │ Voicemail │ Store audio artifact    │   
  │                                    │  detected │                         │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │ run_integrations_post_workflow_run │ Pipeline  │ Fire Zapier / webhook   │   
  │                                    │ end       │ callbacks               │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │ sync_campaign_source               │ Campaign  │ Parse CSV into          │   
  │                                    │ created   │ queued_runs             │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │ process_campaign_batch             │ Campaign  │ Batch-dial queued rows  │   
  │                                    │ runner    │ via telephony           │   
  ├────────────────────────────────────┼───────────┼─────────────────────────┤   
  │ process_knowledge_base_document    │ Upload    │ Chunk + embed + store   │   
  │                                    │           │ vectors in pgvector     │   
  └────────────────────────────────────┴───────────┴─────────────────────────┘   

  3.7 MCP Server (api/mcp_server/)

  Exposes Dograh API as Model Context Protocol tools so Claude (or other MCP     
  clients) can control workflows. Mounted at /api/v1/mcp as Streamable HTTP.     

  3.8 Utilities (api/utils/)

  ┌──────────────────────┬───────────────────────────────────────────────┐       
  │         File         │                    Purpose                    │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ auth.py              │ JWT encode/decode, password hashing           │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ common.py            │ Backend URL resolution                        │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ telephony_helper.py  │ E.164 normalization, webhook parsing          │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ template_renderer.py │ {{variable}} substitution for prompts         │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ artifacts.py         │ Generate presigned/public URLs for recordings │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ audio_converter.py   │ Format conversion (wav/mp3/etc.)              │       
  ├──────────────────────┼───────────────────────────────────────────────┤       
  │ transcript.py        │ Transcript formatting                         │       
  └──────────────────────┴───────────────────────────────────────────────┘       

  ---
  4. Frontend (ui/) — The Interface

  4.1 Build & Tooling

  ┌──────────────────────┬────────────────────────────────────────────────────┐  
  │         File         │                      Purpose                       │  
  ├──────────────────────┼────────────────────────────────────────────────────┤  
  │ package.json         │ Next.js 15, React 19, Tailwind 4, Radix UI,        │  
  │                      │ Zustand, @xyflow/react                             │  
  ├──────────────────────┼────────────────────────────────────────────────────┤  
  │ next.config.ts       │ Standalone output, PostHog/Sentry rewrite rules,   │  
  │                      │ source maps                                        │  
  ├──────────────────────┼────────────────────────────────────────────────────┤  
  │                      │ Generates src/client/ from                         │  
  │ openapi-ts.config.ts │ http://127.0.0.1:8000/api/v1/openapi.json using    │  
  │                      │ @hey-api/client-fetch                              │  
  ├──────────────────────┼────────────────────────────────────────────────────┤  
  │ components.json      │ shadcn/ui config (New York style, neutral base     │  
  │                      │ color, CSS variables)                              │  
  └──────────────────────┴────────────────────────────────────────────────────┘  

  4.2 API Client (src/client/)

  Auto-generated — never hand-edit. Run npm run generate-client after adding     
  backend routes.

  ┌───────────────────────┬──────────────────────────────────────────────────┐   
  │         File          │                     Purpose                      │   
  ├───────────────────────┼──────────────────────────────────────────────────┤   
  │ sdk.gen.ts            │ Typed API function wrappers for every backend    │   
  │                       │ endpoint                                         │   
  ├───────────────────────┼──────────────────────────────────────────────────┤   
  │ types.gen.ts          │ All Pydantic schemas as TypeScript types         │   
  ├───────────────────────┼──────────────────────────────────────────────────┤   
  │ core/auth.gen.ts      │ Auth interceptors and base URL logic             │   
  ├───────────────────────┼──────────────────────────────────────────────────┤   
  │ client.gen.ts /       │ Fetch client configuration and typed request     │   
  │ client/               │ builders                                         │   
  └───────────────────────┴──────────────────────────────────────────────────┘   

  src/lib/apiClient.ts — Runtime config:
  - Server-side calls use BACKEND_URL env var (points to http://api:8000)        
  - Client-side defaults to window.location.origin
  - setupAuthInterceptor() attaches Bearer token lazily after auth loads

  src/lib/apiError.ts — Normalizes FastAPI errors (error.detail can be string or 
  array; rendering raw objects crashes React)

  4.3 Auth Flow (src/lib/auth/)

  ┌────────────────────────────────────┬──────────────────────────────────────┐  
  │                File                │               Purpose                │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │                                    │ Root auth context. Fetches           │  
  │ providers/AuthProvider.tsx         │ /api/config/auth to decide: Stack or │  
  │                                    │  Local?                              │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ providers/StackProviderWrapper.tsx │ SaaS: Stack Auth OAuth wrapper       │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ providers/LocalProviderWrapper.tsx │ OSS: email/password JWT wrapper      │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ config.ts / server.ts / types.ts   │ Auth config helpers and types        │  
  └────────────────────────────────────┴──────────────────────────────────────┘  

  Flow:
  1. Page mounts → AuthProvider fetches /api/config/auth
  2. If provider=stack → lazy-load StackProviderWrapper (OAuth)
  3. If provider=local  → lazy-load LocalProviderWrapper (JWT)
  4. useAuth() exposes: user, isAuthenticated, loading, getAccessToken(),        
  logout()
  5. API interceptor waits for getAccessToken() before attaching Authorization   
  header

  4.4 App Router (src/app/)

  Every directory is a Next.js App Router route.

  ┌──────────────────────────────────────────────┬───────────────────────────┐   
  │                  Route Path                  │          Purpose          │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ page.tsx                                     │ Dashboard / workflow list │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/page.tsx                            │ Workflow library          │   
  │                                              │ (cards/table view)        │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/create/page.tsx                     │ New workflow from scratch │   
  │                                              │  or template              │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/[workflowId]/page.tsx               │ Workflow editor           │   
  │                                              │ (ReactFlow canvas)        │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/[workflowId]/runs/page.tsx          │ Run history for a         │   
  │                                              │ workflow                  │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/[workflowId]/run/[runId]/page.tsx   │ Live call runner (WebRTC  │   
  │                                              │ + WebSocket)              │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ workflow/[workflowId]/settings/page.tsx      │ Workflow                  │   
  │                                              │ metadata/settings         │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ campaigns/page.tsx                           │ Campaign list             │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ campaigns/new/page.tsx                       │ New campaign wizard       │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ campaigns/[campaignId]/page.tsx              │ Campaign detail &         │   
  │                                              │ progress                  │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ telephony-configurations/page.tsx            │ Provider & phone number   │   
  │                                              │ management                │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ telephony-configurations/[configId]/page.tsx │ Edit a telephony config   │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ tools/page.tsx                               │ Reusable HTTP tool        │   
  │                                              │ library                   │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ tools/[toolUuid]/page.tsx                    │ Tool editor               │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ files/page.tsx                               │ Knowledge base document   │   
  │                                              │ management                │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ reports/page.tsx                             │ Analytics charts & CSV    │   
  │                                              │ downloads                 │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ recordings/page.tsx                          │ Browse call recordings    │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ model-configurations/page.tsx                │ Global LLM/TTS/STT        │   
  │                                              │ settings                  │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ api-keys/page.tsx                            │ Manage API keys           │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ auth/login/page.tsx, auth/signup/page.tsx    │ Local auth pages          │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ after-sign-in/page.tsx                       │ Post-auth redirect        │   
  │                                              │ handler                   │   
  ├──────────────────────────────────────────────┼───────────────────────────┤   
  │ superadmin/page.tsx,                         │ Admin views               │   
  │ superadmin/runs/page.tsx                     │                           │   
  └──────────────────────────────────────────────┴───────────────────────────┘   

  Proxy Route

  app/api/v1/[...path]/route.ts — A Next.js route handler that proxies all API   
  calls to the backend. This avoids CORS and allows server-side rendering to call  the internal Docker service.

  4.5 State Management

  Zustand stores (primary global state):
  - workflowStore.ts (src/app/workflow/[workflowId]/stores/) — The single source 
  of truth for the workflow editor
    - Holds nodes, edges, workflow name, history for undo/redo (max 50 states),  
  validation errors, template context variables, workflow configurations
    - Smart history: pushes state only on add/remove/drag-end, not on selection  
  or active dragging
    - Selectors exported: useWorkflowNodes, useWorkflowEdges, useUndoRedo, etc.  

  React Contexts (src/context/):

  ┌────────────────────────────────────┬──────────────────────────────────────┐  
  │              Context               │               Purpose                │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ AppConfigContext.tsx               │ Remote config (version, features,    │  
  │                                    │ telemetry)                           │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ UserConfigContext.tsx              │ Current user's LLM/TTS/STT settings  │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ OnboardingContext.tsx              │ Onboarding tooltip state             │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ TelephonyConfigWarningsContext.tsx │ Validation warnings for phone        │  
  │                                    │ configs                              │  
  ├────────────────────────────────────┼──────────────────────────────────────┤  
  │ UnsavedChangesContext.tsx          │ Browser unload guard for dirty forms │  
  └────────────────────────────────────┴──────────────────────────────────────┘  

  4.6 Workflow Builder (src/components/flow/)

  The centerpiece of the UI. Built on @xyflow/react (React Flow v12).

  ┌─────────────────────────────────┬─────────────────────────────────────────┐  
  │            Component            │                 Purpose                 │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ AddNodePanel.tsx                │ Sidebar palette of draggable node types │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ nodes/GenericNode.tsx           │ Base node renderer used by all node     │  
  │                                 │ types                                   │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ nodes/BaseNode.tsx              │ Wrapper with ports/handles              │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ nodes/NodeHeader.tsx            │ Node title & icon                       │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ nodes/common/NodeEditDialog.tsx │ Modal form to edit a node's properties  │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ nodes/common/NodeContent.tsx    │ Inline property preview on the canvas   │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ edges/CustomEdge.tsx            │ Connection lines with labels            │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ renderer/NodeEditForm.tsx       │ Dynamic form generator based on node    │  
  │                                 │ spec                                    │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ renderer/PropertyInput.tsx      │ Maps schema types to input components   │  
  │                                 │ (text, select, json)                    │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ renderer/useNodeSpecs.ts        │ Fetches node type definitions from      │  
  │                                 │ backend                                 │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ displayOptions.ts               │ Visual layout options                   │  
  │                                 │ (horizontal/vertical)                   │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ DocumentSelector.tsx            │ Pick KB documents to attach to a node   │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ ToolSelector.tsx                │ Pick reusable tools to attach to a node │  
  ├─────────────────────────────────┼─────────────────────────────────────────┤  
  │ MentionTextarea.tsx             │ {{variable}} autocomplete for prompts   │  
  └─────────────────────────────────┴─────────────────────────────────────────┘  

  Flow in the Editor:
  1. workflowStore initializes with workflow definition from backend
  2. ReactFlow renders nodes + edges from the store
  3. User drags a node from AddNodePanel → addNode() action → history push       
  4. User connects nodes → addEdge() → graph validation runs
  5. User clicks node → NodeEditDialog opens with NodeEditForm generated from    
  useNodeSpecs
  6. User clicks "Publish" → frontend calls POST /workflow/{id}/publish with full  graph JSON

  4.7 Live Call / WebRTC (src/app/workflow/[workflowId]/run/[runId]/)

  ┌────────────────────────────────────────┬──────────────────────────────────┐  
  │               Component                │             Purpose              │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ page.tsx                               │ Container for a live call        │  
  │                                        │ session                          │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ hooks/useWebSocketRTC.tsx              │ Manages WebSocket signaling for  │  
  │                                        │ WebRTC                           │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ hooks/useDeviceInputs.tsx              │ Microphone permission & device   │  
  │                                        │ selection                        │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ components/AudioControls.tsx           │ Mute/unmute, speaker toggle      │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ components/ConnectionStatus.tsx        │ ICE / SDP status indicator       │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ components/ContextDisplay.tsx          │ Shows pre_call_fetch context     │  
  ├────────────────────────────────────────┼──────────────────────────────────┤  
  │ components/ContextVariablesSection.tsx │ Editable call variables          │  
  └────────────────────────────────────────┴──────────────────────────────────┘  

  4.8 Testing a Workflow (.../components/WorkflowTesterPanel/)

  ┌────────────────────────────┬───────────────────────────────────────────┐     
  │         Component          │                  Purpose                  │     
  ├────────────────────────────┼───────────────────────────────────────────┤     
  │ EmbeddedVoiceTester.tsx    │ WebRTC voice test inside the editor       │     
  ├────────────────────────────┼───────────────────────────────────────────┤     
  │ ManualTextChatPanel.tsx    │ Text chat test inside the editor          │     
  ├────────────────────────────┼───────────────────────────────────────────┤     
  │ ChatComposer.tsx           │ Message input for text tests              │     
  ├────────────────────────────┼───────────────────────────────────────────┤     
  │ useTextChatSession.ts      │ WebSocket hook for text-chat sessions     │     
  ├────────────────────────────┼───────────────────────────────────────────┤     
  │ AiSimulatorPlaceholder.tsx │ Simulated AI responses for quick UX tests │     
  └────────────────────────────┴───────────────────────────────────────────┘     

  4.9 UI Primitives (src/components/ui/)

  Standard shadcn/ui components — Radix UI wrappers with Tailwind. Key ones:     
  button, dialog, input, textarea, select, tabs, table, badge, tooltip, sonner   
  (toasts), json-editor, choice-chips.

  4.10 Other Reusable Components

  ┌─────────────────────────────────────────────────┬────────────────────────┐   
  │                    Component                    │        Purpose         │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ layout/AppSidebar.tsx                           │ Main navigation rail   │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ layout/AppLayout.tsx                            │ Page shell with        │   
  │                                                 │ sidebar + content area │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ filters/FilterBuilder.tsx                       │ Generic query builder  │   
  │                                                 │ used in runs/tables    │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ filters/DateRangeFilter.tsx, NumberFilter.tsx,  │ Filter primitives      │   
  │ etc.                                            │                        │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ workflow/WorkflowCard.tsx, WorkflowTable.tsx    │ Dashboard list views   │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ workflow-runs/WorkflowRunsTable.tsx             │ Paginated run history  │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ workflow-runs/CampaignRuns.tsx                  │ Campaign-specific run  │   
  │                                                 │ view                   │   
  ├─────────────────────────────────────────────────┼────────────────────────┤   
  │ workflow/conversation/ConversationTimeline.tsx  │ Visual transcript      │   
  │                                                 │ browser                │   
  └─────────────────────────────────────────────────┴────────────────────────┘   

  ---
  5. Cross-Layer Data Flow Maps

  Map A: Creating & Publishing a Workflow

  User clicks "New Workflow"
    │
    ▼
  ui/src/app/workflow/create/page.tsx
    │
    ▼
  POST /api/v1/workflow/create/definition
    │
    ▼
  api/routes/workflow.py → get_user() (JWT/API key)
    │
    ▼
  api/services/workflow/dto.py: ReactFlowDTO.model_validate()
    │
    ▼
  api/services/workflow/workflow_graph.py: build graph, validate edges
    │
    ▼
  api/db/workflow_client.py: INSERT workflows + workflow_definitions
  (status=draft)
    │
    ▼
  returns WorkflowResponse → frontend redirects to /workflow/{id}

  Map B: Workflow Editor → Publish

  User clicks "Publish"
    │
    ▼
  workflowStore holds nodes[] + edges[]
    │
    ▼
  PUT /api/v1/workflow/{id}  (sends full ReactFlow JSON)
    │
    ▼
  api/routes/workflow.py
    │
    ├─► _validate_workflow_definition()
    │     ├─ dto validation
    │     ├─ graph validation (no dangling nodes)
    │     └─ trigger path conflict check
    │
    ▼
  db_client.publish_workflow_draft()
    │     UPDATE workflow_definitions SET status='published'
    │
    ▼
  sync_triggers_for_workflow() → updates agent_triggers table
    │
    ▼
  frontend shows "Published" badge

  Map C: Live WebRTC Call

  User clicks "Test Call" or inbound call arrives
    │
    ▼
  POST /api/v1/workflow/{id}/runs (mode=webrtc)
    │     creates workflow_runs row (status=INITIALIZED)
    │
    ▼
  Frontend loads /workflow/{id}/run/{runId}
    │
    ▼
  useWebSocketRTC.tsx opens WS to /api/v1/webrtc/signal
    │
    ▼
  api/services/pipecat/transport_setup.py: setup WebRTC peer
    │
    ▼
  api/services/pipecat/pipeline_builder.py:
    STT_service → LLM_service → TTS_service
         ↑_________↓ (MCP tools triggered here)
    │
    ▼
  api/services/pipecat/run_pipeline.py: async loop
    │   - processes STT frames → text
    │   - queries LLM with system prompt + history
    │   - executes tools via mcp_tool_session
    │   - streams TTS audio back via WebRTC
    │   - updates metrics (tokens, duration, cost)
    │
    ▼
  Call ends → background ARQ tasks:
    │   - process_workflow_completion (upload recording)
    │   - run_integrations_post_workflow_run (Zapier/webhooks)
    │
    ▼
  DB updated with cost_info, usage_info, transcript_url

  Map D: Campaign Execution

  User uploads CSV → POST /campaign/create
    │
    ▼
  api/services/campaign/source_sync.py: validate columns
    │
    ▼
  Campaign created in DB (state='created')
    │
    ▼
  User clicks "Start"
    │
    ▼
  POST /campaign/{id}/start
    │   checks quota (check_dograh_quota)
    │   UPDATE campaigns SET state='queued'
    │   enqueue sync_campaign_source task (ARQ)
    │
    ▼
  ARQ Worker: sync_campaign_source
    │   parses CSV → INSERT queued_runs (state='queued')
    │
    ▼
  ARQ Worker: process_campaign_batch (re-queues itself until done)
    │   SELECT batch of queued rows
    │   applies rate_limiter + circuit_breaker
    │   for each row:
    │     pick telephony provider + from_number
    │     call provider API (Twilio/Plivo/etc.)
    │     UPDATE queued_runs SET state='executing'
    │   if failure rate > threshold:
    │     UPDATE campaigns SET state='paused' (circuit breaker)
    │
    ▼
  Provider webhooks POST /telephony/callback/{run_id}
    │   parse_webhook_request()
    │   UPDATE workflow_runs + queued_runs with disposition
    │
    ▼
  When all rows processed: UPDATE campaigns SET state='completed'

  ---
  6. Configuration Hierarchy (Who Overrides What)

  ┌─────────────────────────────────────────┐
  │  Workflow-level override (highest)      │
  │     (per-workflow model settings)        │
  ├─────────────────────────────────────────┤
  │  User configuration                      │
  │     (personal API keys, voice prefs)     │
  ├─────────────────────────────────────────┤
  │  Organization configuration              │
  │     (org-wide defaults, quotas)          │
  ├─────────────────────────────────────────┤
  │  Service Defaults / MPS keys (lowest)    │
  │     (fallback credentials provided by    │
  │      Dograh if user didn't bring own)    │
  └─────────────────────────────────────────┘

  Resolver lives in api/services/configuration/resolve.py.

  ---
  7. Where to Make Changes (Extension Points)

  Add a New Telephony Provider

  1. Add provider enum in api/enums.py
  2. Create api/services/telephony/providers/<name>/client.py + callbacks.py     
  3. Register in api/services/telephony/factory.py
  4. Add credentials schema in api/schemas/telephony_config.py
  5. Update UI: src/components/telephony/ConfigFormDialog.tsx

  Add a New Workflow Node Type

  1. Add node type to api/services/workflow/node_data.py (discriminated union)   
  2. Add node behavior in api/services/pipecat/run_pipeline.py or engine
  3. Add node spec to backend route node_types.py
  4. (Re-)generate src/client/ (npm run generate-client)
  5. Add icon + default config in src/components/flow/
  6. Update AddNodePanel.tsx if needed

  Add a New Background Task

  1. Write async function in api/tasks/
  2. Register in api/tasks/arq.py WorkerSettings.functions
  3. Call enqueue_job('function_name', ...) from services/routes

  Change the LLM / TTS / STT Integration

  1. api/services/configuration/registry.py — register provider metadata
  2. api/services/pipecat/service_factory.py — instantiate the service
  3. pipecat/ submodule — if behavior needs low-level pipeline changes

  Add a New Page to the Frontend

  1. Create src/app/<new-route>/page.tsx
  2. Use useAuth() guard pattern
  3. Use generated API client from src/client/sdk.gen.ts
  4. Check src/context/ if you need global config

  Change Database Schema

  1. Modify api/db/models.py
  2. alembic revision --autogenerate -m "description"
  3. Update corresponding api/db/*_client.py
  4. Update Pydantic schemas in api/schemas/
  5. Regenerate frontend client: npm run generate-client

  ---
  8. Security & Multi-Tenancy Critical Rules

  Every route handler and DB client must include organization_id. This is not    
  optional — it's the tenant isolation boundary.

  Good pattern:
  workflow = await db_client.get_workflow(
      workflow_id,
      organization_id=user.selected_organization_id
  )

  Bad pattern (security hole):
  workflow = await db_client.get_workflow(workflow_id)  # missing org filter!    

  The db_client composite pattern makes this straightforward, but always verify  
  new queries include org scoping.

  ---
  This graph covers every major directory, file group, and data flow in Dograh.  
  If you want me to zoom into any specific part (e.g., "I want to modify the     
  Pipecat pipeline" or "I want to add a new report page"), just tell me the area 
  and I'll read the exact files and give you surgical guidance.