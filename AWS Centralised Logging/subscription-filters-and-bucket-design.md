# Subscription Filters and S3 Bucket Design

Two foundational decisions you face when wiring CloudWatch logs to S3: **what a subscription filter actually is**, and **how many S3 buckets to use**.

---

## Part 1 — What is a CloudWatch Logs Subscription Filter?

### The concept in one sentence

A subscription filter is a **rule attached to a CloudWatch log group that says "for every log event that arrives here, also send a copy to this destination in real time."**

### The mental model

Think of a CloudWatch log group as a river. Logs flow into it from your services. A subscription filter is a **tap** on that river that diverts a copy of the water somewhere else — without slowing the original flow.

```
   Lambda / ECS / RDS
         │
         ▼  (logs flow in)
   ┌──────────────────┐
   │  CloudWatch Log  │  ← original logs still here, searchable as normal
   │     Group        │
   └────────┬─────────┘
            │ ← subscription filter (the tap)
            ▼
   ┌──────────────────┐
   │  Destination     │  (Firehose, Lambda, or Kinesis Data Stream)
   └──────────────────┘
```

### Key properties

- **Real-time.** Events are forwarded within seconds of arriving in the log group.
- **Copy, not move.** The original log stays in CloudWatch with whatever retention is set. The filter doesn't delete or interfere with it.
- **Up to 2 filters per log group** (historically 1). Each filter can have its own destination.
- **Filter pattern** — you can choose to forward *everything* (empty pattern `""`) or only matching events (e.g. only `ERROR`-level lines). For archive use cases, you want everything; empty pattern.

### Allowed destinations

Subscription filters can only send to three types of destination:

1. **Kinesis Data Stream** — raw stream; you write the consumer.
2. **Kinesis Data Firehose** — managed pipe to S3, OpenSearch, Splunk, etc. ← **this is what archive pipelines use**.
3. **AWS Lambda** — function invoked per batch of events.

They **cannot** send directly to S3, SQS, SNS, or EventBridge. That's why Firehose sits in the middle in our architecture — it's the bridge from "subscription filter world" to "S3 world."

### General use cases (beyond log archive)

The same pattern recurs all over AWS:

| Use case | Destination |
|---|---|
| Archive logs to S3 (our setup) | Firehose → S3 |
| Ship logs to a SIEM (Splunk, Datadog, Sumo) | Firehose → 3rd-party endpoint |
| Real-time alerting on error patterns | Lambda → SNS/PagerDuty |
| Stream logs into a custom analytics pipeline | Kinesis Data Stream → consumer app |
| Centralise logs from many AWS accounts | Cross-account subscription → central Firehose |
| Replicate logs to OpenSearch | Firehose → OpenSearch |

The pattern is common enough that AWS has a console UI for it under each log group ("Subscription filters" tab).

### How it fits the SMP archive setup

You create one subscription filter **per log group**, all pointing at the same Firehose:

```
/smp/nonprod/smp-internal-api  ──filter──┐
/smp/nonprod/smp-client-api    ──filter──┤
/smp/nonprod/smp-internal-portal ─filter─┤
/smp/nonprod/smp-client-portal ──filter──┼──► Firehose ──► S3
/aws/lambda/image-promoter     ──filter──┤
/aws/codebuild/smp-apps        ──filter──┤
... (all 8+ log groups) ─────────────────┘
```

In Terraform: a `for_each` over the list of log group names, creating one `aws_cloudwatch_log_subscription_filter` per group, all referencing the same Firehose ARN.

### Things that trip people up

- **Filter is on the log group, not the service.** If you delete and recreate a log group, the filter goes with it — you have to recreate.
- **IAM is two-sided.** The subscription filter needs an IAM role that lets CloudWatch Logs put records into Firehose. Trust policy must allow `logs.<region>.amazonaws.com`.
- **No replays.** A subscription filter only forwards events that arrive *after* it's created. Pre-existing logs in the group are not back-filled.
- **Filter pattern syntax is its own thing** — not regex, not SQL. AWS docs call it "filter pattern syntax." For archive use cases (forward everything) just use `""` and never touch it.

