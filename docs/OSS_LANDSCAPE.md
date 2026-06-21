# Mombasa — Open-Source Landscape

A survey of open-source tools and frameworks for each part of Mombasa, with
licenses noted. This complements `TOOLING.md` (which states our picks); this doc
records the **options and trade-offs**, especially for the expansive city view.

> ⚠️ **Verify licenses before committing.** Open-source licenses change (e.g.
> Redis → SSPL in 2024). Treat the license tags here as a starting point and
> re-check each project's current license, especially anything AGPL/SSPL/BSL,
> against how we intend to deploy (hosted SaaS).

---

## 0. The big fork for the expansive city view (§7.4)

Two open-source families solve "a huge, zoomable, living world", differently:

### A. Game-renderer family — bespoke look, build the scaling yourself
- **PixiJS** (MIT) — WebGL/WebGPU 2D renderer. Our current pick.
  - **pixi-viewport** (MIT) — camera, pan/zoom, culling.
  - **@pixi/tilemap** (MIT) — efficient repeated-tile layers (ground/roads).
- **Phaser** (MIT) — full 2D game engine: cameras, tilemaps, **Tiled** import,
  input, tweens. More batteries than Pixi; has isometric plugins.
- **Excalibur.js** (BSD), **melonJS** (MIT) — other OSS 2D engines.
- **Three.js** (MIT) + **react-three-fiber** (MIT), **Babylon.js** (Apache-2.0),
  **PlayCanvas engine** (MIT) — if we ever go 3D low-poly.
- You implement chunk loading, LOD, and the spatial index yourself (helped by
  the libs in §1).

### B. Map-engine family — scaling solved, feed it tiles
- **MapLibre GL JS** (BSD-3, the OSS fork of Mapbox GL) — vector-tile renderer
  with **built-in** smooth zoom, LOD, tile streaming, and viewport culling.
  Works with **custom (non-geographic) coordinate systems** and custom styles,
  so a stylized world is feasible.
- **deck.gl** (MIT, Uber) — WebGL2/WebGPU big-data layers (millions of objects);
  composes with MapLibre.
- **OpenLayers** (BSD) / **Leaflet** (BSD) — alternative map frameworks with
  custom tile grids/projections (Leaflet is lighter, more raster-oriented).
- **Near-off-the-shelf pipeline:** PostGIS (plots) → **Martin** (Apache-2.0) or
  **pg_tileserv** (Apache-2.0) serve Postgres rows as Mapbox Vector Tiles →
  MapLibre renders them with zoom/LOD for free. **tippecanoe** (BSD) bakes
  static vector tiles if we precompute. This gives the "expansive" behavior
  almost for free; we'd lose some of the hand-crafted game look.

**Recommendation:** prototype the expansive view *both* ways early — a MapLibre +
deck.gl spike vs. the PixiJS approach — and pick by how important bespoke
art/animation is versus time-to-expansive. The small-scale Phase 2 view (§7.3)
is fine on PixiJS regardless.

---

## 1. City scaling helpers (spatial index, procgen, ECS, assets)

- **Spatial index:** **rbush** (MIT) / **flatbush** (ISC) — fast JS R-trees for
  "what's in this bbox"; **PostGIS** (GPL-2.0) server-side; **H3** (Apache-2.0,
  Uber) hexagonal global index for chunking/aggregation.
- **Procedural generation:** **simplex-noise** (MIT) for terrain/variation;
  **rot.js** (BSD) roguelike toolkit (procedural maps, FOV) — useful patterns
  for the deterministic `WorldLayout`.
- **Entity-Component-System** (for the "alive" layer — citizens, vehicles):
  **bitECS** (MPL-2.0), **Miniplex** (MIT), **becsy** (MIT). Keeps thousands of
  moving entities cheap.
- **Map authoring & assets:** **Tiled** (editor; GPL-2.0/BSD map format) — loads
  in Phaser & Pixi; **Kenney** asset packs (CC0); **OpenGameArt** (mixed).

## 2. Open-source city-sim *references* (study, not build on)

Not frameworks, but worth mining for simulation logic and feel:
- **Micropolis** (GPL-3.0) — the original SimCity, open-sourced; has JS ports.
- **Unciv** (MPL-2.0) — Civ clone written in **Kotlin/libGDX** (relevant stack).
- **OpenTTD** (GPL-2.0), **Simutrans** (Artistic), **LinCity-NG**, **Widelands**
  (GPL), **Citybound** (Rust, AGPL) — economy/city sims for inspiration.

---

## 3. Real-time collaboration (the Wave layer, §4.2)

- **Yjs** (MIT) — CRDT for live co-editing/presence. Our pick.
  - **Hocuspocus** (MIT) — production Yjs WebSocket server; **y-websocket** (MIT)
    the simple one.
  - **y-codemirror.next** / **y-monaco** (MIT) — editor bindings.
- **Automerge** (MIT) — alternative CRDT (Rust core, JS/WASM).
- **ShareDB** (MIT) — OT-based (if we ever wanted OT instead of CRDT).
- **Editors:** **CodeMirror 6** (MIT) — lighter; **Monaco** (MIT) — VS Code's
  engine, heavier.

## 4. Embedded git + code execution (the GitHub-like layer, §4–5)

- **Embedded git host:** **JGit** (BSD/EDL) — pure-JVM git, our pick;
  **isomorphic-git** (MIT) for any JS-side needs; **libgit2** (GPL-2.0 w/
  linking exception) for native.
