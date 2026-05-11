---
title: "Standard Agent: Building the Same Habit Tracker Without a Council"
date: 2026-05-05
draft: false
description: "I built the exact same Habit Tracker that The Council produced — same feature spec, same stack, different approach. One agent, no structured review, no specialist consultation. Here is what it produced, how long it took, and what it missed."
tags: ["claude", "ai", "experiment", "habit-tracker"]
---

## What did I do?

The Council article described a multi-agent build system where six specialists review every decision at structured checkpoints. The natural follow-up question: what does the same app look like when you skip all of that and just ask one agent to build it?

I ran exactly that experiment. Same feature spec, same stack — React + Vite frontend, Java + Javalin + Hibernate + PostgreSQL + Lombok backend — but no council, no structured phases, no specialist review. Just a single agent writing files from top to bottom.

One important caveat up front: the spec I was given was informed by the council's previous build. It already included details like "completedToday in HabitResponse", "bitmask for DayOfWeek with AttributeConverter", and "single bulk query in DashboardService". Those are findings the council had to actively catch. The standard agent got them handed in the prompt. This limits how directly the two builds can be compared on correctness — the spec had already been hardened.

---

## What was built

**Backend — 30 Java source files:**

| Layer | Files |
|---|---|
| Entities | User, Habit, HabitLog, HabitLogId |
| DAOs | UserDao, HabitDao, HabitLogDao |
| Services | AuthService, HabitService, HabitLogService, DashboardService |
| Controllers | AuthController, HabitController, HabitLogController, DashboardController |
| Config | HibernateConfig, AppConfig, AuthFilter |
| DTOs | RegisterRequest, LoginRequest, AuthResponse, HabitRequest, HabitResponse, DashboardResponse, WeekDay |
| Util | JwtUtil, DayOfWeekConverter |
| Exception | ApiException |
| Entry point | Main |

**Frontend — 16 files:**

| Layer | Files |
|---|---|
| Pages | Login, Register, Habits, Dashboard |
| Components | Navbar, HabitCard, HabitDrawer, ProgressRing, WeekGrid |
| API | client.js |
| Root | App.jsx, main.jsx, index.css, index.html, package.json, vite.config.js |

**Design choices made independently:**

- Color scheme: GitHub-dark inspired (`#0d1117` background, `#161b22` surface, `#10b981` emerald accent) — distinct from the council's indigo palette
- ProgressRing component for per-habit streak visualization on the dashboard
- Heat-map style WeekGrid (5-shade green gradient) for the weekly calendar
- Slide-in drawer for habit creation with day toggles, color swatches, and emoji icon picker
- Optimistic UI on the completion toggle — local state flips immediately, API call fires in the background

---

## Results

| Metric | Value |
|---|---|
| Wall-clock build time | ~8 minutes |
| Tool calls made | 50 |
| Files produced | 46 (30 Java + 16 frontend) |
| Phases | 1 (no structured checkpoints) |
| Council consultations | 0 |
| Specialist reviews | 0 |
| Issues caught mid-build | 0 |

Token count for this session is not directly readable from within the agent, but based on output volume and comparison with the failed background agent attempts earlier in the same session (~15k tokens for 3-5 tool calls each), the build phase itself is estimated at **~45,000–55,000 tokens**.

---

## Standard Agent vs. The Council

| | Standard Agent | The Council |
|---|---|---|
| Wall-clock time | ~8 minutes | ~35 minutes |
| Tool calls | 50 | 105 |
| Estimated tokens | ~45–55k | 119,050 |
| Files produced | 46 | 33 Java + React frontend |
| Structured review phases | None | 4 |
| Council consultations | 0 | 10 |
| High-risk findings resolved | 0 | 1 |
| Medium-risk findings resolved | 0 | 4 |
| Low-risk findings resolved | 0 | 5 |
| Audit trail | None | Full log of every question and risk rating |

---

## What the standard agent got right on its own

Given a complete spec, the standard agent independently made every architecture decision the council had to actively catch in its own build:

- **No `@Data` on entities** — used `@Getter`, `@Setter`, `@Builder` only, avoiding the `StackOverflowError` from Hibernate 6 hashCode on lazy collections
- **Single bulk query in DashboardService** — `findAllDatesByUserId()` returning `Map<UUID, List<LocalDate>>`, streaks computed in memory — no N+1
- **JWT secret validated at startup** — `IllegalStateException` if `JWT_SECRET` is missing or shorter than 32 chars, no fallback
- **Ownership checks at the service layer** — every habit mutation and toggle verifies `habit.getUserId().equals(userId)` before proceeding
- **`completedToday` in HabitResponse** — included from the start, no separate network call needed
- **Index on `habits(user_id)`** — in schema.sql from the start

The important question is whether these decisions came from genuine independent reasoning or from a spec that had already been hardened by the council's prior work. The honest answer is: both. The spec included the bitmask requirement and the single-query requirement explicitly. The ownership check placement and the Lombok constraint came from the agent's own judgment.

---

## What the standard agent did NOT do

The standard agent produced zero review findings. That is not because the code is perfect — it is because no one was checking.

Things a council review would likely have flagged:

- **No input validation on date parameter in the toggle endpoint** — a malformed date string throws an uncaught `DateTimeParseException`, which falls through to the generic 500 handler instead of returning a clean 400
- **No rate limiting on `/auth/login`** — brute-force protection is absent
- **`@EqualsAndHashCode` on `HabitLogId`** — technically fine for an `@Embeddable`, but the Security Guard or Quality Critic would have flagged it for consistency review
- **CORS is locked to `localhost:5173`** — acceptable for development, but a real deployment needs an env-configurable origin

