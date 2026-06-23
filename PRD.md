# RunBook AI — Product Requirements Document

**Hackathon:** HiDevs × Mastra AI Agent Builder  
**Track:** Incident Response & Post-Mortem Agent  
**Stack:** Mastra · Qdrant · Enkrypt AI · TypeScript  
**Submission Round:** Round 1

---

## 1. Problem Statement

Production incidents are a 3 AM problem. When a P1 alert fires, the on-call engineer wakes up, opens a 200-page Confluence runbook, spends 15–20 minutes locating the right section under cognitive pressure, and then executes steps from memory — missing one, and extending the outage. Industry data from PagerDuty's State of Digital Operations report puts the average time-to-mitigate for P1 incidents at 45–90 minutes. Most of that time is not spent fixing. It is spent searching.

The tools that exist today solve the wrong half of the problem.

PagerDuty routes the alert. Incident.io opens a war room. FireHydrant creates a ticket. None of them open the runbook, run `kubectl describe`, or tell the engineer what the last three similar incidents looked like and how they were resolved. The engineer is alone the moment the notification lands.

**The precise gap:** No production tool today autonomously retrieves the relevant runbook section, executes safe read-only diagnostics, surfaces similar historical incidents, validates proposed remediation actions for safety, and presents a pre-approved action for one-click execution — all before the engineer has had time to fully wake up.

That is the gap RunBook AI closes.

---

## 2. Target Users

| User | Specific Pain Point |
|------|-------------------|
| On-call SRE / Backend Engineer | 15–20 min spent on runbook search during active incident; cognitive load causes missed steps |
| Engineering Manager | Post-mortems written from memory 2–3 days after incident; inconsistent format, incomplete timelines |
| Platform / DevOps Lead | Runbooks go stale with no feedback loop; no signal on which sections are actually used during incidents |

**Primary user:** the on-call engineer in the first 10 minutes of a P1/P2 incident. Every design decision optimises for this person under this condition.

---

## 3. Solution

RunBook AI is an agentic incident execution layer that sits between the alert and the engineer. It does not send a notification and wait. It acts — then asks permission before anything irreversible happens.

Three specialised agents, orchestrated by Mastra as a single stateful workflow, cover the full incident lifecycle:

**Triage Agent** fires the moment an alert webhook arrives. It classifies severity, identifies the affected service, checks GitHub for deploys in the last 30 minutes (the most common P1 root cause), and runs a hybrid semantic + keyword search against `incidents_collection` to surface the top-3 most similar historical incidents with their resolutions. The engineer receives a Slack message with full context before they've opened a laptop.

**Remediation Agent** takes the triage output and immediately searches `runbooks_collection` for the relevant runbook section using hybrid search — vector similarity for conceptual match, BM25 for exact error code and service name matching. It then executes read-only diagnostics autonomously: `kubectl describe`, `kubectl logs`, Datadog metric queries, service health endpoints. When it has enough evidence to propose an action, it sends the proposed command to Enkrypt AI for risk evaluation. Only after Enkrypt returns a risk score does the command reach the engineer for approval. Mastra suspends the workflow at this point — nothing executes until a human responds. If no response arrives within a configurable timeout (default: 5 minutes), the workflow escalates to the secondary on-call and pages them via PagerDuty.

**Post-Mortem Agent** activates after incident resolution. It synthesises the full timeline from the Mastra workflow execution log — every diagnostic run, every approval request, every command executed, every timestamp. It retrieves structurally similar past post-mortems from `postmortems_collection` as format templates and root-cause references. It generates a complete structured draft. Before the draft reaches the engineer, Enkrypt AI cross-references every timeline event against the actual execution log to flag invented facts, incorrect attributions, or fabricated root causes.

---

## 4. User Journey

