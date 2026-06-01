# Centralised Logging on AWS — CloudWatch, S3, SQS, Kibana, Fluent Bit

A concept guide. Layman terms, focused on **how the pieces fit together** and **why each one exists**, not on Terraform syntax.

---

## The Big Picture

Imagine your AWS account is a busy restaurant. Every service (ECS apps, Lambda, RDS, DMS) is a cook shouting out what they're doing. You want all those shouts written down, kept long-term, and searchable.

The pipeline we're building looks like this:

```
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐
   │ AWS services│───►│ CloudWatch  │───►│  Firehose   │───►│    S3    │
   │ (cooks)     │    │ Logs        │    │ (conveyor)  │    │(archive) │
   └─────────────┘    └─────────────┘    └─────────────┘    └─────┬────┘
                                                                  │
                                                            (event notify)
                                                                  ▼
                                                            ┌──────────┐
                                                            │   SQS    │
                                                            │(doorbell)│
                                                            └─────┬────┘
                                                                  │
                                                                  ▼
                                                       ┌─────────────────┐
                                                       │ Elastic / Kibana│
                                                       │  (search + UI)  │
                                                       └─────────────────┘
```

Each block has a single job. Understand the job, and the architecture clicks.

---

## The Players

### 1. CloudWatch Logs — the "default notebook"

When you launch most AWS services, they already know how to write logs into CloudWatch. You don't configure anything fancy — it's the default place logs land.

- **Good at:** live tailing during an incident, short-term search (Logs Insights).
- **Bad at:** keeping logs cheap for years, cross-service dashboards, complex queries.
- **Cost gotcha:** you pay **$0.50 per GB ingested**. For chatty apps, this is the bill that surprises teams.

### 2. Kinesis Data Firehose — the "conveyor belt"

Firehose is a managed pipe. You shove records in one end, it batches them up, optionally transforms them, and dumps them at a destination (S3, OpenSearch, Splunk, etc.).

- **Why it exists:** moving logs in real time without you running servers.
- **Knobs:** buffer size (e.g. 64 MB), buffer interval (e.g. 5 min) — whichever fills first triggers a write. Bigger buffers = fewer S3 files = cheaper but higher latency.
- **Common confusion:** Firehose does **not** store anything itself. It's a pipe, not a bucket.

### 3. S3 — the "archive warehouse"

Object storage. Cheap, durable, queryable later with Athena.

- **Good at:** long-term retention, compliance, cost (~$0.023/GB-month for Standard, dramatically less for Glacier).
- **Bad at:** real-time tailing, ad-hoc search without extra tooling.
- **Lifecycle policies:** auto-move objects to Glacier after 90 days, delete after 7 years, etc. Set this once, save thousands.

### 4. SQS — the "doorbell"

A queue. Simple as that. Someone drops a message in; someone else picks it up.

In this architecture, SQS is **not** carrying log data. It's carrying **notifications** ("hey, a new log file just landed in S3 at this path"). Elastic reads the doorbell so it knows exactly which file to fetch — instead of constantly walking the S3 aisles checking shelves.

- **Why a queue?** Decoupling. S3 doesn't wait for Elastic; it just rings the bell and moves on. If Elastic is down, messages pile up in SQS and get processed when it's back.
- **Mental model:** SQS = the pager. S3 = the warehouse. The two are unrelated until you wire them.

### 5. Elastic / Kibana — the "search UI"

Elastic is a search engine optimised for logs. Kibana is its web UI.

- **What it gives you:** Discover (full-text search), Dashboards (graphs), Alerting (paging on patterns), cross-service correlation.
- **Where it lives:** can be self-hosted (you run servers), or Elastic Cloud (managed SaaS).
- **How it gets data here:** it reads from S3 + SQS using IAM credentials you give it.

### 6. Fluent Bit — the "log forwarder sidecar" (ECS only)

A tiny log shipper. It runs as a container next to your app inside an ECS task. The app writes logs to stdout/stderr or a file; Fluent Bit reads them and ships them wherever you tell it — CloudWatch, Firehose, S3 directly, even multiple destinations at once.

This is what AWS markets as **FireLens** — FireLens is just AWS's name for "use Fluent Bit (or Fluentd) as a sidecar in your ECS task, configured through the task definition."

---

## Two Architectures for ECS

ECS is the interesting case because, unlike Lambda or RDS, **you have a choice**.

### Architecture A — Keep CloudWatch, archive to S3 (recommended starting point)

```
ECS app ──awslogs driver──► CloudWatch ──subscription filter──► Firehose ──► S3
```

