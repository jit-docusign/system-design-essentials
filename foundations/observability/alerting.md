# Alerting

## What it is

**Alerting** is the automated process of notifying the right people when a system deviates from expected behavior in a way that requires human action. Good alerting is the bridge between observability data (metrics, logs, traces) and your on-call team.

The goal is **actionable alerts with minimal noise**: every alert should represent a situation that a human needs to investigate and resolve. Alerts that don't require action are noise that leads to alert fatigue — engineers learn to ignore alerts, and the next real incident gets missed.

---

## How it works

### The Alerting Pipeline

```
Metrics/Logs  ──► Alert Rules ──► Alert Manager ──► Notification Channels
(Prometheus)      (PromQL/       (Group, Route,      (PagerDuty, Slack,
                  LogQL rules)   Silence, Inhibit)    Email, OpsGenie)
```

### Alert Rule Anatomy (Prometheus)

```yaml
# prometheus-rules.yaml
groups:
  - name: order-service.rules
    interval: 30s   # how often to evaluate
    rules:

      # Symptom-based: user-facing impact
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{service="order-service", status_code=~"5.."}[5m])
            /
            rate(http_requests_total{service="order-service"}[5m])
          ) > 0.01    # > 1% error rate
        for: 5m       # must be true for 5 consecutive minutes (avoids flapping)
        labels:
          severity: critical
          team: orders
        annotations:
          summary: "Order service error rate above 1%"
          description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
          runbook: "https://runbooks.example.com/order-service/high-error-rate"
          dashboard: "https://grafana.example.com/d/orders/order-service?from=now-1h"

      # Latency SLO alert
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket{service="order-service"}[5m])
          ) > 1.0    # p99 > 1 second
        for: 5m
        labels:
          severity: warning
          team: orders
        annotations:
          summary: "Order service p99 latency above 1s"
          runbook: "https://runbooks.example.com/order-service/high-latency"

      # Saturation alert
      - alert: DBConnectionPoolExhausted
        expr: |
          db_connection_pool_size{state="waiting"} > 5
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Database connection pool has waiting requests"
```

### Alertmanager Configuration

Alertmanager handles deduplication, grouping, routing, and silencing:

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'service', 'team']
  group_wait: 30s        # wait 30s before sending initial group notification
  group_interval: 5m     # wait 5m before sending next notification for same group
  repeat_interval: 4h    # re-notify after 4h if still firing

  receiver: default
  routes:
    # Critical alerts go to PagerDuty (wake someone up)
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: false

    # Warnings go to Slack (inform, don't wake up)
    - match:
        severity: warning
      receiver: slack-warnings

    # Database alerts go to DBA team
    - match:
        team: platform
      receiver: platform-team-slack

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_INTEGRATION_KEY}"
        description: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
        links:
          - href: "{{ (index .Alerts 0).Annotations.runbook }}"
            text: "Runbook"
          - href: "{{ (index .Alerts 0).Annotations.dashboard }}"
            text: "Dashboard"

  - name: slack-warnings
    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL}"
        channel: "#alerts-warnings"
        title: "[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"

inhibit_rules:
  # If the service is completely down (no traffic), suppress latency alerts
  - source_match:
      alertname: ServiceDown
    target_match_re:
      alertname: (HighP99Latency|HighErrorRate)
    equal: ['service']
```

### SLO-Based Alerting (Multi-Burn-Rate)

The gold standard for user-impacting alerts. Alert on **error budget burn rate** rather than raw error rate. This gives both speed (detect fast burns early) and accuracy (avoid waking people up for transient spikes).

```
Error budget: 30-day SLO of 99.9% availability
  Budget: 0.1% of 30 days = 43.8 minutes of downtime allowed per month

Burn rate examples:
  1x burn rate: consuming budget at exactly the SLO rate (no alert needed)
  14x burn rate: will exhaust 30-day budget in ~51 hours → alert
  6x burn rate:  will exhaust 30-day budget in ~5 days → warning