- **Self-hosted git server (reference or component):** **Gitea** (MIT) /
  **Forgejo** (MIT/GPL) / **Gitiles** — if we'd rather embed a server than build
  on JGit directly.
- **In-browser code editors/runtimes:** **Sandpack** (Apache-2.0, CodeSandbox) —
  in-browser bundling/running for web apps; **code-server** (MIT) — VS Code in
  the browser; **Eclipse Theia** (EPL) — IDE framework.
- **Sandboxed execution (run/verify code):**
  - Isolation: **gVisor** (Apache-2.0), **Firecracker** (Apache-2.0),
    **Kata Containers** (Apache-2.0), **nsjail** (Apache-2.0).
  - Orchestration: **Docker** (Apache-2.0), **Kubernetes** (Apache-2.0),
    **Knative** (Apache-2.0) for scale-to-zero.
  - Ready-made code-runners: **Judge0** (GPL-3.0) and **Piston** (MIT) — OSS
    "run this code, get output" engines worth evaluating before building ours.
  - Dev-environment platforms: **Coder** (AGPL-ish/enterprise), **Eclipse Che**
    (EPL) — heavier but solve workspace provisioning.

## 5. Backend, data & infra (with license caveats)

- **Backend:** **Spring Boot** (Apache-2.0), **Ktor** (Apache-2.0). **Gradle**
  (Apache-2.0), **Flyway** (Apache-2.0).
- **Database:** **PostgreSQL** (PostgreSQL license) + **PostGIS** (GPL-2.0) for
  the spatial/expansive world.
- **Cache / pub-sub:** ⚠️ **Redis** moved to SSPL/RSAL (2024) — use **Valkey**
  (BSD, the Linux Foundation fork) for a cleanly-licensed drop-in.
- **Object storage:** ⚠️ **MinIO** is **AGPL-3.0** (matters for SaaS) —
  alternatives: **SeaweedFS** (Apache-2.0), **Garage** (AGPL), or just managed S3.
- **Events/streaming:** **Apache Kafka** (Apache-2.0), **Redpanda** (BSL→Apache),
  **NATS** (Apache-2.0).
- **Search:** ⚠️ avoid Elasticsearch licensing ambiguity — **OpenSearch**
  (Apache-2.0), **Meilisearch** (MIT), **Typesense** (GPL-3.0).

## 6. Credits, metering & billing (the economy, §6.4 / DOMAIN_MODEL §8)

Directly relevant to the credits economy — these meter usage and handle plans:
- **OpenMeter** (Apache-2.0) — usage metering built for AI/compute billing.
- **Lago** (AGPL-3.0) — open-source metering & billing (subscriptions + usage),
  can sit in front of Stripe.
- **Kill Bill** (Apache-2.0) — mature JVM billing platform.
- Payments processor itself (Stripe/Adyen) is not OSS, but the metering layer can
  be.

## 7. Auth & identity (§10 trust)

- **Keycloak** (Apache-2.0) — full OIDC/SAML IdP (JVM, pairs with Spring).
- **Ory** Hydra/Kratos (Apache-2.0), **Zitadel** (Apache-2.0/enterprise),
  **Authentik** (MIT/enterprise), **Supabase Auth/GoTrue** (MIT).

## 8. AI agents on the JVM (§8 / TOOLING §7)

- **Anthropic Java SDK** (MIT) — Kotlin uses it directly; default
  `claude-opus-4-8`.
- **Spring AI** (Apache-2.0) — Spring-native LLM/agent integration (Anthropic
  supported); **LangChain4j** (Apache-2.0) — JVM agent/tool framework;
  **Embabel** (agent framework, JVM).
- **Anthropic Managed Agents** is a hosted option (not OSS) that we may use to
  shortcut sandbox orchestration early (TOOLING §7).

## 9. DevOps & observability (build-time)

- **OpenTelemetry** (Apache-2.0), **Prometheus** (Apache-2.0), **Grafana**
  (AGPL-3.0), **Loki** (AGPL-3.0).
- Error tracking: ⚠️ **Sentry** is BSL/FSL — OSS alternative **GlitchTip**
  (MIT/various).
- IaC: **Terraform** ⚠️ moved to BSL — OSS fork **OpenTofu** (MPL-2.0); or
  **Pulumi** (Apache-2.0). CI: **GitHub Actions** (build-time, §0 boundary).

---

## 10. Summary picks (if forced to choose today)

| Concern | Open-source pick | Note |
| --- | --- | --- |
| Expansive city view | **MapLibre GL + deck.gl** *or* **PixiJS** | Spike both; map-engine path scales fastest |
| Tile pipeline | PostGIS → **Martin**/pg_tileserv → MVT | Off-the-shelf chunking/LOD |
| Spatial index | PostGIS + **rbush** (client) | |
| Real-time editing | **Yjs** + **Hocuspocus** + CodeMirror 6 | |
| Embedded git | **JGit** | + **Piston**/**Judge0** to evaluate for execution |
| Sandbox isolation | **gVisor**/**Firecracker** on **Kubernetes** | **Knative** for scale-to-zero |
| Cache/pubsub | **Valkey** | not Redis (license) |
| Object storage | managed S3 or **SeaweedFS** | MinIO is AGPL |
| Metering/billing | **OpenMeter** or **Lago** | in front of Stripe |
| Auth | **Keycloak** | JVM-native |
| Agents | **Anthropic Java SDK** + **Spring AI** | model `claude-opus-4-8` |
</content>
