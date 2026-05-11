# Capstone Lab — Resilient, Scalable & Recoverable AWS Architecture

> **Course:** Moringa AWS DevOps Capstone (DAWSB-PT02M2)
> **Author:** Omao Machoka (Joash)
> **Primary Region:** eu-west-1 (Ireland) · **DR Region:** eu-central-1 (Frankfurt)

## TL;DR

A three-tier web application infrastructure on AWS that survives AZ failures (Multi-AZ HA), scales automatically under load (target-tracking autoscaling on requests and queue depth), and recovers from regional outages (warm-standby DR with 30-min RTO / 5-min RPO). All four rubric pillars demonstrated end-to-end with live infrastructure, executed load tests, and CloudWatch observability.

## Architecture

![Architecture](Architecture%20Diagram.png)

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

| Pillar                | Points | Evidence                                                                                     |
| --------------------- | ------ | -------------------------------------------------------------------------------------------- |
| **High Availability** | 25     | `docs/screenshots/1.1` – `1.13`, [`spofs.md`](spofs.md)                                      |
| **Scalability**       | 25     | `docs/screenshots/2.x`, `load-test-*`, [`docs/load-test/`](docs/load-test)                   |
| **Disaster Recovery** | 25     | `docs/screenshots/dr-*` and `ha-primary-*`, [`docs/dr/dr-runbook.md`](docs/dr/dr-runbook.md) |
| **Observability**     | 15     | `docs/screenshots/observability-*`, [`docs/observability/`](docs/observability)              |
| **Documentation**     | 10     | This README, runbook, SPOF analysis                                                          |

---

## Day 1 — High Availability (25 pts)

Built a three-tier VPC in `eu-west-1` spanning two AZs (`eu-west-1a`, `eu-west-1b`) with isolated public / private-app / database subnets. Every layer that could be a single point of failure was duplicated:

- **Per-AZ NAT Gateways** instead of a single NAT — eliminates the cross-AZ failure dependency that the single-NAT shortcut introduces.
- **ALB and ASG** spread across both AZs; ASG uses ELB-based health checks (catches "instance up, app down").
- **RDS MySQL Multi-AZ** with synchronous standby in a second AZ; encrypted at rest; not publicly accessible.
- **DynamoDB Global Tables** for session state, replicated to `eu-central-1` for cross-region resilience.
- **Security baseline:** SSM Session Manager only (no SSH exposure), IMDSv2 required, layered SG chain (ALB → EC2 → RDS).

The complete single-point-of-failure analysis — including remaining SPOFs the design intentionally leaves for the DR phase — is in [`spofs.md`](spofs.md).

**Visual evidence:** screenshots `1.1` through `1.13` in `docs/screenshots/`.

---

## Day 2 — Scalability (25 pts)

Three independent scaling mechanisms:

1. **Web tier** — ASG target tracking on `ALBRequestCountPerTarget` (target 200/min) with asymmetric cooldowns (60s out / 300s in) — scale up fast, scale down conservatively.
2. **Cache tier** — ElastiCache Redis primary + replica; reader endpoint for read distribution.
3. **Worker tier** — ECS Fargate driven by SQS; Application Auto Scaling on `ApproximateNumberOfMessagesVisible` (target 5).

### Web tier load test

Artillery 2.x in three phases (baseline 5 req/s → sustained 50 req/s → cooldown). The ASG responded by jumping straight from 2 instances to the max of 6 in a single scaling decision, then scaled back to 2 over ~3.5 minutes once load ended. Full activity log in `docs/load-test/asg-scaling-history-web.txt`.

Evidence (chronological): `load-test-web-a-config.png` → `b-artillery-run.png` → `c-scale-out.png` → `d-scale-in.png`.

### Worker tier load test

200 messages injected into `capstone-orders`. CloudWatch's `AlarmHigh` fired in ~3.5 minutes, Application Auto Scaling jumped the Fargate service from 1 → 6 tasks. After queue purge, `AlarmLow` eventually fired and tasks scaled back to 1.

