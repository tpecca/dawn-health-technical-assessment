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

I would propose a **99.9% request success rate** over a rolling 28-day window as the starting point for a patient-facing API. The justification has two sides:

To engineering, 99.9% translates to ~40 minutes of error budget per 28 days - enough to absorb a real incident and several deployments, which preserves velocity. A higher target (99.99%) would leave only ~4 minutes of budget, making any deployment that touches production a near-zero-tolerance exercise. That is not pragmatic at current team and platform maturity.

To product stakeholders, the error budget *is* the velocity guarantee. The conversation I'd have is: "You have 40 minutes of budget per month. How you spend it is your choice - on deployments, on experiments, on unavoidable failures. SPend it wisely and you can deploy as of ten as you like. Burn it on reliability failures and releases stop until the next period starts." That reframes SLO compliance as a high business enabler rather than an engineering constraint.

For SaMD-classified flows specifically (patient safety-relevant), I would recommend a separate, stricter SLO, potentially 99.95%, because the clinical or compliance impact of failures is asymmetric. The 99.9% target covers general API availablility.

**How would you use the error budget to govern releases?**

| Budget remaining | Release policy |
|------------------|----------------|
| > 50% | Normal cadence; standard review process |
| 25-50% | Canary required; releases need explicit SRE sign-off |
| < 25% | Freeze non-critical releases; reliability work takes priority |
| Exhausted | Full feature freeze; only security patches and rollbacks allowed |

The key principle: the release decision is automatic, not political. If the budget is exhausted, the freeze is not punishment. It is the pre-agreed policy that protected everyone from over-riding the SLO.

**What should happen when the error budget is exhausted?**

Feature releases stop immediately for the remainder of the budget period. The team runs a retrospective to understand whether the budget was consumed by an incident (which needs a postmortem and prevention actions) or by a series of slow-burn errors (which needs an SLI investigation). At the start of the next period, the budget resets and the retrospective findings drive the roadmap. Exhaustion more than twice in a row should trigger a reliability sprint and a review of whether the SLO target itself needs adjustment.

---

## Part 2 — Incident Response

### 2.1 — Runbook

> No written answer needed — your completed [`part2/runbook.md`](./part2/runbook.md) is the answer.

### 2.2 — Postmortem

> Write a postmortem for the incident described in the README. Cover: timeline, root cause, contributing factors, impact, and as many action items as the incident warrants (at least three).

**Timeline** 

| Time | Event |
|------|-------|
| 01:55 | New version of `team-alpha-backend` deployed via ArgoCD. All three pods updated simultaneously with no canary. |
| 02:14 | First HTTP 503 responses observed from `team-alpha-backend`. Memory leak is causing rapid heap growth. |
| 02:17 | `TeamAlphaBackendHighErrorBudgetBurn` fires as error rate exceeds 1.4%. First OOMKill events recorded in pod state. |
| 02:19 | 40% of requests failing. All three pods in crash-loop: each pod is OOMKilled, restarted by kubelet, and crashes again within ~90 seconds as the leak refills memory. `livenessProbe` failures during restarts extend each recovery window. |
| ~02:20 | On-call engineer paged. Begins investigation. |
| ~02:35 | Root cause identified as the 01:55 release via memory growth charts and `kubectl describe` showing `OOMKilled` / exit code 137. |
| 02:42 | Rollback initiated via ArgoCD to the previous stable revision. |
| 02:47 | All three pods healthy and serving traffic on the rolled-back version. Error rate returns to baseline. Incident resolved. |

**Root Cause**

A memory leak introduced in the 01:55 release caused unbounded heap growth on every pod. The leak was fast enough to push each pod over its 512Mi memory limit within ~4 minutes of startup. The kernel OOMKilled all three pods in close succession. Kubelet restarted each pod per the Deployment's restart policy, but the leak caused each restarted pod to hit the limit again within ~90 seconds, creating a sustained crash-loop. The `livenessProbe` also failed during the brief restart windows, preventing any pod from reaching a fully healthy state. Service was fully restored only after the release was rolled back.

**Contributing Factors**

