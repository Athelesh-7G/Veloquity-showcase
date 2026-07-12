<div align="center">

![Veloquity Banner](https://capsule-render.vercel.app/api?type=waving&color=0:0f0c29,50:302b63,100:24243e&height=200&section=header&text=VELOQUITY&fontSize=70&fontColor=ffffff&animation=fadeIn&fontAlignY=35&desc=Agentic%20Evidence%20Intelligence%20%E2%80%94%20Raw%20Feedback%20to%20Evidence-Driven%20Decisions&descAlignY=55&descSize=18)

[![AWS APJC Regional Champion](https://img.shields.io/badge/AWS%2010%2C000%20AIdeas-APJC%20Regional%20Champion%20%F0%9F%8F%86-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](#recognition)
[![Demo Video](https://img.shields.io/badge/Demo%20Video-Watch%20on%20YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtu.be/wEG5jTQxlJ4?si=l1tH72icmjTMdh_H)
[![AWS Builder Center Article](https://img.shields.io/badge/Full%20Technical%20Write--up-AWS%20Builder%20Center-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://builder.aws.com/content/3AzrKpJbhJwEP6EZbm87vdxufgi/aideas-finalist-veloquity-the-agentic-evidence-intelligent-platform-turning-raw-feedback-into-evidence-driven-decisions)
[![Live Platform](https://img.shields.io/badge/Live%20Platform-veloquity1.vercel.app-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://veloquity1.vercel.app)

<br/>

![Tests](https://img.shields.io/badge/tests-158%20passing-brightgreen?style=flat-square)
![Cost](https://img.shields.io/badge/pipeline%20cost-%240.029%2Frun-blue?style=flat-square)
![Runtime](https://img.shields.io/badge/end--to--end-91s-informational?style=flat-square)
![Agents](https://img.shields.io/badge/agentic%20pipeline-4%20stages-orange?style=flat-square)
![Domains](https://img.shields.io/badge/validated%20on-2%20domains-9cf?style=flat-square)

</div>

---

## About This Repository

This is the public documentation showcase for Veloquity. The source code is maintained in a private repository. What's here is a complete record of what was designed, how it was validated, and what it actually produced.

| Document | What It Covers |
|---|---|
| This README | System overview, architecture summary, validation results, recognition |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Agent-by-agent design, AWS service roles, key architectural decisions, clustering evolution |
| [EVALUATION.md](EVALUATION.md) | Test suite, cost benchmarks, latency breakdown, domain-agnostic validation, real production failure modes |

The full technical write-up is on the [AWS Builder Center](https://builder.aws.com/content/3AzrKpJbhJwEP6EZbm87vdxufgi/aideas-finalist-veloquity-the-agentic-evidence-intelligent-platform-turning-raw-feedback-into-evidence-driven-decisions) and the system is live at [veloquity1.vercel.app](https://veloquity1.vercel.app). Source code available on request.

---

## The Problem

Most product and operations teams don't fail because they lack data. They fail because raw feedback arrives as noise — thousands of disconnected reviews, tickets, and survey responses — and by the time a human has read enough of it to spot a real pattern, the signal has already decayed or been acted on by instinct instead of evidence. The gap was never data collection. It's the missing layer between raw information and a decision someone can actually stand behind.

Veloquity is that layer: a fully serverless, agentic pipeline that turns raw, unstructured feedback into prioritized, evidence-backed decisions — with every single recommendation traceable back to the original source item that produced it.

---

## What Was Built

Four purpose-built agents, each responsible for one clean transformation of the data, chained into a single pipeline:

```
Raw Feedback (reviews, tickets, survey responses)
        │
        ▼
┌───────────────────────┐
│   Ingestion Agent      │   Cleans, protects, and deduplicates raw input
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│ Evidence Intelligence  │   Groups related feedback into confidence-scored evidence
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│   Reasoning Agent      │   Reasons over evidence into ranked, explainable actions
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│   Governance Agent     │   Keeps the evidence base honest and current over time
└───────────────────────┘
        │
        ▼
  Evidence-Backed Decision
```

| Stage | Responsibility | Core AWS Technology |
|---|---|---|
| Ingestion | Normalize, protect, and deduplicate raw feedback | AWS Lambda, Amazon S3 |
| Evidence Intelligence | Embed and cluster feedback into confidence-scored evidence | Amazon Titan Embed V2, Amazon RDS (pgvector, HNSW) |
| Reasoning | Reason across evidence into ranked, source-linked recommendations | Amazon Bedrock (Nova Pro) |
| Governance | Detect staleness, promote signals, maintain an audit trail | AWS Lambda, Amazon EventBridge |

Every stage hands off a structured, well-defined output to the next — no stage guesses what it received, and no stage does more than one job.

---

## How It's Different

**Traceability, not a black box.**
Every recommendation Veloquity produces links back through its evidence to the exact original feedback items that generated it — source, timestamp, and context, not just a generated paragraph. If a recommendation can't be traced to real evidence, it isn't surfaced.

**Confidence scoring, not keyword counting.**
Evidence isn't ranked by how often a word appears. Related feedback is embedded and grouped semantically, then scored on how tightly that group actually agrees with itself — tight, coherent clusters route differently than loose, weakly-related ones, with the more expensive reasoning step reserved for evidence that has earned it.

**Agentic reasoning, not static rules.**
The final recommendation step isn't a fixed if/else rulebook. A reasoning agent retrieves the relevant evidence, weighs it against multiple real-world factors, and generates a structured, explainable recommendation — consistent and comparable across every run, but not hardcoded to any one domain.

---

## Engineering Evolution

The evidence-clustering approach wasn't the first thing that shipped. It started as a simpler, greedy similarity-based grouping method, and was later iterated toward density-based clustering in a dedicated experimentation branch to test whether cluster quality could improve further. That kind of iteration — ship the simple version, measure it, then deliberately explore an alternative — is the normal shape of how this system was actually built, not a one-shot design.

---

## Validated On

Veloquity's core claim is that the pipeline is domain-agnostic — the same code, unmodified, produces meaningful evidence and recommendations regardless of what kind of feedback it's given.

| Domain | Volume | Result |
|---|---|---|
| SaaS product feedback (app reviews + support tickets) | 547 items | Coherent evidence clusters, ranked recommendations |
| Healthcare patient experience (portal + survey data) | 310 items | Same pipeline, same confidence approach, zero code changes |
| **Combined** | **857 items** | **One pipeline, two unrelated domains** |

**Performance and cost, measured end-to-end:**

| Metric | Result |
|---|---|
| Full pipeline runtime | ~91 seconds |
| Cost per full run | $0.029 |
| Automated test suite | 158 tests, 100% passing |
| Test suite runtime | 0.72 seconds (fully mocked — no live AWS or DB calls) |

---

## AWS Services

| Service | Role in Veloquity |
|---|---|
| AWS Lambda | Hosts all four pipeline agents |
| Amazon Bedrock — Nova Pro | Powers the reasoning and recommendation generation stage |
| Amazon Bedrock — Titan Embed V2 | Generates embeddings for evidence clustering |
| Amazon RDS (PostgreSQL + pgvector) | Vector similarity search (HNSW) and relational storage |
| Amazon S3 | Raw feedback storage and reasoning run archival |
| Amazon EventBridge | Triggers scheduled governance runs |
| AWS IAM | Access control across all services |
| AWS Secrets Manager | Credential and secret management |

---

## Recognition

<a name="recognition"></a>

Veloquity won the **AWS 10,000 AIdeas Asia Pacific & Japan (APJC) Regional Championship 2026** — selected from 10,000+ teams across 115 countries.

- 🏆 **AWS APJC Regional Champion** — $15,000 prize support · $1,500 AWS credits · AWS re:Invent Las Vegas invitation
- 📋 Official winners list: [`assets/aws-apjc-winners-list.png`](assets/aws-apjc-winners-list.png)
- 📰 Full technical write-up: [AWS Builder Center Article](https://builder.aws.com/content/3AzrKpJbhJwEP6EZbm87vdxufgi/aideas-finalist-veloquity-the-agentic-evidence-intelligent-platform-turning-raw-feedback-into-evidence-driven-decisions)
- 🎥 Demo video: [Watch on YouTube](https://youtu.be/wEG5jTQxlJ4?si=l1tH72icmjTMdh_H)
- 🌐 Live platform: [veloquity1.vercel.app](https://veloquity1.vercel.app)
- 📰 Featured in **Business Today** alongside **Jeff Barr** *(VP & Chief Evangelist, AWS)*

---

<div align="center">
<img src="https://capsule-render.vercel.app/api?type=waving&color=0:24243e,50:302b63,100:0f0c29&height=100&section=footer" width="100%"/>
</div>
