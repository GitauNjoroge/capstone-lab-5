# Single Points of Failure — Day 1 Audit

## Overview

A single point of failure (SPOF) is any component whose failure would take the
whole system down. In a mission-critical e-commerce application — the scenario
for this capstone — every SPOF translates directly into customer-visible
outage and lost revenue.

This document audits the architecture built in Day 1 against a naïve baseline
(single-AZ, single-region), enumerates the SPOFs each Day 1 design choice
removes, and notes the SPOFs that remain at the end of Day 1. Those remaining
items are the explicit focus of Day 2 (scalability) and Day 3 (disaster
recovery).

The architecture diagram is at [`docs/architecture.svg`](architecture.svg).
References below to specific AWS resources use the variable names captured in
the deployment runbook (`lab.env`).

## Architecture in one paragraph

In `eu-west-1`, a single VPC (`10.0.0.0/16`) spans two availability zones
(`eu-west-1a` and `eu-west-1b`). Each AZ contains a public subnet (for the
ALB and NAT gateway), a private application subnet (for EC2 ASG instances and,
later, Fargate workers), and a database subnet (for RDS and ElastiCache, with
no internet route). An Application Load Balancer distributes traffic across
the two AZs. An EC2 Auto Scaling Group keeps at least two instances running,
spread across both AZs. RDS MySQL runs in Multi-AZ mode with a synchronous
standby in the second AZ. DynamoDB hosts session data as a Global Table
replicated to `eu-central-1`.

## SPOFs removed in Day 1

| SPOF in naïve design | Day 1 fix | Resources |
|----------------------|-----------|-----------|
| Single AZ | Multi-AZ across every layer | Subnets, ALB, ASG, RDS, NAT |
| Single EC2 instance | Auto Scaling Group with ELB health checks | `capstone-web-asg` |
| Single load balancer | AWS-managed Multi-AZ ALB | `capstone-alb` |
| Single database primary | RDS Multi-AZ with automatic failover | `capstone-mysql-primary` |
| Single NAT gateway | Per-AZ NAT + per-AZ private route tables | `NAT_GW_A`/`B`, `PRIV_RT_A`/`B` |
| Single region for session/catalog | DynamoDB Global Tables in two regions | `capstone-sessions` |

### Single AZ

**Failure scenario:** all resources in `eu-west-1a` become unreachable
simultaneously — power, network, or hardware failure at the AZ level.

**Design change:** the VPC is carved into six subnets — public, private-app,
and database tiers — with each tier present in two different AZs. Every
workload was placed across both AZs:

- ALB has ENIs in both public subnets (`PUB_SUBNET_A`, `PUB_SUBNET_B`); the
  loss of one AZ's ENI does not take down the ALB.
- The ASG's `--vpc-zone-identifier` lists both private app subnets, so
  instances are distributed across AZs. `min-size=2` guarantees at least one
  instance per AZ.
- RDS Multi-AZ places the synchronous standby in the AZ opposite the primary.
- Per-AZ NAT gateways mean each AZ's outbound is independent.

**Behaviour on failure:** AZ-a outage → ALB stops forwarding to AZ-a targets
within ~30 seconds (health check); RDS promotes the AZ-b standby within
60–120s with no committed transaction loss; ASG launches replacement instances
in AZ-b within 3–5 minutes. The application keeps serving from AZ-b
throughout, with at most ~60 seconds of degraded service.

### Single EC2 instance

**Failure scenario:** the EC2 process crashes, the host hardware dies, or the
instance becomes network-isolated.

**Design change:** the Auto Scaling Group (`capstone-web-asg`) has
`min-size=2`, `max-size=6`, `desired-capacity=2`, and crucially
`--health-check-type ELB` — which means the ASG uses the ALB's HTTP health
check, not just EC2 instance status. An instance whose OS is up but whose
nginx process has crashed is still detected as unhealthy and replaced.

**Behaviour on failure:** ALB health check fails within ~60 seconds (two
consecutive 30-second probes); target is removed from rotation; ASG
terminates the instance and launches a replacement from the launch template
(`capstone-web-lt`); replacement passes health check within ~2 minutes of
nginx starting. Total recovery: 3–5 minutes, no customer-visible loss because
the surviving instance(s) continue serving.

### Single load balancer

The ALB itself is an AWS-managed multi-AZ component. By specifying both
public subnets at creation, the load balancer maintains an ENI in each AZ,
and AWS handles the cross-AZ HA of the load-balancing layer transparently.
This SPOF is "removed" by virtue of choosing ALB rather than a single-instance
load balancer (Classic ELB Single-AZ, or a self-managed nginx on EC2). No
configuration on our part directly protects against ALB internal failures —
AWS does.