1. **No canary rollout** — all three pods received the new version simultaneously, eliminating all healthy capacity. A 10% canary would have exposed the leak while two healthy pods continued serving traffic.
2. **No pre-OOMKill alert** — the team was alerted only after OOMKills started, at which point service was already degraded. An alert at 80% memory utilisation would have given a ~1-minute warning window.
3. **Overlapping `livenessProbe` failures** — the liveness probe fired during the short restart/recovery windows because `initialDelaySeconds: 30` was insufficient for a pod recovering under memory pressure, extending each pod's downtime.
4. **No memory regression test in CI/CD** — the leak was not detected in staging because no load test with a memory growth assertion existed.
5. **Fast restart policy without circuit breaking** — Kubernetes restarted pods into the same broken version indefinitely. There is no mechanism to "give up and hold" after multiple OOMKills in quick succession.

**Impact**

- **Duration**: 33 minutes of degraded service (02:14–02:47).
- **Peak error rate**: 40% of requests failing.
- **Affected scope**: all patient-facing workflows routed through `team-alpha-backend`.
- **Data integrity**: no data loss; degraded availability only.
- **SaMD relevance**: any clinical decision support or patient data submission flows dependent on this API were unavailable during the incident. This must be documented in the audit trail.

**Action Items**

| # | Action | Owner | Due |
|---|--------|-------|-----|
| 1 | Add a pre-OOMKill warning alert that fires at 80% memory utilisation, before the kernel kills the container | Platform SRE | 1 week |
| 2 | Introduce canary rollouts via Argo Rollouts (10% traffic, 5-minute memory soak period) for all `team-alpha-backend` releases | Dev Team + Platform SRE | 2 weeks |
| 3 | Add memory regression assertion to CI/CD pipeline: fail the build if container memory grows > 10% under constant load in staging | Dev Team | 2 weeks |
| 4 | Increase `livenessProbe.initialDelaySeconds` to 60s to survive restart cycles under memory pressure | Dev Team | 1 week |
| 5 | Update this runbook to make ArgoCD rollback the explicit first-line response to OOMKill incidents | Platform SRE | 3 days |

---

## Part 3 — Reliability Engineering

### 3.1 — HPA and PodDisruptionBudget

> Your completed [`part3/reliability.yaml`](./part3/reliability.yaml) is the primary answer.
>
> One short paragraph here: is memory-based HPA appropriate for `team-alpha-backend` given the Part 2.2 failure mode (memory leak triggering OOMKill)? Implement the spec as written, but explain whether you would actually keep the memory metric in production and why.

Memory-based HPA is implemented as specified, but I would not keep the memory metric in production for this workload.

The failure mode from Part 2.2 illustrates why: a memory leak causes memory growth independently of request volume. If the HPA scales out in response to growing per-pod memory, every new pod carries the same leak and will grow identically. Adding pods does not reduce the memory pressure on the originating pods — it just adds more leaking pods, which together exhaust cluster-level memory faster than the original three would have.

The HPA is designed to handle load-driven resource increases. A memory leak is not load-driven — it is a code defect. The right response to a leak is a human-triggered rollback, not autoscaling. For this specific service, I would remove the memory metric and rely on the pre-OOMKill warning alert (recommended in the runbook) to notify the on-call engineer instead.

Memory-based scaling *is* appropriate for services that hold in-memory caches proportional to request volume (e.g., a caching proxy that loads more data into memory as request diversity grows). `team-alpha-backend` is not documented as that type of service.

### 3.2 — Resource Strategy

> How would you approach setting CPU and memory `requests` and `limits` for a service you have never seen before? What are the risks of setting limits too low? Too high? On a shared multi-tenant cluster?  How would you enforce resource quotas consistently across all teams?

**Setting requests and limits for an unfamiliar service:**

I would not guess. I would start by running the service in staging under a representative load test — even a rough one — while observing `container_memory_working_set_bytes` and `container_cpu_usage_seconds_total`. From that I derive a p95 baseline. I then enable the [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) in recommendation mode (`updateMode: Off`) for 48–72 hours in staging or production, which gives a data-driven suggestion based on actual usage patterns.

Initial sizing rule of thumb: requests at ~70–80% of observed p95 usage (so HPA has meaningful signal before limits are approached); limits at ~2× requests to allow burst without immediate OOMKill.

**Risks:**

*Limits too low*: memory OOMKill (sudden, visible, covered above). CPU throttling is worse in some ways — it is **silent**: the container stays running and serving traffic but responses slow down. CPU throttling is a common source of unexplained latency increases that are hard to attribute because there are no error signals.

*Limits too high*: the node scheduler still considers requests for placement, so over-provisioning requests wastes bin-packing capacity. High limits without proportionally high requests are dangerous on a shared cluster: a pod can quietly burst to its limit and starve neighbours. On a multi-tenant cluster handling PHI this is a "noisy neighbour" problem with regulatory implications — one team's memory surge can degrade or even crash workloads belonging to another team.

