# What Actually Lands in S3 from CloudWatch

Trace one log line from your app, through CloudWatch → Firehose → S3, and see exactly what the final file in S3 looks like.

This is the kind of detail you don't need to *configure* — it happens automatically. But understanding the format demystifies the pipeline and helps when you ever open an S3 file expecting raw log lines and see JSON envelopes instead.

---

## End-to-end walkthrough — one log line

### Step 1 — The app writes a log

Your `smp-internal-api` ECS container does this:

```
[2026-06-01T14:23:45Z] INFO request_id=abc-123 user=alice action=login status=200 duration=42ms
```

It's just a line on stdout. The `awslogs` driver in the ECS task definition picks it up and ships it to CloudWatch.

### Step 2 — CloudWatch stores it as a log event

In the log group `/smp/nonprod/smp-internal-api`, this becomes one **log event**, stored internally as:

```json
{
  "id": "37195383933748202054930386541714857586",
  "timestamp": 1717252425000,
  "message": "[2026-06-01T14:23:45Z] INFO request_id=abc-123 user=alice action=login status=200 duration=42ms"
}
```

You never see this JSON in the CloudWatch console — the console only shows the `message` field. But this is the underlying shape.

### Step 3 — Subscription filter forwards a batch to Firehose

The filter is attached to that log group. Every new log event triggers it. CloudWatch **batches** events (it doesn't send one-by-one) and wraps a batch in a **CloudWatch Logs envelope**:

```json
{
  "messageType": "DATA_MESSAGE",
  "owner": "123456789012",
  "logGroup": "/smp/nonprod/smp-internal-api",
  "logStream": "ecs/app/task-abc123",
  "subscriptionFilters": ["smp-archive-filter"],
  "logEvents": [
    {
      "id": "37195383933748202054930386541714857586",
      "timestamp": 1717252425000,
      "message": "[2026-06-01T14:23:45Z] INFO request_id=abc-123 user=alice action=login status=200 duration=42ms"
    },
    {
      "id": "37195383933748202054930386541714857587",
      "timestamp": 1717252425100,
      "message": "[2026-06-01T14:23:45Z] INFO request_id=def-456 user=bob action=logout status=200 duration=12ms"
    }
  ]
}
```

This blob is then **gzipped + base64-encoded** under the hood and shipped to Firehose.

### Step 4 — Firehose buffers and flushes to S3

Firehose holds incoming records in a buffer until either:
- 5 minutes pass, **or**
- 64 MB accumulates,

…whichever comes first. Then it flushes everything to S3 as **one object**.

### Step 5 — What lands in S3

The file shows up at a path like:

```
s3://nonprod-smp-logs-archive/2026/06/01/14/smp-archive-stream-1-2026-06-01-14-25-30-abc12345.gz
```

(With dynamic partitioning, the path can use `service=...` prefixes — see the bucket design note.)

If you download that `.gz` file and `gunzip` it, here is what's literally inside (newline-delimited JSON, one envelope per line):

```json
{"messageType":"DATA_MESSAGE","owner":"123456789012","logGroup":"/smp/nonprod/smp-internal-api","logStream":"ecs/app/task-abc123","subscriptionFilters":["smp-archive-filter"],"logEvents":[{"id":"37195383933748202054930386541714857586","timestamp":1717252425000,"message":"[2026-06-01T14:23:45Z] INFO request_id=abc-123 user=alice action=login status=200 duration=42ms"},{"id":"37195383933748202054930386541714857587","timestamp":1717252425100,"message":"[2026-06-01T14:23:45Z] INFO request_id=def-456 user=bob action=logout status=200 duration=12ms"}]}
{"messageType":"DATA_MESSAGE","owner":"123456789012","logGroup":"/smp/nonprod/smp-internal-api","logStream":"ecs/app/task-abc124","subscriptionFilters":["smp-archive-filter"],"logEvents":[{"id":"...","timestamp":1717252430000,"message":"[2026-06-01T14:23:50Z] ERROR request_id=ghi-789 ..."}]}
{"messageType":"DATA_MESSAGE", ... }
```

Each **line** is one CloudWatch envelope containing **a batch of log events**. A single S3 file might contain hundreds or thousands of these envelope lines, each wrapping 1 to ~100 actual log events.

---

## Visualising the nesting

```
S3 file (gzipped, .gz)
└── many lines, each line is:
    └── CloudWatch envelope (JSON)
        ├── messageType, owner, logGroup, logStream  ← metadata
        └── logEvents[]                              ← array of actual logs
            └── each item: {id, timestamp, message}
                └── message: the literal log line your app printed
```

To recover your original log line, a consumer has to:

1. `gunzip` the file
2. Parse each line as JSON
3. Walk `logEvents[]`
4. Pull `.message` out of each event

This is what Elastic's `aws-cloudwatch-logs` ingest pipeline does automatically. **You don't write that code** — Elastic, Datadog, Splunk, Sumo all ship with this parser built in.

---

## Why the wrapping exists

Three reasons, so it doesn't feel arbitrary:

1. **Batching efficiency** — one envelope with 50 events is far cheaper than 50 separate API calls.
2. **Provenance** — the envelope tells the receiver *which log group, stream, and account* the events came from. When multiple sources merge into one Firehose, this is what keeps them distinguishable.
3. **Reliability** — `messageType` distinguishes real data (`DATA_MESSAGE`) from CloudWatch's heartbeat pings (`CONTROL_MESSAGE`), so consumers don't process junk as logs.

It's not over-engineering — it's the minimum metadata needed for a multi-source log pipeline to be usable downstream.

---

## What this means for your work

- **For pipeline setup:** nothing. Subscription filter → Firehose → S3, walk away. The wrapping is automatic.
- **For Elastic ingestion:** the team's Elastic side uses the built-in `aws-cloudwatch-logs` parser. No custom code.
- **For Athena queries later:** either add a Firehose **transform Lambda** to unwrap envelopes into flat events, *or* use Athena's `json_extract` and `UNNEST(logEvents)` to dig into the nested structure. Decision for later, not now.
- **For debugging:** if you ever download a file from S3 to inspect, `gunzip` and you'll see the JSON envelopes. That is correct, not a misconfiguration.

---

## Quick reference — anatomy of one envelope

| Field | Meaning |
|---|---|
| `messageType` | Either `DATA_MESSAGE` (real logs) or `CONTROL_MESSAGE` (heartbeat — ignore). |
| `owner` | AWS account ID that owns the log group. |
| `logGroup` | Full log group name (e.g. `/smp/nonprod/smp-internal-api`). |
| `logStream` | Stream the events came from (identifies the source instance — see the hierarchy note). |
| `subscriptionFilters` | Names of filters that forwarded this batch — usually one. |
| `logEvents[]` | Array of events. Each has `id`, `timestamp` (epoch ms), `message` (the raw log line). |

---

## Two-line summary

> Files in S3 are gzipped, newline-delimited JSON. Each line is a CloudWatch envelope wrapping a batch of log events with metadata about where they came from. Elastic and other SIEMs parse this format natively; for Athena you either unwrap with a Lambda transform or query the nested JSON directly.