### Single database primary — RDS Multi-AZ failover behaviour

**Failure scenario:** RDS primary becomes unreachable (AZ failure, hardware
failure, or RDS-detected infrastructure issue).

**Design change:** RDS was launched with `--multi-az`, provisioning a hot
synchronous standby in the second AZ. The DB subnet group
(`capstone-db-subnet-group`) spans both database subnets across the two AZs.

**Behaviour on failover (the rubric-required documentation):**

1. RDS internal monitoring detects the primary as unhealthy, typically within
   30–60 seconds of the failure.
2. The standby is promoted to primary. Because replication is synchronous, no
   committed transactions are lost — RPO is effectively zero for this failure
   mode.
3. The endpoint DNS (`$RDS_ENDPOINT`) is updated to resolve to the new primary.
   The old primary's IP is no longer used.
4. Application connection pools see existing connections drop. The next
   connection attempt resolves the endpoint DNS to the new primary and
   succeeds — provided the application's MySQL driver re-resolves DNS rather
   than caching IPs (most modern drivers do).
5. The old primary, once recovered, is re-provisioned as the new standby.
   The system returns to fully redundant state.

**Total downtime:** typically 60–120 seconds of unavailability for writes.
Applications with retry logic experience a brief blip rather than an outage.

**What Multi-AZ does *not* protect against:**

- **Region-level failure.** The standby is in the same region, so a regional
  outage takes down both. Addressed in Day 3 via cross-region read replica.
- **Logical data corruption.** A `DELETE FROM users` runs on the primary and
  is synchronously replicated to the standby. That is what backups are for —
  addressed in Day 3 via AWS Backup with point-in-time restore.
- **Failure modes RDS does not auto-detect.** Slow queries, lock contention,
  application-level deadlocks. These require application-level monitoring,
  not infrastructure HA.

### Single NAT gateway

**Failure scenario:** the NAT gateway for an AZ becomes unhealthy or is
destroyed by an AZ failure.

**Design change:** two NAT gateways, one per AZ. `NAT_GW_A` lives in
`PUB_SUBNET_A`; `NAT_GW_B` in `PUB_SUBNET_B`. Two private route tables —
`PRIV_RT_A` pointing its default route at `NAT_GW_A`, `PRIV_RT_B` at
`NAT_GW_B`. Each AZ's private subnet uses its own AZ's NAT.

**Behaviour on failure:** NAT-A fails → AZ-a's private subnet loses outbound
internet (intra-VPC routing is unaffected — local route stays). AZ-a EC2
instances lose the ability to call public AWS APIs and external services;
ALB health checks against them eventually fail; ALB shifts all traffic to
AZ-b instances. **AZ-b is entirely unaffected** because `PRIV_RT_B` routes
through `NAT_GW_B`, not `NAT_GW_A`.

**Why this matters:** the single-NAT alternative was deliberately rejected.
Had we used one NAT for both AZs to save ~$32/month, the AZ of that NAT
would have been a cross-AZ SPOF — both private subnets would lose outbound
when that single AZ failed, defeating the entire Multi-AZ posture for ~$32
of savings. For an HA-themed architecture this trade is wrong.

### Single region for session/catalog — Why DynamoDB Global Tables

**Failure scenario:** the entire primary region (`eu-west-1`) becomes
unavailable.

**Design choice rationale:** session and catalog data are read-heavy and
painful to lose. Sessions especially — losing a customer's cart mid-checkout
is direct revenue impact. We chose DynamoDB Global Tables specifically
because:

1. **Active-active replication, not active-passive.** Reads and writes
   succeed in either region simultaneously. When traffic shifts via Route 53
   failover (Day 3), no promotion step is needed — the secondary region's
   table is already a fully writable replica.
2. **Sub-second replication latency.** A write to `eu-west-1` typically
   appears in `eu-central-1` within a second. RPO for session/catalog data
   is effectively zero in normal operation.
3. **Built-in conflict resolution.** Concurrent writes to the same item from
   different regions resolve "last writer wins" deterministically.
   Acceptable for session data (typically single-user) and catalog updates
   (typically low-conflict).
4. **No application complexity.** The app calls `DynamoDB.PutItem` against
   whichever region is local. There is no replication code, no custom
   failover logic. The replication is invisible to application developers.

This was verified in Step 1.13: `put-item` in `eu-west-1` followed by
`get-item` in `eu-central-1` returned the identical item, and the reverse
direction worked too. Both with replication latency observed at under three
seconds.

