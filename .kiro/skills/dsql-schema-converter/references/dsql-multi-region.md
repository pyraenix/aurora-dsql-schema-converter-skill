# Multi-Region DSQL Clusters

Aurora DSQL's key differentiator: active-active multi-region with strong consistency.

Sources:
- [What is Aurora DSQL](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/what-is-aurora-dsql.html)
- [Multi-Region Clusters (Starter Kit)](https://awslabs.github.io/aurora-dsql-starter-kit/multi-region-clusters.html)
- [Multi-Region Endpoint Routing](https://aws.amazon.com/blogs/database/implement-multi-region-endpoint-routing-for-amazon-aurora-dsql/)
- [DSQL for Global-Scale Financial Transactions](https://aws.amazon.com/blogs/database/amazon-aurora-dsql-for-global-scale-financial-transactions/)
- [Introducing Aurora DSQL](https://aws.amazon.com/blogs/database/introducing-amazon-aurora-dsql/)
- [Aurora DSQL Samples](https://github.com/aws-samples/aurora-dsql-samples)

---

## Overview

| Configuration | Availability SLA | Regions | Use Case |
|---|---|---|---|
| Single-Region | 99.99% | 1 | Standard workloads, single-geography |
| Multi-Region | 99.999% | 2 + witness | Global apps, disaster recovery, compliance |

### Key Properties

- **Active-active**: Both regions handle reads AND writes simultaneously
- **Strongly consistent**: All reads and writes to any endpoint are strongly consistent and durable
- **Synchronous replication**: Cross-region replication is synchronous (not eventual)
- **Automatic failover**: Zero data loss failover between regions
- **Single schema**: Same tables, indexes, and data in both regions

---

## Architecture

```
┌─────────────────┐         ┌─────────────────┐
│   Region 1      │         │   Region 2      │
│   (us-east-1)   │◄───────►│   (us-east-2)   │
│                 │  sync    │                 │
│  ┌───────────┐  │  repli-  │  ┌───────────┐  │
│  │ Endpoint  │  │  cation  │  │ Endpoint  │  │
│  │ (read +   │  │         │  │ (read +   │  │
│  │  write)   │  │         │  │  write)   │  │
│  └───────────┘  │         │  └───────────┘  │
└─────────────────┘         └─────────────────┘
         │                           │
         └───────────┬───────────────┘
                     │
              ┌──────┴──────┐
              │  Witness    │
              │  Region     │
              │ (us-west-2) │
              │             │
              │ Transaction │
              │ logs for    │
              │ quorum      │
              └─────────────┘
```

### Components

| Component | Role | Client Connections |
|---|---|---|
| Region 1 cluster | Full read/write endpoint | Yes |
| Region 2 cluster | Full read/write endpoint | Yes |
| Witness region | Quorum decisions, transaction logs | No |

---

## Schema Migration Implications

### What's the Same Across Regions

- Table definitions (schema is synchronized)
- Index definitions
- Sequence state
- View definitions
- Function definitions
- Role/permission grants

### What You Deploy Once

Schema DDL only needs to be executed against ONE region — it automatically propagates:

```sql
-- Connect to Region 1 endpoint
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id uuid NOT NULL,
  total numeric(10,2),
  created_at timestamptz DEFAULT now()
);

-- This table is immediately available in Region 2 as well
-- (after DDL propagation, which is synchronous)
```

### DDL Considerations for Multi-Region

| Behavior | Impact |
|---|---|
| DDL propagates synchronously | Slightly higher DDL latency than single-region |
| CREATE INDEX ASYNC | Index builds in both regions; monitor in both |
| One DDL per transaction | Same rule applies |
| Schema changes visible after reconnect | May need to refresh connections in both regions |

---

## Application Design for Multi-Region

### Connection Strategy

```
┌──────────────┐     ┌─────────────────────────────┐
│  Application │────►│  Route 53 / Global          │
│  (any region)│     │  Accelerator / CloudFront   │
└──────────────┘     └──────────┬──────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
              ┌─────┴─────┐         ┌──────┴────┐
              │ Region 1  │         │ Region 2  │
              │ Endpoint  │         │ Endpoint  │
              └───────────┘         └───────────┘
```

**Options:**
1. **Latency-based routing** — Route to nearest region (lowest latency)
2. **Failover routing** — Primary/secondary with health checks
3. **Application-level** — Connection string per region, app decides

### OCC Behavior in Multi-Region

Cross-region write conflicts are resolved the same way as single-region:
- SQLSTATE 40001 (serialization failure)
- Retry with exponential backoff
- The transaction with the earlier commit timestamp wins

**Additional consideration:** Cross-region writes have higher latency (~50-100ms additional for synchronous replication). This means:
- Transactions touching the same rows from different regions are more likely to conflict
- Design for low contention across regions (partition by geography if possible)
- Read-only transactions have zero commit latency and no conflicts

### Multi-Region Design Patterns

| Pattern | Description | When to Use |
|---|---|---|
| Geographic partitioning | Route users to nearest region; most writes are local | User-facing apps with geographic distribution |
| Active-passive reads | Write to one region, read from nearest | Read-heavy workloads |
| Active-active writes | Both regions write freely | Low-contention workloads, global availability |
| Conflict-free writes | Each region owns different data subsets | High-write workloads needing zero conflicts |

### Geographic Partitioning Example

```sql
-- Design PK to include region affinity
CREATE TABLE user_sessions (
  region varchar(20) COLLATE "C",
  session_id uuid DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  data json,
  PRIMARY KEY (region, session_id)
);

-- Region 1 writes sessions with region='us-east-1'
-- Region 2 writes sessions with region='us-east-2'
-- No cross-region conflicts on the same rows
```

---

## Setting Up Multi-Region Clusters

### Prerequisites

- Two AWS regions selected for endpoints
- One witness region (geographically separate)
- IAM permissions for dsql:CreateCluster, dsql:UpdateCluster
- Deletion protection recommended for production

### Creation Flow

1. Create cluster in Region 1 (with witness region specified)
2. Create cluster in Region 2 (with witness region + Region 1 as peer)
3. Update Region 1 cluster to add Region 2 as peer
4. Wait for both clusters to reach ACTIVE state

### Cluster States

| State | Meaning |
|---|---|
| CREATING | Being provisioned |
| ACTIVE | Ready for connections |
| UPDATING | Configuration change in progress |
| DELETING | Being removed |
| PENDING_DELETE | Waiting for peer cluster deletion |

### Deletion (Important)

Multi-region clusters must be deleted together:
1. Delete cluster in Region 1 → enters PENDING_DELETE
2. Delete cluster in Region 2 → both transition to DELETING
3. Wait for both to complete deletion

You cannot delete just one cluster from a linked pair.

---

## Performance Considerations

### Latency

| Operation | Single-Region | Multi-Region | Difference |
|---|---|---|---|
| Read (local) | ~2-5ms | ~2-5ms | None |
| Write (local) | ~10-20ms | ~50-100ms | Cross-region sync adds latency |
| Read-only transaction commit | 0ms | 0ms | No commit needed |
| DDL propagation | Immediate | Synchronous (slightly slower) | Minimal |

### Throughput

- Each region can handle full read/write throughput independently
- Cross-region synchronization does not limit per-region throughput
- OCC conflicts increase with cross-region writes to same rows

### Optimization Tips

1. **Minimize cross-region write conflicts** — Partition data by region where possible
2. **Use UUIDs for PKs** — Random distribution prevents hot spots in both regions
3. **Keep transactions short** — Shorter transactions = less conflict window
4. **Leverage read-only transactions** — Zero latency, zero conflicts
5. **Use covering indexes** — Avoid cross-region storage fetches

---

## Monitoring Multi-Region Clusters

```sql
-- Check cluster status (run in each region)
-- Use AWS CLI or SDK:
-- aws dsql get-cluster --identifier <cluster-id> --region us-east-1

-- Monitor async index status (both regions)
SELECT indexrelid::regclass, indisvalid
FROM pg_index WHERE NOT indisvalid;

-- Monitor for OCC conflicts (application metrics)
-- Track SQLSTATE 40001 frequency per region
-- Alert if retry rate exceeds threshold (e.g., >5% of transactions)
```

---

## Migration to Multi-Region

If you're migrating from single-region PostgreSQL to multi-region DSQL:

### Step 1: Single-Region DSQL First
1. Convert schema (using this skill)
2. Deploy to single-region DSQL cluster
3. Migrate data
4. Validate application works
5. Run under production load

### Step 2: Add Second Region
1. Create multi-region cluster (link existing + new)
2. Data automatically synchronizes
3. Update application to use multi-region endpoints
4. Configure routing (Route 53 / Global Accelerator)
5. Test failover scenarios

### Step 3: Optimize
1. Monitor cross-region conflict rates
2. Implement geographic partitioning if conflicts are high
3. Tune retry logic for cross-region latency
4. Set up monitoring and alerting in both regions

---

## Quotas for Multi-Region

| Quota | Value | Increasable |
|---|---|---|
| Multi-region clusters per account | 5 | Yes (via Service Quotas) |
| Regions per cluster | 2 (+ 1 witness) | No |
| Witness region | Must be different from both endpoint regions | — |
| Storage per cluster | 10 TiB (up to 256 TiB) | Yes |
