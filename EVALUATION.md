# Veloquity — Evaluation

## Overview

The goal of Veloquity's validation work was never just "does the pipeline run
end-to-end without crashing." The real question was whether the system reasons
*correctly* — whether it groups related feedback the way a careful human would,
whether its confidence scores actually track how coherent a group of evidence is,
whether the same input reliably produces the same output, and whether every
recommendation it surfaces can be traced back to real, specific source records rather
than a plausible-sounding generated summary.

Three things were validated as a result: correctness of the underlying logic (via an
automated test suite), real-world cost and latency (via full pipeline runs), and
domain generality (via two structurally unrelated real datasets run through the exact
same, unmodified pipeline).

---

## Test Suite

**158 automated tests, 100% passing, 0.72 seconds total runtime.**

The suite runs fully mocked — zero live AWS or database calls — which is why it
completes in under a second and can be run safely and repeatedly in any environment.
Tests are organized around the categories of logic that actually matter for
correctness, not just code coverage for its own sake:

- **Deduplication and normalization** — confirming that near-identical feedback
  records are correctly identified and collapsed, and that malformed or partial input
  doesn't silently corrupt downstream data.
- **Clustering stability** — confirming that evidence grouping behaves consistently
  and that confidence scoring responds correctly to tight versus loose clusters.
- **Priority and routing logic** — confirming that evidence is routed to the correct
  next step (rejected, escalated for validation, or accepted) based on its confidence,
  and that ranking behaves predictably as inputs change.
- **Failure isolation** — confirming that a failure in one stage (a malformed record,
  an unreachable dependency) is contained and doesn't silently corrupt or block
  unrelated evidence moving through the rest of the pipeline.

---

## Cost Benchmark

Measured on a full run across the combined validation dataset (857 items):

| Component | Cost |
|---|---|
| Embeddings (Evidence Intelligence) | $0.016 |
| Reasoning (Nova Pro) | $0.013 |
| **Total — full run** | **$0.029** |
| Total — cached embeddings (re-run, unchanged corpus) | ~$0.013 |

Keeping the LLM confined to a single stage of the pipeline, with everything else
handled deterministically, is what keeps the cost this low even at full volume.

---

## Latency Breakdown

| Stage | Latency |
|---|---|
| Ingestion | 18s |
| Evidence Intelligence | 34s |
| Reasoning | 27s |
| Governance | 12s |
| **Total — end to end** | **91s** |

---

## Domain-Agnostic Validation

The strongest validation claim Veloquity makes is that the pipeline required zero
domain-specific code changes to move from one domain to a completely unrelated one.

| Domain | Volume | Example Evidence Identified |
|---|---|---|
| SaaS product feedback (app reviews + support tickets) | 547 items | Crash patterns tied to specific user actions, regressions following a recent release |
| Healthcare patient experience (portal + survey data) | 310 items | Extended emergency department wait times, recurring appointment-booking failures |

Same embeddings model, same clustering approach, same confidence scoring, same
reasoning agent — applied to feedback about software crashes and feedback about
emergency room wait times, with the system correctly identifying coherent, distinct
evidence groups in both, without being told anything in advance about either domain.

---

## Real-World Failure Modes Resolved

These were genuine production issues encountered during the build, not hypothetical
edge cases — evidence of an engineered system that survived contact with real AWS
infrastructure, not a prototype that only ever ran in a clean, ideal environment.

**Model access restricted by billing entity.** Accounts under AISPL billing (used for
Indian AWS accounts) could not access Anthropic models on Bedrock. Resolved by moving
the reasoning stage to Amazon Nova Pro, which is available universally regardless of
billing entity.

**Private networking blocked model access.** A Lambda function deployed inside a VPC
could not reach the Bedrock API without a NAT gateway or VPC endpoint in place.
Resolved by adjusting the network configuration so the relevant stage could reach
Bedrock directly.

**Deployment configuration mismatch.** A Lambda's configured handler path didn't
match where its actual code lived after a deployment, causing invocations to fail.
Resolved by correcting the deployment configuration to point at the right entry
point.

**Missing runtime dependency.** File upload handling failed silently in production
because a required server-side dependency wasn't declared. Resolved by identifying
the missing dependency and adding it to the deployment's requirements.

---

## Confidence Routing

Every evidence cluster is scored for confidence, and that score determines what
happens to it next — not every cluster is worth spending an expensive reasoning call
on, and not every cluster deserves to be silently discarded either.

| Confidence Score | Routing Outcome |
|---|---|
| Below 0.40 | Automatically rejected — no reasoning cost spent on weak evidence |
| 0.40 – 0.60 | Routed for LLM-based validation before proceeding |
| Above 0.60 | Automatically accepted into the evidence base |

This three-tier routing is what keeps the pipeline's cost low without sacrificing
quality at the top end: only evidence that's either strong enough to trust outright,
or ambiguous enough to be worth a second look, ever reaches the more expensive stages
of the system.
