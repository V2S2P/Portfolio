---
title: "The Council: Building a Multi-Agent Code Review System"
date: 2026-05-04
draft: false
description: "Instead of asking one AI to do everything, I built a council of six specialist agents — each owning a narrow domain — and had them review every decision in a structured build process. Here is how it works, what it caught, and how it compares to the standard agent."
tags: ["claude", "ai", "multi-agent", "experiment"]
---

## What did I do?

Most developers use an AI coding assistant the same way they use a search engine: ask a question, get an answer, move on. The assistant is a generalist. It knows a little about everything and a lot about nothing in particular. That is fine for small tasks. For building a real system, it starts to show cracks.

When one agent is simultaneously the architect, the security reviewer, the database designer, the UX designer, and the code quality enforcer, it will make tradeoffs you never asked for. It will focus on what it was most recently asked about. It will miss things that fall between concerns. And it will not push back on its own decisions.

I wanted something better. So I built **The Council** — a multi-agent system where six specialist agents each own a narrow domain, and every significant decision in a build is routed to the relevant specialist.

---

## The Council Members

| Agent | Domain |
|---|---|
| **Architect** | System design, API contracts, data flow, component boundaries |
| **Security Guard** | Auth flows, data exposure, input validation, OWASP Top 10 |
| **DB Whisperer** | Hibernate mapping, schema design, N+1 detection, PostgreSQL |
| **Quality Critic** | SOLID principles, naming, patterns, React hooks, Lombok usage |
| **Structure Warden** | Package and folder organization, layer discipline, class placement |
| **UI Artisan** | UX patterns, visual design, accessibility, frontend performance |

Each agent responds in the same structured format every time — no open-ended answers, no lengthy explanations:

```
[Risk]: high / medium / low / none
[Concern]: <the specific issue>
[Recommendation]: <the concrete fix>
```

All high-risk findings must be resolved before the build continues. A seventh agent, the **Builder**, acts as the orchestrator — running builds in four structured phases and consulting the council at defined checkpoints.

---

## Why Six Specialists?

The stack I work with — React + Vite on the frontend, Java + Javalin + Hibernate + PostgreSQL + Lombok on the backend — has enough moving parts that a single generalist agent regularly makes inconsistent decisions across layers. It designs a clean API and then forgets about it halfway through the service layer. It maps Hibernate entities correctly and then suggests `@Data` on an entity (which causes `StackOverflowError` in Hibernate 6 due to `hashCode` on lazy collections). It builds a beautiful component and ships it without ARIA labels.

The Council mirrors how a real engineering team works. Senior engineers do not review everything themselves. They route security questions to the person who thinks about security all day. They route schema decisions to the person who has been burned by N+1 queries before. Specialization produces better outcomes than generalism at scale.

Each council member is a Claude subagent with a tightly scoped system prompt. The scope is enforced explicitly — every member's prompt includes a **Do NOT comment on** clause listing the domains owned by other members. This prevents the sprawl that comes from a generalist trying to comment on everything at once.

The **UI Artisan** was the last member added. Originally the council had five members covering the backend-heavy concerns. The frontend and UX were getting reviewed by the Quality Critic and Structure Warden, which left visual design, accessibility, and UX patterns without a dedicated voice.

---

## The Test: Building a Habit Tracker

To stress-test the full system, I had the Builder construct a Habit Tracker application from scratch:

- User accounts with register and login (JWT auth, BCrypt passwords)
- Habits with name, category, target frequency (daily or specific weekdays), color, and icon
- Per-day completion tracking with toggles
- A dashboard — design decisions delegated entirely to the UI Artisan

The Builder ran four phases, consulting the council at each checkpoint.

### Phase 1 — Planning (All 6 in Parallel)

Before writing a single line of code, the Builder presented the full plan to all six council members simultaneously. This is the highest-leverage consultation: catching design problems before they are baked into code.

**What the council caught:**

