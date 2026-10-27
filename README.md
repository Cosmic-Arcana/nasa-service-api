# Cosmic Arcana — NASA Service

The window to the real sky. This service brings real astronomical information into Cosmic Arcana, so predictions can be inspired by what is actually happening above us.

## The idea

Cosmic Arcana is an AI-native fortune-telling experience.

A user asks a question — about a decision, a relationship, a career move, or simply what the coming week might bring — and an AI agent answers the way a digital fortune-teller would. It draws tarot cards, looks at what is happening in the sky, remembers previous readings, and weaves everything into a personal, entertaining prediction.

The predictions are **fictional and reflective** by design. They are an invitation to think, not a claim about the future. Real-world data such as NASA imagery or astronomical events is used as storytelling material — symbolism, themes, atmosphere — and is never presented as evidence that the future can be predicted. The product always keeps a visible line between what was *retrieved from the real world* and what was *imagined*.

The core loop:

```text
User question
    ↓
Prediction / tarot request
    ↓
AI agent
    ↓
Relevant tools and context
    ↓
Optional cosmic, historical or personal context
    ↓
AI interpretation
    ↓
Fictional prediction
    ↓
Saved reading
```

## How it is being built

Cosmic Arcana is also an experiment: how far can one software engineer push Claude Code as an engineering partner?

The project is **vibe-coded**. It is built through **Claude Code Remote Control** (a session on a development machine, driven from a smartphone) **and through Cursor**. After the hackathon, about **$190 of Cursor usage credits** remain. There is no fixed schedule and no desk required. Work happens wherever the engineer happens to be — in small pockets of free time (a commute, a queue, a quiet evening) and whenever there are tokens left that are worth spending. The roadmap is shaped as much by spontaneous ideas as by a plan. A feature often starts as a thought typed on a phone and ends as a reviewed commit.

That way of working shapes the engineering:

- **Claude implements, the engineer steers.** Most of the code is delegated; architecture, service boundaries and review stay in human hands.
- **Context lives in the repositories.** Any session must be able to pick up where the previous one stopped, so knowledge is kept in `CLAUDE.md` files, progress notes, conventions, skills and hooks — not in anyone's memory.
- **Small slices.** Tasks are cut small enough to plan, implement, review and commit from a phone screen.
- **Automated quality gates.** Tests, structured logging and consistent conventions catch what a small screen might miss.
- **Multi-repository by design.** Every service lives in its own repository, which makes cross-repository context sharing part of the experiment.
- **Deliberate context budgeting.** Short sessions reward tight prompts, focused tools and small outputs — the same discipline the product asks of its own AI agent.

## The system

Cosmic Arcana is split into independent services, each in its own repository:

| Service | Role |
| --- | --- |
| **cosmic-arcana-storefront** | Web application and BFF — everything the user sees, and the only API the browser talks to |
| **ai-service-api** | The fortune-teller's mind — predictions, tarot readings and all AI-specific business logic |
| **nasa-service-api** *(this repository)* | The window to the real sky — retrieves, normalizes and caches NASA and astronomical data |
| **mcp-service-api** | The AI agent's doorway — exposes application capabilities as MCP tools, with agent authentication and on-behalf-of access |

Around them sit a few parts that do not have their own repositories yet: a **CQRS command layer** that orchestrates use cases, a **Redis / BullMQ broker** that carries domain events, a **PostgreSQL read model** that stores readings, and a **PostgreSQL MCP server** that gives the agent restricted, user-scoped database access.

```text
User
  │
  ▼
Storefront (Next.js + BFF) ◄─────────────── queries ───────────────┐
  │                                                                │
  │ commands                                                       │
  ▼                                                                │
Command layer (NestJS CQRS)                                        │
  │                                                                │
  ├──► AI service ─────┐                                           │
  └──► NASA service ───┤                                           │
                       │ domain events                             │
                       ▼                                           │
             Broker (Redis / BullMQ) ──► Read model (PostgreSQL) ──┘


AI agent (Claude) ── MCP ──► MCP service ──┬──► Command layer
                                           └──► PostgreSQL MCP ──► cosmic_agent schema (RLS)
```

## What this service does

- **NASA integration** — the only place in the system that talks to NASA's public APIs. Examples of what it can draw on: the Astronomy Picture of the Day, near-Earth asteroid approaches, space weather such as solar flares and geomagnetic storms, and imagery of Earth and Mars.
- **Normalization** — every NASA API speaks its own dialect. The service turns them into one small, consistent shape: a few meaningful facts with their date and source instead of raw payloads. An AI agent needs "a solar flare peaked this morning", not a page of JSON.
- **Caching** — the sky of a given day does not change, and public APIs have rate limits. Data is fetched once and reused.
- **Domain events** — when relevant cosmic data is retrieved, the service publishes events so the rest of the system can react to it and store it.

### Real information, not fortune-telling

This service deals only in facts. It retrieves and describes what NASA reports; it does not interpret it mystically or connect it to anyone's future — that is the AI service's job.

Keeping that boundary in a separate service is what lets the product stay honest: everything that comes out of here is real, sourced information, and everything built on top of it is clearly fiction.

### How it is reached

The NASA service is internal. The command layer asks it for cosmic context over service-to-service messaging, and the AI agent can reach the same capability through the MCP service — but only when a question actually calls for it.

### What it has to handle well

- NASA being slow or unavailable must never break a reading — a prediction can always be made without cosmic context;
- a retried request must not trigger duplicate calls to NASA;
- every request carries the correlation id it was given, so it can be traced across services;
- API keys stay out of logs and out of responses.

### What it does not do

It does not generate predictions, call the AI model or hold any user data.