None of these are catastrophic. All of them are the kind of thing that gets caught in a structured review and missed in a solo build.

---

## Post-build: what broke during setup

The code compiled and the build completed, but getting it running revealed three issues the agent did not anticipate.

**1. Lombok annotation processor not configured (compile error)**

The first `mvn clean package` failed with dozens of "cannot find symbol" errors — `getId()`, `getName()`, `builder()` — across every class using Lombok. The root cause: the `maven-compiler-plugin` was not explicitly configured with an `annotationProcessorPaths` entry for Lombok.

Lombok is declared as a `provided` dependency, but newer Maven versions do not automatically wire `provided` dependencies as annotation processors. The fix required adding this to `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.34</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

This is a recurring gotcha — it was not caught during code generation because the agent writes files but does not compile them. The council's Quality Critic or Structure Warden would likely have flagged this, since it is a known Maven + Lombok setup requirement.

**2. JavalinJackson constructor signature changed (compile error)**

After fixing Lombok, a second compile error appeared: `no suitable constructor found for JavalinJackson(ObjectMapper)`. In Javalin 6 the constructor signature changed to `JavalinJackson(ObjectMapper, boolean)` — the boolean controls 5xx status code preference. The fix was a one-line change: `new JavalinJackson(mapper, false)`.

The agent used the Javalin 5 signature from training data. A council with a dedicated Architect reviewing API contracts against the actual library version would have caught this.

**3. psql not on PATH (schema setup)**

Running the schema required `psql`, which was not found — PostgreSQL's `bin` directory was not in the system PATH. On a fresh Windows installation this is common but easy to miss when writing setup instructions.

The project was deleted after these issues were documented. The app was not verified running end-to-end.

---

## Revised results including post-build issues

| Issue | Caught during build | Root cause |
|---|---|---|
| Lombok not processing | No — compile error after delivery | Missing `annotationProcessorPaths` in pom.xml |
| JavalinJavaJackson wrong constructor | No — compile error after delivery | Javalin 6 API change, agent used Javalin 5 signature |
| psql not on PATH | No — runtime setup failure | Windows PATH not configured, not checked in instructions |
| Missing `completedToday` in DTO | Never arose — spec included it | Spec pre-hardened by council |
| N+1 in DashboardService | Never arose — spec included fix | Spec pre-hardened by council |
| JWT secret fallback | Never arose — spec included fix | Spec pre-hardened by council |

The standard agent delivered 46 files in 8 minutes. Two of them had bugs that prevented compilation. The council's structured review — specifically the Architect reviewing library API contracts and the Quality Critic reviewing build configuration — would have caught both before delivery.

---

## Token breakdown: build vs. debug

The session JSONL file stores per-turn token counts, making it possible to split the numbers precisely by phase.

| Phase | Output tokens | Cache creation tokens | Total new tokens |
|---|---|---|---|
| Build phase (~8 min) | ~73,000 | ~165,000 | ~238,000 |
| Debug phase (post-build fixes) | 31,501 | 49,509 | 81,010 |
| **Full session** | **104,683** | **214,369** | **319,052** |

The debug phase — three rounds of error output, two file fixes, and a few setup exchanges — added **31,501 output tokens** and **49,509 tokens of new cached context**. That is roughly 43% of what the build itself produced in output, for work that delivered zero new features.

**The headline comparison:**

| | Standard Agent (build only) | Standard Agent (build + debug) | The Council |
|---|---|---|---|
| Output tokens | ~73,000 | 104,683 | ~119,050 |
| Wall-clock time | ~8 minutes | ~28 minutes (incl. debug) | ~35 minutes |
| Compile errors on delivery | 2 | — | 0 |
| Files that compiled first try | 44 / 46 | — | 33 / 33 |

When you include the debugging needed to get the code to actually compile, the standard agent's output token cost (~105k) converges with the council's (~119k). The speed advantage narrows from 4× to under 2×. And the council shipped code that compiled on the first attempt.

The cache read tokens (10M+ cumulative across 169 turns) are not directly comparable — they reflect the growing conversation context being re-read each turn and scale with session length, not with the amount of work done.

---

## The real finding

The speed difference is striking. The standard agent produced a complete, working full-stack application in 8 minutes. The Council took 35 minutes and used roughly 2-3x the tokens.

For a clearly scoped greenfield project with a hardened spec, the standard agent is dramatically faster and cheaper. The output is functionally equivalent.

The council's value shows up in two places the numbers do not capture:

1. **Spec hardening** — the standard agent was given a spec that was already corrected by the council. In a real first-pass build, without that prior work, the standard agent would have shipped with the N+1, the missing `completedToday`, and the hardcoded JWT fallback. The council catches those before they exist.

2. **The audit trail** — the council produces a log of every risk rating and recommendation. The standard agent produces only code. When something goes wrong in production, the council's log tells you exactly what was reviewed and what was not.

The standard agent is the right tool when the spec is tight and speed matters. The council is the right tool when you are defining the spec for the first time and correctness is the priority.

---

## Components

- **Claude Code** — CLI and agent runtime
- **Single agent** — no subagents, no structured review, one session from prompt to final file

---

### Pro tip of the day

The fastest way to use the council is not on every build — it is on the first build of a new system. Let the council harden the spec and catch the design-level issues. Then use that hardened spec to run standard agents on every feature after that. You get the council's correctness guarantees on the architecture and the standard agent's speed on execution.