- **Security Guard (high risk):** Ownership checks must be enforced on every single endpoint — not just the obvious ones. Every habit mutation and completion toggle must verify the habit belongs to the requesting user at the service layer, not just the controller.
- **Architect (medium risk):** `completedToday` was missing from the `HabitResponse` DTO. Without it, the frontend would need a separate network call per habit just to know whether to show a completed state.
- **DB Whisperer (medium risk):** Storing target weekdays as a `List<String>` in a join table would produce unnecessary rows and joins. Better: a bitmask integer with an `AttributeConverter<Set<DayOfWeek>, Integer>` — one column, no join table, O(1) reads.
- **UI Artisan (low risk):** Defined the design system upfront — dark theme (`#0f0f13` background, `#1a1a24` surface, `#6366f1` indigo accent), ProgressRing component for per-habit completion rate, a weekly calendar grid, a streak counter, and a slide-in drawer for the habit creation form. Optimistic UI on the completion toggle so it feels instant.

### Phase 2 — Schema + API Design

- **DB Whisperer:** Confirmed composite primary key mapping for `Completion` in Hibernate 6. Flagged a missing index on `habits(user_id)` that would cause full table scans on every habit list.
- **Architect:** Caught that `DashboardStatsResponse` was missing a `weeklyData` field needed to render the weekly calendar widget. Without it the widget would have had no data source.

### Phase 4 — Pre-Done Checkpoint

- **Security Guard (medium risk):** The JWT secret had a hardcoded fallback value in source code. Fix: throw `IllegalStateException` at startup if `JWT_SECRET` env var is missing or shorter than 32 characters. No fallback, ever.
- **Quality Critic (medium risk):** `DashboardService.getStats()` was calling `findAllDatesByHabitId()` inside a loop over all habits — an N+1 query at the service layer. Fix: single `findAllDatesByUserId()` query returning `Map<UUID, List<LocalDate>>`, then compute all streaks in memory from that map.

---

## Results

| Metric | Value |
|---|---|
| Total tokens used | 119,050 |
| Tool calls made | 105 |
| Wall-clock build time | ~35 minutes |
| Council consultations | 10 across 4 phases |
| High-risk findings resolved | 1 |
| Medium-risk findings resolved | 4 |
| Low-risk findings resolved | 5 |
| Files produced | 33 Java source files + full React frontend |

---

## Council vs. Standard Agent

A standard agent running the same build would likely use 40–70k tokens. The Council used 119k. That is the cost of structured review — more total tokens, but with a higher signal-to-noise ratio on what gets shipped.

| | Standard Agent | The Council |
|---|---|---|
| Token cost | Lower (~40–70k) | Higher (~119k) |
| Audit trail | None | Full log of every question and risk rating |
| Ownership check catch | Unlikely | Caught in Phase 1 (high risk) |
| N+1 service layer catch | Unlikely | Caught in Phase 4 (medium risk) |
| JWT fallback secret catch | Unlikely | Caught in Phase 4 (medium risk) |
| UX design ownership | Ad hoc | Dedicated specialist, defined upfront |

The two findings that matter most — the N+1 at the service layer and the hardcoded JWT fallback — are exactly the kind of issues that fall between concerns. The N+1 shows up at the intersection of the service layer and the query layer. The JWT fallback is a security issue that looks like a convenience decision. A generalist building at speed misses both. The specialists whose entire job is to think about those intersections catch them before they ship.

---

## Components

- **Claude Code** — the CLI and agent runtime
- **6 specialist subagents** — each with a scoped system prompt and a structured response format
- **The Builder** — orchestrator agent running the 4-phase build protocol
- **council-history.json** — shared audit log of every consultation
- **council.py** — interactive terminal CLI for consulting the council directly

---

### Pro tip of the day

When building the council, the most important design decision was the **Do NOT comment on** clause in every agent's prompt. Without it, every specialist tries to cover everything — and you end up with six generalists instead of six specialists. Narrow scope is what makes specialization work.
