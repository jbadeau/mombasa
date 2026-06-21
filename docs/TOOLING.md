# Mombasa — Tooling & Technology Choices

How we build a platform that is, at once, a **game** (the city), a **GitHub**
(collaborative coding), a **Google Wave** (real-time co-editing), and a **social
network** (feed, profiles, reputation). This maps each of those pillars to
concrete tools, on top of the locked stack: **React + TypeScript** frontend,
**Kotlin + Spring Boot** backend (see `DESIGN.md` and `DOMAIN_MODEL.md`).

> Reminder (DESIGN §0.4): GitHub is **build-time only** — these are tools we
> develop Mombasa *with*; the product itself never depends on GitHub at runtime.

---

## 0. The framing

It's not one kind of app, it's four overlapping surfaces, and each wants
different tooling:

| Pillar | What it needs | Primary tools |
| --- | --- | --- |
| **Game (the city)** | Render a living isometric city in the browser | **PixiJS** (WebGL 2D) |
| **GitHub (code)** | Host git + run code, inside Mombasa | **JGit** + container sandboxes |
| **Wave (real-time)** | Character-level co-editing, presence | **Yjs (CRDT)** + WebSocket |
| **Social network** | Feed, profiles, graph, notifications | React + Spring + Postgres/Redis |
| **AI agents** | Humans + agents as peers | **Anthropic API** (Claude) |

---

## 1. Frontend (social UI + the app shell)

- **React + TypeScript**, built with **Vite**.
- **Routing:** React Router (or TanStack Router).
- **Server state:** TanStack Query (caching, mutations, infinite feed).
- **Client/UI state:** Zustand.
- **UI kit:** Tailwind CSS + Radix UI primitives (or shadcn/ui).
- **Forms/validation:** React Hook Form + Zod.
- **Feed performance:** TanStack Virtual for long virtualized lists.
- **Real-time client:** native `WebSocket` subscribing to the channels in
  `DESIGN.md §9.4` (`wave:`, `workspace:`, `city`, `app:`, `user:`).

## 2. The city (game layer)

- **PixiJS** (+ `@pixi/react`) for the 2D isometric WebGL render (decision in
  `DESIGN.md §7.3`).
- **pixi-viewport** for camera pan/zoom + culling.
- **Tiled** map editor for authoring district/street layouts; **Kenney** free
  isometric asset packs to prototype.
- Upgrade path if we ever go 3D: **Three.js + react-three-fiber**.
- It stays a *renderer* — not a game engine. No Unity/Godot/physics; the backend
  computes `CityState`, the client draws it.

## 3. Real-time collaboration (the Wave layer)

- **Yjs** — the CRDT that powers live, character-level co-editing, presence, and
  cursors (`DESIGN.md §4.2`). Chosen over OT for simpler reasoning and graceful
  offline/reconnect.
- **Code editor component:** CodeMirror 6 (lighter) or Monaco (VS Code engine),
  bound to Yjs via `y-codemirror` / `y-monaco`.
- **Transport:** our own WebSocket gateway (Spring) relaying Yjs updates; Redis
  pub/sub fans out across backend nodes.
- Yjs document updates are themselves an op-log → they fit the "everything is a
  replayable log" spine and feed **playback**.

## 4. Collaborative coding (the GitHub-like layer), inside Mombasa

- **JGit** (pure-Java git, runs on the JVM with Kotlin) — Mombasa's **own git
  host**: create repos, read/write refs, and **commit programmatically** (how an
  agent contributes). Bare repos on object storage.
- **Execution sandboxes** (run + verify code): containers via **Docker**,
  orchestrated by **Kubernetes**. For stronger isolation of untrusted/agent code:
  **gVisor** or **Firecracker** microVMs, or **Kata Containers**. Scale-to-zero
  with **Knative** keeps idle cost down (ties to the credits economy, §6).
- **Verification/CI** runs in the sandbox against a submitted git ref
  (`DESIGN.md §5.3`).

## 5. Backend (Kotlin)

- **Spring Boot** (Web/WebFlux, Security, Data JPA, WebSocket).
- **Build:** Gradle (Kotlin DSL), multi-module mirroring the bounded contexts.
- **Migrations:** Flyway. **Persistence:** Spring Data JPA / Hibernate; consider
  **jOOQ** or **Exposed** (Kotlin) for type-safe SQL where JPA chafes.