### Happy path
```
[Alert fires — PagerDuty / Grafana webhook]
        │
        ▼
Backend API receives webhook → triggers Mastra workflow (incident_id assigned)
        │
        ▼
[Triage Agent — target: < 30s]
  • Classifies severity (P1–P4) and affected service from alert payload
  • GitHub API → checks for deploys on affected service in last 30 min
  • Qdrant incidents_collection → hybrid search → top-3 similar incidents + resolutions
  • Slack → pushes context card to on-call engineer
        │
        ▼
[Remediation Agent — target: < 2 min from alert]
  • Qdrant runbooks_collection → hybrid search → retrieves relevant runbook section
  • Executes read-only diagnostics autonomously:
      kubectl describe <pod>, kubectl logs --tail=100, Datadog metric query, health check
  • Synthesises diagnostic findings into proposed remediation action
  • Enkrypt AI → command validation → risk score (LOW / MEDIUM / HIGH / CRITICAL)
  • ⏸ Mastra waitForEvent → suspends workflow, pushes approval request to dashboard + Slack
        │
        ▼
[Engineer approves via Dashboard or Slack]
  • Approved → Mastra resumes, command executes against Kubernetes API
  • Rejected → agent proposes next runbook alternative, loops back to Enkrypt validation
  • No response in 5 min → escalation triggered (secondary on-call paged)
        │
        ▼
[Incident resolved — engineer marks resolved on dashboard]
        │
        ▼
[Post-Mortem Agent — target: < 60s to draft]
  • Synthesises timeline from Mastra workflow execution log
  • Qdrant postmortems_collection → retrieves similar past post-mortems as templates
  • Generates structured draft (summary, timeline, root cause, impact, action items)
  • Enkrypt AI → hallucination check → flags events not in execution log
  • Engineer edits annotated draft and publishes
```

### Failure & edge case handling

| Scenario | Behaviour |
|----------|-----------|
| Qdrant returns zero results for runbook search | Agent falls back to full-text search on raw runbook store; if still empty, surfaces the top-level runbook index and asks engineer to point to the right section |
| Enkrypt scores proposed command as CRITICAL | Command is blocked entirely; agent generates 2 alternative approaches and re-runs Enkrypt on each |
| Engineer rejects all proposed alternatives | Workflow enters manual mode — agent continues running read-only diagnostics and surfacing data while engineer takes manual control; full audit trail maintained |
| HiTL timeout (5 min, no approval response) | Mastra triggers escalation step: pages secondary on-call via PagerDuty, pushes full context to incident Slack channel |
| Post-Mortem Agent hallucination detected | Flagged sections shown to engineer with warning label; original execution log attached as source of truth |

---

## 5. Architecture

### 5.1 Layer Map

| Layer | Components |
|-------|-----------|
| Client | React + TypeScript On-Call Dashboard — REST + WebSocket |
| Control Plane | Node.js/Express Backend API · Mastra Orchestrator |
| Agent Swarm | Triage Agent · Remediation Agent · Post-Mortem Agent (TypeScript + Mastra SDK) |
| Memory & Retrieval | Qdrant — `incidents_collection` · `runbooks_collection` · `postmortems_collection` |
| Safety | Enkrypt AI SDK — Command Risk Scoring · Hallucination Detection |
| Integrations | PagerDuty · Grafana · Datadog · Slack · GitHub · Kubernetes API |

### 5.2 Mastra Orchestration

Mastra manages the incident lifecycle as a single named, stateful, resumable workflow. This is not three separate agent calls — it is one workflow with persistent state that survives process restarts.

**Mastra primitives used:**

- `createWorkflow` — defines the incident lifecycle as a typed step graph: `triage → remediation_loop → resolution → post_mortem`
- `waitForEvent` — the critical primitive. The Remediation Agent calls this after generating a Enkrypt-validated action. Mastra serialises the full workflow state to durable storage and suspends. The workflow resumes only when the approval event arrives (or the timeout triggers the escalation branch).
- `createAgent` with tool registries — each subagent has a typed tool set. Triage Agent tools: `getAlertPayload`, `queryIncidentsCollection`, `getRecentDeploys`, `sendSlackContextCard`. Remediation Agent tools: `queryRunbooksCollection`, `execKubectl`, `queryDatadog`, `validateWithEnkrypt`, `requestApproval`. Post-Mortem Agent tools: `getWorkflowExecutionLog`, `queryPostmortemsCollection`, `validateDraftWithEnkrypt`.
- Structured context passing — `incident_context` object is typed and passed between steps. No string-passing between agents.
- Escalation branch — `waitForEvent` accepts a timeout parameter. On timeout, Mastra routes to the `escalation` step rather than failing.

### 5.3 Qdrant Memory & Retrieval

**Embedding model:** `text-embedding-3-small` (OpenAI, 1536 dimensions) for all three collections. Chosen for cost-to-quality ratio on short-to-medium technical text. Qdrant named vectors allow switching the model per collection in future without schema migration.

