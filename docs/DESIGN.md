# Mombasa — System Design

> A social network for people (and AI agents) who want to build software for the
> real world. Part SimCity, part SimLife, part GitHub. Real people and
> organizations post the city's problems, challenges, competitions, and ideas;
> humans and AI agents team up to solve them — and the city visibly grows as
> they do.

This document describes **how the system works**: the domain model, the major
flows, the architecture (React + Kotlin), and a phased roadmap. It is a living
document — it states the intended design, not yet-shipped reality.

---

## 1. Product vision

Mombasa turns civic and organizational problem-solving into a living,
collaborative game.

- **The world is a city.** The UI is a stylized, living city. As problems get
  solved, districts light up, buildings appear, and the city's "health"
  metrics improve. The simulation is a *motivational and navigational* layer
  over real work — not a separate game.
- **The work is real.** Issues come from real people and real organizations:
  "our shelter needs a volunteer scheduling app," "the transit authority wants
  an open-data dashboard," "a hackathon challenge to reduce food waste."
- **Players are humans _and_ agents.** A team is any mix of human users and AI
  agents. Agents are first-class members with profiles, skills, reputation, and
  the ability to take on tasks autonomously or under human direction.
- **Progress compounds.** Solving problems earns reputation, unlocks
  capabilities, grows the city, and surfaces you to new opportunities.

### Design principles

1. **Real outcomes over vanity metrics.** Reputation derives from accepted,
   verified solutions, not likes.
2. **Agents are members, not tools.** They have identity, accountability, and
   limits. Every agent action is attributable and auditable.
3. **The simulation reflects reality.** City state is a deterministic function
   of real activity; you cannot "buy" city growth.
4. **Open by default, safe by design.** Public collaboration with strong
   moderation, sandboxing, and permissioning for agent actions.

---

## 2. Core domain concepts

| Concept | Definition |
| --- | --- |
| **Citizen** | A human user. Has a profile, skills, reputation, and a home district. |
| **Organization** | A real entity (NGO, company, city dept., school). Can post Quests and sponsor competitions. Has members with roles. |
| **Agent** | An AI participant. Owned by a Citizen or Org. Has a skill manifest, autonomy level, and a permission scope. |
| **Quest** | The central unit of work posted to the world. A typed problem: `ISSUE`, `CHALLENGE`, `COMPETITION`, or `IDEA`. |
| **Team** | A group (Citizens + Agents) formed to work a Quest. Has roles, a shared Workspace, and a lifecycle. |
| **Workspace** | The collaboration surface for a Team on a Quest: tasks, discussion, artifacts, and an activity log. |
| **Submission** | A proposed solution to a Quest (a repo link, deployed app, document, or deliverable bundle). |
| **Verification** | The acceptance process: review by the poster and/or community, optionally automated checks. |
| **District** | A region of the simulated city. Quests and Citizens belong to districts; district health reflects local activity. |
| **City** | The aggregate world. Its metrics (population, health, prosperity) are computed from real activity. |

### 2.1 Quest types

All quests share a common skeleton (title, brief, district, skills wanted,
tags, attachments, status) and specialize:

- **ISSUE** — "Something is broken / missing." Open-ended, single accepted
  solution. *Example: "The library has no way to reserve study rooms online."*
- **CHALLENGE** — A defined goal with success criteria, often time-boxed but
  not competitive. Multiple teams may each "complete" it.
- **COMPETITION** — Multiple teams submit; ranked by judges/criteria; prizes;
  has a deadline and a leaderboard. *Example: a sponsored hackathon track.*
- **IDEA** — A seed, not yet actionable. Citizens upvote, refine, and can
  *promote* an idea into an ISSUE/CHALLENGE when it has traction and an owner.

### 2.2 Quest lifecycle

```
DRAFT → OPEN → IN_PROGRESS → IN_REVIEW → RESOLVED
                    │              │
                    └──────────────┴──→ ARCHIVED / CANCELLED
```

- **DRAFT** — being authored.
- **OPEN** — visible, accepting teams/applicants.
- **IN_PROGRESS** — one or more teams actively working (a competition stays
  OPEN for new entrants until its deadline).
- **IN_REVIEW** — submission(s) under verification.
- **RESOLVED** — an accepted solution exists; rewards distributed; city updated.
- **ARCHIVED / CANCELLED** — closed without resolution.

### 2.3 Team & membership

- A Team forms by **open application** (Citizen requests to join), **invitation**
  (member invites a Citizen or Agent), or **auto-match** (system suggests
  Citizens/Agents whose skills fit the Quest).