- **No changes to the ECS task definition.** The `awslogs` log driver stays.
- A **CloudWatch subscription filter** tees a copy of every log line to Firehose.
- **Works identically for Lambda, RDS, DMS, CodeBuild** — anything already in CloudWatch.
- **Trade-off:** you still pay CloudWatch ingestion. Mitigation: drop retention from 30d → 7d once S3 archive is verified. CW becomes the short-term tail, S3 is the long-term store.

### Architecture B — Fluent Bit sidecar (skip CloudWatch for ECS)

```
ECS app ──stdout──► Fluent Bit sidecar ──► Firehose ──► S3
                                       └──► CloudWatch (optional dual-shipping)
```

- **Modify the ECS task definition** to:
  1. Add a Fluent Bit container alongside the app container.
  2. Change the app container's `logConfiguration` from `awslogs` to `awsfirelens`.
  3. Mount a Fluent Bit config file (via S3, SSM, or baked into the image) that defines outputs.
- **Pro:** logs bypass CloudWatch entirely → no CW ingestion cost for ECS.
- **Pro:** Fluent Bit can parse, filter, enrich logs *before* they leave the task.
- **Pro:** dual-shipping is easy — send the same log to CW (for live tail) and Firehose (for S3).
- **Con:** every task definition changes — a real deployment touching every ECS service.
- **Con:** Fluent Bit is software you now operate. Misconfigure it and logs vanish.
- **Con:** **only works for ECS.** Lambda/RDS/DMS can't run sidecars.

### How to choose

| If your priority is… | Pick |
|---|---|
| Get S3 archive quickly with minimal change | **A** (subscription filter) |
| Cut CloudWatch ingestion bill significantly | **B** (Fluent Bit), but only for ECS |
| Unified pipeline across ECS + Lambda + RDS + DMS | **A** — it's the only one that covers non-ECS |
| Need parsing/filtering before logs leave the host | **B** |

Most teams start with **A** for everything, then move just the high-volume ECS services to **B** later if CW cost becomes painful. You don't have to pick one for the whole estate.

---

## What Logs Where, by AWS Service

The reason Architecture A is appealing is this: almost everything in AWS already logs to CloudWatch by default, so one downstream pipe covers them all.

| Service | Native logging | Can it skip CloudWatch? |
|---|---|---|
| **ECS (with `awslogs` driver)** | CloudWatch | Yes, via FireLens/Fluent Bit |
| **Lambda** | CloudWatch (`/aws/lambda/*`) | No — always lands in CW first |
| **RDS** | CloudWatch (when `enabled_cloudwatch_logs_exports` set) | No |
| **DMS** | CloudWatch | No |
| **CodeBuild** | CloudWatch or S3 (native option) | Yes — has native `s3_logs` config |
| **API Gateway** | CloudWatch access + execution logs | No |
| **ALB / NLB** | **S3 directly** via `access_logs` | N/A — already S3-native |
| **CloudFront** | **S3 directly** | N/A — already S3-native |
| **VPC Flow Logs** | CloudWatch *or* S3 directly | Yes — `log_destination_type = "s3"` |
| **WAF** | CloudWatch, Firehose, *or* S3 directly | Yes |

**Takeaway:** ALB, CloudFront, VPC Flow Logs, and WAF can write straight to S3 — skip CloudWatch entirely if that's your archive target. Everything else funnels through CloudWatch.

---

## The "Doorbell" Pattern in Detail

This is the part most engineers haven't seen before. Let's unpack it.

**Problem:** Elastic needs to ingest log files from S3. How does it know when a new file arrives?

**Naive solution — polling:** Elastic asks S3 every 30 seconds, "any new files?" S3 lists the bucket, Elastic compares against what it's seen, fetches new ones. Wasteful and slow.

**Better solution — push notifications:**

1. You enable S3 event notifications on the bucket: *"on every `ObjectCreated` event, send a message to this SQS queue."*
2. The message contains the bucket name and object key.
3. Elastic's S3 input plugin reads from the SQS queue, fetches each referenced object, deletes the SQS message on success.

**Why SQS specifically and not, say, SNS?**
- **Buffering** — if Elastic is down, messages persist in SQS for up to 14 days.
- **Acknowledgement** — Elastic only deletes the message *after* successful ingestion. Crash mid-way and it'll be redelivered.
- **Parallelism** — multiple Elastic workers can consume the same queue.

You could use SNS → Lambda → Elastic too, but SQS is the pattern Elastic's own docs recommend.

---

## IAM — Who's Allowed to Do What

