---
tags: [eventing, scaling, architecture, kubernetes, agents]
---
# AI Agent Platforms That Use Eventing for Scaling

## Overview

Event-driven architectures are emerging as the primary mechanism for scaling AI agent workloads on Kubernetes. Rather than keeping agents running (and burning resources) while waiting for external tasks, eventing allows agents to be suspended at near-zero cost and resumed only when relevant events arrive — task completions, user messages, or upstream data changes.

## Key Platforms and Approaches

### Agent Substrate (google/agent-substrate)

A Kubernetes-native system that maps a larger set of "actors" (agents) onto a smaller set of ready "workers" (Pods), achieving 30x+ oversubscription. It exploits the observation that agent workloads are idle most of the time. Uses gVisor runsc checkpoint/restore (not CRIU) for full-state snapshots including volatile RAM and filesystem.

**Architecture:**
- ateapi: gRPC control plane for actor/worker lifecycle
- atelet: Node-level DaemonSet coordinating snapshots and state transfers
- atecontroller: Reconciles WorkerPool and ActorTemplate CRDs
- atenet: DNS + Envoy routing + proxy sidecars for traffic-driven activation
- ateom-gvisor: Executes runsc checkpoint/restore inside sandboxed pods

**Key features:**
- Sub-second suspend/resume ("instant session teleport")
- Zero-idle self-suspension — actors suspend themselves when idle
- Full process memory + filesystem preserved across hibernation
- Framework-agnostic: hosts ADK, LangChain, Claude Code, MCP servers
- Live migration between nodes without cold restart
- Takes Kubernetes control plane out of critical path for lower latency

**Status:** Very early development (v0.0.0, May 2026). Apache 2.0.

### Google AX (Agent eXecutor)

A distributed agent runtime that coordinates agentic loops with durable event logging and resumption. Designed to run on Agent Substrate for production Kubernetes deployments. Provides the orchestration layer above Substrate's compute layer.

**Architecture:**
- Single-writer controller pattern for consistent state
- Event log (SQLite-backed) for durable execution and automatic recovery
- Resumable gRPC streams between client, controller, and remote agents
- Registry of remote agents, MCP tools, and skills

**Key features:**
- Event log with sequence numbers — clients catch up after disconnects
- Execution resumption after failures or interruptions
- Forking — branch from any checkpoint into new conversations
- Model-agnostic and framework-agnostic (ADK, A2A, custom agents)
- Built-in Gemini agent, MCP tool integration, A2A bridge
- Central auditing of all user and agent calls

**Status:** Early development (v0.1.0, May 2026). Apache 2.0.

### KEDA (Kubernetes Event-Driven Autoscaling)

The most natural fit for event-triggered agent scaling. Extends the Kubernetes HPA with 70+ scalers (Kafka, RabbitMQ, SQS, Redis, Prometheus, HTTP, cron) and drives workloads from 0 to 1 on an event via activation thresholds.

- Supports multiple event sources simultaneously (takes max desired count)
- Fail-safe: holds last known scale on scaler failure rather than scaling to zero
- HTTP add-on queues inbound requests during scale-from-zero
- Fire-and-forget pattern: agent dispatches task, suspends, KEDA watches completion queue

### Knative Eventing and Serving

Knative provides two complementary capabilities:
- **Knative Serving**: Request/concurrency-driven scale-to-zero with activator buffering
- **Knative Eventing**: CloudEvents-based event routing, brokers, triggers, and subscriptions

Hybrid pattern: Knative for synchronous ingress, KEDA for asynchronous queue-driven work.

### Kagenti (High-Density AI Agents with Eventing)

Combines event-driven triggers with state-preserving suspend/resume (CRIU or microVM snapshots). Pairs KEDA-style event triggers with checkpoint/restore rather than cold-start.

- Event trigger: KEDA scaler on task-completion queue
- State-preserving resume: CRIU (zeropod-style) or Firecracker snapshot-restore
- Density: Copy-on-Write from shared snapshots (ZeroBoot pattern)
- Shared memory: Wiki-based durable long-term memory across agents

### zeropod

A containerd shim combining CRIU + eBPF for pause/resume on Kubernetes:
- eBPF tracks TCP activity; after idle timeout, CRIU checkpoints the container
- Restores on first incoming connection (~185 ms checkpoint, ~200 ms restore)
- Supports experimental live migration between nodes
- Network-activity triggered (not event-queue aware)

### Durable Execution Platforms (Semantic Eventing)

Avoids CRIU by persisting semantic state:
- **Temporal**: Workflows capture state at every step; resume after failure
- **DBOS**: Durability via Postgres and function decorators
- **LangGraph**: State as checkpoints in threads; interrupt primitive pauses indefinitely

## Platform Comparison

### Feature Matrix