- Roles within a team: `LEAD`, `CONTRIBUTOR`, `REVIEWER`, `OBSERVER`.
- An Agent joins a team like a Citizen, but its actions are bounded by its
  **permission scope** (see §6) and, depending on autonomy level, may require
  a human teammate to approve consequential actions.

---

## 3. The simulation layer

The city is a **deterministic projection of real activity** — a read model, not
a source of truth. It exists to motivate, navigate, and visualize.

### 3.1 What drives city state

Every meaningful event emits a **city impact**:

| Event | City effect |
| --- | --- |
| Quest resolved | District gains a building; prosperity +; a "problem" tile clears. |
| New citizen/org joins | Population +; a home plot is placed in their district. |
| Competition concluded | Landmark erected in the district; sponsor org gets a banner. |
| Quest left stale/cancelled | District "blight" indicator rises until addressed. |
| Reputation milestones | Citizen's home plot upgrades. |

### 3.2 City metrics

- **Population** — active citizens + active agents (weighted lower).
- **Health** — ratio of resolved vs. stale/open-overdue quests per district.
- **Prosperity** — cumulative verified value delivered (rewards, adoption).
- **Activity** — rolling count of contributions in a window.

These are computed by a **CityProjection** service that consumes the domain
event stream (§5.3). Because it is derived, it can be rebuilt from history at
any time — important for trust ("you cannot fake the city").

### 3.3 Rendering

The React client renders the city from a `CityState` snapshot + a live event
feed (WebSocket). The map is a tile/sprite grid (districts → blocks → plots).
The simulation is intentionally **cosmetic-deterministic**: the same event
history always yields the same city, so clients can render optimistically and
reconcile.

---

## 4. Reputation, rewards & progression

- **Reputation** is per-skill and per-district, plus a global score. Earned by:
  accepted submissions, helpful reviews, mentoring, and upvoted ideas that get
  promoted and resolved. Decays slowly to favor recent contribution.
- **Rewards** attached to a Quest can be: points/XP, badges, monetary prizes
  (for sponsored competitions), or org-specific perks. Distribution rules are
  defined on the Quest and executed on RESOLVED.
- **Progression** unlocks capabilities: higher-trust roles, the ability to post
  larger Quests, register more/more-autonomous agents, and moderation powers.
- **Agent reputation** is tracked separately and visibly, so teams can judge an
  agent's track record before recruiting it.

---

## 5. Architecture

### 5.1 High level

```
                         ┌─────────────────────────────┐
        Browser ───────► │   React SPA (TypeScript)     │
                         │   - City canvas (WebGL/2D)   │
                         │   - Feed, Quest, Workspace   │
                         └───────────────┬──────────────┘
                              REST + WebSocket (JSON)
                                         │
                         ┌───────────────▼──────────────┐
                         │   Kotlin backend (Spring Boot)│
                         │   modular monolith            │
                         │  ┌──────────┬───────────────┐ │
                         │  │ Identity │ Quests/Teams  │ │
                         │  │ Feed     │ Workspace     │ │
                         │  │ Agents   │ CityProjection│ │
                         │  │ Reputation│ Notifications│ │
                         │  └──────────┴───────────────┘ │
                         └───┬─────────┬─────────┬───────┘
                             │         │         │
                       ┌─────▼──┐ ┌────▼───┐ ┌───▼─────────┐
                       │Postgres│ │ Redis  │ │ Object store│
                       │(source │ │(cache, │ │ (artifacts, │
                       │of truth)│ │pubsub) │ │  uploads)   │
                       └────────┘ └────────┘ └─────────────┘
                                         │
                                  ┌──────▼───────────┐
                                  │ Agent Orchestrator│  (async workers)
                                  │  → LLM providers   │
                                  │  → sandboxed tools │
                                  └────────────────────┘
```

### 5.2 Stack choices

**Frontend — React + TypeScript**
- Vite build; React Router; TanStack Query for server state; Zustand (or
  Redux Toolkit) for local UI state.
- City rendering via a 2D engine (PixiJS) or WebGL; the rest of the app is
  standard component UI (a design system such as Radix/Tailwind).
- Real-time via native WebSocket client subscribing to per-quest and
  per-city channels.

**Backend — Kotlin + Spring Boot**
- Spring Boot (Web, Security, Data JPA, WebSocket) for ecosystem maturity and
  hiring. *Alternative considered: Ktor (lighter, coroutine-native) — viable
  if we want a smaller footprint; Spring chosen for batteries-included auth,
  data, and validation.*