Evidence: `load-test-worker-a-scale-out.png`, `load-test-worker-b-scale-in.png`. Scaling activity log in `docs/load-test/fargate-scaling-history-worker.txt`.

---

## Day 3 — Disaster Recovery (25 pts)

**Pattern:** Warm Standby · **RTO:** 30 min · **RPO:** 5 min

Built a scaled-down replica of the production stack in `eu-central-1`:

- **Compute** — ALB + ASG (1 warm instance) ready to scale up on failover
- **Relational data** — RDS cross-region **read replica** (`capstone-mysql-dr-replica`)
- **State** — DynamoDB Global Table replica (Day 1)
- **DNS** — Route 53 hosted zone (`capstone.local`) with failover routing: primary alias to `eu-west-1` ALB with HTTP health check, secondary to `eu-central-1` ALB
- **Backups** — AWS Backup vaults in both regions, daily plan with cross-region copy, on-demand backup executed as evidence

Full failover procedure (replica promotion, ASG scale-up, DNS verification, config update), failback, and limitations are in [`docs/dr/dr-runbook.md`](docs/dr/dr-runbook.md).

**Evidence:** `ha-primary-ec2-multi-az.png`, `ha-primary-rds-multi-az.png`, `ha-primary-alb-serving.png`, `ha-primary-target-group-health.png`, `dr-alb-serving.png`, `dr-rds-replica.png`. CLI captures in [`docs/dr/`](docs/dr).

---

## Day 4 — Observability (15 pts)

**CloudWatch dashboard `capstone-platform`** with seven widgets covering every tier: ALB requests vs 5xx, Web ASG capacity, RDS CPU and connections, SQS depth and message age, Fargate CPU/memory, ElastiCache health, and the **DR RDS replica lag** (RPO indicator, target < 300s).

**Custom alarms** beyond AWS's auto-created target-tracking ones:

| Alarm                          | Triggers on                             |
| ------------------------------ | --------------------------------------- |
| `capstone-alb-5xx-errors`      | > 10 errors/min sustained for 2 min     |
| `capstone-alb-unhealthy-hosts` | Any unhealthy target for 3 min          |
| `capstone-rds-high-cpu`        | > 80% CPU for 10 min                    |
| `capstone-sqs-message-age`     | Oldest message > 5 min (worker lagging) |
| `capstone-dr-replica-lag`      | Replica lag > 5 min (RPO violation)     |

**Evidence:** `observability-dashboard-a.png`, `observability-dashboard-b.png`. Dashboard definition and alarm exports in [`docs/observability/`](docs/observability).

---

## Repository Structure

```
.
├── README.md                  ← this file
├── architecture.svg           ← architecture diagram (SVG source)
├── Architecture Diagram.png   ← rendered diagram
├── spofs.md                   ← single-point-of-failure analysis
├── lab.env                    ← captured resource IDs (gitignored in practice)
└── docs/
    ├── dr/                    ← DR runbook + CLI evidence
    ├── load-test/             ← Artillery configs + scaling activity logs
    ├── observability/         ← Dashboard JSON + alarm exports
    └── screenshots/           ← All visual evidence (1.x = HA, 2.x = scale, load-test-*, ha-*, dr-*, observability-*)
```

---

## Trade-offs & Limitations

Honest decisions made under the lab's time and cost constraints:

1. **Manual RDS master password** — disabled `ManageMasterUserPassword` on the primary because MySQL cross-region read replicas don't support managed passwords (current AWS limitation). The secret in Secrets Manager remains valid; auto-rotation is a documented follow-up.
2. **Single AZ in DR compute** — warm standby runs 1 EC2 in `eu-central-1a` only. Multi-AZ DR is documented as a scale-up step in the failover runbook.
3. **HTTPS deferred** — ALBs serve HTTP only; ACM cert + HTTPS listener would be the production add-on but was out of scope for the lab window.
4. **Placeholder worker task** — the Fargate task is a sleep loop, not a real consumer. Queue-depth scaling was tested in isolation (which is what the rubric measures); a production worker would drain the queue itself.

---
