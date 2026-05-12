# Capstone Lab — Resilient, Scalable & Recoverable AWS Architecture

> **Primary Region:** eu-west-1 (Ireland) · **DR Region:** eu-central-1 (Frankfurt)

## Summary

A three-tier web application infrastructure on AWS that survives AZ failures (Multi-AZ HA), scales automatically under load (target-tracking autoscaling on requests and queue depth), and recovers from regional outages (warm-standby DR with 30-min RTO / 5-min RPO). All four rubric pillars demonstrated end-to-end with live infrastructure, executed load tests, and CloudWatch observability.

## Architecture

![Architecture Diagram](docs/architecture-diagram.png)

| Tier            | Components                        | Resilience mechanism                            |
| --------------- | --------------------------------- | ----------------------------------------------- |
| Edge            | ALB ×2 (one per region), Route 53 | Multi-AZ ALBs, DNS failover with health check   |
| Web             | EC2 ASG (AL2023 + nginx)          | 2 instances across 2 AZs (primary), 1 warm (DR) |
| Worker          | ECS Fargate, SQS-driven           | Application Auto Scaling on queue depth         |
| Cache           | ElastiCache Redis                 | Multi-AZ with automatic failover                |
| Relational data | RDS MySQL Multi-AZ                | Cross-region read replica in eu-central-1       |
| State           | DynamoDB Global Tables            | Bidirectional sync, eu-west-1 ↔ eu-central-1    |
| Backup          | AWS Backup                        | Daily snapshots, cross-region copy              |

---

## Rubric Coverage