**`incidents_collection`**
```
Payload schema:
{
  incident_id: string,
  service: string,
  severity: "P1" | "P2" | "P3" | "P4",
  error_signature: string,       // e.g. "OOMKilled", "CrashLoopBackOff"
  symptoms: string,              // free-text description
  root_cause: string,
  resolution_steps: string[],
  resolved_at: timestamp,
  mttr_minutes: number
}
Embedding input: `${error_signature} ${symptoms}`
Search type: hybrid (vector + BM25)
Queried by: Triage Agent on every new alert
Purpose: surfaces similar past incidents and proven resolutions
```

**`runbooks_collection`**
```
Payload schema:
{
  doc_id: string,
  service: string,
  section_title: string,
  chunk_text: string,
  chunk_index: number,
  parent_doc: string,
  last_updated: timestamp
}
Chunking strategy: 512 tokens, 64-token overlap (preserves step context across chunk boundaries)
Embedding input: `[${service}] ${section_title}: ${chunk_text}`  (service + title prepended for grounding)
Search type: hybrid (vector + BM25)
Queried by: Remediation Agent after triage classification
Purpose: retrieves the exact runbook procedure for the current incident symptoms
```

**`postmortems_collection`**
```
Payload schema:
{
  pm_id: string,
  service: string,
  root_cause_category: string,   // e.g. "resource_exhaustion", "bad_deploy", "dependency_failure"
  impact_scope: string,
  duration_minutes: number,
  action_items: string[],
  published_at: timestamp
}
Embedding input: `${root_cause_category}: ${impact_scope}`
Search type: hybrid (vector + BM25)
Queried by: Post-Mortem Agent after resolution
Purpose: retrieves structurally similar post-mortems as format and content templates
```

**Why hybrid search on all three collections:** Pure vector search misses exact strings — error codes like `OOMKilled`, pod names like `payments-service-7d9f`, and Kubernetes resource identifiers are single-token terms that embeddings compress poorly. BM25 catches exact matches; vector search catches conceptual similarity. Together they handle both "what happened before that looked like this" and "find the step that mentions this exact error code."

### 5.4 Enkrypt AI Safety Layer

Enkrypt is not a passive logger. It is a synchronous blocking gate in the agent workflow — the Remediation Agent cannot surface a command to the engineer until Enkrypt has returned a verdict.

**Checkpoint 1 — Command Risk Validation**
- Trigger: Remediation Agent has a proposed action ready
- Input to Enkrypt: proposed command string + execution context (service, environment, current incident state)
- Evaluation dimensions: data loss potential, irreversibility, blast radius, known dangerous pattern matching (e.g. `rm -rf`, broad namespace deletions, certificate rotations in production)
- Output: `{ risk_level: "LOW"|"MEDIUM"|"HIGH"|"CRITICAL", flags: string[], suggested_safer_alternative?: string }`
- LOW/MEDIUM: command surfaces to engineer with risk label attached
- HIGH: engineer sees command with prominent warning and must type a confirmation string to approve
- CRITICAL: command blocked; Enkrypt's `suggested_safer_alternative` is passed back to the Remediation Agent to propose instead

**Checkpoint 2 — Post-Mortem Hallucination Detection**
- Trigger: Post-Mortem Agent has generated a full draft
- Input to Enkrypt: draft text + Mastra workflow execution log (ground truth)
- Evaluation: cross-references every factual claim in the draft (timestamps, service names, commands run, root cause attribution) against the execution log
- Output: annotated draft with flagged sentences, confidence scores, and the execution log line that either supports or contradicts each claim
- Engineer sees the annotated draft with highlighted warnings before publishing

### 5.5 Data Flow (end-to-end)

```
PagerDuty/Grafana
  → POST /webhook → Backend API
  → triggerWorkflow(incident_payload) → Mastra Orchestrator
  → Triage Agent
      → GitHub API (recent deploys on affected service)
      → Qdrant incidents_collection.search(hybrid, symptoms+error_sig, top_k=3)
      → Slack API (context card push)
  → Remediation Agent
      → Qdrant runbooks_collection.search(hybrid, symptoms, top_k=5)
      → kubectl describe / kubectl logs / Datadog query (read-only)
      → Enkrypt AI SDK (command validation → risk score)
      → Backend API (POST /approval-request)
      → WebSocket → On-Call Dashboard (approval UI push)
      ← PATCH /approval-request/:id {decision: "approved"}
      → Mastra waitForEvent resolves → command executes
  → Post-Mortem Agent
      → Mastra getWorkflowExecutionLog()
      → Qdrant postmortems_collection.search(hybrid, root_cause, top_k=3)
      → Enkrypt AI SDK (hallucination check on draft)
      → WebSocket → On-Call Dashboard (draft editor push)
```