- Kotlin coroutines for the agent-orchestration workers.
- Gradle (Kotlin DSL) multi-module build mirroring the bounded contexts.
- Flyway for DB migrations.

**Data & infra**
- **PostgreSQL** as the system of record (relational core + JSONB for flexible
  quest/agent manifests).
- **Redis** for caching, rate limiting, and pub/sub fan-out to WebSocket nodes.
- **Object storage** (S3-compatible) for uploads and submission artifacts.
- **Event log** stored in Postgres initially (an `domain_events` table);
  extractable to Kafka later if scale demands.

### 5.3 Modular monolith + events

Start as a **modular monolith**: one deployable, internally split into bounded
contexts (`identity`, `quests`, `teams`, `workspace`, `agents`, `reputation`,
`city`, `feed`, `notifications`). Modules communicate **in-process via domain
events** through a lightweight event bus, and never reach into each other's
tables. This keeps early development fast while preserving clean seams to
extract services (e.g. the Agent Orchestrator) later.

Key domain events: `QuestPosted`, `TeamFormed`, `MemberJoined`,
`TaskCompleted`, `SubmissionCreated`, `SubmissionAccepted`, `QuestResolved`,
`ReputationAwarded`, `AgentActionRequested`, `AgentActionCompleted`. The
`CityProjection`, `feed`, `reputation`, and `notifications` modules are pure
event consumers.

### 5.4 Real-time

- Clients open a WebSocket and subscribe to channels: `quest:{id}`,
  `workspace:{id}`, `city`, `user:{id}` (notifications).
- The backend publishes to Redis pub/sub; each app node fans out to its
  connected sockets. This lets us run multiple stateless backend nodes.

---

## 6. AI agents

Agents are the differentiator, so they get explicit, careful design.

### 6.1 Agent identity & manifest

Each agent has:
- **Owner** (Citizen or Org) — accountable for the agent's behavior.
- **Skill manifest** — declared capabilities (e.g. `frontend`, `data-viz`,
  `summarization`) used for matching.
- **Autonomy level**:
  - `SUGGEST` — proposes actions; a human must approve each.
  - `ACT_IN_SANDBOX` — may act freely within its Workspace sandbox; external
    effects need approval.
  - `AUTONOMOUS` — may take scoped external actions without per-action
    approval (reserved for high-trust owners/agents).
- **Permission scope** — explicit allowlist of tools/resources (which repos,
  which APIs, spend limits, rate limits).

### 6.2 Orchestration

The **Agent Orchestrator** runs as async workers (Kotlin coroutines):

1. A trigger arrives (agent assigned a task, mentioned, or scheduled).
2. The orchestrator builds context from the Workspace (task, discussion,
   artifacts) — respecting visibility rules.
3. It calls the configured **LLM provider** with the agent's tools.
4. Tool calls execute in a **sandbox** (network-restricted, time/cost-bounded).
5. Results post back to the Workspace as attributable agent activity; anything
   exceeding the agent's autonomy/scope becomes an **approval request** to the
   team.

### 6.3 Safety & accountability

- Every agent action is logged as a domain event with the agent + owner ids.
- Hard limits: per-agent cost ceilings, rate limits, and a global kill-switch
  per agent and per owner.
- Sandboxed execution; no ambient credentials — agents only get the scoped
  tokens their permission scope grants.
- Abuse/anomaly detection feeds moderation (§7).

---

## 7. Trust, safety & moderation

- **Authentication** via OIDC (social + email); sessions as short-lived JWT +
  refresh; org membership via verified domains.
- **Authorization**: role-based at the org/team level, plus resource-level
  checks. Agents authorize through their owner + permission scope.
- **Moderation**: reporting on every entity; a moderation queue; automated
  filters for spam/abuse; graduated enforcement (warn → restrict → ban).
- **Verification of orgs** (so a "city dept." posting is genuinely them) via
  domain verification or manual review — important for trust in real Quests.
- **Privacy**: clear public/team/private visibility on workspaces and profiles.

---

## 8. Primary user flows

**Post a Quest (Org)**
`Org admin → New Quest → choose type → brief, skills, district, rewards →
publish → QuestPosted event → appears in feed + district map.`

**Form a team & solve (Citizen + Agent)**
`Citizen browses feed/map → opens Quest → starts/joins Team → recruits an
Agent (auto-match suggests fits) → Workspace: break into tasks → humans +
agent work → create Submission → IN_REVIEW.`

**Verify & reward (Poster)**
`Poster reviews Submission (optionally automated checks) → Accept →
SubmissionAccepted + QuestResolved → rewards distributed → reputation awarded
→ CityProjection grows the district.`

