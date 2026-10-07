# AWS Interview Prep: 25 Questions, With Reasoning

Each answer has the **short answer** you'd give in an interview, then the **why**, so you can handle follow-ups. For the experience questions (10.10, 10.11, 10.13, 10.23, 10.24), I've given strong templates. Swap in your own real stories, because interviewers probe deeply on those.

---

## 10.1 Highly available multi-tier architecture

**Answer:** Spread every tier across at least 2 (ideally 3) AZs, with no single point of failure at any layer.

```
Route 53 → CloudFront (+WAF) → ALB (public subnets, 3 AZs)
        → App tier: ASG across 3 AZs (private subnets)
        → Cache: ElastiCache (Multi-AZ)
        → DB: RDS/Aurora Multi-AZ (isolated DB subnets)
        S3 for static/assets, NAT GW per AZ for egress
```

**Why:**
- **Three subnet layers** (public / private app / isolated data) limit blast radius. Only the ALB is internet-facing.
- **Security groups chain by reference:** the ALB SG allows 443 from the internet, the app SG allows traffic only from the ALB SG, and the DB SG allows 3306/5432 only from the app SG.
- **Stateless app tier** (sessions in ElastiCache or DynamoDB, files in S3) lets the ASG add or remove instances freely.
- **One NAT Gateway per AZ.** A NAT GW lives in one AZ, so a single shared one is a hidden SPOF and creates cross-AZ charges.
- **Health checks at two levels:** ALB target health and ASG ELB health checks, so unhealthy instances get replaced automatically.
- **State is the hard part:** Multi-AZ RDS gives synchronous standby with 60-120s failover. Aurora fails over faster and has up to 15 read replicas.

For multi-region, add Route 53 health-check failover plus Aurora Global Database or DynamoDB Global Tables. Don't suggest it unless the RTO demands it, since it roughly doubles cost and complexity.

---

## 10.2 NAT Gateway mechanics and use cases

**Answer:** A managed service that lets private-subnet resources initiate outbound connections to the internet while blocking unsolicited inbound connections.

**How it works:**
1. It sits in a **public subnet** with an **Elastic IP**.
2. The private route table has `0.0.0.0/0 → nat-xxxx`.
3. The NAT GW translates the instance's private source IP to its own EIP (source NAT / PAT), then sends traffic out via the IGW.
4. It tracks connection state, so return traffic is mapped back to the right instance.

**Key facts:**
- AZ-scoped, and highly available *within* its AZ only.
- Scales from 5 Gbps up to 100 Gbps automatically.
- Supports TCP, UDP, ICMP. Can't be associated with a security group (use NACLs on its subnet).
- **Private NAT Gateway** (no EIP) exists for private-to-private translation, e.g., overlapping CIDRs over Transit Gateway.

**Use cases:** OS patching, pulling container images, calling third-party APIs, and giving a whitelisted fixed egress IP to partners.

**Cost trap:** You pay hourly plus **per GB processed**. The classic surprise bill is EC2 pulling terabytes from S3 or DynamoDB through NAT. The fix is a **gateway VPC endpoint** (free for S3 and DynamoDB).

---

## 10.3 Internet access for private subnet workloads

**Answer:** Add a route from the private subnet to a NAT Gateway.

**Steps:**
1. Create an IGW and attach it to the VPC.
2. Public subnet route table: `0.0.0.0/0 → igw-xxx`.
3. Create the NAT GW in the public subnet with an EIP.
4. Private subnet route table: `0.0.0.0/0 → nat-xxx`.
5. Check the SG outbound rules and NACL (ephemeral ports 1024-65535 inbound for return traffic).

**Debugging checklist when it doesn't work:** is the NAT GW actually in a *public* subnet (one whose route table has the IGW route)? Is the private route table associated with the right subnet? Does the NACL block return traffic? Is DNS resolving?

