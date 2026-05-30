# Runbook — team-alpha-backend: Memory Exhaustion / OOMKill

## Overview

**Alert**: `ContainerMemoryUsageCritical`
**Severity**: Critical
**Service**: `team-alpha-backend` in namespace `team-alpha`
**Typical cause**: Memory leak in the application, or a sudden traffic spike exceeding configured resource limits

---

## Triage (first 2 minutes)

**Step 1 - Check pod status and restart counts.**
High `RESTARTS` combined with a recent `AGE` value confirms crash-looping.

```bash
kubectl get pods -n team-alpha -l app=team-alpha-backend -o wide
```

**Step 2 - Check live resource usage.**
Memory near 512Mi (the configured limit) confirms memory pressure is the issue, not the CPU or network.

```bash
kubectl top pods -n team-alpha -l app=team-alpha-backend
```

**Step 3 - Check for OOMKilled containers across all pods.**
Exit code 137 = kernel SIGKILL due to memory limit breach. This confirms OOMKill rather than an app crash.

```bash
kubectl describe pods -n team-alpha -l app=team-alpha-backend | grep -A10 -iE "Last State:|Reason:|Exit Code:"
```

**Step 4 - Confirm service-level error rate in Mimir/Grafana.**

```promql
sum(rate(http_requests_total{namespace="team-alpha", job="team-alpha-backend", status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total{namespace="team-alpha", job="team-alpha-backend"}[5m]))
```

If the ratio is `> 0` and rising, service impact is confirmed. Proceed to Diagnosis.

*Order matters. Steps 1-3 take under 30 seconds and establish scope before you start making changes. Never skip to Resolution without confirming the blast radius.*

---

## Diagnosis

### Is this a memory leak or a traffic spike?

**Check memory trend over the past 2 hours:**

```promql
container_memory_working_set_bytes{
     namespace="team-alpha",
     pod=~"team-alpha-backend.*",
     container="backend"
}
```
- **Steady upward slope over hours with flat request rate -> `memory leak`**
- **Sudden jump that correlates with a request rate increase -> `traffic spike`**


**Check request rate over the same window to compare:**

```promql
sum(rate(http_requests_total{namespace="team-alpha", job="team-alpha-backend"}[5m]))
```
If request rate is flat but memory grows linearly = `leak`. If both spiked together = `traffic`

### Is the container OOMKilling?

```bash
kubectl describe pod <pod-name> -n team-alpha
```

Look for this block in the output:
```
Last State:      Terminated
     Reason:     OOMKilled
     Exit Code:  137
```
Exit code **137** = OOMKilled (128 + signal 9/SIGKILL). A normal application restart has exit code 0 or 1. A crash due to unhandled exception typically shows 1 or 2.

### Is this isolated to one pod or affecting the node?

**Identify the node and check its memory pressure condition:**

```bash
kubectl get pods -n team-alpha -l app=-team-alpha-backend -o wide
kubectl describe node <node-name> | grep -A5 -iE "memorypressure|allocatable|allocated"
```

- `MemoryPressure: True` on the node -- node-level issue; other tenants may be affected. Escalate to platform team immediately.
- `MemoryPressure: False` -- issue is pod-level only; stay with the application team.

**Check what else is running on the node:**

```bash
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> | grep -v Running
```
Any non-running pods from other namespaces on the same node indicate broader impact.

---

## Resolution

### Immediate mitigation

**If a recent deployment is the cause (memory leak introduced by a release):**

Roll back via ArgoCD (preferred in a GitOps model - this keeps git as the source of truth):

1. Open ArgoCD UI > select `team-alpha-backend` app
2. Click **History and Rollback** > select the last known-good revision
3. Click **Rollback** and confirm

**Or** via `kubectl` to buy seconds while ArgoCD syncs - this creates drift, so update Git immediately after:

```bash
kubectl rollout undo deployment/team-alpha-backend -n team-alpha
# And check
kubectl rollout status deployment/team-alpha-backend -n team-alpha
```

**If no recent deployment and the issue appears to be a traffic spike:**

Scale out to spread load across more pods and reduce per-pod memory pressure:

```bash
# Temporary only - raise a PR to update replicas in the Deployment manifest before end of incident
kubectl scale deployment/team-alpha-backend -n team-alpha --replicas=6
```

### Root cause fix

Once the service is stable:

1. Notify the owner development team with the memory growth chart and the deployment that introduced the leak.
2. Request heap profiling or memory analysis in staging under a representative load test.
3. Require the fix to include a memory regression assertion in the CI pipeline: if `container_memory_working_set_bytes` grows more than 10%  under constant load, the build test fails.
4. Verify the fix by monitoring memory for at least 2 hours post-deployment without growth under normal traffic.


---

## Escalation

| Condition | Escalate to |
|-----------|-------------|
| All pods OOMKilled, error rate > 50%, service fully down | Page dev team lead immediately; open a P1 incident channel |
| Node `MemoryPressure: True`, other tenant pods affected | Reach Platform team on-call - this is now a cluster level incident |
| OOMKill recurs within minutes of rollback (issue predates the release) | Engage app team for emergency fix; consider temporarily setting a higher memory limit as a bridge |
| Any patient-safety-relevant workflow confirmed disrupted | Security and compliance team; document impact for SaMD audit trail |

---

## Prevention

**1. Add a pre-OOMKilll memory warning alert** (fires at 80%, before the kernel kills the container):

```yaml
- alert: TeamAlphaBackendMemoryHighWatermark
  expr: |
     container_memory_working_set_bytes{
          namespace="team-alpha", pod=~"team-alpha-backend.*", container="backend"
     }
     /
     container_spec_memory_limit_bytes{
          namespace="team-alpha", pod=~"team-alpha-backend.*", container="backend"
     }
     > 0.8
  for: 5m
  labels:
    severity: warning
```

**2. Add a memory growth rate check to detect leaks early** (steady growth without a traffic increase is a strong leak indicator):

```promql
deriv(
     container_memory_working_set_bytes{
     namespace="team-alpha", pod=~"team-alpha-backend.*"
     }[30m]
) > 0
```

If this is positive and sustained for 30+minutes with flat request rate, open a P2.

**3. Introduce canary rollouts via Argo Rollouts** - a 10% canary with a 5-minute memory soak would have exposed the leak before it reached all three pods and caused a complete outage.

**4. Add a memory regression gate to CI/CD** - run a load test in staging and assert that memory does not grow beyond a fixed threshold under constant load before promoting to production.

---

## Related Grafana panels and queries

**Memory working set per pod (view - use to identify leak vs. spike pattern)**
```promql
container_memory_working_set_bytes{namespace="team-alpha", pod=~"team-alpha-backend.*", container="backend"}
```

**OOMKill event counter (non-zero = pod has been OOMKilled recently):**
```promql
kube_pod_container_status_last_terminated_reason{namespace="team-alpha", reason="OOMKilled"}
```

**5xx error rate (use alongside memory chart to confirm correlation):**
```promql
sum(rate(http_requests_total{namespace="team-alpha", job="team-alpha-backend", status_code=~"5.."}[5m]))
/ sum(rate(http_requests_total{namespace="team-alpha", job="team-alpha-backend"}[5m]))
```

