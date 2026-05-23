# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

This repo currently contains only the product spec — no code, no commits. The authoritative source is `docs/yozakura-spec.md` (Japanese). Read it before making architectural suggestions; it records not only decisions but the reasoning behind what was deliberately rejected.

When code lands, update this file with the actual build/test/run commands.

## What this product is

**yozakura (夜桜)** — a single-user iOS app to sustain 3x/week gym attendance over months. The user is ADHD-leaning, beginner-level, and historically drops off due to (1) post-workout soreness from poorly recovered programs and (2) unguarded calendars (drinking the night before gym). The app exists to solve those two structural problems, not to gamify consistency.

### Non-negotiable design constraints (the "north star")

These are explicit anti-patterns in the spec. Do not propose features that violate them:

- **No streaks, no Duolingo-style gamification.** The user's failure mode is soreness + social calendar, not motivational gaps. Streak UI amplifies guilt on broken weeks.
- **No blame language.** Recovery copy (LLM-generated) must propose comebacks, not scold.
- **Three tabs only** (Today / History / Graph). New screens are rejected by default — gym-time tapping must stay minimal.
- **The Record screen is the highest-priority UX surface** (longest cumulative dwell time, one open per set).
- Secretary / task-management / memo / HealthKit features are explicitly out of scope for this repo. If those become real, they go in sibling repos.

## Architectural lines that must not be crossed

### Rule engine owns numbers; LLM owns selection

The menu generator is a **hybrid**:

- **Go rule engine** computes weight, reps, sets, deload thresholds, Linear Progression (+2.5kg upper / +5kg lower on full clear; hold on miss; −10% deload on two consecutive misses), and A/B rotation.
- **LLM** picks which exercises and in what order, adapts to today's condition (好調/普通/疲れ), suggests substitutes when a machine is busy, and writes encouragement/recovery copy.

**The LLM must never produce or modify weight/rep numbers.** Hallucinated load is a direct injury risk. Any code path that lets LLM output flow into the numeric fields of a workout prescription is a bug.

### LLM is behind an interface

Provider selection (Claude / OpenAI / Gemini / local) is deliberately deferred. Code against a `MenuPlanner`-style Go interface. If the LLM call fails, the rule engine must still be able to emit a workable menu — don't make the LLM a hard dependency for "can the user train today."

### Auth is intentionally absent at the app layer

Single user, exposed only via Tailscale or Cloudflare Tunnel. `user_id` is a fixed constant; there is no users table. Google OAuth exists **only** for Calendar access, with the refresh token baked into a server Secret. Do not add app-level login flows "for future-proofing" — the spec explicitly rejects that.

### Menu generation timing

A k8s CronJob generates the day's menu at 07:00 so the home screen renders instantly with no LLM latency. Synchronous regeneration only fires on (a) user-toggled condition change or (b) machine-busy reroll during the session. Don't move generation onto the request path of the home screen.

## Tech stack

| Layer | Choice |
|---|---|
| iOS client | SwiftUI |
| API | Go |
| DB | PostgreSQL |
| Infra | k8s on a home server |
| IaC | Terraform |
| Batch | k8s CronJob |
| Exposure | Tailscale or Cloudflare Tunnel |
| External auth | Google OAuth 2.0 (Calendar only) |
| LLM | TBD — interface-abstracted |

k8s is chosen because the author already runs it at work, not as a learning exercise. Don't suggest simpler alternatives (Compose, single-binary on a VM) — the choice is informed.

## Phase order

Build strictly in this order. Earlier phases must be usable on their own.

1. **Phase 1 — foundation:** training log, growth graph, record-mode interrupt.
2. **Phase 2 — core:** recovery-aware menu suggestion, machine-busy reroll.
3. **Phase 3 — defense:** Google Calendar pre-day warning, recovery-week plans.

Phase 3 is intentionally last because the OAuth plumbing is the most likely thing to stall everything else.

## Open questions still owned by engineering

Section 7 of the spec enumerates unresolved design points (data model, LLM prompt structure, calendar heuristic for "drinking event", offline behavior in the gym, k8s/Terraform layout). When picking up work in those areas, treat the spec's section 7 as the live backlog.