| Pillar                | Points | Evidence section                                                                                  |
| --------------------- | ------ | ------------------------------------------------------------------------------------------------- |
| **High Availability** | 25     | [Day 1 below](#day-1--high-availability-25-pts), [`spofs.md`](spofs.md)                           |
| **Scalability**       | 25     | [Day 2 below](#day-2--scalability-25-pts), [`docs/load-test/`](docs/load-test)                    |
| **Disaster Recovery** | 25     | [Day 3 below](#day-3--disaster-recovery-25-pts), [`docs/dr/dr-runbook.md`](docs/dr/dr-runbook.md) |
| **Observability**     | 15     | [Day 4 below](#day-4--observability-15-pts), [`docs/observability/`](docs/observability)          |
| **Documentation**     | 10     | This README, runbook, SPOF analysis                                                               |

---

## Day 1 — High Availability (25 pts)

Built a three-tier VPC in `eu-west-1` spanning two AZs (`eu-west-1a`, `eu-west-1b`) with isolated public / private-app / database subnets. Every layer that could be a single point of failure was duplicated.

### Per-AZ NAT Gateways

Two NAT Gateways, one per AZ — rejects the common single-NAT shortcut that introduces cross-AZ failure dependency.

![NAT Gateways](docs/screenshots/1.7%20NAT%20Gateways%20Created.png)

### Multi-AZ Application Load Balancer

ALB spans both AZs; ASG uses ELB-based health checks (catches "instance up, app down" cases).

![ALB created](docs/screenshots/1.10%20create%20ALB.png)

### Auto Scaling Group across two AZs

ASG with `min=2` instances spread across `eu-west-1a` and `eu-west-1b`.

![Launch Template and ASG](docs/screenshots/1.11%20create%20LT%20and%20ASG.png)

### RDS MySQL Multi-AZ

Synchronous standby in a second AZ, encrypted at rest, no public access, credentials in Secrets Manager.

![RDS Multi-AZ](docs/screenshots/1.12%20%E2%80%94%20RDS%20MySQL%20Multi-AZ.png)

### DynamoDB Global Tables (cross-region state)

Bidirectional replication between `eu-west-1` and `eu-central-1` for session data.

![DynamoDB Global Tables](docs/screenshots/1.13%20%E2%80%94%20DynamoDB%20Global%20Tables.png)

> Full single-point-of-failure analysis in [`spofs.md`](spofs.md). Network and security baseline screenshots (`1.1` – `1.9`) in [`docs/screenshots/`](docs/screenshots).

---

## Day 2 — Scalability (25 pts)

Three independent scaling mechanisms:

1. **Web tier** — ASG target tracking on `ALBRequestCountPerTarget` (target 200/min), asymmetric cooldowns (60s out / 300s in).
2. **Cache tier** — ElastiCache Redis primary + replica; reader endpoint for read distribution.
3. **Worker tier** — ECS Fargate driven by SQS; Application Auto Scaling on `ApproximateNumberOfMessagesVisible` (target 5).

### Scaling Policy

Target tracking automatically creates `AlarmHigh` (scale-out) and `AlarmLow` (scale-in) alarms.

![ASG Scaling Policy](docs/screenshots/2.1%20%E2%80%94%20ASG%20scaling%20policy.png)

### Web tier load test — Scale-out

Under simulated 50 req/s, target tracking jumped the ASG straight from 2 instances to the max of 6 in a single decision.

![Web tier scale-out](docs/screenshots/load-test-web-d-scale-out.png)

### Web tier load test — Scale-in

Once load ended, scale-in unwound the 4 extra instances over ~3.5 minutes (staggered to avoid capacity cliffs).

![Web tier scale-in](docs/screenshots/load-test-web-c-scale-in.png)

### Worker tier — Scale-out

200 messages injected into `capstone-orders`. CloudWatch's `AlarmHigh` fired in ~3.5 minutes, Application Auto Scaling jumped Fargate from 1 → 6 tasks.

![Worker tier scale-out](docs/screenshots/load-test-worker-a-scale-out.png)

### Worker tier — Scale-in

After queue purge, `AlarmLow` eventually fired and Fargate scaled back to 1 task.

![Worker tier scale-in](docs/screenshots/load-test-worker-b-scale-in.png)

> Worker scaling activity log: `docs/load-test/fargate-scaling-history-worker.txt`. State snapshot at max: `docs/load-test/worker-state-at-max.txt`.

---

## Day 3 — Disaster Recovery (25 pts)

**Pattern:** Warm Standby · **RTO:** 30 min · **RPO:** 5 min

Built a scaled-down replica of the production stack in `eu-central-1` with cross-region data replication and DNS failover.

### Primary EC2 — Multi-AZ

Web tier instances distributed across `eu-west-1a` and `eu-west-1b`.

![Primary EC2 Multi-AZ](docs/screenshots/ha-primary-ec2-multi-az.png)

### Primary RDS — Multi-AZ

Multi-AZ deployment with synchronous standby (intra-region HA before DR considerations).

![Primary RDS Multi-AZ](docs/screenshots/ha-primary-rds-multi-az.png)

### DR Region — Cross-Region RDS Read Replica

`capstone-mysql-dr-replica` in eu-central-1, continuously replicating from the primary.

![DR RDS Replica](docs/screenshots/dr-rds-replica.png)

### DR Region — Warm Standby Serving

DR ALB live and serving from `eu-central-1a` — proof the warm standby is real, not just provisioned.

![DR ALB serving](docs/screenshots/dr-alb-serving.png)

### Failover plumbing

- **Route 53** hosted zone (`capstone.local`) with failover routing — primary alias to `eu-west-1` ALB with HTTP health check, secondary to `eu-central-1` ALB
- **AWS Backup** vaults in both regions, daily plan with cross-region copy rule, on-demand backup executed as evidence

> Full failover procedure (replica promotion, ASG scale-up, DNS verification, config update), failback, and limitations in [`docs/dr/dr-runbook.md`](docs/dr/dr-runbook.md). CLI captures (RDS status, Route 53 records, vault listings, backup plan) in [`docs/dr/`](docs/dr).

---

### DR Simulation — Detection → Failover → Recovery

A failover drill was executed by temporarily pointing the Route 53 primary health check at a non-existent path (`/does-not-exist`), forcing it to fail without disrupting actual traffic. The mechanism was exercised end-to-end and reversed within ~7 minutes.

**Route 53 failover configuration** — `app.capstone.local` has two A records with failover routing policy: primary aliased to the eu-west-1 ALB (with health check), secondary aliased to the eu-central-1 ALB.

![DR Failover Records](docs/screenshots/dr-failover-records.png)

**During failure** — Health check observations report `Failure: HTTP Status Code 404` across all five AWS probe regions. `app.capstone.local` now resolves to the DR ALB IPs (`52.57.x`, `63.184.x` — eu-central-1 range). DR ALB independently verified serving `HTTP/1.1 200 OK`.

![DR Simulation — Failure State](docs/screenshots/dr-simulation-failure.png)

**After recovery** — Health check path restored to `/`. All five probe regions report `Success: HTTP 200` within ~3 minutes. `app.capstone.local` now resolves back to the primary ALB IPs (`54.155.x`, `54.229.x` — eu-west-1 range).

![DR Simulation — Recovery State](docs/screenshots/dr-simulation-recovery.png)

| Phase             | Health check status                 | DNS resolves to | IPs observed                         |
| ----------------- | ----------------------------------- | --------------- | ------------------------------------ |
| Before            | Success (HTTP 200)                  | Primary ALB     | `54.155.x`, `54.229.x` (eu-west-1)   |
| Failure simulated | Failure (HTTP 404) across 5 regions | DR ALB          | `52.57.x`, `63.184.x` (eu-central-1) |
| Recovery          | Success (HTTP 200)                  | Primary ALB     | `54.155.x`, `54.229.x` (eu-west-1)   |

> Complete timestamped CLI evidence (8 files spanning before / during / after) in [`docs/dr/simulation/`](docs/dr/simulation).

## Day 4 — Observability (15 pts)

### CloudWatch Dashboard (top: web + data tier)

ALB requests vs 5xx errors, Web ASG capacity, RDS CPU and connections, SQS depth and message age.

![Observability Dashboard A](docs/screenshots/observability-dashboard-a.png)

### CloudWatch Dashboard (bottom: workers + cache + DR)

Fargate worker CPU/memory (the spike at 11:30 and drop to 0% at 12:15 capture the full worker scale-out → scale-in cycle), ElastiCache health, and the DR RDS replica lag — the RPO indicator (target < 300s).

![Observability Dashboard B](docs/screenshots/observability-dashboard-b.png)

### Custom alarms (beyond AWS's auto-created target-tracking ones)

| Alarm                          | Triggers on                             |
| ------------------------------ | --------------------------------------- |
| `capstone-alb-5xx-errors`      | > 10 errors/min sustained for 2 min     |
| `capstone-alb-unhealthy-hosts` | Any unhealthy target for 3 min          |
| `capstone-rds-high-cpu`        | > 80% CPU for 10 min                    |
| `capstone-sqs-message-age`     | Oldest message > 5 min (worker lagging) |
| `capstone-dr-replica-lag`      | Replica lag > 5 min (RPO violation)     |

> Dashboard definition (`dashboard-definition.json`) and alarm exports (`alarms-primary.txt`, `alarms-dr.txt`) in [`docs/observability/`](docs/observability).

---

## Repository Structure

```
.
├── README.md                   ← this file
├── architecture-diagram.png    ← rendered architecture diagram
├── architecture.svg            ← architecture diagram (SVG source)
├── spofs.md                    ← single-point-of-failure analysis
├── lab.env                     ← captured resource IDs
└── docs/
    ├── dr/
    │   ├── dr-runbook.md
    │   ├── backup-job-status.txt
    │   ├── backup-plan.json
    │   ├── backup-vaults.txt
    │   ├── dr-alb-targets.txt
    │   ├── dynamodb-global-table.txt
    │   ├── rds-replica-status.txt
    │   └── route53-failover-records.txt
    ├── load-test/
    │   ├── fargate-scaling-history-worker.txt
    │   └── worker-state-at-max.txt
    ├── observability/
    │   ├── alarms-primary.txt
    │   ├── alarms-dr.txt
    │   └── dashboard-definition.json
    └── screenshots/            ← all visual evidence (HA 1.x, Scalability 2.x, load-test-*, ha-*, dr-*, observability-*)
```

---

## Trade-offs & Limitations

Honest decisions made under the lab's time and cost constraints:

1. **Manual RDS master password** — disabled `ManageMasterUserPassword` on the primary because MySQL cross-region read replicas don't support managed passwords (current AWS limitation). The secret in Secrets Manager remains valid; auto-rotation is a documented follow-up.
2. **Single AZ in DR compute** — warm standby runs 1 EC2 in `eu-central-1a` only. Multi-AZ DR is a documented scale-up step in the failover runbook.
3. **HTTPS deferred** — ALBs serve HTTP only; ACM cert + HTTPS listener would be the production add-on but was out of scope for the lab window.
4. **Placeholder worker task** — the Fargate task is a sleep loop, not a real consumer. Queue-depth scaling was tested in isolation (which is what the rubric measures); a production worker would drain the queue itself.

---
