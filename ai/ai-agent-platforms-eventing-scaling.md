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

**Status:** Very early development (v0.0.0, May 2026). Apache 2.0. 478 stars.

**Links:**
- Repository: https://github.com/agent-substrate/substrate
- CNCF Slack: #substrate-users, #substrate-dev

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

**Status:** Early development (v0.1.0, May 2026). Apache 2.0. 1.6k stars.

**Links:**
- Repository: https://github.com/google/ax

### KEDA (Kubernetes Event-Driven Autoscaling)

CNCF Graduated project. The most natural fit for event-triggered agent scaling. Extends the Kubernetes HPA with 60+ built-in scalers and drives workloads from 0 to 1 on an event via activation thresholds. KEDA feeds the HPA with external metrics, extending it beyond CPU and memory to any event source.

Three core components:
- keda-operator: manages ScaledObjects, handles zero-to-one scaling
- keda-metrics-apiserver: serves external metrics to the HPA
- keda-admission-webhooks: validates resources at apply time

Supported event sources include: Apache Kafka, RabbitMQ, AWS SQS, Azure Service Bus, GCP Pub/Sub, Redis, PostgreSQL, Prometheus, NATS, cron, HTTP, and 50+ more. Custom external scalers via gRPC.

- Fire-and-forget pattern: agent dispatches task, suspends, KEDA watches completion queue
- Fail-safe: holds last known scale on scaler failure rather than scaling to zero
- Scale-to-zero not supported with CPU/Memory triggers (no running pods = no metrics)

**Status:** CNCF Graduated. v2.20.0 (June 2026). 10.3k stars.

**Links:**
- Repository: https://github.com/kedacore/keda
- Documentation: https://keda.sh
- Concepts: https://keda.sh/docs/latest/concepts/
- Scalers list: https://keda.sh/docs/latest/scalers/
- Samples: https://github.com/kedacore/samples
- Slack: #KEDA on Kubernetes Slack

### Knative Eventing and Serving

**Knative Eventing** provides APIs for event-driven architecture on Kubernetes, routing events from producers (sources) to consumers (sinks) using loosely coupled components. All events conform to the CloudEvents specification.

Core components:
- **Event Sources**: ApiServerSource, Apache Kafka, PingSource, IntegrationSource (AWS S3/SQS/DDB), RabbitMQ, Redis Streams, custom via SinkBinding
- **Brokers**: Channel-based, Apache Kafka, RabbitMQ — receive events and distribute to subscribers
- **Triggers**: Filter and route events to consumers based on CloudEvent attributes
- **Event Sinks**: JobSink, Kafka Sink, IntegrationSink (AWS S3/SNS/SQS)
- **Flows**: Sequences and Parallel constructs for complex event processing

**Knative Serving** builds on Kubernetes for deploying serverless containers with automatic scale-to-zero. Request-driven compute with an activator that buffers traffic during scale-from-zero.

Hybrid pattern: Knative Serving for synchronous ingress, KEDA for asynchronous queue-driven work.

**Status:** v1.22.1 (June 2026). Eventing: 1.5k stars. Serving: 6.1k stars.

**Links:**
- Eventing repo: https://github.com/knative/eventing
- Serving repo: https://github.com/knative/serving
- Eventing docs: https://knative.dev/docs/eventing/
- Serving docs: https://knative.dev/docs/serving/
- CloudEvents spec: https://cloudevents.io
- Slack: #eventing on cloud-native.slack.com

### Kagenti (High-Density AI Agents with Eventing)

Combines event-driven triggers with state-preserving suspend/resume (CRIU or microVM snapshots). Pairs KEDA-style event triggers with checkpoint/restore rather than cold-start.

- Event trigger: KEDA scaler on task-completion queue
- State-preserving resume: CRIU (zeropod-style) or Firecracker snapshot-restore
- Density: Copy-on-Write from shared snapshots (ZeroBoot pattern)
- Shared memory: Wiki-based durable long-term memory across agents

### zeropod

A containerd shim (Kubernetes runtime) combining CRIU + eBPF for pause/resume:
- After configurable idle period with no TCP connections, CRIU checkpoints container to disk
- eBPF redirector sends incoming packets to a userspace activator while container is suspended
- On first incoming connection, container restores — "tens to a few hundred milliseconds"
- Once restored, eBPF redirect disabled — subsequent connections go directly to app with no proxy overhead
- In-place resource scaling: adjusts resource requests when scaled down
- Experimental live migration between nodes without starting up

**Status:** v0.12.0 (May 2026). Apache 2.0. 895 stars.

**Links:**
- Repository: https://github.com/ctrox/zeropod
- Docs: https://github.com/ctrox/zeropod/blob/main/docs/README.md
- Migration docs: https://github.com/ctrox/zeropod/blob/main/docs/experimental/migration.md