| Feature | Agent Substrate | Google AX | KEDA + zeropod/CRIU | Kagenti |
|---------|----------------|-----------|---------------------|---------|
| State preservation | Full (RAM + FS via gVisor) | Semantic (event log) | Full (CRIU process) | Full (CRIU/microVM) |
| Checkpoint mechanism | gVisor runsc | Event log replay | CRIU | CRIU or Firecracker |
| Oversubscription | 30x+ demonstrated | N/A (orchestration layer) | Depends on workload | Target: high density |
| Resume latency | Sub-second | Instant (event replay) | ~200 ms (zeropod) | Sub-second target |
| Event-driven activation | Traffic-driven (Envoy/DNS) | Event log + resumable streams | KEDA scalers (70+ sources) | KEDA scalers |
| Isolation | gVisor (kernel-level) | Process isolation | Container namespaces | microVM or gVisor |
| Framework support | ADK, LangChain, Claude, MCP | ADK, A2A, MCP, custom | Any containerized workload | Any containerized workload |
| Kubernetes-native | Yes (CRDs, DaemonSets) | Yes (runs on Substrate) | Yes (KEDA operator) | Yes (operator-based) |
| Cross-node migration | Yes (state transfer) | N/A | Limited (zeropod experimental) | Planned |
| Shared memory/wiki | Not built-in | Event log as shared state | Not built-in | Wiki-based shared memory |

### Pros and Cons for AI Agent Platforms

#### Agent Substrate

**Pros:**
- Highest demonstrated density (30x+ oversubscription)
- Full transparent state preservation — no agent code changes needed
- gVisor provides strong kernel-level isolation for untrusted agent code
- Sub-second restore removes cold-start penalty entirely
- Framework-agnostic: runs any OCI container
- Takes K8s control plane off critical path for latency
- Zero-idle self-suspension is automatic

**Cons:**
- Very early (v0.0.0), APIs unstable, not production-ready
- Tied to gVisor — limits environments where it runs (no GPU passthrough)
- GCP-centric tooling (GKE, GCS) though Kind supported locally
- No built-in event-queue integration (traffic-driven only)
- No semantic memory layer — just raw process state
- Checkpoint images grow with agent memory usage
- Not an officially supported Google product

#### Google AX

**Pros:**
- Clean event-driven architecture with durable event log
- Resumption protocol handles disconnects and failures gracefully
- Forking enables branching agent executions from checkpoints
- Central auditing and control of all agent actions
- Model-agnostic and framework-agnostic
- Native integration with Agent Substrate for compute density
- MCP and A2A protocol support built-in
- Semantic state (event log) is portable and debuggable

**Cons:**
- Very early (v0.1.0), external contributions paused
- Single-writer controller is a potential bottleneck at scale
- SQLite event log may not scale for very large deployments
- Requires Agent Substrate for production density (not standalone)
- Gemini-first built-in agent (other models need custom integration)
- Resumption of subagents still on roadmap (not shipped)
- Not an officially supported Google product

#### KEDA + CRIU/zeropod (Composable Approach)

**Pros:**
- Mature event trigger ecosystem (70+ scalers, production-proven)
- KEDA is CNCF graduated — strong community, wide adoption
- Decoupled: choose any checkpoint mechanism independently
- Works with any event source (Kafka, SQS, Redis, webhooks, cron)
- zeropod proves CRIU loop works on stock Kubernetes
- No vendor lock-in, fully open-source

**Cons:**
- Assembly required — no integrated solution exists
- CRIU has kernel/library version coupling and fragility
- zeropod is network-triggered only, not event-queue aware
- No built-in agent framework awareness
- CRIU struggles with open file descriptors and network connections
- Cross-node migration is experimental and unreliable
- No shared memory or semantic state built-in

#### Kagenti

**Pros:**
- Combines event triggers (KEDA) with state preservation (CRIU/microVM)
- Wiki-based shared memory provides semantic layer across agents
- CoW density optimization (ZeroBoot pattern) for memory efficiency
- Designed specifically for the agent scheduling use case
- Hybrid approach: CRIU as optimization, wiki as correctness layer

**Cons:**
- Research project, not yet production-ready
- Novel composition — less community validation
- Depends on multiple moving parts (KEDA + CRIU + wiki)
- CRIU fragility inherited unless microVM path chosen

## Architectural Relationships

```
Google AX (orchestration, event log, resumption)
    |
    v
Agent Substrate (compute density, suspend/resume, gVisor isolation)
    |
    v
Kubernetes (infrastructure, pods, networking)
    |
    v
KEDA / Knative (event-driven scaling triggers)
```

AX and Substrate are complementary: Substrate provides the compute layer (how to run and suspend agents efficiently), while AX provides the orchestration layer (how to coordinate, audit, and recover agent executions). KEDA/Knative operate at the infrastructure trigger level, below both.

Kagenti's architecture occupies a similar space to the Substrate+AX combination but with different choices: CRIU instead of gVisor, wiki instead of event log, and KEDA as the explicit event trigger rather than traffic-driven activation.

## Design Principles

1. Release live connections before checkpoint — dispatch external tasks fire-and-forget
2. Density ceiling is storage and restore latency, not CPU
3. Separate ephemeral state (checkpoint) from durable state (wiki/event log)
4. Agents should not hold local GPU/model state — call inference endpoints instead
5. The event trigger layer and the state-preservation layer are independent choices — compose them based on workload needs