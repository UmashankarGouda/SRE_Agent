# SRE Agent: Multi-Agentic Autonomous Site Reliability Agent

**Evidence-driven incident investigation and human-approved remediation for Kubernetes.**

![Python](https://img.shields.io/badge/python-3.12-blue)
![Kubernetes](https://img.shields.io/badge/kubernetes-minikube-326ce5)
![Tests](https://img.shields.io/badge/tests-252%20passing-brightgreen)
![Status](https://img.shields.io/badge/status-research%20prototype-orange)

A set of cooperating agents, each with a narrow job and limited authority, that carry an incident from its
first symptom to a **verified fix**. The system is validated on a live Kubernetes application into which
failures are injected by scripts that tell the investigator nothing. It has to work out what broke from
evidence alone.

> Major Project (BCS705), B.E. Computer Science and Business Systems, BMS Institute of Technology and
> Management (VTU), 2026-27.

---

## Why this exists

When a production service fails, most of the incident is spent *understanding* it, not fixing it. The
evidence is also perishable: Kubernetes keeps only a container's current and previous run, forgets a deleted
pod at once, and expires events after about an hour. Existing approaches each fall short in one way:

- **Monitoring** raises alerts without explaining them.
- **Graph-based root-cause methods** name a suspicious service but not *why* it failed, and need metrics that
  may already be gone.
- **LLM assistants** write fluent explanations that are not always traceable to the real system state.
- **Very few tools** carry an incident safely from diagnosis to a *verified* fix.

This project treats **evidence as the single source of truth**, keeps every executable action deterministic,
puts a human in control of every change, and lets verification say "not resolved".

## Key features

| Property | What it means in practice |
|---|---|
| **Scenario-blind** | No scenario flag or hint exists anywhere in the investigation path (asserted by a test). |
| **Evidence-first** | Every conclusion cites identified, time-qualified facts. Facts are kept apart from interpretation. |
| **Post-mortem capable** | A recorder retains what Kubernetes forgets (24 h), so incidents can be reconstructed after the failing pods are gone. |
| **Honest about uncertainty** | Times print only as precisely as they are known (ranges, "at or before"). Confidence drops when competing explanations exist. "Undetermined" is preferred to an invented answer. |
| **Provider-independent reasoning** | Reasoning code uses a neutral capability and fact vocabulary. Kubernetes and Prometheus sit behind adapters. Architecture tests enforce this. |
| **Safe in action** | Typed plans, per-action human approval bound to the plan digest, a separate Execute, policy check, live recheck, dry run, exactly-once change, then verification. |
| **Grounded LLM** | The LLM agent is read-only. Every citation is validated in code and the executable action never comes from the model. |
| **Graduated autonomy** | A policy-gated Remediation Agent acts without a click only for high-confidence, low-risk fixes, and reverts anything it cannot verify. |

## How it works

```
Detection (symptoms only)
   -> Incident context -> Investigation planner <-> Capabilities (read-only; Kubernetes / Prometheus adapters)
   -> Facts -> Diagnosis (taxonomy, scoring, dependency chaining) -> Timeline + impact -> Report
   -> Hybrid routing: rules lead  --or--  LLM Investigator Agent leads
   -> Typed remediation plan -> Slack review (per action, bound to plan digest)
   -> Execute: policy -> claim -> live recheck -> dry run -> ONE typed change
   -> Verification: RESOLVED | NOT_RESOLVED | INCONCLUSIVE (rollback plan offered on failure)
```

### The agents and their authority

| Agent | Responsibility | Authority |
|---|---|---|
| **Detector** | Samples the system every 5 s and opens an incident when a symptom persists (two consecutive polls). | Reports symptoms only, never causes. |
| **Evidence recorder** | Retains logs of earlier runs, deleted pods, expired events, configuration versions and service availability for 24 h in SQLite. | Read-only observer. |
| **Rule engine** | Plans which evidence to collect, records facts, diagnoses, reconstructs the timeline and the impact. Deterministic. | Read-only. |
| **Investigator Agent (LLM)** | Verifies the rules on familiar incidents and leads on unfamiliar ones, using read-only tools. | Read-only. Citations validated in code. |
| **Remediation planner** | Turns the diagnosis into a typed, evidence-backed plan. | No system access, executes nothing. |
| **Remediation Agent** | Deterministic, policy-gated. Replaces the engineer's two Slack clicks only for high-confidence, low-risk fixes. | At most 3 attempts, automatic revert, circuit breaker. |
| **Engineer** | Approves, rejects or defers each action in Slack, supplies missing values, presses Execute. | Final authority over every change the policy gate does not cover. |

### Investigation methodology

Every incident follows one evidence chain:

```
raw source -> normalised evidence -> fact -> correlation -> diagnosis -> root cause -> plan -> change -> verification
```

1. **Symptoms first, causes later.** Detection never guesses why.
2. **Follow the evidence.** The planner decides each next step from what it has found and records a reason.
3. **Facts before interpretation.** Each fact has an ID, source, subject and time basis (exact, bounded, observed, before window or unknown).
4. **A general failure taxonomy.** Every component is checked against eight categories: memory exhaustion, application crash, dependency misconfiguration, dependency unavailable, image pull failure, container configuration error, unschedulable and health-check failure.
5. **Root cause versus impact.** Findings are chained along the dependency graph derived from configuration. A consumer whose errors are explained by a failing dependency is marked *impacted*, not *the cause*.
6. **Change only under authority, then verify.**

**Confidence** is `c = min(0.97, max(0.05, s1 * (1 - 0.5 * s2)))`, where `s1` is the primary finding's score and
`s2` the strongest independent competing finding. It is multiplied by 0.9 if any capability call failed.
`>= 0.75` is High, `>= 0.5` is Medium, otherwise Low.

**Hybrid routing.** If the rules reach at least 75% confidence on a known pattern, they lead and the agent only
verifies (at most 5 tool calls, 150 s). Otherwise the agent leads (at most 14 tool calls, 300 s). Measured live,
leading costs about four times as much as verifying (about 180 s and 89k tokens against about 37 s and 21k).

## Architecture

```
 loadtest ns             shop ns (system under investigation)                      host
┌──────────┐  HTTP  ┌──────────┐  HTTP  ┌──────────────┐  SQL  ┌────────────┐   ┌───────────────────────────┐
│ loadgen  │ ─────► │ frontend │ ─────► │ backend (x2) │ ────► │ PostgreSQL │   │ python -m demo / investigator │
│ 100 rps  │        │ gateway  │        │ orders API + │       │            │   │  detector -> planner      │
└──────────┘        └──────────┘        │ invoice job  │       └────────────┘   │  capabilities -> adapters │
                                        └──────────────┘                        │  diagnose -> plan -> exec │
                                                                                └───────────────────────────┘
                              Slack (Socket Mode)  ·  NVIDIA NIM (LLM)  ·  Web console :8800
```

The service is layered with strict boundaries:

| Layer | Modules | Notes |
|---|---|---|
| Detection | `detector.py` | Symptoms only, via capabilities |
| Context and planner | `context.py`, `planner.py` | Incident window, suspects, evidence-following steps |
| Capability interface | `capabilities/base.py`, `capabilities/__init__.py` | Neutral, read-only, traced |
| Adapters | `capabilities/kubernetes.py`, `capabilities/prometheus.py` (+ `kube.py`, `prom.py`) | Provider-specific by design |
| Evidence history | `capabilities/kubernetes_recorder.py`, `history_store.py` | Retains what Kubernetes forgets |
| Composition root | `providers.py` | The one place that chooses adapters and the actuator |
| Evidence | `collect.py`, `dependencies.py`, `metrics.py`, `logparse.py`, `evidence.py` | Capability results become identified facts |
| Diagnosis | `diagnosis.py` | Taxonomy, scoring, dependency chaining, confidence, alternatives |
| Timeline and impact | `timeline.py`, `impact.py` | Bounded times, blast radius |
| Remediation plan | `remediation.py`, `remediation_model.py` | Deterministic typed `RemediationPlan` |
| Human review | `review.py`, `slack_view.py`, `slack.py`, `slack_app.py` | Digest-bound decisions, Block Kit, Socket Mode |
| Execution | `executor.py`, `execution_policy.py`, `execution_store.py`, `recheck.py`, `verification.py` | Ordered safety checks, one change, audit |
| Cluster writer | `actuators/kubernetes.py` | The only code that writes |
| Demo layer | `demo/engine.py`, `demo/agent/*`, `demo/remediation_agent.py` | Web console, LLM agent, Remediation Agent |

Two architectural rules hold throughout, and both are enforced by `tests/test_architecture.py`:

1. Reasoning modules never import provider code or use provider vocabulary such as `OOMKilled`. Adapters map provider values to neutral ones (for example the termination cause `memory_limit`).
2. There is exactly **one write path** to the cluster: `Execute -> executor -> actuator`.

### Typed remediation actions

The executor supports four typed, compare-and-set actions, each with a dry run and a `resourceVersion` precondition:

- adjust memory limit
- scale workload (replicas)
- restore a configuration value (with restart)
- roll back a release image

There is no shell, `kubectl`, `exec` or YAML execution anywhere. Every change is a fixed patch.

## Safety model

- **Read and write separation.** Capabilities are read-only. Only the executor holds an actuator.
- **Authorisation.** Only Slack user IDs in `SLACK_APPROVERS` may decide, and by default only approvers may execute. Unauthorised clicks get a private notice and are audited.
- **Approval bound to the exact plan.** Each plan is identified by the SHA-256 digest of its canonical JSON. A newer plan supersedes the old one and its decisions. Duplicate or conflicting clicks are refused.
- **Policy limits.** Only listed action types, namespace `shop`, memory up to 1 Gi, at most 3 replicas, plans no older than 15 minutes.
- **Exactly-once execution.** An atomic, permanent claim per incident, plan digest and action. An interrupted execution is marked *uncertain* and never retried automatically.
- **Nine ordered checks on Execute:** executor allowed, effective approval on the current digest, plan age, policy, claim, live recheck, dry run, one change, verification.
- **Secret protection.** Secret values are never read into evidence or reports. Secret-backed configuration is recorded only as a 16-character SHA-256 fingerprint, which shows *that* a value changed without revealing it.
- **LLM containment.** Read-only tools. Tool output is marked untrusted (a defence against prompt injection from log lines). Citations, root-cause component and suggested action are validated in code. The executable action always comes from the deterministic plan.
- **Audit.** Every decision, refusal, claim, recheck, dry run, change and verification is recorded with actor, time and plan digest.

### Remediation Agent gate

All of these must hold, otherwise the incident goes to an engineer in Slack:

- automation is switched on and the pattern is known (rules led)
- rule confidence is at least 85% (`demo/remediation_agent_policy.json`)
- the Investigator Agent agrees and every finding is evidence-checked
- the action is `scale_workload` or `adjust_resource_limit`, with a validated value
- the component is not locked

Memory attempts follow a ladder: the agent's value, then x1.5 rounded up to 64 Mi and capped at 1 Gi, at most 3
attempts. Replica changes get one attempt. If nothing was verified as resolved, the original value is restored
through the executor and the component stays locked until an engineer acknowledges.

## Results

All results come from live runs on a minikube cluster with injected failures. They are demonstrations rather
than statistics.

**Retained evidence changes the diagnosis**

| Case | Without history | With history |
|---|---|---|
| OOM, pods replaced | Undetermined, 0% | Memory exhaustion in `backend`, **85%** |
| OOM, crash-looping | Memory exhaustion, 70% | Memory exhaustion, **85%** |
| Database down, then recovered | Application crash, 75% (**wrong**) | Dependency unavailable, root cause `postgres`, 57% |

**Approved execution and verification**

| Incident | Approved change | Outcome |
|---|---|---|
| Database down | scale postgres 0 -> 1 | RESOLVED (settled in 26 s, 5/5 criteria) |
| Wrong DB host, run 1 | restore host | NOT_RESOLVED (fix worked, but an unrelated stall pushed errors to 15.1%) |
| Wrong DB host, run 2 | same | RESOLVED (21 s, 4/4 criteria) |
| Overload (210 req/s) | backend memory 192 -> 256 Mi | NOT_RESOLVED (8 new kills, as the plan's own risk statement predicted) |
| Rollback of the above | memory 256 -> 192 Mi | RESOLVED, under a separate approval |

**LLM Investigator Agent and Remediation Agent**

| Incident | Outcome |
|---|---|
| Database down (rules 97%) | Rules led, agent verified, Remediation Agent scaled 0 -> 1, RESOLVED, **no human click** |
| Memory limit set too low (rules 97%) | Remediation Agent raised 32 -> 64 Mi, RESOLVED on attempt 1, **no human click** |
| Stale database credentials (rules 60%, wrong setting) | Agent led and found the rotated Secret from configuration history |
| Broken release (missing image) | Rollback plan from recorded history, human approve and execute |
| Sustained overload (rules 49%) | Gate refused, a human decides |

**Tests.** All **252 automated tests pass** offline in about 18 s against a fake Kubernetes API (103 unit,
35 functional, 114 integration, across 27 test files).

**Limitations.** Kubernetes is the only provider implemented, so provider independence is designed and
enforced but not yet shown with a second provider. Verification judges observed state, not cause: an unrelated
stall can give NOT_RESOLVED. Nothing is retained while the recorder is not running. A measured MTTR comparison
has not yet been carried out.

## Failure scenarios (test cases)

Injection scripts live in `scripts/inject/`. Each changes the cluster exactly as a real fault would.

| ID | Script | What breaks | Expected handling |
|---|---|---|---|
| TC1 | `oom.ps1` | Traffic spike to 800 req/s exceeds the 192 Mi limit | Memory exhaustion in `backend` |
| TC2 | `db-misconfig.ps1` | `DATABASE_URL` host changed to `postgres-wrong` | Dependency misconfiguration, restore recorded value |
| TC3 | `db-down.ps1` | PostgreSQL scaled to 0 | Dependency unavailable, root cause `postgres`, frontend impacted |
| TC4 | `app-crash.ps1` | Order with quantity 0 makes the invoice job divide by zero | Application crash (not OOM), manual steps |
| TC5 | `oom-then-replace.ps1` | OOM, then an operator replaces the pods | Memory exhaustion only with evidence history |
| TC6 | `db-credentials.ps1` | Backend reads a stale password from a new Secret | Agent leads and finds the credential change |
| TC7 | `bad-image.ps1` | Backend points at an image tag that does not exist | Image pull failure, roll back from recorded history |
| TC8 | `memory-limit-low.ps1` | Backend limit lowered to 32 Mi | Memory exhaustion, eligible for the Remediation Agent |
| TC9 | `backend-down.ps1` | Backend scaled to 0 | No rule pattern, agent leads, manual steps |
| TC10 | `overload.ps1` | 800 req/s for 15 minutes | Low confidence, Remediation Agent gate must refuse |

`scripts/restore.ps1` recovers the system from any of them.

## Getting started

### Prerequisites

- Windows host with **Docker Desktop**, **minikube** and **kubectl** (scripts are PowerShell)
- **Python 3.12**
- A **Slack app** with Socket Mode and Interactivity enabled (bot token `xoxb-`, app token `xapp-`)
- An **NVIDIA NIM API key** (`nvapi-`) for the LLM Investigator Agent. Without it the agent is disabled and the rule engine still works.

The experiments in the report ran on one Windows host with a single-node minikube cluster (profile
`incident-demo`, Docker driver, 4 CPUs, 3.5 GB).

### 1. Install

```powershell
git clone https://github.com/UmashankarGouda/SRE_Agent.git
cd SRE_Agent
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure

```powershell
copy .env.example .env
# fill in SLACK_BOT_TOKEN, SLACK_CHANNEL, SLACK_APP_TOKEN, SLACK_APPROVERS, NVIDIA_API_KEY
```

`.env` is git-ignored. Never commit real tokens. Every variable is documented inside `.env.example`.

### 3. Start the cluster and the demo shop

```powershell
.\scripts\setup.ps1
python -m investigator check     # verifies Kubernetes, the entry probe, metrics and Slack
```

This starts minikube, builds the app image, and deploys the shop (frontend, two-replica backend, PostgreSQL),
the load generator (100 req/s) and Prometheus.

### 4. Run

**Web console** (recommended for a demo):

```powershell
python -m demo                   # opens http://127.0.0.1:8800
```

Click *Start services*, then use the incident lab to trigger a real failure. The status turns red, and after 20 s
of symptom collection the rule engine's decisions and the Investigator Agent's tool calls appear, followed by
the report. *Reset to healthy* restores the system.

**Command line:**

```powershell
python -m investigator watch                 # detection, recorder, investigation, reporting and Slack listener
python -m investigator investigate --since 10m   # on-demand investigation over retained history
python -m investigator replay <evidence-file>    # re-diagnose a saved evidence file offline
```

`status`, `record`, `post` and `review` run the individual parts.

### 5. Inject a failure

```powershell
.\scripts\inject\db-down.ps1     # example: scale PostgreSQL to zero
# watch Slack, read the plan, Approve, then Execute
.\scripts\restore.ps1            # back to healthy
```

In Slack each incident appears as headed messages (detected, investigation, remediation, final status).
Approve each action first, then click **Execute** as a separate step. The console takes no decisions. It mirrors
the shared review and execution records.

### Run the tests

```powershell
python -m pytest tests -q        # 252 passed, offline, about 18 s
```

## Repository layout

```
app/            The shop under investigation (frontend, backend, load generator)
k8s/            Kubernetes manifests (namespaces, PostgreSQL, shop, loadgen, Prometheus)
investigator/   Detection, capabilities, evidence, diagnosis, planning, review, execution
  capabilities/ Provider-neutral interface and Kubernetes / Prometheus adapters
  actuators/    Typed cluster writer, the only code that changes the cluster
demo/           Web console, LLM Investigator Agent, Remediation Agent
config/         Execution policy
scripts/        Setup, restore and failure-injection scripts
tests/          252 tests and a fake Kubernetes API
docs/           Architecture, roadmap, incident notes, live validation records
archive/        Iteration 1 prototype, kept for reference
```

## Technology stack

| Layer | Technology |
|---|---|
| Application under test | Python 3.12, aiohttp, psycopg 3, prometheus_client, PostgreSQL 16 |
| Orchestration | Kubernetes on minikube (Deployments, Services, ConfigMaps, Secrets) |
| Evidence sources | Kubernetes API (Python client), Prometheus HTTP API (PromQL) |
| Investigation service | Python 3.12, dataclasses, SQLite for evidence history and audit |
| Engineer interface | Slack Web API (Block Kit) and Socket Mode via `slack_sdk` |
| LLM agent | NVIDIA NIM (OpenAI-compatible), Nemotron 3 Super (120B), native tool calling |
| Web console | FastAPI, Uvicorn, Server-Sent Events |
| Testing | pytest with a fake Kubernetes API |

## Development history

Built incrementally, each iteration validated live on the cluster and kept as regression tests.

| Iteration | Theme |
|---|---|
| 1 | First prototype: OOM detection with metric timeline and LLM narrative (archived) |
| 2 | Evidence-driven investigation of four failure types without being told which |
| 3 | Capability abstraction, planner, neutral fact vocabulary, architecture tests |
| 4 | Historical evidence recorder, timeline with bounded times, impact assessment |
| 5 | Deterministic, typed remediation planning (nothing executed) |
| 6 | Slack review with per-action decisions bound to the plan digest |
| 7 | Approved execution, verification and rollback plans |
| Demo layer | LLM Investigator Agent with hybrid routing, Remediation Agent, web console |

More detail is in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md), [`docs/ROADMAP.md`](docs/ROADMAP.md),
[`docs/INCIDENTS.md`](docs/INCIDENTS.md), [`docs/PROJECT_VISION.md`](docs/PROJECT_VISION.md) and
[`docs/validation/`](docs/validation/). A walkthrough for presenting the project is in [`DEMO.md`](DEMO.md).

## Future work

- A second infrastructure provider, to demonstrate the provider independence the architecture already enforces
- A multi-tenant SaaS control plane with a read-only customer connector
- Incident memory: similarity search over past incidents as a hint, never as evidence
- A measured MTTR study with and without the system, with several engineers
- Wider graduated autonomy, extended per action type from advisory to semi-autonomous
- Better detection and verification: merge duplicate incidents, add CPU and node metrics, tell an unrelated transient stall from an incomplete fix
- An external pilot on a real service with on-call engineers

## Team

BMS Institute of Technology and Management, Department of Computer Science and Business Systems.

| Name | USN |
|---|---|
| R Deepak Reddy | 1BY23CB042 |
| Saanvi M S | 1BY23CB046 |
| Umashankar S | 1BY23CB058 |
| Utkarsh Jain | 1BY23CB059 |

**Guide:** Prof. Uday Prasad P K, Associate Professor, Dept. of CSBS.

## Acknowledgements

Builds on the ideas of Google's SRE practice, chaos engineering, and the AIOps and LLM-agent literature
(MicroRCA, RCAgent, Flow-of-Action, STRATUS, among others) surveyed in the project report.
