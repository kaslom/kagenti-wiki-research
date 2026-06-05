---
tags: [eventing, scaling, architecture, kubernetes, agents]
---
# AI Agent Platforms That Use Eventing for Scaling

## Overview

Event-driven architectures are emerging as the primary mechanism for scaling AI agent workloads on Kubernetes. Rather than keeping agents running (and burning resources) while waiting for external tasks, eventing allows agents to be suspended at near-zero cost and resumed only when relevant events arrive — task completions, user messages, or upstream data changes.

## Key Platforms and Approaches

### KEDA (Kubernetes Event-Driven Autoscaling)

KEDA is the most natural fit for event-triggered agent scaling. It extends the Kubernetes HPA with 70+ scalers (Kafka, RabbitMQ, SQS, Redis, Prometheus, HTTP, cron) and can drive workloads from 0 to 1 on an event via activation thresholds. For AI agents, the pattern is fire-and-forget: the agent dispatches a long-running task, suspends, and KEDA watches the completion queue to trigger resume.

- Supports multiple event sources simultaneously (takes max desired count)
- Fail-safe: holds last known scale on scaler failure rather than scaling to zero
- HTTP add-on queues inbound requests during scale-from-zero

### Knative Eventing and Serving

Knative provides two complementary capabilities:
- **Knative Serving**: Request/concurrency-driven scale-to-zero with an activator that buffers cold traffic
- **Knative Eventing**: CloudEvents-based event routing, brokers, triggers, and subscriptions

The common hybrid pattern uses Knative for synchronous ingress paths and KEDA for asynchronous queue-driven background work.

### Kagenti (High-Density AI Agents with Eventing)

Kagenti combines event-driven triggers with state-preserving suspend/resume (CRIU or microVM snapshots) to achieve high-density agent scheduling. The key innovation is pairing KEDA-style event triggers with checkpoint/restore rather than cold-start, preserving the agent's in-memory execution state across suspensions.

Architecture:
- Event trigger: KEDA scaler on task-completion queue
- State-preserving resume: CRIU (zeropod-style) or Firecracker snapshot-restore
- Density: Copy-on-Write from shared snapshots (ZeroBoot pattern)
- Shared memory: Wiki-based durable long-term memory across agents

### zeropod

A containerd shim combining CRIU + eBPF for pause/resume on Kubernetes:
- eBPF tracks TCP activity; after idle timeout, CRIU checkpoints the container
- Restores on first incoming connection (~185 ms checkpoint, ~200 ms restore as of v0.12.0)
- Supports experimental live migration between nodes
- Network-activity triggered (not event-queue aware), but proves the CRIU loop works

### Durable Execution Platforms (Semantic Eventing)

An alternative approach that avoids CRIU entirely by persisting semantic state:
- **Temporal**: Workflows capture state at every step; resume after failure; native ADK/OpenAI integrations
- **DBOS**: Durability via Postgres and function decorators; auto-persists and resumes
- **LangGraph**: State as checkpoints in threads; interrupt primitive pauses execution indefinitely and resumes on event

## The Gap

No single platform ships the full combination: event-driven trigger + CRIU/microVM live restore + microVM isolation + CoW density optimization + shared semantic memory, packaged as a Kubernetes scheduler for AI agents. Each piece is proven individually; the contribution is integration and density tuning.

## Design Principles

1. Release live connections before checkpoint — dispatch external tasks fire-and-forget
2. Density ceiling is storage and restore latency, not CPU
3. Separate ephemeral state (CRIU) from durable state (wiki/journal)
4. Agents should not hold local GPU/model state — call inference endpoints instead