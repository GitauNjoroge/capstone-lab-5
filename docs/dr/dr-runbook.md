# Disaster Recovery Runbook — Capstone Platform

## Recovery Objectives

| Metric                             | Target     | Mechanism                                                                                    |
| ---------------------------------- | ---------- | -------------------------------------------------------------------------------------------- |
| **RTO** (Recovery Time Objective)  | 30 minutes | Warm standby pre-provisioned in eu-central-1                                                 |
| **RPO** (Recovery Point Objective) | 5 minutes  | RDS cross-region read replica (~5 min replication lag) + DynamoDB Global Tables (sub-second) |

## DR Pattern: Warm Standby

A scaled-down version of the production stack runs continuously in eu-central-1. On primary region failure:

1. Route 53 health check detects the outage (~90s)
2. DNS automatically fails over \`app.capstone.local\` to the DR ALB
3. Operator promotes the RDS read replica to standalone primary
4. Operator scales DR ASG from 1 to production capacity

## Cross-Region Components

| Component            | Primary (eu-west-1)        | DR (eu-central-1)            | Replication                        |
| -------------------- | -------------------------- | ---------------------------- | ---------------------------------- |
| ALB + ASG            | 2 baseline / 6 max         | 1 warm / 2 max               | Independent                        |
| RDS MySQL            | Multi-AZ primary           | Cross-region read replica    | Async, ~5 min lag                  |
| DynamoDB sessions    | Global Table replica       | Global Table replica         | Bidirectional, sub-second          |
| Backups (AWS Backup) | \`capstone-vault-primary\` | \`capstone-vault-dr\`        | Daily cross-region copy            |
| DNS                  | Route 53 \`PRIMARY\` alias | Route 53 \`SECONDARY\` alias | Failover routing with health check |

## Failover Procedure

### Step 0 — Detection (automatic)

Route 53 health check on the primary ALB fails 3 consecutive 30-second probes (~90s detection). DNS automatically switches \`app.capstone.local\` to the DR ALB.

### Step 1 — Promote the read replica

\\\`\\\`\\\`bash
aws rds promote-read-replica --region eu-central-1 \\
--db-instance-identifier capstone-mysql-dr-replica
\\\`\\\`\\\`
Wait for status to reach \`available\` (~5 min). The instance becomes a standalone primary that accepts writes.

### Step 2 — Scale up DR ASG to handle production load

\\\`\\\`\\\`bash
aws autoscaling update-auto-scaling-group --region eu-central-1 \\
--auto-scaling-group-name capstone-web-asg-dr \\
--min-size 2 --max-size 6 --desired-capacity 4
\\\`\\\`\\\`

### Step 3 — Update application config

Point applications at the new primary RDS endpoint:
\\\`\\\`\\\`bash
aws rds describe-db-instances --region eu-central-1 \\
--db-instance-identifier capstone-mysql-dr-replica \\
--query 'DBInstances[0].Endpoint.Address'
\\\`\\\`\\\`

### Step 4 — Verify

- \`app.capstone.local\` resolves to DR ALB
- DR ASG target group shows healthy targets
- Database accepts test writes

## Failback Procedure

When primary region recovers:

1. Confirm eu-west-1 health (all services GREEN in AWS Health Dashboard)
2. Configure eu-west-1 RDS as a replica of the (now-primary) eu-central-1 instance
3. Schedule a maintenance window
4. Cutover:
   - Stop writes to eu-central-1
   - Promote eu-west-1 replica back to primary
   - Update DNS to restore PRIMARY routing
   - Resume writes
5. Recreate eu-central-1 as the read replica → back to warm standby

## Verification Checklist

Use after any DR drill or actual failover:

- [ ] Route 53 health check status: \`HEALTHY\`
- [ ] Active region's ALB target group: all targets \`healthy\`
- [ ] RDS replica status (steady state): \`available\`
- [ ] CloudWatch \`ReplicaLag\` metric: < 5 minutes
- [ ] Latest AWS Backup job: \`COMPLETED\`
- [ ] Cross-region copy job: \`COMPLETED\`

## Alternative: Backup-Based Recovery

If the read replica is unavailable, restore from cross-region backup vault:

\\\`\\\`\\\`bash

# 1. List recovery points

aws backup list-recovery-points-by-backup-vault --region eu-central-1 \\
--backup-vault-name capstone-vault-dr

# 2. Start restore job (uses metadata from the recovery point)

aws backup start-restore-job --region eu-central-1 \\
--recovery-point-arn <ARN> \\
--metadata file://restore-metadata.json \\
--iam-role-arn $BACKUP_ROLE_ARN \\
--resource-type RDS
\\\`\\\`\\\`

Higher RTO (~60 min for restore) but works without a pre-provisioned replica.

## Known Limitations

1. **Manual replica promotion** — currently requires operator action. Could be automated via Lambda + EventBridge triggered by the Route 53 health check.
2. **Manual app config update** — RDS endpoint change requires application redeployment. Could be mitigated by Route 53 private hosted zone pointing app at a CNAME.
3. **DNS TTL** — Route 53 alias records ignore TTL; failover is near-instant. Direct A records would have inherited TTL delays.
4. **Master password management** — disabled \`ManageMasterUserPassword\` on primary to enable read replica (AWS limitation). Secret in Secrets Manager remains valid but no longer auto-rotates. Follow-up: implement manual rotation via Lambda.