---

## Part 2 — One S3 Bucket or Many?

When you build the archive, you have a choice: one centralised bucket for all log groups, or a separate bucket per service / per log group. **Almost always: one centralised bucket per environment, with prefixes inside.**

### Why one bucket wins

| Concern | One bucket | Bucket per service |
|---|---|---|
| Number of buckets to manage | 1 | 8+ (and growing) |
| IAM policies | One for Elastic to read | One per bucket, multiplied |
| Lifecycle policy (Glacier, expiry) | Set once | Repeat per bucket |
| KMS key | One key | Multiple keys, more rotation surface |
| Athena setup | One Glue table, one location | One table per bucket |
| AWS bucket quota | Soft limit 100 buckets/account | Burns through fast |
| Cross-service correlation (ECS + RDS for one request) | Trivial — same bucket | Query multiple locations |

### How you separate things *inside* one bucket — prefixes

The trick: separation happens via **prefixes** (folder-like paths inside the bucket), not separate buckets. Firehose can build these prefixes automatically using **dynamic partitioning**.

Recommended layout:

```
s3://nonprod-smp-logs-archive/
├── service=smp-internal-api/year=2026/month=06/day=01/hour=14/
│   └── smp-logs-1-2026-06-01-14-23-45-abc.gz
├── service=smp-client-api/year=2026/month=06/day=01/hour=14/
│   └── ...
├── service=aws-lambda-image-promoter/year=2026/month=06/day=01/hour=14/
│   └── ...
└── service=aws-codebuild-smp-apps/year=2026/month=06/day=01/hour=14/
    └── ...
```

Benefits of this prefix layout:
- **Athena partitioning** — queries scan only one service/day, not the whole bucket. Cuts query cost ~100x.
- **Lifecycle granularity** — you can apply different lifecycle rules to different prefixes (e.g. keep audit logs longer than build logs) even within one bucket.
- **IAM scoping** — you can grant read access to *one* prefix without giving the whole bucket.

The `service=` / `year=` syntax follows the **Hive partition convention**, which Athena and most query engines recognise natively.

### When you *would* want separate buckets

Only in these specific cases:

- **Compliance / data classification** — logs containing PII/PHI need a bucket with different access controls, KMS keys, or audit settings.
- **Cross-account isolation** — logs from different AWS accounts must never co-mingle in storage.
- **Massively different retention** — e.g. 7-year audit logs vs. 30-day debug logs, *and* you want hard separation rather than lifecycle rules.

For a single non-prod environment with a handful of services, none of these apply. **One bucket, multiple prefixes.**

### One bucket per *environment* — yes

A separate consideration: should `nonprod`, `stage`, `uat`, `prod` share a bucket?

**No.** Use one bucket per environment:
- `nonprod-smp-logs-archive`
- `stage-smp-logs-archive`
- `prod-smp-logs-archive`

Reasons:
- **Different AWS accounts** — prod typically lives in its own account; the bucket should too.
- **Blast radius** — a misconfigured non-prod IAM policy mustn't expose prod logs.
- **Independent lifecycle / compliance** — prod often has longer retention requirements.
- **Standard industry pattern** — every centralised logging setup at scale follows this.

### Quick decision rubric

> Default: **one bucket per environment**, all log groups in that environment write to that bucket, separated by `service=` prefix.
>
> Deviate only if compliance, account isolation, or hard retention differences force you to.

---

## How these two pieces fit together

```
For each log group:
   subscription filter ──► Firehose ──► s3://<env>-smp-logs-archive/service=<name>/...
```

- **One Firehose stream** (per environment) receives from all subscription filters.
- **Dynamic partitioning** in Firehose extracts the `logGroup` field from the CloudWatch envelope and routes it to the right prefix automatically.
- **One bucket** holds everything in that environment.
- **Elastic** reads the whole bucket via SQS notifications, but `logGroup` is still searchable per-event, so service-level views still work.

This is the simplest design that scales to hundreds of log groups without per-service Terraform sprawl.
