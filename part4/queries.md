# Part 4.1 — Observability Investigation

## Scenario

The on-call engineer is paged at 02:30: the error rate on `team-alpha-backend` has
risen to 8% over the past 10 minutes. The pods are running. No deployment has
occurred in the last 4 hours.

You have access to:

- **Grafana / Mimir** — Prometheus-compatible query interface (metrics)
- **Loki** — LogQL query interface (logs)
- **Tempo** — trace search and waterfall view
- **kubectl** on the cluster

---

## Task A — Metrics Triage (PromQL)

### A1. What is the current error rate for `team-alpha-backend`?

```promql
# 5xx error rate as a fraction of all requests over the last 5 minutes.
# 5m matches the alert cadence and is short enough to reflect current state.
# rate() handles counter resets (e.g. pod restart mid-window) gracefully.

sum(
  rate(http_requests_total{
    namespace="team-alpha",
    job="team-alpha-backend",
    status_code=~"5.."
  }[5m])
)
/
sum(
  rate(http_requests_total{
    namespace="team-alpha",
    job="team-alpha-backend"
  }[5m])
)
```

### A2. Is the error rate affecting all pods or just one?

```promql
# Per-pod error ratio — if one pod shows ~50% errors and others show 0%,
# the issue is isolated to that pod (bad restart, config drift, etc.).
# A uniform error rate across all pods suggests a shared dependency failure.

sum by (pod) (
  rate(http_requests_total{
    namespace="team-alpha",
    job="team-alpha-backend",
    status_code=~"5.."
  }[5m])
)
/
sum by (pod) (
  rate(http_requests_total{
    namespace="team-alpha",
    job="team-alpha-backend"
  }[5m])
)
```

### A3. Has memory or CPU usage changed in the last hour?

```promql
# Memory: working_set_bytes excludes reclaimable page-cache memory, making it
# the most accurate measure of how close a container is to its memory limit.
# container_memory_usage_bytes would overstate usage by including file cache.

container_memory_working_set_bytes{
  namespace="team-alpha",
  pod=~"team-alpha-backend.*",
  container="backend"
}
```

```promql
# CPU usage rate per pod expressed in cores (0.2 = 200m).
# A spike here without a traffic spike may indicate a runaway goroutine / thread.

sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="team-alpha",
    pod=~"team-alpha-backend.*",
    container="backend"
  }[5m])
)
```

### A4. Are request latencies elevated?

```promql
# p95 latency using histogram_quantile.
# le must be in the sum by clause — omitting it collapses the histogram
# buckets and produces a meaningless result.
# Compare this value against the normal baseline (check the 24h graph in Grafana).

histogram_quantile(0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{
      namespace="team-alpha",
      job="team-alpha-backend"
    }[5m])
  )
)
```

---

## Task B — Log Investigation (LogQL)

### B1. Show all error-level logs from `team-alpha-backend` in the last 15 minutes

```logql
# Structured JSON logs — parse the level field and filter.
# json unpacks the log line; level filters on the parsed field.

{namespace="team-alpha", pod=~"team-alpha-backend.*"}
  | json
  | level =~ "(?i)error|fatal"
```

```logql
# Fallback for unstructured or mixed-format logs.
# Line filter with case-insensitive regex captures ERROR, Error, error, FATAL, etc.

{namespace="team-alpha", pod=~"team-alpha-backend.*"}
  |~ "(?i)(error|exception|fatal)"
```

### B2. Count errors by message pattern to identify the most common failure

```logql
# Metric query — counts error log lines grouped by the parsed msg field.
# topk(10) surfaces the most frequent error messages in a table or bar chart.
# Use this to distinguish a single noisy error from many distinct failures.

topk(10,
  sum by (msg) (
    count_over_time(
      {namespace="team-alpha", pod=~"team-alpha-backend.*"}
        | json
        | level =~ "(?i)error|fatal"
      [15m]
    )
  )
)
```

### B3. Is there a timing pattern — are errors arriving in bursts or steadily?

```logql
# Rate of error log lines per second over 1-minute buckets.
# A burst pattern (spikes separated by silence) suggests a periodic job or
# connection retry loop. A steady rate suggests a persistent failure.

rate(
  {namespace="team-alpha", pod=~"team-alpha-backend.*"}
    |~ "(?i)(error|exception|fatal)"
  [1m]
)
```

---

## Task C — Connecting the Dots

Based on what the above queries reveal, here is how I would use Tempo to narrow down the root cause.

**What to search for in Tempo:**

Filter traces by service name `team-alpha-backend`, status = `error`, and time range `02:20–02:35`. Sort by duration descending to surface the slowest failing requests first. A trace ID from a Loki error log line (if the app emits one) is the fastest path to the relevant trace.

**What to look for in the trace waterfall:**

In a healthy trace the spans are narrow and sequential. In this incident I am looking for:

- A span that accounts for the majority of the total trace duration (wall-clock time is spent waiting there).
- A span with a red error tag — `error=true` or `http.status_code=5xx`.
- Whether the long/erroring span belongs to `team-alpha-backend` itself or to a child span representing an outbound call to a database, a downstream service, or an external API.

**Distinguishing where the error originates:**

- **Error in `team-alpha-backend` itself**: the root span carries the error tag; child spans (if any) complete normally and are short. The application code is the problem — look at the stack trace attached to the root span.
- **Error originating in a downstream dependency**: the root span is long but the error tag is on a child span (e.g. `db.query` or `http.client → service-b`). The parent span is simply waiting. This is the classic cascading failure pattern — the downstream slowness fills the upstream thread pool. In this case I would pivot to investigating the downstream service's own metrics and traces.

Since no deployment has occurred in 4 hours, I would first check whether a downstream dependency's latency or error rate changed around 02:20, which would explain why `team-alpha-backend` started failing without any local change.