### Durable Execution Platforms (Semantic Eventing)

Avoids CRIU by persisting semantic state — workflows capture state at every step and resume exactly where they left off after failures.

**Temporal** — Durable execution platform where workflows automatically handle intermittent failures and retry failed operations. Originated as a fork of Uber's Cadence. Supports microservice orchestration, distributed cron, and workflow management.
- Status: v1.31.0 (April 2026). MIT. 20.8k stars.
- Links: https://github.com/temporalio/temporal | https://docs.temporal.io

**DBOS** — Lightweight durable workflow library using PostgreSQL as persistence. Checkpoints workflow state in Postgres; if program fails, workflows automatically resume from last completed step. Supports durable queues, exactly-once event processing, and durable scheduling.
- Status: v2.23.0 (June 2026). MIT. 1.4k stars.
- Links: https://github.com/dbos-inc/dbos-transact-py | https://docs.dbos.dev

**LangGraph** — Low-level orchestration framework for building stateful agents. Durable execution with automatic resumption from exactly where execution left off. Supports human-in-the-loop via interrupts, short-term working memory, and long-term persistent memory across sessions. Inspired by Google Pregel and Apache Beam.
- Status: v1.2.4 (June 2026). MIT. 34k stars.
- Links: https://github.com/langchain-ai/langgraph | https://docs.langchain.com/oss/python/langgraph/durable-execution

**Firecracker** — Open-source microVM technology by AWS (powers Lambda and Fargate). Lightweight VMs combining hardware virtualization security with container-like speed. Cold boot 125-200 ms; snapshot-restore 5-30 ms.
- Status: v1.16.0 (June 2026). Apache 2.0. 34.8k stars.
- Links: https://github.com/firecracker-microvm/firecracker | https://firecracker-microvm.io

## Platform Comparison

### Feature Matrix

| Feature | Agent Substrate | Google AX | KEDA + zeropod/CRIU | Kagenti |
|---------|----------------|-----------|---------------------|---------|
| State preservation | Full (RAM + FS via gVisor) | Semantic (event log) | Full (CRIU process) | Full (CRIU/microVM) |
| Checkpoint mechanism | gVisor runsc | Event log replay | CRIU | CRIU or Firecracker |
| Oversubscription | 30x+ demonstrated | N/A (orchestration layer) | Depends on workload | Target: high density |
| Resume latency | Sub-second | Instant (event replay) | ~200 ms (zeropod) | Sub-second target |
| Event-driven activation | Traffic-driven (Envoy/DNS) | Event log + resumable streams | KEDA scalers (60+ sources) | KEDA scalers |
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
- Tied to gVisor — limits environments (no GPU passthrough)
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
- Mature event trigger ecosystem (60+ scalers, production-proven)
- KEDA is CNCF Graduated — strong community, wide adoption (10.3k stars)
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

Kagenti occupies a similar space to Substrate+AX but with different choices: CRIU instead of gVisor, wiki instead of event log, and KEDA as the explicit event trigger rather than traffic-driven activation.

## Design Principles

1. Release live connections before checkpoint — dispatch external tasks fire-and-forget
2. Density ceiling is storage and restore latency, not CPU
3. Separate ephemeral state (checkpoint) from durable state (wiki/event log)
4. Agents should not hold local GPU/model state — call inference endpoints instead
5. The event trigger layer and the state-preservation layer are independent choices

## Sources and References

| Project | Repository | Documentation | Stars |
|---------|-----------|---------------|-------|
| Agent Substrate | https://github.com/agent-substrate/substrate | CNCF Slack #substrate-users | 478 |
| Google AX | https://github.com/google/ax | In-repo docs | 1.6k |
| KEDA | https://github.com/kedacore/keda | https://keda.sh | 10.3k |
| Knative Eventing | https://github.com/knative/eventing | https://knative.dev/docs/eventing/ | 1.5k |
| Knative Serving | https://github.com/knative/serving | https://knative.dev/docs/serving/ | 6.1k |
| zeropod | https://github.com/ctrox/zeropod | In-repo docs | 895 |
| Temporal | https://github.com/temporalio/temporal | https://docs.temporal.io | 20.8k |
| DBOS | https://github.com/dbos-inc/dbos-transact-py | https://docs.dbos.dev | 1.4k |
| LangGraph | https://github.com/langchain-ai/langgraph | https://docs.langchain.com/oss/python/langgraph/overview | 34k |
| Firecracker | https://github.com/firecracker-microvm/firecracker | https://firecracker-microvm.io | 34.8k |
| CloudEvents | https://github.com/cloudevents/spec | https://cloudevents.io | - |

*Last updated: June 2026. Star counts and versions are point-in-time snapshots.*