```

```yaml
# Multi-burn-rate alert (Google SRE Workbook pattern)
- alert: ErrorBudgetBurnFast
  # Fast burn: exhausts 2% of budget in 1 hour = 14.4x burn rate
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[1h]) / rate(http_requests_total[1h])
    ) > 14.4 * (1 - 0.999)  # 1 - SLO
    AND
    (
      rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
    ) > 14.4 * (1 - 0.999)
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Fast error budget burn: at this rate budget exhausts in ~2 days"

- alert: ErrorBudgetBurnSlow
  # Slow burn: exhausts 5% of budget in 6 hours = 6x burn rate
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[6h]) / rate(http_requests_total[6h])
    ) > 6 * (1 - 0.999)
    AND
    (
      rate(http_requests_total{status=~"5.."}[30m]) / rate(http_requests_total[30m])
    ) > 6 * (1 - 0.999)
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Slow error budget burn: at this rate budget exhausts in ~5 days"
```

### Alert Severity Levels

| Severity | Meaning | Response | Examples |
|---|---|---|---|
| **Critical / P1** | User-facing impact right now | Page on-call immediately (24/7) | Error rate > 1%, service down, data loss |
| **Warning / P2** | Trending toward impact, not there yet | Notify during business hours | Error rate rising, high latency, disk >80% |
| **Info / P3** | Informational, investigate proactively | Slack notification, no action required | Deployment completed, cert expires in 30 days |

### Alert Fatigue Prevention

```
Symptoms of alert fatigue:
  - On-call engineers routinely acknowledge and ignore alerts
  - Alerts fire but get resolved without investigation
  - No runbook — engineer doesn't know what to do
  - Same alert fires multiple times for the same root cause

Fixes:
  1. Every alert must have a runbook (what to investigate, what to do)
  2. Every alert must be actionable — if no action is needed, demote to log/metric
  3. Tune thresholds — an alert that fires daily isn't useful if it's "normal"
  4. Use inhibit rules — suppress child alerts when parent (cause) is already alerting
  5. Use `for` duration — avoid single-datapoint flapping alerts
  6. Track alert frequency — delete or fix alerts that fire more than 3x/week without human action
```

### Runbooks

Every alert must link to a runbook — a document that answers:
1. What is this alert detecting?
2. What is the user impact?
3. Quick checks: what do I look at first?
4. Diagnosis steps: how to narrow down root cause?
5. Mitigation steps: how to stop the bleeding?
6. Escalation path: who to wake up next?

---

## Key Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| **Symptom-based (error rate)** | Directly measures user impact | May fire late (after impact started) |
| **Cause-based (CPU > 90%)**  | Early warning | Often doesn't → user impact; high false positive rate |
| **SLO burn rate** | Precision — exhaustion-proportional urgency | PromQL complexity; requires SLO definition first |
| **Aggressive thresholds** | Catches problems early | Alert fatigue; training people to ignore alerts |
| **Conservative thresholds** | Low noise | Real incidents missed or caught late |

**Prefer symptom-based over cause-based alerting.**

---

## When to use

- Always alert on user-facing symptoms first: error rate, latency SLO, availability.
- Use warning-level alerts for leading indicators: rising error trends, approaching resource limits.
- Use cause-based alerts (CPU, disk, memory) as warnings only, never as critical pages, unless they directly and quickly → user impact.

---

## Common Pitfalls

- **Missing `for` duration**: without a `for: 5m` clause, a single bad scrape or transient spike triggers a page. Always add a minimum duration to prevent flapping.
- **No runbook link**: an alert that fires at 3 AM without a runbook forces the on-call engineer to debug from scratch under pressure. Every alert must have a runbook URL in its annotations.
- **Alerting on every resource metric**: not every `CPU > 80%` is an incident. Alert on CPU only if it's impacting request latency or causing errors — tie cause-based alerts to symptom evidence.
- **Not testing alert rules**: PromQL is easy to get wrong. Use `promtool test rules` to unit test alert conditions before deploying them.
- **Sending all alerts to all teams**: use routing rules to send alerts to the team that owns the service. All-hands alerts cause "someone else will handle it" diffusion of responsibility.