**Alternatives you should mention** (this is what separates seniors from juniors):
- **VPC endpoints** (gateway for S3/DynamoDB, interface endpoints/PrivateLink for other services): no internet path at all, cheaper and more secure.
- **Egress-only IGW** for IPv6.
- **Proxy / Network Firewall** if you need egress filtering by domain.

---

## 10.4 Inter-subnet communication within a VPC

**Answer:** It works by default, via the implicit **local route**.

Every route table contains `VPC-CIDR → local`. You can't delete or override it. So subnet A (in any AZ) can reach subnet B with no gateway, NAT, or extra route.

**What actually controls access:**
- **Security groups:** the main control. The target SG must allow the source (best practice: reference the source SG, not CIDRs).
- **NACLs:** the default NACL allows everything, but a custom NACL can block subnet-to-subnet traffic. Rules must be right in *both* directions, since NACLs are stateless.
- **OS firewall** (iptables, Windows Firewall).

**Trap:** Route tables decide *where* packets go, not *whether they're allowed*. People often debug routing when the real culprit is an SG or NACL. Use **VPC Reachability Analyzer** to find the blocking component quickly.

Cross-AZ traffic works but costs about $0.01/GB each direction, so keep chatty tiers AZ-aware where possible.

---

## 10.5 NACL (stateless) vs Security Group (stateful)

| | Security Group | NACL |
|---|---|---|
| Level | ENI / instance | Subnet |
| State | **Stateful**: return traffic auto-allowed | **Stateless**: must allow both directions |
| Rules | Allow only | Allow **and Deny** |
| Evaluation | All rules evaluated together | Numbered order, first match wins |
| Default | Deny all in, allow all out | Default NACL allows all |
| Reference | Can reference other SGs | CIDRs only |

**Why stateless matters:** If you allow inbound 443 on a NACL, the response goes out on a random **ephemeral port** (1024-65535). You must also allow *outbound* ephemeral ports, or the connection hangs. An SG remembers the connection and handles this automatically.

**When to use which:**
- **SGs** do 95% of the work: the primary, fine-grained control.
- **NACLs** are a coarse backstop. Their unique ability is **explicit deny**, e.g., block a malicious IP range, which an SG can't do. Also useful for compliance "defense in depth."

**Interview line:** "I keep NACLs simple and use SGs for real policy, because stateless rules double the debugging surface."

---

## 10.6 EC2 terminated unexpectedly: CloudTrail triage

**Step 1: Find out who/what terminated it.** CloudTrail → Event history (last 90 days) → filter `EventName = TerminateInstances` or `ResourceName = i-xxxx`. Check `userIdentity` (user, role, or `autoscaling.amazonaws.com`), `sourceIPAddress`, and `userAgent` (console vs CLI vs Terraform).

**Step 2: If CloudTrail shows nothing, it wasn't an API call.** Check:
- **ASG activity history:** scale-in, failed health check replacement, AZ rebalancing, or instance refresh.
- **Spot interruption:** EC2 console "State transition reason" shows `Server.SpotInstanceTermination`.
- **Instance-initiated shutdown:** `shutdown -h` with "shutdown behavior = terminate" gives `Client.InstanceInitiatedShutdown`.
- **EBS or capacity issues:** `Client.VolumeLimitExceeded`, `Server.InsufficientInstanceCapacity`.
- **Automation:** Lambda cleanup scripts, SSM Automation, AWS Backup, Instance Scheduler, or someone's Terraform `apply` that replaced it.

**Step 3: Investigate at scale.** Query CloudTrail logs in S3 with **Athena**, and use **AWS Config** timeline to see what changed.

**Step 4: Prevent recurrence:**
- Enable **termination protection** (and set shutdown behavior to `stop`).
- An **SCP or IAM deny** on `ec2:TerminateInstances` without an MFA or tag condition.
- **EventBridge rule** on termination events → SNS/Slack.
- Set `DeleteOnTermination=false` for critical data volumes.

---

## 10.7 Lambda fails intermittently: timeout / memory