**Enforcement across teams:**

Three layers, each catching what the previous one misses:

1. **`LimitRange`** per namespace — sets a default request and limit so any pod without resource specs still gets bounded. Without this, a forgotten `resources: {}` in a manifest means the scheduler places the pod as if it has zero resource requests.
2. **`ResourceQuota`** per namespace — caps aggregate consumption for the team (e.g., total CPU ≤ 4 cores, total memory ≤ 8Gi). This prevents one team from consuming disproportionate cluster capacity.
3. **Admission controller (OPA Gatekeeper or Kyverno)** — rejects any Deployment manifest that does not include explicit `resources.requests` and `resources.limits`. This is the only enforcement that stops the bad manifest from entering Git at all.


### 3.3 — Capacity Planning

> The business expects a 3x traffic increase over the next 6 weeks as a new partner integration goes live. How would you assess whether the platform can absorb this? What signals would you monitor in the run-up? What would you change proactively vs. reactively?

**Assessing whether the platform can absorb 3× traffic:**

Before writing any capacity proposals I would build a baseline picture:

- Current p95 and p99 CPU and memory per pod for each affected service, at peak traffic.
- HPA state: which services are already near `maxReplicas`? If a service is running at 8/10 maxReplicas today, 3× load will saturate it.
- Node pool utilisation: what percentage of node CPU/memory is currently allocated? At what percentage does the cluster autoscaler start adding nodes, and what is the provisioning latency (typically 2–4 minutes on AKS)?
- Downstream dependencies: does the database connection pool, the external partner API, or any backing queue have a documented rate limit or throughput ceiling?

**Signals to monitor in the run-up:**

- Pod pending time (`kube_pod_status_phase{phase="Pending"}` duration) — scheduling failures indicate the cluster needs more nodes before the traffic arrives.
- HPA current vs. max replicas for each service in scope.
- Node CPU and memory allocatable vs. requested.
- Database connection pool saturation and query latency.
- Any upstream API rate-limit counter.

**Proactive vs. reactive:**

Proactive (do before traffic arrives):
- Increase `maxReplicas` on affected HPAs. The HPA can only scale within its maximum; hitting the ceiling is silent until pods become saturated.
- Raise the cluster autoscaler's minimum node count to pre-warm capacity. Autoscaler provisioning latency is too slow for a sudden 3× spike.
- Review database connection limits and add read replicas or connection pooling if needed.
- Run a synthetic load test at 3× baseline and observe where the first bottleneck appears.

Reactive (leave to the system):
- Cluster autoscaler adding nodes beyond the pre-warmed minimum.
- HPA scaling pods within the new `maxReplicas` ceiling.
- Alerting and runbooks already cover the degradation scenarios if something is underestimated.

---

## Part 4 — Observability Investigation

### 4.1 — Queries

> No written answer needed — your completed [`part4/queries.md`](./part4/queries.md) is the answer.

### 4.2 — Dashboard Strategy

> What dashboards would you maintain as a platform SRE for a multi-tenant cluster? Describe 3–4 and what they show. How would you structure dashboards so that a product team can investigate their own service without needing to ask the platform team? What is the difference between a triage dashboard and a deep-dive dashboard, and when do you use each?

**3–4 platform-level dashboards I would maintain:**

1. **Platform health overview** — cluster node count, overall CPU and memory allocation ratios, pod pending rate, control plane API server latency, and cluster autoscaler events. This is the "is the cluster itself healthy?" single-pane-of-glass that an on-call engineer opens first when they receive any alert. No per-service detail.

2. **SLO compliance across tenants** — one row per team: current error ratio vs. SLO target, error budget remaining as a percentage and in minutes, and a 28-day trend sparkline. This is used in weekly reliability reviews, release-gate decisions, and during incidents to understand cross-team blast radius.

3. **Capacity and bin-packing** — per-namespace CPU and memory quota utilisation, HPA current vs. max replicas, node pool size vs. autoscaler target. Used for capacity planning conversations and for the run-up to anticipated traffic increases.

4. **Incident triage** — per-namespace error rates, OOMKill events, pod restart rates, and top-10 Loki error lines by namespace. Sparse, high-density, and scoped to the last 1 hour. Opened on every page as the first-stop context before diving into service-specific dashboards.