- **Concurrency:** Kotlin coroutines (agent orchestration workers).
- **Auth:** Spring Security + OAuth2/OIDC. Identity provider: **Keycloak**
  (self-host) or a hosted IdP (Auth0/Clerk/Ory).
- **Events:** start with Spring's in-process events for the domain-event bus;
  extract to **Kafka** later if scale demands (`DESIGN.md §9.3`).

## 6. Data, infra & the credits economy

- **PostgreSQL** — system of record (relational + JSONB).
- **Redis** — cache, rate limiting, and WebSocket pub/sub fan-out.
- **Object storage** — S3 / self-hosted **MinIO** — for uploads, **bare git
  repos**, artifacts, and app deploys.
- **Search/discovery:** Postgres full-text first; **OpenSearch** if the feed/
  search outgrows it.
- **Metering → billing:** the credits/usage model (`DOMAIN_MODEL.md §8`) meters
  compute + storage; **Stripe** (Billing + metered usage / subscriptions)
  handles credit purchases and plan tiers. This is the revenue engine.

## 7. AI agents — the differentiator

Agents are first-class participants (`DESIGN.md §8`). They run on **Claude via
the Anthropic API**.

- **SDK:** the official **Anthropic Java SDK** (`com.anthropic:anthropic-java`)
  — Kotlin uses the Java SDK directly. The Spring/Kotlin orchestrator calls it
  from coroutine workers.
- **Default model:** `claude-opus-4-8` (most capable Opus-tier). Use
  `claude-fable-5` for the hardest long-horizon agent runs; a cheaper tier
  (`claude-haiku-4-5`) for lightweight/auto-match tasks. Model choice maps
  naturally onto agent autonomy level + the credit budget.
- **Two build options for the agent loop + sandbox:**
  1. **Self-hosted (Claude API + tool use):** we run the agent loop and the
     execution sandbox (§4). Maximum control over isolation, attribution, and
     the credits metering — fits "we own the substrate."
  2. **Managed Agents (Anthropic-hosted):** Anthropic runs the agent loop *and*
     provisions a per-session container where the agent's tools (bash, file ops,
     code) execute, streaming events back. This could dramatically shortcut the
     sandbox-orchestration work in early phases.
  - *Lean:* prototype agents on **Managed Agents** to move fast (Phase 3), and
    evaluate self-hosting once the credits/cost model and isolation requirements
    are firm — self-hosting gives us the metering and data-ownership the design
    calls for.
- **Safety mechanics** (`DESIGN.md §8.3`): per-agent credit budget, rate limits,
  kill-switch, sandboxed execution with scoped tokens — all enforced by our
  orchestrator regardless of which option above we pick.

## 8. DevOps / platform (build-time)

- **Source & CI:** GitHub + **GitHub Actions** (this is the build-time GitHub
  use; see the boundary note up top).
- **Containers:** Docker. **Orchestration:** Kubernetes (managed: EKS/GKE).
- **IaC:** Terraform (or Pulumi).
- **Observability:** OpenTelemetry → Prometheus/Grafana (metrics), Loki (logs),
  **Sentry** (frontend + backend error tracking).

## 9. Design / prototyping tools (designing the system)

- **Figma** — UI/UX and the city's visual language.
- **Excalidraw / Miro** — architecture sketches; **EventStorming** for the
  domain (it's how `DOMAIN_MODEL.md` was shaped).
- **Mermaid** — diagrams-as-code, already used in the design docs.

---

## 10. Minimal starting set ("start small", DESIGN §0.14)

For the **Phase 1 core loop**, the toolbox shrinks to:

> Vite + React + TS + Tailwind + TanStack Query (frontend) · Kotlin + Spring
> Boot + Postgres + Flyway + JGit (backend) · OIDC auth · Docker.

Everything else — PixiJS (Phase 2 city), Yjs live editing + Anthropic agents
(Phase 3), Stripe/Kafka/Knative (Phase 4) — layers on per the roadmap. We don't
need the full list to begin.
</content>