### 5.6 Tech Stack

| Component | Technology |
|-----------|-----------|
| Agent Orchestration | Mastra SDK (TypeScript) |
| Backend API | Node.js · Express · TypeScript · WebSocket (ws) |
| Frontend | React 18 · TypeScript · TanStack Query · WebSocket |
| Vector Database | Qdrant Cloud (managed) |
| Embedding Model | OpenAI text-embedding-3-small (1536d) |
| Safety & Guardrails | Enkrypt AI SDK (TypeScript) |
| Diagnostic Execution | kubectl (Kubernetes JS client) · Datadog API v2 |
| Alert Ingestion | PagerDuty Webhooks v3 · Grafana Alerting Webhooks |
| Notifications | Slack Bolt SDK (TypeScript) |
| Deploy Context | GitHub REST API v3 |
| Containerisation | Docker Compose (local dev) |
| LLM | OpenAI GPT-4o (agent reasoning) |

---

## 6. Key Design Decisions & Tradeoffs

**Why three agents instead of one?**
A single agent handling triage, remediation, and post-mortem would have conflicting tool registries, ambiguous state, and no clean suspension point for HiTL. Three agents with distinct responsibilities allow Mastra to manage each as a typed step with a clear input/output contract. The orchestrator, not the agent, manages transitions.

**Why Mastra `waitForEvent` instead of polling?**
Polling for approval burns compute and introduces latency. `waitForEvent` serialises workflow state and exits the process — the workflow resumes from exact state when the approval arrives. This is the correct pattern for human-in-the-loop systems where approval latency is measured in minutes, not milliseconds.

**Why hybrid search instead of pure vector?**
Error codes, pod names, and Kubernetes resource identifiers are high-signal exact-match terms. A vector embedding for `CrashLoopBackOff` and `OOMKilled` will be similar — which is wrong. BM25 separates them precisely. Hybrid search gives both semantic recall and exact-term precision.

**Why block on Enkrypt before showing to human, not after?**
The approval UI is the engineer's mental model of "safe to execute." If a CRITICAL-risk command reaches that UI, the engineer may approve it under pressure. Enkrypt must be a pre-filter, not a post-audit log. The human should only ever see commands that have already passed safety evaluation.

**Escalation timeout: 5 minutes**
Configurable. Default is 5 minutes based on common SRE on-call SLA expectations. If the primary on-call is incapacitated or unreachable, a 5-minute wait before escalation is aggressive enough to protect MTTR without being so short it creates false escalations from network lag.

---

## 7. Success Metrics

| Metric | Target | Rationale |
|--------|--------|-----------|
| Alert → triage context delivered | < 30 seconds | Engineer should have context before they've opened their laptop |
| Alert → first remediation proposal | < 2 minutes | Matches SRE industry expectation for automated first-response |
| Runbook retrieval — correct section in top-3 | > 80% | Measured against a labelled test set of incident-runbook pairs; 80% is achievable with hybrid search on well-structured runbooks |
| Enkrypt CRITICAL false positive rate | < 2% | A false CRITICAL block during a P1 forces the engineer to manually override — must be rare |
| Post-mortem draft accepted without major edits | > 70% | Proxy for hallucination rate; if the engineer rewrites more than 30%, the draft adds no value |
| MTTR reduction vs baseline | 30–50% | Conservative estimate based on eliminating manual runbook search time |

---

## 8. What This Is Not

- Not a chatbot. The engineer does not type questions. The agent acts on the alert automatically.
- Not a runbook replacement. RunBook AI reads and executes existing runbooks — it does not generate new operational procedures.
- Not fully autonomous. Every destructive action requires human approval. The agent is a force multiplier for the engineer, not a replacement.
- Not a monitoring tool. RunBook AI does not generate alerts. It consumes them from PagerDuty and Grafana.

---

*RunBook AI — autonomous incident execution with a human in the loop.*