**Structuring dashboards for product team self-service:**

Each team gets a namespace-scoped service dashboard built from a shared Grafana dashboard template. The template uses a `namespace` variable as the filter so each team can change it to their own namespace. I believe in the power of variables, and I think it's good to use them as much as possible. The template covers: error rate, request rate, p95 latency, pod memory and CPU, and links to Loki and Tempo filtered to that namespace. When a team needs to investigate their own service they should never need to ask the platform team — everything is scoped to their namespace and the template is in Git so they can extend it.

The critical practice is not repeating yourself: common panels (cluster node health, control plane latency) live only in platform dashboards. If a team copies those panels into their service dashboard, they create a maintenance burden and a source of inconsistency.

**Triage dashboard vs. deep-dive dashboard:**

A **triage dashboard** answers one question in under two minutes: is there a problem and how bad is it? It is sparse, uses large font sizes, and covers 5–10 key metrics at the 1h time range. The on-call engineer opens it immediately after being paged and makes a go/no-go decision on whether to escalate. No links to other systems, no panels that require interpretation.

A **deep-dive dashboard** answers: why is there a problem? It has per-pod breakdowns, log links, trace links, longer time ranges (6h, 24h), and multiple sub-panels for correlated signals. It is opened after triage has confirmed an active problem and the engineer needs root-cause context.

The mistake I have seen most often is building a single dashboard that tries to do both. It ends up being too detailed to triage quickly and not detailed enough to diagnose properly.

---

## Part 5 — Toil & Automation

### 5.1 — Identify and Prioritise Toil

> Describe three examples of toil you would expect to find on a Kubernetes platform team running a shared multi-tenant cluster. For each one: what makes it toil (rather than valuable engineering work), how you would measure how much time it consumes, and how you would prioritise which to automate first.

**Example 1 — Manual tenant onboarding**

When a new product team or Life Sciences partner is added to the shared cluster, someone on the platform team must manually: create the namespace, apply a `ResourceQuota`, a `LimitRange`, namespace-scoped RBAC, network policies, a Loki logging config, and a monitoring namespace label. This is ~15 sequential steps with no judgement required after the first time.

*Why it is toil*: it is repetitive, manual, and grows linearly with team count. It adds no engineering value beyond the first time someone designed the process.

*How to measure*: count onboarding tickets in Jira/Linear over the last quarter, multiply by average time-per-ticket from time-tracking. In practice teams underreport this because it feels like "just a few minutes" — a better measure is calendar interruptions, since onboarding usually blocks someone mid-task.

*How to automate*: a GitOps-driven onboarding template repository. A new team opens a pull request adding a `teams/team-name.yaml` config file. A CI pipeline validates the config and an ArgoCD `ApplicationSet` generates all the required manifests automatically. The platform team reviews one PR instead of executing 15 manual steps.

**Example 2 - Manually responding to "which team or what is causing node pressure"**

On a shared cluster, node memory or CPU pressure alerts frequently require a platform engineer to manually run `kubectl describe node`, find the top resource consumers, cross-reference pod labels to teams, and then notify the right team via Slack. The same investigation steps are repeated each time. I have actually developed a tool in my current position that runs `kubectl top` for comparing the utilization of the resources to what we have provisioned. The percentile we get over a prolonged period of time helps us scale them down and save money, but also catch those we have underprovisioned.

*Why it is toil*: the investigation is scripted — anyone who has done it once can write the exact commands. The value is in the notification and the conversation that follows, not in the lookup itself.

*How to measure*: count Slack threads containing "which pod is using all the memory on node X" — a strong proxy for unautomated investigation toil.

*How to automate*: add a Grafana alert that fires when any namespace exceeds a threshold percentage of node resources, with an annotation that names the namespace and links to the relevant dashboard. The alert routes to the team's own channel. No platform engineer involvement needed.

**Example 3 — Manual rotation of Kubernetes secrets for external credentials**

API keys, database passwords, and Azure service principal secrets expire or must be rotated on a schedule. Each rotation requires: updating the secret in Azure Key Vault, updating the corresponding Kubernetes Secret, and restarting the relevant pods to pick up the new value. With many services this is a weekly interruption.

*Why it is toil*: it is manual, error-prone (the wrong namespace or secret name causes an incident), and purely mechanical. There is no engineering judgement involved.

*How to measure*: count rotation tickets, and count incidents caused by missed or botched rotations (a hidden cost that makes the real toil burden higher than it appears).