**Diagnose first, don't guess.** CloudWatch Logs → look at the `REPORT` line:
```
Duration: 2999 ms  Billed: 3000 ms  Memory Size: 128 MB  Max Memory Used: 127 MB
```
- **"Task timed out after X seconds"** → timeout.
- **"Runtime exited with error: signal: killed"** → out of memory (Max Memory Used ≈ Memory Size).
- **Throttles (429)** → check `Throttles` and `ConcurrentExecutions` metrics against account or reserved concurrency.

**Why it's *intermittent*:**
1. **Cold starts** push duration over the timeout. This is worse in VPC-attached functions or heavy runtimes (Java).
2. **Slow downstream:** RDS connection exhaustion, a third-party API slowing down, DynamoDB throttling.
3. **Variable payload size.**
4. **Retries** make it look random (async invocations retry twice).

**Fixes:**
- Raise memory. Lambda allocates **CPU proportional to memory**, so more memory often *reduces* duration and cost. Use **AWS Lambda Power Tuning** to find the sweet spot.
- Set downstream client timeouts *below* the Lambda timeout, so you fail gracefully with a useful error.
- Reuse connections outside the handler; use **RDS Proxy** for databases.
- **Provisioned concurrency** for latency-critical paths.
- Make handlers **idempotent**, add a **DLQ/on-failure destination**, and trace with **X-Ray** or Lambda Insights.

---

## 10.8 RDS storage full: autoscaling and vacuum

**Immediate firefight:** If status is `storage-full`, the DB is effectively down for writes. Modify the instance and increase allocated storage (apply immediately). It's online for most engines, but note you can **only grow, never shrink** RDS storage.

**Prevent: Storage Autoscaling.** Enable it and set a **maximum storage threshold**. It triggers when free space is below ~10% (or 10 GiB), the condition lasts ~5 minutes, and ~6 hours have passed since the last modification. So it won't rescue you from a sudden fast fill. Pair it with a CloudWatch alarm on `FreeStorageSpace`.

**Find what's eating space:**
- **PostgreSQL bloat:** UPDATE/DELETE leaves **dead tuples** (MVCC). Autovacuum reclaims them *for reuse* but usually doesn't shrink the files.
  - `VACUUM`: marks space reusable, no heavy lock.
  - `VACUUM FULL`: rewrites the table and returns space to the OS, but takes an **exclusive lock** (downtime for that table).
  - **pg_repack**: reclaims space with minimal locking. Production favorite.
  - Check `pg_stat_user_tables.n_dead_tup`, tune autovacuum, and hunt **long-running transactions** and **idle-in-transaction** sessions, which block vacuum.
- **WAL accumulation:** an **inactive replication slot** (a forgotten replica or DMS task) makes Postgres retain WAL forever. This is a very common cause.
- **MySQL:** binary logs retention, large temp tables, undo logs.
- **Logs and temp files:** slow query/general logs.

**Teaching point:** Autoscaling treats the symptom. If growth is abnormal, find the root cause (bloat, replication slot, runaway logging, missing archival/partitioning).

---

## 10.9 Accidental deletion of S3 / RDS / EC2: disaster recovery

Frame it as **prevention + recovery + objectives (RTO/RPO)**.

**S3**
- Prevent: **Versioning** (a delete just adds a *delete marker*; remove the marker to restore), **MFA Delete**, **Object Lock**, bucket policies denying `s3:DeleteObject*`.
- Recover: delete the marker or restore a prior version. Cross-Region Replication (CRR) is a copy for DR, but note replicating deletes is optional.

**RDS**
- Prevent: **Deletion protection**, IAM deny on `DeleteDBInstance`.
- Recover: **Point-in-time restore** (to any second within retention) or restore from a snapshot into a *new* instance, then repoint the app.
- **Gotcha:** Automated backups are deleted with the instance unless you choose to retain them. Always take a **final snapshot**, and use **AWS Backup** with cross-region/cross-account copies.

