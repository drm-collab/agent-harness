# Continuity Framework

This document explains how to preserve conversational and task context across sessions.

For conceptual background, see:

- <https://use-ash.github.io/ash/>
- <https://use-ash.github.io/ash/memory-and-continuity/>

## The problem

An agent's context window ends when the session ends.

Without continuity:

- half-finished work disappears into chat history
- model handoffs lose important detail
- channel switches create parallel confusion
- the user has to restate what was already known

## The solution

Keep session state in files.

Use a persistent thread ledger that records:

- what work is active
- what work is closed
- what the current summary says
- what the next step should be
- which model or channel last touched the thread

The state file becomes the handoff point between sessions.

## Core components

Build continuity from three parts:

1. A persistent conversation state file
2. A session start process that reads and interprets that file
3. A handoff process that updates the file before context changes

## Relationship to memory

Continuity and memory are related but different.

Continuity stores the current shape of ongoing work.

Memory stores durable facts, rules, and references that should survive longer than a thread.

Use continuity for active thread tracking.
Use memory for lasting knowledge.

## State model

Use a state structure with at least:

- active threads
- closed threads
- global context
- session metadata

The exact field names may differ in your system. The meaning should not.

## Thread model

A thread is a unit of ongoing work that can survive:

- a new session
- a channel change
- a delegated subtask
- an interrupted workflow

Each active thread should be compact enough to reload quickly and clear enough that a different model can continue it.

## Summary rule

A thread summary should explain:

- what the thread is about
- what has already happened
- what matters now

Do not store full transcripts in the state file. Store decision-quality summaries.

## Next-step rule

Every active thread should have a next step.

If the thread has no next step, the agent will reopen the thread and still not know what to do.

The next step should be specific enough to act on immediately.

Bad:

```text
Continue later.
```

Good:

```text
Review the guardrails draft and verify each required file exists.
```

## Closure rule

A thread should move from active to closed when:

- the work is complete
- the user explicitly cancels it
- it is superseded by a newer thread

Closed threads still matter. They provide history and prevent the same work from being reopened accidentally.

## Session recovery

Thread state files handle planned handoffs. Session recovery handles the unplanned ones: context window full, server restart, crash, or compaction.

Recovery uses two layers that serve different purposes.

### Layer 1: Recovery briefing

A fast, cheap model generates a structured summary from the recent transcript. The summary follows a fixed format:

- **Task** — one line describing what was being worked on
- **Intent** — why the user is doing this and what they will do with the output. This is the most commonly lost signal in naive recovery systems. Without intent, the agent knows the task but not the goal.
- **Status** — in-progress, completed, blocked, or idle
- **Last Action** — what was happening right before the reset
- **Pending** — specific, actionable next steps. Not vague ("user to decide later") but concrete ("user needs to review the 606 session summaries and decide which to feed into redigest")
- **Key Decisions** — choices made during the conversation that should persist
- **Guidance** — tagged directives extracted from the conversation:
  - `[enforce]` — things that must be done a specific way
  - `[avoid]` — things that failed or were rejected
  - `[correction]` — user corrections to agent mistakes
  - `[decision]` — choices that should persist
  - `[pending]` — unresolved items requiring follow-up

The briefing generator must never include raw JSON, tool results, or API responses. Everything is distilled to plain English. A guidance item that says `[correction] {"tool_use_id": "toolu_01..."}` is worthless. One that says `[correction] price data must come from Tradier, not Alpaca` changes behavior.

Use a fallback chain for generation. The primary model should be fast and cheap (Grok, Haiku). If that fails, fall back to a local model (Ollama). If that also fails, the transcript tail alone still provides continuity.

### Layer 2: Prior session transcript

The raw tail of the conversation (~1500 characters), pulled directly from the database or transcript files. This is not a summary. It is the literal last exchange before the reset.

The transcript tail serves a different purpose than the briefing. The briefing gives topic orientation — what the project is about. The transcript tail gives moment-of-pause precision — exactly where the conversation stopped, what was just said, what was about to happen next.

Together, the two layers are complementary. The agent reads the briefing to understand the project. It reads the transcript tail to pick up the thread.

### Recovery injection

Both layers are assembled into a single recovery block and injected into the next message as a system-level context injection. The agent sees:

1. The structured briefing
2. The raw transcript tail in a fenced block
3. An instruction to continue from where the conversation left off

After injection, the recovery block is consumed (removed from the queue). It fires exactly once per reset event.

### When recovery fires

Recovery is triggered by three events:

- **Server restart** — the most recent active chat gets a recovery block generated during startup
- **Auto-compaction** — when cumulative tokens exceed a threshold, the system compacts and generates recovery
- **On-demand** — when a chat receives a message after an idle period with no active session

In all three cases, the same two-layer block is generated and injected.

## Success criteria

Continuity is working when:

- a new session can recover ongoing work from the ledger alone
- the user does not need to repeat recently active context
- a delegated subtask can return without losing the parent thread
- channel changes do not split the same work into multiple conflicting threads
- a server restart or compaction does not lose the current task, intent, or pending actions
- the recovery briefing contains intent and specific next steps, not vague summaries
- the transcript tail picks up the exact moment of pause, not a generic project description