*How to automate*: the [External Secrets Operator](https://external-secrets.io/) with Azure Key Vault as the backend syncs secrets into Kubernetes automatically and triggers pod restarts via annotations. Secret rotation becomes a Key Vault operation with no manual Kubernetes step. This is also the right answer for SaMD audit trail requirements — Key Vault has immutable audit logs for every secret access and rotation.

**Prioritisation approach:**

I prioritise by `(time consumed per week) × (frequency of interruption) × (incident risk if done wrong)`. Onboarding (#1) is high frequency with low incident risk; secret rotation (#3) is lower frequency but has caused incidents when done manually; node pressure investigation (#2) is frequent but low-risk. I would automate #3 first (highest incident risk), then #1 (highest frequency), then #2.

### 5.2 — Implement the Automation

> Your completed [`part5/cronjob.yaml`](./part5/cronjob.yaml) is the primary answer.
>
> Briefly address here: (a) Kubernetes already provides `ttlSecondsAfterFinished` and a pod GC, so what does your CronJob add on top, and would you pick a different target? (b) Cluster-wide `get/list/delete` on pods and jobs in a PHI-handling multi-tenant cluster: what are the security implications and how would you mitigate them?

**What this CronJob adds on top of native Kubernetes GC:**

Kubernetes provides `ttlSecondsAfterFinished` on Jobs and a pod garbage collector (default threshold: 12,500 terminated pods cluster-wide). These cover the common case, but they miss three categories that accumulate on a busy shared cluster:

- **Evicted pods** — pods with `status.phase=Failed` and `reason=Evicted` are cleaned up by the GC but only after the threshold is reached. On a cluster with many tenant namespaces the threshold is effectively never hit, so evicted pods persist indefinitely.
- **Jobs without `ttlSecondsAfterFinished`** — many teams do not set this field, particularly for Jobs created by older CI/CD pipelines or third-party operators. The CronJob provides a backstop.
- **Completed pods from deleted Jobs** — when a Job's owner reference is removed (e.g., the Job was force-deleted), the pods it owned become orphaned and are not touched by the Job GC. The phase-based field selector in this CronJob catches them.

A potentially more valuable target for a cluster running Argo Workflows would be completed `Workflow` objects — these accumulate heavily and consume etcd space — but that requires the `argoproj.io` API group in the ClusterRole and falls outside the scope of this task.

**Security implications of cluster-wide pod/job access in a PHI-handling multi-tenant cluster:**

This ClusterRole grants `get`, `list`, and `delete` on pods and jobs across all namespaces. The risks are:

- **Tenant data leakage via metadata**: pod names, labels, owner references, and image tags can reveal tenant identifiers, deployment versions, and internal service names. A compromised or misbehaving cleanup job could exfiltrate this metadata.
- **Accidental deletion of non-stale resources**: a bug in the jq filter or a clock skew issue could cause deletion of pods that are actually in use.
- **Privilege escalation surface**: if the ServiceAccount token were stolen (e.g., via a compromised CI system), an attacker could delete pods across all namespaces as a denial-of-service.

Mitigations I would apply in production:

1. **Kubernetes audit logging** — enable audit logging for all `delete` actions by this ServiceAccount. Any unexpected deletion is immediately visible and attributable.
2. **Namespace exclusion list** — add logic to skip `kube-system`, `kube-public`, and other platform namespaces from pod deletion to reduce the blast radius of a bug.
3. **Admission policy (OPA/Kyverno)** — enforce that the ServiceAccount can only delete pods with specific phase labels, not arbitrary pods. This is belt-and-braces but meaningful in a regulated environment.
4. **Image digest pinning** — pin `alpine/k8s:1.33.0` to a specific image digest in production to prevent a supply-chain substitution of the cleanup binary.

---

## Part 6 — Operational Scenarios

### 6.1 — Cascading Failure

> Walk through your investigation of the cascading failure described in the README. How do you identify the root cause as Service B (rather than Service A), and what do you do about it?

**Initial alert**: `TeamAlphaBackendHighErrorBudgetBurn` fires — Service A is returning 503s.

**First check — is Service A itself broken?**
```bash
kubectl get pods -n service-a -l app=service-a -o wide
kubectl top pods -n service-a -l app=service-a
kubectl describe pods -n service-a -l app=service-a | grep -A5 "OOMKilled\|Reason"
```
Pods are running, no OOMKill, no resource pressure. Service A looks healthy internally.

**Second check — Service A's own logs:**
```logql
{namespace="service-a"} | json | level =~ "error" | line_format "{{.msg}}"
```
The logs show: `upstream request to service-b timed out after 30s` and `connection pool exhausted`. Service A is failing because it is waiting too long for Service B responses, and its thread pool has filled up waiting.

**Third check — Service B metrics:**
```promql
# Service B error rate — this will show near zero
sum(rate(http_requests_total{job="service-b", status_code=~"5.."}[5m]))
/ sum(rate(http_requests_total{job="service-b"}[5m]))

# Service B p99 latency — this will show 8s instead of 200ms
histogram_quantile(0.99, sum by (le) (
  rate(http_request_duration_seconds_bucket{job="service-b"}[5m])
))
```

This is the key insight: **Service B is not returning errors, so no error-rate alert fired for it**. But its latency has increased 40×. This is a classic slow-poison cascading failure — the downstream is responsive but slow, which is worse than being down because the upstream keeps waiting instead of failing fast.

**How I confirm the root cause is Service B:**
Distributed traces are the fastest path. In Tempo I search for traces where `service-a` is the root span and look at the waterfall. The majority of wall-clock time will be inside a child span labelled `http.client → service-b`. The service-b span has a 7–8 second duration. Service A is not doing anything wrong — it is just waiting.

**What I do about it:**
Immediate: if Service A has a configurable timeout on its Service B calls, reduce it to ~500ms and add a fallback response (degrade gracefully). This drains the thread pool and restores Service A within 30–60 seconds.

Follow-up: investigate why Service B's latency increased. Check its own dependencies (database, external API). Add a latency-based SLO alert to Service B — `TeamAlphaBBackendHighP99Latency` — so that future slow-poison failures are caught at the source before they cascade.

### 6.2 — On-Call Handoff During an Active Incident

> You are 30 minutes into a P1 incident (as defined in the README) when your shift ends. How do you hand off? What do you communicate, in what format, to ensure the incoming engineer can pick up without losing ground?

The handoff note goes into the incident Slack channel and is linked from the PagerDuty ticket. I use a fixed structure so the incoming engineer can orient in under 90 seconds:

---

**P1 HANDOFF — team-alpha-backend — 02:44**

**Current state**: DEGRADED — service is partially restored after ArgoCD rollback; error rate dropped from 40% to ~5% and still declining. Not fully resolved yet.

**Impact**: ~40% peak error rate for 30 minutes. All patient-facing workflows through team-alpha-backend were affected. See #incident-20260528 for timeline.

**What I've done**:
- 02:20 — confirmed OOMKill via `kubectl describe` (exit code 137, all 3 pods)
- 02:30 — ruled out traffic spike (request rate was flat since 01:30)
- 02:35 — confirmed root cause is the 01:55 release via memory growth chart
- 02:42 — initiated ArgoCD rollback to revision 47 (v0.9.8)

**What I have NOT done**:
- Have not confirmed whether any in-flight patient data submissions were lost — needs verification against the audit log in Azure Monitor.
- Have not notified the dev team lead (Faisal) — this should happen now.

**Current hypothesis**: rollback will fully resolve the issue within the next 3–5 minutes as the remaining 5% error rate is residual from pods still restarting.

**Active handles**:
- Incident: #incident-20260528 in Slack
- ArgoCD rollout: [link to ArgoCD app]
- Grafana dashboard: [link filtered to team-alpha, last 1h]

**Next step**: watch error rate in Grafana for 5 minutes. If it reaches 0%, close as resolved. If it remains elevated after the rollback completes, the issue predates the 01:55 release — run `kubectl logs --previous` on any still-restarting pod to look for a different root cause.

---

The three principles behind this format: lead with current state (not history), clearly separate what is ruled out from what has not been tried, and leave exactly one concrete next action so the incoming engineer can act immediately without having to re-read everything.

### 6.3 — Noisy Alerting _(Optional)_

> How would you audit and reduce 40–60 daily alert notifications (with ~5 after-hours pages per week, ~80% non-actionable) without reducing coverage for real incidents?

I see alerting as a thing that requires constant altering, as it's sometimes quite difficult to find the sweet spot for thresholds. I believe it is often not quite possible to set a one-off perfect threshold from the go, so I would say that it takes some time and care.

---

## Anything Else?

> Note any tasks you skipped and how you would have approached them, assumptions you made, or anything else you would like the reviewers to know.

N/A
