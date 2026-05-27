# Submission — Dawn Health SRE Technical Assessment

> Fill in this file with your written answers. For code and manifest tasks, your files in the `part*/` directories are the answer — you do not need to repeat code here.

---

## Part 1 — SLOs & Burn-Rate Alerting

### 1.1 — Recording Rules

> No written answer needed — your completed [`part1/recording-rules.yaml`](./part1/recording-rules.yaml) is the answer.

### 1.2 — Multi-Window Burn-Rate Alerts

> No written answer needed — your completed [`part1/slo-alerts.yaml`](./part1/slo-alerts.yaml) is the answer.

### 1.3 — SLO Policy

> What SLO target would you propose for a patient-facing API, and how would you justify it to product stakeholders who want to move fast? How would you use the error budget to govern releases, in particular to decide whether a release should proceed? What should happen when the error budget is exhausted?

_Your answer here._

---

## Part 2 — Incident Response

### 2.1 — Runbook

> No written answer needed — your completed [`part2/runbook.md`](./part2/runbook.md) is the answer.

### 2.2 — Postmortem

> Write a postmortem for the incident described in the README. Cover: timeline, root cause, contributing factors, impact, and as many action items as the incident warrants (at least three).

**Timeline** _(add as many rows as the incident warrants)_

| Time | Event |
|------|-------|
|      |       |

**Root Cause**

_Your root cause here._

**Contributing Factors**

_Your contributing factors here._

**Impact**

_Your impact assessment here. Quantify where you can: failed requests,
affected user cohort, any patient-safety-relevant flows touched._

**Action Items** _(add as many rows as warranted)_

| # | Action | Owner | Due |
|---|--------|-------|-----|
| 1 |        |       |     |

---

## Part 3 — Reliability Engineering

### 3.1 — HPA and PodDisruptionBudget

> Your completed [`part3/reliability.yaml`](./part3/reliability.yaml) is the primary answer.
>
> One short paragraph here: is memory-based HPA appropriate for `team-alpha-backend` given the Part 2.2 failure mode (memory leak triggering OOMKill)? Implement the spec as written, but explain whether you would actually keep the memory metric in production and why.

_Your evaluation here._

### 3.2 — Resource Strategy

> How would you approach setting CPU and memory `requests` and `limits` for a service you have never seen before? What are the risks of setting limits too low? Too high? On a shared multi-tenant cluster?  How would you enforce resource quotas consistently across all teams?

_Your answer here._

### 3.3 — Capacity Planning

> The business expects a 3x traffic increase over the next 6 weeks as a new partner integration goes live. How would you assess whether the platform can absorb this? What signals would you monitor in the run-up? What would you change proactively vs. reactively?

_Your answer here._

---

## Part 4 — Observability Investigation

### 4.1 — Queries

> No written answer needed — your completed [`part4/queries.md`](./part4/queries.md) is the answer.

### 4.2 — Dashboard Strategy

> What dashboards would you maintain as a platform SRE for a multi-tenant cluster? Describe 3–4 and what they show. How would you structure dashboards so that a product team can investigate their own service without needing to ask the platform team? What is the difference between a triage dashboard and a deep-dive dashboard, and when do you use each?

_Your answer here._

---

## Part 5 — Toil & Automation

### 5.1 — Identify and Prioritise Toil

> Describe three examples of toil you would expect to find on a Kubernetes platform team running a shared multi-tenant cluster. For each one: what makes it toil (rather than valuable engineering work), how you would measure how much time it consumes, and how you would prioritise which to automate first.

**Example 1**

_Your answer here._

**Example 2**

_Your answer here._

**Example 3**

_Your answer here._

### 5.2 — Implement the Automation

> Your completed [`part5/cronjob.yaml`](./part5/cronjob.yaml) is the primary answer.
>
> Briefly address here: (a) Kubernetes already provides `ttlSecondsAfterFinished` and a pod GC, so what does your CronJob add on top, and would you pick a different target? (b) Cluster-wide `get/list/delete` on pods and jobs in a PHI-handling multi-tenant cluster: what are the security implications and how would you mitigate them?

_Your notes here._

---

## Part 6 — Operational Scenarios

### 6.1 — Cascading Failure

> Walk through your investigation of the cascading failure described in the README. How do you identify the root cause as Service B (rather than Service A), and what do you do about it?

_Your answer here._

### 6.2 — On-Call Handoff During an Active Incident

> You are 30 minutes into a P1 incident (as defined in the README) when your shift ends. How do you hand off? What do you communicate, in what format, to ensure the incoming engineer can pick up without losing ground?

_Your answer here._

### 6.3 — Noisy Alerting _(Optional)_

> How would you audit and reduce 40–60 daily alert notifications (with ~5 after-hours pages per week, ~80% non-actionable) without reducing coverage for real incidents?

_Your answer here._

---

## Anything Else?

> Note any tasks you skipped and how you would have approached them, assumptions you made, or anything else you would like the reviewers to know.

_Your answer here._
