---
title: "PromptPetrol: What It Does and Why I Built It"
date: 2026-02-24
---
If AI is a performance machine, tokens are fuel.

`PromptPetrol` started from a simple problem: I wanted a clear, real-time way to see how fast I was burning that fuel across different models and providers.

Most dashboards show usage after the fact. I wanted something that feels live, local, and operational.

## The idea behind PromptPetrol

The ideation was straightforward:

- AI usage is getting harder to reason about in real time
- Costs can drift quietly across providers
- Limits (especially during coding-heavy sessions) can block momentum
- Existing tooling was either too cloud-heavy, too delayed, or too generic

So the concept became: **a local terminal dashboard for token usage, cost, limits, and trends**.

Fast feedback. Low friction. No heavy setup.

## What PromptPetrol does

At its core, PromptPetrol is a Rust TUI app that monitors and normalizes AI usage data.

### 1) Unified usage view
It ingests usage data and normalizes it into a common schema:

- input tokens
- output tokens
- estimated or provided cost
- provider and model context

This makes cross-provider comparison practical instead of messy.

### 2) Multi-provider adaptation
It supports provider-specific token formats and maps them into one model so the dashboard stays consistent.

### 3) Codex session import
It can import local Codex session JSONL data, including token snapshots and available rate-limit events.

That means you can monitor active coding usage without manual copy/paste workflows.

### 4) Budget and limit awareness
PromptPetrol highlights budget pressure and limit usage so you catch risk early:

- budget burn signals
- token load signals
- activity and freshness indicators
- Codex limit visibility when present

### 5) Live, local workflow
Everything is designed for terminal-native operation:

- fast refresh
- keyboard-driven control
- no dependency on a remote analytics stack for day-to-day monitoring

## Why this matters

When AI workflows scale, visibility becomes a product requirement, not a nice-to-have.

PromptPetrol is meant to answer practical questions quickly:

- Which provider is consuming most tokens right now?
- Is cost trending into risky territory?
- Are we close to short-window or weekly limits?
- Is imported session data fresh and reliable?

The goal is not vanity metrics. The goal is operational clarity.

## What I care about in this project

- **Speed:** data should be visible quickly
- **Clarity:** one common language across providers
- **Pragmatism:** useful signals over decorative charts
- **Extensibility:** easy to improve parser quality, diagnostics, and UI over time

## Closing

PromptPetrol is a small idea with a practical focus: make AI usage observable while work is happening, not hours later.

If you build with multiple models daily, you already know the pain this solves.

And if you have ever asked, “Where did all my tokens go?”, this is exactly what PromptPetrol is for.