**EC2**
- Prevent: termination protection, IaC so you can rebuild.
- Recover: **AMIs + EBS snapshots** (via DLM or AWS Backup), or relaunch from Terraform/CloudFormation.

**Strategy by RTO/RPO (cost increasing):** Backup & Restore → Pilot Light → Warm Standby → Multi-site Active/Active.

**Key principle:** Backups you've never restored aren't backups. Schedule **restore drills**. Protect backups in a **separate account** with vault lock so a compromised admin can't delete them.

---

## 10.10 Real-world cost optimization (example answer)

Use a **STAR** structure with real numbers. Example:

> "Our monthly bill had grown about 30% with no traffic growth. I used **Cost Explorer + CUR in Athena**, grouped by tag and service, and found four big levers:"

| Finding | Action | Typical impact |
|---|---|---|
| NAT GW data processing from S3/DynamoDB traffic | Added **gateway endpoints** | Hundreds of $/month, instantly |
| Oversized EC2 (CPU <15%) | **Compute Optimizer** rightsizing, moved to **Graviton** | 20-40% on those workloads |
| gp2 volumes, orphaned EBS/snapshots, idle EIPs | gp2 → **gp3**, cleanup job, snapshot lifecycle | 20% on EBS |
| Steady baseline compute | **Compute Savings Plans** (covering ~70% of baseline) | up to ~30-60% vs on-demand |
| Dev/QA running 24/7 | **Instance Scheduler** (nights/weekends off) | ~65% on non-prod |
| S3 growth | **Lifecycle + Intelligent-Tiering** | storage cost down |
| CloudWatch Logs forever | Set **retention** | quiet but real savings |

Also: **Spot** for stateless/batch, **tagging + Budgets + Anomaly Detection** for governance.

**Teaching point:** Order of operations matters: eliminate waste first, then rightsize, *then* buy commitments. Buying Savings Plans before rightsizing locks in waste.

---

## 10.11 Recent challenge and resolution (template)

Pick a real one. A solid shape, using intermittent Lambda or ASG issues as the example:

1. **Situation:** "Checkout API had intermittent 5xx during peak."
2. **Investigation:** ALB metrics, then CloudWatch Logs Insights, then X-Ray traces pointed to DB connection exhaustion from Lambda concurrency spikes.
3. **Action:** Introduced **RDS Proxy**, capped reserved concurrency, added SQS buffering.
4. **Result:** 5xx dropped from ~2% to <0.05%, p99 latency improved, added alarms and a runbook.
5. **Lesson:** "We load-tested and added connection-limit alarms so it can't recur silently."

**Interviewers want:** a methodical diagnosis (not guessing), a measurable result, and a lasting preventive fix.

---

## 10.12 Auto Scaling Group not launching instances

Start at **EC2 → ASG → Activity tab**. It almost always contains the exact error. Common causes:

