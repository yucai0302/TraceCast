# TraceCast

**English | [简体中文](./README.zh-CN.md)**

> Turn AI agent execution traces into replayable, danmu-annotated content — bringing back the "living internet" feeling to the age of working with AI.

## What is this

TraceCast takes the full execution process of a task you run with Claude Code (or another agent CLI) — planning, tool calls, errors and retries, final success — and turns it into a structured "event timeline" uploaded to the cloud. Viewers can replay this process like a Bilibili video and drop danmu (scrolling comments) at key moments.

The core idea isn't "live-streaming the raw token output." Instead, the raw execution trace is turned into a **narrative sequence of event nodes**: low-information segments like thinking/loading can be skipped or compressed, while high-information nodes — errors, decision turning points, successes — are preserved or even highlighted. The timeline reflects "information density," not physical time.

## Product Shape (Three Deliverables)

```
tracecast/
├── capture-agent/     # Local capture client: CLI tool hooking into Claude Code's execution flow, captures + normalizes + uploads events
├── backend/           # Cloud service: event storage, permission engine, danmu, auto-ranking, redaction scanning
├── web/                # Web frontend: login required, read-only consumption (browse + replay + danmu), project management (public/hidden permission)
├── sandbox-runtime/   # Sandbox service (Phase 2): hosts the project's final deliverable for viewers to experience directly
└── docs/               # This folder: README / CLAUDE.md / SPEC.md
```

- **capture-agent**: Installed on the developer's own machine, runs alongside Claude Code. Handles "capture" and "upload" only — no display logic.
- **backend**: Handles storage, permission decisions, auto-ranking, and danmu anchoring logic. This is the core of the product.
- **web**: Serves two kinds of people — viewers (can only watch and post danmu after logging in, no project-level control) and project owners (additionally get the "public/hidden" project management permission).
- **sandbox-runtime** (Phase 2): Continuously hosts a project's final runnable deliverable (e.g. a web app or small tool). **One public project maps to one sandbox instance** — all viewers see and interact with the same running product, with no per-visitor isolation. This closes the loop of "watch the process + experience the result," while avoiding runaway cost from spinning up a per-visitor instance.

## From "Watching the Process" to "Playing with the Result": The Full Loop

TraceCast's experience has two layers:
1. **Process layer**: Replay the event timeline of the agent's execution, with danmu anchored to event nodes, discussing "how was this step actually done."
2. **Outcome layer**: If the project produced a runnable deliverable (a web app, a small tool, etc.), viewers can experience the final product directly inside the project's sandbox, with danmu continuing around that experience, discussing "how good is the thing that got built."

These two layers are independent: the sandbox only reflects the project's **current, latest** deliverable state and doesn't need to correspond to every historical session (historical process replay remains a separate timeline feature). Not every project has a sandbox — only projects that produce a runnable deliverable enable this layer.

## Core Design Principles (see SPEC.md for details)

1. **One-time project-level authorization**: once a user makes a project public, all future sessions under that project default to public without needing per-session confirmation (similar to how a GitHub public repo works).
2. **Retroactive per-session revocation**: default-public doesn't mean uncontrollable — the owner can hide any individual session at any time, as a safety fallback.
3. **Event-driven, not character-driven**: the smallest unit for capture and display is an "event" (plan / tool call / error / success, etc.), not the raw token stream.
4. **Danmu is anchored to event nodes**, not to absolute timestamps — this guarantees that "what people reacted to at that step" always lines up correctly on replay.
5. **Sensitive-data redaction is a precondition, not optional** — local pre-screening plus a cloud-side second pass, with a preview/confirmation step before anything goes public.

## MVP Scope

Phase one only includes:
- Claude Code as the sole capture source
- Project-level toggle + session-level hiding
- Basic event type detection (plan / tool_call / error / retry / success)
- Simplified player (can skip thinking segments, danmu anchored to events)
- Email login + basic project homepage

Not doing yet (later iterations): multi-agent tool source support, real-time live-streaming mode, danmu engagement/incentive mechanics, automatic highlight-clip generation and sharing.

**Phase 2 (post-MVP)**:
- **Sandbox-hosted deliverables**: once a project produces a runnable result, deploy it to the project's dedicated sandbox so viewers can experience it directly (not just watch the process).
- **Commercialization quota**: free users can have at most 1 public project with a sandbox at a time; paid users can have multiple. This is a commercial-layer restriction only — it doesn't affect the core free experience of "process replay + danmu." The quota only gates "sandbox hosting."

## Getting Started with Development

See [SPEC.md](./SPEC.md) for the data model, event schema, and API design; see [CLAUDE.md](./CLAUDE.md) for development conventions and priority guidance for Claude Code.

> Note: SPEC.md and CLAUDE.md are currently maintained in Chinese only, as they are internal working documents for development rather than public-facing material. Let us know if English versions are needed.
