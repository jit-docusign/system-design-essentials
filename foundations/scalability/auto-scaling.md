# Auto-Scaling

## What it is

**Auto-scaling** is the ability to automatically adjust the number of running instances of a service — adding capacity when demand rises and removing it when demand falls — without manual intervention. It ensures the system handles traffic spikes without over-provisioning during quiet periods.

Auto-scaling is a critical component of modern cloud infrastructure and is required to operate cost-effectively at variable load.

---

## How it works

### Auto-Scaling Loop

Auto-scaling works on a continuous monitoring and adjustment cycle:

```
Monitor metrics → Evaluate scaling policy → Make scaling decision → Apply change
      ↑                                                                    │
      └────────────────────────────────────────────────────────────────────┘
```

1. **Metrics are collected**: CPU utilization, memory, request rate, response time, queue depth.
2. **Scaling policy is evaluated**: is the metric above the scale-out threshold? Below the scale-in threshold?
3. **Scaling decision**: add N instances (scale-out) or remove N instances (scale-in).
4. **Instances are provisioned/deprovisioned**: typically takes 1–5 minutes for a new VM; under 30 seconds for a container.
5. **Loop repeats**.

### Scale-Out Triggers (Add Capacity)

| Metric | Scale-Out Rule (example) |
|---|---|
| CPU utilization | > 70% for 5 consecutive minutes → add 2 instances |
| Memory utilization | > 80% → add instances |
| HTTP request rate | > 5,000 req/s per instance → add instances |
| Response time (P99) | > 500ms → add instances |
| Message queue depth | > 1,000 unprocessed messages → add worker instances |
| Custom business metric | Active sessions > 10,000 per instance |

### Scale-In (Remove Capacity)

Scale-in is more conservative than scale-out to prevent oscillation (rapidly adding and removing instances):
- Scale-in threshold is lower than scale-out threshold.
- Cooldown period: after scaling in, wait N minutes before evaluating again.
- Scale-in happens gradually: remove 1–2 instances at a time.
- Before terminating an instance, gracefully drain in-flight requests (connection draining).

### Target Tracking vs Step Scaling

**Target tracking scaling (preferred)**: set a target metric value and let the auto-scaler determine how many instances to add/remove to maintain it.
```
Target: CPU utilization = 50%
Current: CPU = 80% → add instances automatically
Current: CPU = 20% → remove instances automatically
The number added/removed is calculated to bring CPU back to 50%
```

**Step scaling**: define specific steps for different metric ranges.
```
CPU 70–80% → add 1 instance
CPU 80–90% → add 2 instances
CPU > 90%  → add 5 instances
```
More control; more configuration.

**Scheduled scaling**: add capacity at predictable times (before known traffic spikes):
```yaml
Schedule:
  - name: morning-peak
    recurrence: "0 8 * * 1-5"   # 8am weekdays
    min_capacity: 20
    max_capacity: 50
  - name: night-low
    recurrence: "0 23 * * *"     # 11pm every day
    min_capacity: 5
    max_capacity: 15
```

Scheduled scaling is useful for predictable patterns (business hours, end-of-month billing runs, Black Friday).

### Pre-Warming vs Reactive Scaling

**Reactive scaling (default)**: scale *after* the metric breaches the threshold. Problem: a sudden traffic spike hits while new instances are still booting (typically 1–5 min for VMs, 10–30s for containers).

```
Traffic spike at T=0
New instances requested at T=1min (threshold crossed + evaluation period)
Instances ready at T=4min (provisioning time)
Users experience degradation for ~4 minutes
```

**Pre-warming strategies:**
1. **Predictive auto-scaling** (AWS Fargate predictive, etc.): uses ML to predict traffic patterns and adds capacity proactively.
2. **Scheduled scaling**: add minimum capacity ahead of known peaks.
3. **Monitoring and alerting on trend** (not just threshold): if request rate is rising 20% per minute, scale before hitting the CPU threshold.
4. **Smaller, faster-starting containers vs large VMs**: containers start in seconds; VMs take minutes.
5. **Minimum instance count**: never scale to zero for latency-sensitive services. Keep a minimum "always on" capacity.

### Auto-Scaling for Different Tiers

| Tier | Auto-scale on | Notes |
|---|---|---|
| Stateless application servers | CPU, request rate, P99 latency | Easiest to auto-scale — stateless |
| Message consumers / workers | Queue depth | Scale workers proportional to backlog |
| Background job processors | Job queue length | Same pattern as queue consumers |
| Cache (Redis, Memcached) | Memory utilization, hit rate | Harder; cluster resharding is disruptive |
| Database (read replicas) | Read query rate, CPU | Add read replicas; never auto-scale writes |
| Database (primary) | Not recommended | Vertical scaling first; sharding is complex |

### Kubernetes Horizontal Pod Autoscaler (HPA)

In Kubernetes, the HPA scales the replica count of a Deployment based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60      # target: 60% average CPU
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"        # 1000 req/s per pod
```

HPA also supports KEDA (Kubernetes Event-Driven Autoscaling) for scaling based on external queues (Kafka, SQS, RabbitMQ).

### Scale-In Graceful Termination

When an instance is terminating, in-flight requests must complete or be re-routed:

1. Instance receives SIGTERM.
2. Instance stops accepting new connections (removed from LB target group).
3. Instance processes in-flight requests (connection draining period: typically 30–300 seconds).
4. Instance exits and is terminated.

```yaml
# Kubernetes: allow 60s for graceful shutdown
spec:
  terminationGracePeriodSeconds: 60
```

In AWS, ASG lifecycle hooks allow a running instance to complete work before terminating.

---

## Key Trade-offs

| Benefit | Challenge |
|---|---|
| Cost efficiency — pay for what you use | Provisioning lag → degradation during sudden spikes |
| Handles variable load automatically | Wrong metric selection leads to poor decisions |
| No manual capacity planning for routine changes | Scale-in must handle graceful shutdown / draining |
| Reduces engineer toil | Stateful services can't scale out easily |
| Improves availability (adds capacity before failure) | Configuration tuning required (thresholds, cooldowns) |

---

## Common Pitfalls

- **Scaling on the wrong metric**: CPU is not always the right metric. If your service is I/O-bound, CPU stays low while latency degrades. Scale on P99 latency or queue depth instead.
- **Scale-in without drain**: terminating instances while requests are in-flight causes errors. Always configure connection draining and graceful shutdown.
- **Oscillation**: low cooldown periods cause add/remove cycles that disrupt users. Use sufficient cooldown after scale-in (5–10 minutes typical).
- **Scaling to zero**: for latency-sensitive services, scaling to zero means the first user after idle must wait for a cold start (30s–5min). Keep a minimum instance count.
- **Not testing auto-scaling**: auto-scaling policies should be load-tested before production reliance. Surprise behavior during an actual traffic spike is the worst time to discover misconfiguration.