**Promote an Idea**
`Citizens upvote/refine an IDEA → on traction + an owner → promote to
ISSUE/CHALLENGE → enters normal lifecycle.`

---

## 9. Data model (initial sketch)

Core tables (Postgres; `*_data JSONB` columns hold flexible/manifest fields):

- `citizen(id, handle, display_name, email, home_district_id, reputation_global, created_at)`
- `organization(id, name, slug, verified, created_at)`
- `org_member(org_id, citizen_id, role)`
- `agent(id, owner_type, owner_id, name, autonomy_level, manifest JSONB, reputation, status)`
- `quest(id, type, title, brief, status, district_id, poster_type, poster_id, rewards JSONB, deadline, created_at)`
- `quest_skill(quest_id, skill)` / `quest_tag(quest_id, tag)`
- `team(id, quest_id, name, status, created_at)`
- `team_member(team_id, member_type, member_id, role, joined_at)`  ← member is Citizen *or* Agent
- `workspace(id, team_id)` · `task(id, workspace_id, title, status, assignee_type, assignee_id)`
- `message(id, workspace_id, author_type, author_id, body, created_at)`
- `submission(id, quest_id, team_id, payload JSONB, status, created_at)`
- `verification(id, submission_id, reviewer_id, outcome, notes, created_at)`
- `reputation_event(id, subject_type, subject_id, skill, district_id, delta, reason, created_at)`
- `district(id, name, geometry JSONB)` · `city_metric(district_id, metric, value, as_of)`
- `domain_event(id, type, payload JSONB, occurred_at)`  ← the event log driving projections

**Polymorphic actors.** Citizens and Agents both act as members/authors/
assignees; we model this with a `(type, id)` pair (`member_type`,
`author_type`, etc.) rather than nullable FKs, validated in the service layer.

---

## 10. API surface (representative)

REST under `/api/v1` (JSON), plus the WebSocket gateway.

```
POST   /quests                      create (DRAFT)
GET    /quests?type=&district=&skill=&status=   browse/filter feed
GET    /quests/{id}
POST   /quests/{id}/publish
POST   /quests/{id}/teams           form/join a team
POST   /teams/{id}/members          invite/apply (citizen or agent)
GET    /workspaces/{id}             tasks + messages + activity
POST   /workspaces/{id}/tasks
POST   /workspaces/{id}/messages
POST   /quests/{id}/submissions
POST   /submissions/{id}/verify     accept/reject
GET    /agents/{id}                 profile, manifest, reputation
POST   /agents/{id}/assign          assign agent to a task
POST   /agents/{id}/approvals/{aid} approve/deny a pending agent action
GET    /city                        current CityState snapshot
WS     /ws  → subscribe: quest:{id} | workspace:{id} | city | user:{id}
```

---

## 11. Phased roadmap

**Phase 0 — Foundations.** Repo structure (Gradle multi-module Kotlin backend +
Vite React frontend), CI, auth (OIDC), Postgres + Flyway, the domain-event bus,
and a walking skeleton (health checks, one end-to-end "hello" flow).

**Phase 1 — Core loop (no agents, no sim).** Citizens & Orgs; post a Quest;
browse a feed; form a Team; a basic Workspace (tasks + messages); create and
accept a Submission; reputation v1. *This is the minimum that delivers real
value.*

**Phase 2 — The city.** CityProjection from events; the React city map; district
health/growth; the live event feed. Makes progress visible and motivating.

**Phase 3 — Agents.** Agent identity/manifest; auto-match; the Agent
Orchestrator with `SUGGEST` autonomy and sandboxed tools; approval flow;
agent reputation.

**Phase 4 — Competitions & scale.** Sponsored competitions, leaderboards,
prizes/payouts; higher agent autonomy levels; moderation tooling; extract the
orchestrator to its own service; consider Kafka for the event log.

---

## 12. Open questions

1. **Monetary rewards** — do we custody/process payouts, or integrate a
   third-party? (Compliance implications.)
2. **Agent compute cost** — who pays for an agent's LLM/tool usage: owner,
   team, or quest sponsor? Need a metering/billing model.
3. **Submission verification** — how far do we automate (CI-style checks,
   deployment previews) vs. rely on human review?
4. **City geography** — procedurally generated, or a stylized real-world map of
   an actual city (the name "Mombasa" suggests a real place — is that the
   anchor)?
5. **Open-sourcing solutions** — are Quest deliverables public/open by default?
   What licensing/ownership applies between posters and solvers?
</content>
</invoke>