Every arrow in the diagram needs a permission. Common gotchas:

1. **CloudWatch → Firehose** needs an IAM role trusted by `logs.<region>.amazonaws.com` with `firehose:PutRecord*`.
2. **Firehose → S3** needs an IAM role trusted by `firehose.amazonaws.com` with `s3:PutObject` + `kms:GenerateDataKey` (if S3 uses a KMS key).
3. **S3 → SQS** is *not* a role — it's an **SQS queue policy** that allows the `s3.amazonaws.com` principal to `SendMessage`, scoped to the bucket ARN.
4. **Elastic → S3 + SQS** needs an IAM user *or* role with read-only S3 + consume-only SQS + KMS decrypt (if encrypted).

**KMS double-handshake:** if your bucket uses a customer-managed key, the principal needs `kms:Decrypt` in *their* policy **and** the KMS key's own key policy must list them as allowed. Forgetting the key policy side is the #1 silent failure in this pipeline.

---

## Costs — Where the Bill Comes From

Knowing the cost model helps you defend the design.

| Component | Cost driver | Order of magnitude |
|---|---|---|
| CloudWatch Logs ingestion | $0.50/GB | This is usually the biggest line item |
| CloudWatch Logs storage | $0.03/GB-month | Cheap, but it accumulates |
| Firehose | ~$0.029/GB ingested | Smaller than CW, but additive |
| S3 Standard | $0.023/GB-month | Tiny compared to CW |
| S3 → Glacier (lifecycle) | $0.004/GB-month | Effectively free at scale |
| SQS | $0.40 per million requests | Negligible for log-event volumes |
| KMS | $0.03 per 10k requests | Negligible |

**Cost optimisation moves, in order:**
1. **Drop CW retention** to 7 days once S3 archive is verified.
2. **Compress** in Firehose (`GZIP`) — typically 10x reduction in S3 storage.
3. **Partition** S3 by `service/year/month/day/` so Athena scans less.
4. **Lifecycle to Glacier** at 90 days, expire at 7 years.
5. **(Advanced)** Move highest-volume ECS services to FireLens to skip CW ingestion entirely.

---

## Why This Matters for Your Career

Every mid-to-senior infra/DevOps/SRE role expects fluency in this exact pattern. Specifically:

- **Observability** is one of three pillars of platform engineering (the others being CI/CD and IaC). You can't run production without it.
- **"Pipe one system into another via a queue with a notification"** is a pattern that recurs everywhere — S3→SQS→worker, EventBridge→Lambda, Kinesis→Firehose→Splunk. Internalise the shape once, recognise it forever.
- **Cost-aware logging** is a real differentiator in interviews. Most candidates know "logs go to CloudWatch." Few can articulate when to bypass it.
- **Fluent Bit / FireLens** is increasingly common — knowing when it's worth the operational cost vs. just using subscription filters is a judgment call seniors make.

If you can sketch the diagram at the top of this doc on a whiteboard and defend the trade-offs for each arrow, you're at a senior-infra level on logging architecture.

---

## Glossary (one-liners)

- **CloudWatch Logs subscription filter** — a rule on a CW log group that copies every matching log event to a destination (Firehose, Lambda, Kinesis Data Streams).
- **`awslogs` driver** — the default ECS log driver that ships container stdout/stderr to a CW log group.
- **FireLens** — AWS-branded name for using Fluent Bit / Fluentd as a sidecar log router in ECS.
- **Sidecar container** — a second container in the same task as your app, sharing its lifecycle, doing a supporting job (logs, secrets, proxying).
- **S3 event notification** — bucket-level configuration that emits an event (to SQS/SNS/Lambda/EventBridge) when objects are created, deleted, etc.
- **Visibility timeout (SQS)** — how long a message stays "in flight" after a worker picks it up before it's redelivered to another worker.
- **Buffering (Firehose)** — accumulating records before flushing to S3, controlled by size and time thresholds.
- **Athena** — serverless SQL over S3. Cheap way to query archived logs without standing up a database.

---

## Questions Worth Asking on Any New Project

1. Where do logs land *by default*? (Usually CloudWatch.)
2. What's the **retention** there, and is it costing more than it should?
3. Is there an **archive** path (S3) for compliance / long-term?
4. Is anything **visualised** beyond CloudWatch Logs Insights? (Kibana, Grafana, Datadog?)
5. Who can **read** logs, and is that access **audited**?
6. If the visualisation tool dies, are logs still being collected? (i.e. is the archive independent of the search layer?)

Good logging architectures answer all six. Bad ones answer only the first.