| Cause | Clue / Fix |
|---|---|
| **Max size reached**, or desired = current | Check min/desired/max |
| **Quota exceeded** (vCPU limits) | `VcpuLimitExceeded`; request a Service Quota increase |
| **No capacity** for instance type in AZ | `InsufficientInstanceCapacity`; use multiple instance types/AZs (mixed instances policy) |
| **Launch template problems** | AMI deleted/not shared, key pair missing, SG in wrong VPC |
| **KMS permissions** | Encrypted AMI/EBS: the ASG **service-linked role** needs access to the key (a classic!) |
| **Subnet out of free IPs** | Add subnets/larger CIDR |
| **IAM instance profile** | Missing or no `iam:PassRole` |
| **Launch-terminate loop** | Instances fail ELB health checks (app didn't start in grace period) and get replaced repeatedly. Fix health check grace period/user-data |
| **Suspended processes** | `Launch` or `AZRebalance` suspended |
| **Spot** | Price/capacity unavailable |

**Teaching point:** The ASG's own activity history gives the error. Don't start by SSHing around.

---

## 10.13 Day-to-day AWS services (template)

Group by function to sound organized:
- **Compute:** EC2, ASG, ECS/EKS, Lambda
- **Network:** VPC, ALB/NLB, Route 53, CloudFront, Transit Gateway, VPC endpoints
- **Storage/DB:** S3, EBS, EFS, RDS/Aurora, DynamoDB, ElastiCache
- **Security/IAM:** IAM, KMS, Secrets Manager, GuardDuty, Security Hub, WAF, Config
- **Observability:** CloudWatch (metrics/logs/alarms), CloudTrail, X-Ray
- **Delivery/IaC:** CloudFormation/Terraform, CodePipeline/CodeBuild, Systems Manager
- **Governance/Cost:** Organizations, SCPs, Control Tower, Cost Explorer, Budgets

Then say what you *do* with them: "Monday I review GuardDuty findings and cost anomalies; during the week it's PR reviews on Terraform, ASG tuning, incident response, and patching via SSM Patch Manager."

---

## 10.14 EFS performance and bursting credits

**Throughput modes:**
- **Elastic** (default recommendation today): scales automatically with workload, pay per use. Best for spiky/unpredictable.
- **Provisioned:** you set throughput independent of size. For steady, high throughput on small datasets.
- **Bursting:** throughput scales with **stored size**.

**Bursting mechanics:** Baseline is ~50 KiB/s per GiB stored (≈50 MiB/s per TiB). Below baseline you *earn credits*; above it you *spend credits* to burst (≈100 MiB/s per TiB, with a floor for small file systems). The credit balance starts large (~2.1 TiB worth), but a **small file system with sustained heavy I/O drains credits**, then throughput collapses to baseline, which looks like a mysterious slowdown.

**Diagnose:** CloudWatch metrics `BurstCreditBalance` (trending to 0), `PermittedThroughput` vs `MeteredIOBytes`, and `PercentIOLimit`.

**Fixes:** switch to **Elastic** or **Provisioned** throughput, parallelize I/O, avoid many tiny files, and use the **General Purpose** performance mode (lowest latency).

**Teaching point:** The "slow on day 12 but fine on day 1" pattern is a signature of credit exhaustion.

---

## 10.15 EFS vs EBS: real-world selection

| | **EBS** | **EFS** |
|---|---|---|
| Type | Block storage | Shared NFS file system |
| Scope | **Single AZ**, usually one instance | **Multi-AZ**, thousands of concurrent clients |
| Latency | Lowest (sub-ms) | Higher (network file system) |
| OS | Linux/Windows | **Linux only** (NFSv4) |
| Cost (approx.) | ~$0.08/GB (gp3) | ~$0.30/GB (Standard) |
| Sizing | Provision size | Elastic, pay for what's used |

**Choose EBS** for databases, boot volumes, and latency-sensitive single-writer workloads.

**Choose EFS** when multiple instances/pods across AZs need the *same files*: CMS uploads (WordPress), shared config/content, ML training datasets, home directories, Lambda/container shared storage.

**Rule of thumb:** If only one machine needs it, EBS. If many need it simultaneously, EFS (or FSx for Windows/Lustre/NetApp, or S3 if objects will do). And if EFS is mostly cold data, use **EFS IA/Archive lifecycle** to cut cost.

---

## 10.16 Disabling console access for IAM users

**Fastest, clean method:** remove the login profile (the console password).
```bash
aws iam delete-login-profile --user-name alice
```
Or console: IAM → Users → Security credentials → **Manage console access → Disable**.

**Important caveats (this is what interviewers test):**
- This **doesn't** disable programmatic access. Also deactivate/delete **access keys**.
- Existing console sessions may remain valid until expiry. To revoke immediately, attach an inline deny policy with `aws:TokenIssueTime` condition ("revoke older sessions").
- Remove MFA devices only if offboarding fully.

**At scale or as best practice:**
- Move humans to **IAM Identity Center (SSO)** with a central IdP. Then disabling a user in one place cuts access to all accounts, with no long-lived IAM users to chase.
- Use an **SCP** or group-attached deny to restrict console access for specific classes of users (e.g., service accounts that should be API-only).

---

## 10.17 Cross-account Lambda (Account A) → S3 (Account B)

**Core rule:** For cross-account access, **both sides must allow it**: identity policy in A *and* resource policy in B. (Within one account, either is enough.)

**Option 1: Bucket policy (simplest)**

*Account A:* Lambda execution role policy
```json
{ "Effect":"Allow", "Action":["s3:GetObject","s3:PutObject"],
  "Resource":"arn:aws:s3:::bucket-b/*" }
```
*Account B:* bucket policy
```json
{ "Effect":"Allow",
  "Principal":{"AWS":"arn:aws:iam::<A-ID>:role/lambda-exec-role"},
  "Action":["s3:GetObject","s3:PutObject"],
  "Resource":"arn:aws:s3:::bucket-b/*" }
```

**Option 2: Assume a role in B.** Create `RoleInB` (trust policy allows the Lambda role from A; permissions allow S3). Lambda calls `sts:AssumeRole`, then uses temp credentials. Better when you need to touch multiple services in B.

**Gotchas:**
- **SSE-KMS:** the KMS **key policy** in B must also allow A's role (`kms:Decrypt`, `kms:GenerateDataKey`).
- **Object ownership:** objects written by A are owned by A unless the bucket uses **Bucket owner enforced** (recommended, disables ACLs).
- Using the exact **role ARN** (not `:root`) as principal follows least privilege.

---

## 10.18 AWS STS and temporary credentials

**What it is:** **Security Token Service** issues short-lived credentials so you avoid long-lived access keys.

**What you get back:** `AccessKeyId` + `SecretAccessKey` + **`SessionToken`** + `Expiration`. The session token is what makes it temporary and mandatory in requests.

**Key APIs:**
| API | Use |
|---|---|
| `AssumeRole` | Cross-account, or role switching |
| `AssumeRoleWithWebIdentity` | OIDC (EKS IRSA, GitHub Actions) |
| `AssumeRoleWithSAML` | Enterprise SSO |
| `GetSessionToken` | MFA-protected API access for an IAM user |
| `GetFederationToken` | Legacy federation |

**Mechanics:** The caller proves identity; STS checks the target role's **trust policy**; if allowed, it returns credentials scoped to the role's permissions (optionally narrowed by a **session policy**). Duration is 15 min to 12 hours (default 1 hour; **role chaining caps at 1 hour**).

**Where you already use it unknowingly:** EC2 instance profiles, Lambda execution roles, and ECS task roles are all STS under the hood, auto-rotated. That's why they beat hardcoded keys.

**Extras worth mentioning:** `ExternalId` (confused deputy protection for third parties), session tags, and source identity for auditing in CloudTrail.

---

## 10.19 Trust policy vs permissions policy

An IAM **role** has two separate policy types answering two separate questions:

| | **Trust policy** | **Permissions policy** |
|---|---|---|
| Question | **Who** can assume this role? | **What** can the role do once assumed? |
| Type | Resource-based policy on the role | Identity-based policy |
| Key element | `Principal` + `sts:AssumeRole` | `Action` + `Resource` |

Trust policy:
```json
{ "Effect":"Allow",
  "Principal":{"Service":"lambda.amazonaws.com"},
  "Action":"sts:AssumeRole" }
```
Permissions policy:
```json
{ "Effect":"Allow", "Action":"s3:GetObject",
  "Resource":"arn:aws:s3:::my-bucket/*" }
```

**Analogy:** Trust policy is the **guest list at the door**, and the permissions policy is **what you're allowed to do inside**.

**Debug rule:** "AccessDenied on AssumeRole" means look at the **trust policy** (and the caller's own permission to call `sts:AssumeRole`). "AccessDenied on an S3/DynamoDB call" means look at the **permissions policy**, plus any SCPs, permission boundaries, session policies, or resource policies.

---

## 10.20 Cross-account Lambda (A) → DynamoDB (B)

**Approach 1: Assume a role in Account B (classic, works everywhere)**

*Account B*, `DDBAccessRole`:
- Trust: allow Lambda's role in A to `sts:AssumeRole`.
- Permissions: `dynamodb:GetItem/PutItem/Query` on the table ARN.

*Account A*, Lambda role needs `sts:AssumeRole` on `arn:aws:iam::<B-ID>:role/DDBAccessRole`.

Code flow: Lambda calls STS `AssumeRole`, builds a DynamoDB client with the temp credentials, and queries the table.

**Approach 2: Resource-based policy on the table (modern, simpler)**
DynamoDB now supports resource policies on tables. Account B attaches a policy allowing A's Lambda role. In Account A, the role's identity policy allows the actions, and the code references the **full table ARN** as `TableName`. No role-hopping, no credential refresh logic.

**Gotchas:**
- Customer-managed **KMS key** on the table needs a key policy allowing A.
- If using Approach 1, **cache the credentials** until near expiry. Don't call STS on every invocation (adds latency, risks throttling).
- Set the SDK region correctly, and make sure Lambda can reach STS/DynamoDB (VPC endpoints if in a VPC).

**When to choose:** Approach 2 for a single table and simplicity; Approach 1 when you need multiple services or tighter control in B.

---

## 10.21 Disadvantages of EBS in multi-AZ Kubernetes

**Root problem: EBS volumes are AZ-locked and single-attach (RWO).**

1. **Pod rescheduling failures:** A StatefulSet pod using a volume in `us-east-1a` can only run on nodes in 1a. If 1a has no capacity or nodes, the pod is stuck `Pending` ("volume node affinity conflict") even though other AZs are healthy.
2. **Poor HA:** An AZ failure takes the data offline until you restore a snapshot elsewhere.
3. **Slow failover:** If a node dies, detach/attach can take minutes (force-detach delays), causing longer recovery.
4. **Binding mistakes:** With `Immediate` binding, the PV is created in a random AZ *before* the scheduler picks a node, which can strand pods.
5. **Not shareable:** No ReadWriteMany, so multiple pods can't share a volume (except io1/io2 Multi-Attach, with major caveats and a cluster-aware filesystem).
6. **Attachment limits** per instance type, plus Cluster Autoscaler struggles when node groups span AZs but PVs don't.

**Mitigations:**
- Use `volumeBindingMode: WaitForFirstConsumer` so the volume is created in the AZ where the pod is scheduled.
- **One node group (ASG) per AZ** for Cluster Autoscaler, or use **Karpenter**.
- Spread replicas with topology spread constraints; the app should replicate data itself (e.g., Kafka, Postgres operator).
- Use **EFS** for shared/RWX needs.
- Regular **VolumeSnapshots** for restore into another AZ.

---

## 10.22 Secrets Manager vs SSM Parameter Store

| | **Secrets Manager** | **SSM Parameter Store** |
|---|---|---|
| Purpose | Secrets (DB creds, API keys) | Config + secrets |
| **Automatic rotation** | **Yes** (native for RDS, Redshift, DocumentDB; Lambda for custom) | No (DIY) |
| Cost | ~$0.40/secret/month + API calls | **Standard tier free** (advanced tier paid) |
| Size | 64 KB | 4 KB standard / 8 KB advanced |
| Cross-account sharing | Resource policy | Limited (advanced tier sharing via RAM) |
| Multi-region replication | Built in | No |
| Encryption | Always KMS | Optional (`SecureString`) |
| Hierarchy/versioning | Versions | Hierarchical paths (`/app/prod/db`) + versions |

**Rule of thumb I use:**
- **Parameter Store** for application config and non-rotating values (feature flags, endpoints, AMI IDs), where free hierarchical config is great.
- **Secrets Manager** for anything needing **rotation**, cross-account/cross-region access, or audit-sensitive credentials, especially RDS passwords.

**Pattern:** The app reads secrets at runtime via IAM role (never baked into images or env files), caches them (Lambda extension/SDK caching) to cut latency and API cost, and handles rotation without redeploy.

---

## 10.23 Day-to-day database operational tasks (template)

- **Backups & recovery:** verify automated backups/PITR, copy snapshots cross-region, and run **restore drills**.
- **Monitoring:** CloudWatch + **Performance Insights** + Enhanced Monitoring: CPU, `FreeableMemory`, `FreeStorageSpace`, connections, `ReplicaLag`, IOPS/latency alarms.
- **Performance tuning:** slow query log, `EXPLAIN`, index review, parameter groups tuning.
- **Maintenance:** minor/major version upgrades in a test environment first, maintenance windows, certificate rotation.
- **HA/DR:** Multi-AZ failover tests, read replica health, promoting replicas.
- **Capacity:** storage autoscaling, rightsizing, connection pooling (**RDS Proxy**).
- **Housekeeping:** vacuum/bloat (Postgres), archiving/partitioning old data, replication slot checks.
- **Security:** least-privilege DB users, encryption at rest (KMS) and in transit (SSL), secrets rotation, audit logging, no public accessibility.
- **Migrations:** DMS/Schema Conversion Tool for moves or engine changes.

---

## 10.24 Production Lambda usage and architectures

**Patterns:**
1. **API backend:** API Gateway/ALB → Lambda → DynamoDB/RDS Proxy.
2. **Event processing:** S3 upload → Lambda (thumbnail, virus scan, ETL).
3. **Queue consumers:** SQS → Lambda with **partial batch responses** and a DLQ.
4. **Streams:** Kinesis/DynamoDB Streams → Lambda.
5. **Scheduled jobs:** EventBridge Scheduler → Lambda (cleanup, reports).
6. **Orchestration:** **Step Functions** coordinating Lambdas for multi-step workflows with retries.
7. **Ops automation:** Config rule → Lambda auto-remediation, snapshot cleanup, tag enforcement, Slack alerts from CloudWatch alarms.

**Production hardening (the part that matters):**
- **Idempotency** (events are at-least-once; Powertools has helpers).
- **DLQs / failure destinations**, retries with backoff.
- **Reserved concurrency** to protect downstream systems; **provisioned concurrency** for latency.
- **Versions + aliases + CodeDeploy canary** deployments for safe rollout/rollback.
- Least-privilege role per function, secrets from Secrets Manager.
- Observability: structured logs, X-Ray, custom metrics, alarms on Errors/Throttles/Duration.

**Know the limits:** 15-minute max runtime, 10 GB memory, 6 MB sync payload, 1000 default regional concurrency (adjustable). If you hit those, think containers (Fargate) or batch instead.

---

## 10.25 IAM User vs IAM Role

| | **IAM User** | **IAM Role** |
|---|---|---|
| Identity | A specific person/app | An assumable identity, no owner |
| Credentials | **Long-lived** (password, access keys) | **Temporary** (STS, auto-expire) |
| Used by | Legacy/humans/some third-party tools | AWS services, cross-account, federation |
| Risk | Keys leak, rarely rotated | Low: short-lived, nothing to leak long-term |

**Modern best practice:**
- **Humans** → federate via **IAM Identity Center/SSO**, not IAM users.
- **AWS workloads** → roles (EC2 instance profile, Lambda execution role, ECS task role, IRSA for EKS).
- **CI/CD** → OIDC federation to a role (e.g., GitHub Actions), no stored keys.
- **Cross-account** → roles, always.
- IAM users are a last resort (e.g., a third-party system that can't assume roles). If you must, enforce MFA, rotate keys, and apply least privilege.

**One-liner:** "Users are *who you are*; roles are *what you temporarily become*."

---
