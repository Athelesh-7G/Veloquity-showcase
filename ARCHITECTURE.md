# Veloquity — Architecture

## What This Architecture Solves

At a system level, Veloquity solves a single problem: raw, unstructured feedback
arrives continuously and in volume, and nothing about it is inherently trustworthy,
deduplicated, or prioritized. A useful decision-support system has to take that raw
stream and, without human triage, turn it into a small number of well-supported,
explainable conclusions — while preserving a path back to the original evidence for
every conclusion it produces, and while continuing to hold up as the volume, source
mix, and domain change underneath it.

The architecture is a four-stage agentic pipeline, plus one cross-cutting agent that
keeps the whole system honest over time. Each stage is a bounded, independently
deployable unit with one job. No stage does more than one transformation of the data,
and no stage needs to understand what happens inside any other stage — only the
structure of what it receives and what it must hand off.

---

## Pipeline Diagram

```
                              ┌───────────────────────┐
                              │    Governance Agent     │
                              │  (cross-cutting concern) │
                              └───────────┬───────────┘
                                          │  monitors / maintains
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
 Feedback Sources        Ingestion            Evidence               Reasoning
  (reviews, tickets,  →    Agent      →     Intelligence   →   →       Agent      →   Decision Layer
   survey responses)                          Agent                                  (ranked, traceable
                                                 │                                    recommendations)
                                                 ▼
                                          Evidence Store
                                      (confidence-scored,
                                       clustered evidence)
```

Feedback flows strictly left to right through four stages. Governance does not sit in
that line — it runs independently and continuously, watching the evidence store and
the decisions produced, rather than gating any single request.

---

## Agent-by-Agent

### Ingestion Agent
**Role:** Take raw feedback in whatever shape it arrives and turn it into clean,
safe, non-duplicated records the rest of the pipeline can trust.
**AWS services:** AWS Lambda, Amazon S3.
**Input:** Raw feedback records from any connected source (reviews, tickets, survey
exports).
**Output:** Normalized, deduplicated, PII-safe records written to durable storage.

### Evidence Intelligence Agent
**Role:** Turn a pile of individually meaningless feedback records into a small
number of coherent, confidence-scored evidence groups.
**AWS services:** Amazon Bedrock (Titan Embed V2), Amazon RDS (PostgreSQL + pgvector,
HNSW index).
**Input:** Normalized feedback records from the Ingestion Agent.
**Output:** Evidence clusters, each carrying a confidence score describing how tightly
its members actually agree with one another.

### Reasoning Agent
**Role:** Reason across the current evidence and produce ranked, explainable,
source-linked recommendations.
**AWS services:** Amazon Bedrock (Nova Pro).
**Input:** Confidence-scored evidence from the Evidence Store.
**Output:** A ranked set of recommendations, each traceable back to the evidence — and
through it, back to the original feedback records — that produced it.

### Governance Agent
**Role:** Keep the evidence base and decision history honest as time passes —
independent of and running alongside the other three.
**AWS services:** AWS Lambda, Amazon EventBridge (scheduled trigger).
**Input:** The current state of the evidence store and the decision history.
**Output:** Staleness flags, signal promotions, and a permanent, append-only record of
every governance action taken.

---

## Key Architectural Decisions

**1. Serverless and event-driven, not a standing service.**
Every agent is a Lambda function, triggered by the event that makes it relevant
rather than running continuously. This buys two things at once: a failure in one
stage cannot cascade into another (each stage is isolated by the event boundary
between them), and cost scales to zero at idle — there's no fixed compute cost for a
system that runs in bursts rather than constantly.

**2. pgvector with HNSW for semantic clustering, not a dedicated vector database.**
Evidence clustering needs fast approximate nearest-neighbor search over embeddings,
but it also needs that search to sit next to ordinary relational data (timestamps,
sources, statuses) that the rest of the system already depends on. Running HNSW
inside PostgreSQL via pgvector avoids standing up and syncing a second, separate
vector store — one database, one source of truth, sub-10ms search at the scale this
system runs at.

**3. Amazon Nova Pro over Anthropic Claude, for reasoning.**
The reasoning stage was originally designed around Claude, but AWS accounts under the
AISPL billing entity (used for Indian AWS accounts) cannot access Anthropic models on
Bedrock. Nova Pro is first-party AWS and available universally regardless of billing
entity, which made it the model that could actually be relied on in every deployment
context — not just in the ones where the billing setup happened to allow it.

---

## Clustering Evolution

The Evidence Intelligence Agent's clustering approach has gone through more than one
iteration. It started as a simpler, greedy cosine-similarity grouping method (v0) —
straightforward to reason about and fast to ship. Later, a density-based clustering
approach was explored in a separate experimentation branch (v1) specifically to test
whether cluster boundaries could better reflect natural groupings in the data rather
than the more mechanical greedy-threshold approach. This matters for signal quality
because how evidence gets grouped directly determines what confidence score it
receives and, downstream, whether it ever reaches a human decision-maker at all — so
the grouping method itself is treated as something worth continuing to improve, not a
detail settled once and left alone.