**Trade-off accepted:** writes are billed in *each* replica region, so total
write cost approximately doubles versus a single-region table. In exchange
we gain operational simplicity and RPO ≈ 0 disaster recovery for the items
stored here. For session and catalog data this trade is the right one.

## SPOFs not yet addressed (Day 2 / Day 3 roadmap)

| Remaining SPOF | Planned fix | Day |
|----------------|-------------|-----|
| Single region for ALB, ASG, RDS | Route 53 failover routing + warm-standby region | Day 3 |
| RDS not yet cross-region | Cross-region read replica in `eu-central-1`, promoted on regional failover | Day 3 |
| Synchronous order processing | SQS + ECS Fargate worker tier (decoupling) | Day 2 |
| RDS as hot-read bottleneck | ElastiCache Redis for cached reads | Day 2 |
| Manual capacity (fixed at 2 instances) | ASG scaling policies on ALB request count | Day 2 |
| No fixed customer-facing domain | Route 53 hosted zone + ACM cert + HTTPS listener | Day 3 |
| No automated backup retention | AWS Backup with cross-region snapshot copy | Day 3 |
| No observability | CloudWatch metrics, dashboards, alarms | Day 2/3 |

## How to simulate failures

The rubric requires SPOFs to be both identified and *demonstrably* removed.
The following commands trigger each failure scenario in a controlled way
within the deployed environment.

### Simulate EC2 instance failure

```bash
# Pick one instance
EC2_VICTIM=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names capstone-web-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' \
  --output text)

# Terminate it
aws ec2 terminate-instances --instance-ids $EC2_VICTIM

# Watch ALB target health and ASG response
watch -n 15 'aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[].[Target.Id,TargetHealth.State]" \
  --output table'
```

**Expected observation:** the terminated instance shows `unhealthy` within
60 seconds; ASG launches a replacement within 2–3 minutes; new instance
shows `healthy` within ~5 minutes total. The ALB never returns 5xx to
customers because the surviving instance keeps serving throughout.

### Simulate RDS failover

```bash
aws rds reboot-db-instance \
  --db-instance-identifier capstone-mysql-primary \
  --force-failover
```

**Expected observation:** the RDS endpoint stays reachable but the underlying
primary AZ flips. Existing MySQL connections drop and need to be re-opened.
The `AvailabilityZone` and `SecondaryAvailabilityZone` fields in
`describe-db-instances` swap with each other within 60–120 seconds.

Verify the swap:

```bash
aws rds describe-db-instances \
  --db-instance-identifier capstone-mysql-primary \
  --query 'DBInstances[0].[AvailabilityZone,SecondaryAvailabilityZone]'
```

### Simulate NAT gateway failure (proxy)

A NAT gateway cannot be "broken" without deleting it. The closest controlled
proxy is to remove the default route from one of the private route tables:

```bash
# Break AZ-a outbound
aws ec2 delete-route \
  --route-table-id $PRIV_RT_A \
  --destination-cidr-block 0.0.0.0/0
```

**Expected observation:** AZ-a private subnet immediately loses outbound
connectivity. AZ-a EC2 instances lose access to external services; ALB
shifts all traffic to AZ-b instances over the next minute.

**Restore** (do this before moving on, or AZ-a will stay broken):

```bash
aws ec2 create-route \
  --route-table-id $PRIV_RT_A \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_A
```

### DynamoDB regional replication check

A regional DynamoDB outage cannot be triggered. Instead, the bidirectional
replication property is re-verified by writing to one region and reading
from the other:

```bash
SID="failsim-$(date +%s)"
aws dynamodb put-item --table-name capstone-sessions \
  --item "{\"sessionId\":{\"S\":\"$SID\"},\"region\":{\"S\":\"eu-west-1\"}}" \
  --region eu-west-1

sleep 3

aws dynamodb get-item --table-name capstone-sessions \
  --key "{\"sessionId\":{\"S\":\"$SID\"}}" \
  --region eu-central-1 \
  --query 'Item.region.S' --output text
# Expected output: eu-west-1
```

If the item appears in `eu-central-1` after a write to `eu-west-1`, the
replication path is healthy. The Day 3 region-failover test exercises the
read-side of this property under actual traffic redirection.

## Summary

After Day 1, the application is resilient to failures of:

- any single EC2 instance,
- either AZ in `eu-west-1`,
- the RDS primary node,
- either NAT gateway,
- any single component within the data tier within a region.

The remaining systemic risks — full-region failure of the application tier,
synchronous coupling of web and order processing, and RDS load on hot reads
— are explicit targets for Day 2 (scalability) and Day 3 (disaster recovery).
