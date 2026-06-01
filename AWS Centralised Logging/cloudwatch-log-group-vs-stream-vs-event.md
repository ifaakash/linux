# CloudWatch Logs Hierarchy — Group vs Stream vs Event

How CloudWatch organises log data, and why this matters for subscription filters, debugging, and Elastic ingestion.

---

## The three-level hierarchy

CloudWatch Logs is organised like a filing cabinet:

```
Log Group         ←  the cabinet  (e.g. "/smp/nonprod/smp-internal-api")
 └── Log Stream   ←  a folder     (e.g. "ecs/app/task-abc123")
      └── Log Event  ← a single page (one log line + timestamp)
```

### Log Event — the leaf

A **log event** is one entry. Just two fields that matter:
- `timestamp` — when it happened
- `message` — the text your app printed

Example: `"[2026-06-01T14:23:45Z] INFO user=alice action=login"` with timestamp `1717252425000`.

Every line your app writes to stdout becomes one log event. That's it.

### Log Stream — the source

A **log stream** is a sequence of events that come from the **same source instance**. The defining property: events within a stream are guaranteed to be in **chronological order** and come from **one writer**.

In practice, you get one stream per:
- **ECS task instance** — each running container gets its own stream. If you have 3 tasks of `smp-internal-api` running, you have 3 streams.
- **Lambda invocation container** — one stream per "execution environment" (roughly one per warm Lambda container).
- **EC2 instance** — one stream per host (if using the CloudWatch agent).

When an ECS task starts, CloudWatch creates a new log stream (e.g. `ecs/smp-internal-api/abc123def456` using the task ID). When the task dies, the stream stays around (it's just data) but nothing new gets written to it. A new task spawns a new stream.

### Log Group — the logical bucket

A **log group** is a collection of streams that share **configuration** — retention period, encryption key, metric filters, subscription filters, IAM access policies.

Think of it as "all the streams for one application." For example:
- `/smp/nonprod/smp-internal-api` is one log group containing streams from **every** task that ever ran for that service.
- Retention (30 days, 7 days, etc.) is set at the **group** level, applies to all streams within.
- Subscription filters attach to the **group**, so they capture events from every stream inside.

---

## Concrete picture for one service

Say `smp-internal-api` is running 3 ECS tasks today, and yesterday 5 different tasks ran (different IDs because of restarts/scaling). Here's what CloudWatch holds:

```
Log Group: /smp/nonprod/smp-internal-api
│  retention: 30 days
│  encryption: KMS key XYZ
│  subscription filter: → Firehose smp-archive
│
├── Log Stream: ecs/smp-internal-api/task-aaa111  (yesterday's task, now dead)
│   ├── 14:00:01  "INFO server started"
│   ├── 14:00:05  "INFO request_id=1 GET /health 200"
│   └── ...  (thousands of events)
│
├── Log Stream: ecs/smp-internal-api/task-bbb222  (yesterday's, dead)
│   └── ...
│
├── Log Stream: ecs/smp-internal-api/task-ccc333  (running now)
│   ├── 13:55:10  "INFO server started"
│   ├── 14:23:45  "INFO request_id=abc-123 user=alice action=login"
│   └── ...  (still receiving)
│
├── Log Stream: ecs/smp-internal-api/task-ddd444  (running now)
│   └── ...
│
└── Log Stream: ecs/smp-internal-api/task-eee555  (running now)
    └── ...
```

Each stream is an immutable, ordered timeline of events from one task. The group is the umbrella that holds them all and applies policy to them all.

---

## Why both layers exist

It's a separation of concerns:

| Layer | Answers the question | Used for |
|---|---|---|
| **Stream** | *Where did this log come from?* | Debugging — "which container instance produced this error?" |
| **Group** | *What policy applies?* | Governance — "how long do we keep these logs? who can read them? where else do they get shipped?" |

If CloudWatch had no streams, you couldn't tell which of 10 tasks produced an error. If it had no groups, you'd be setting retention and IAM individually on every task instance — unmanageable at scale.

---

## How this connects to subscription filters and S3 archive

This hierarchy matters for the centralised logging pipeline because:

### 1. Subscription filters attach to log groups, not streams

When you set up a filter on `/smp/nonprod/smp-internal-api`, it automatically captures events from **every stream** in that group — including streams that don't exist yet (future tasks). You don't have to update anything when ECS scales out or replaces tasks.

### 2. The CloudWatch envelope in S3 carries both fields

Recall the JSON envelope that lands in S3:

```json
{
  "messageType": "DATA_MESSAGE",
  "logGroup":  "/smp/nonprod/smp-internal-api",
  "logStream": "ecs/smp-internal-api/task-ccc333",
  "logEvents": [ ... ]
}
```

So in S3 you can still tell *which specific task* a log line came from — not just which service. That's gold for debugging "why did task ccc333 crash at 14:23 but task ddd444 was fine?"

### 3. In Elastic/Kibana, both fields are searchable

You can filter by `logGroup` (= service) for service-level views, or drill into `logStream` (= task instance) when investigating an incident. Without the stream field, you'd lose per-instance traceability.

---

## Mapping to other AWS services

The hierarchy is identical for every service, just named differently in practice:

| Service | Log Group | Log Stream |
|---|---|---|
| **ECS** (awslogs driver) | `/smp/nonprod/<service>` | `ecs/<container>/<task-id>` |
| **Lambda** | `/aws/lambda/<function-name>` | `<date>/[$LATEST]<execution-env-id>` |
| **CodeBuild** | `/aws/codebuild/<project>` | `<build-id>` |
| **RDS** | `/aws/rds/instance/<db>/<log-type>` | `<db-instance-id>` |
| **API Gateway** | `API-Gateway-Execution-Logs_<api-id>/<stage>` | `<date>/<request-id-prefix>` |
| **EC2 / on-host agents** | (you choose the name) | typically `{hostname}` or `{instance-id}` |

In every case: group = "the service or resource", stream = "this specific running instance of it".

---

## Limits worth remembering

- **Streams per group:** soft limit, effectively unlimited (you can have millions). Don't worry about creating too many.
- **Events per stream:** unlimited.
- **Event size:** max 256 KB per event. Larger lines get truncated. Relevant if your app logs huge stack traces or JSON blobs.
- **Subscription filters per group:** 2 (historically 1). Plan accordingly if you want to ship to multiple destinations.
- **Retention values:** must be one of AWS's preset values (1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, ...). You can't set "10 days".

---

## One-line summary

> A **log group** is the service-level bucket where retention, encryption, and subscription filters are configured. A **log stream** is one specific source instance writing into that bucket. A **log event** is a single log line with a timestamp.

---

## Mental model to take into interviews

If asked *"how does CloudWatch Logs organise data?"* — answer in this order:
1. **Group = configuration scope** (retention, IAM, filters).
2. **Stream = source identity** (one writer, ordered).
3. **Event = the actual log line** (timestamp + message).

This is the same shape as: **Kafka topic / partition / message**, or **Kinesis stream / shard / record**. Recognise the pattern once, recognise it everywhere.
