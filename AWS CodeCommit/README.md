# AWS CodeCommit (GitHub → CodeCommit Mirror)

> **Status: design phase.** Approach is being validated with the team before build. This doc captures the *mental model* for reasoning about the mirror — not final implementation.

## What it is

AWS CodeCommit is a **fully managed private Git host** that lives inside your AWS account — AWS's own GitHub/GitLab. It speaks plain Git (clone, push, pull, branches, PRs), but access is governed by **IAM** instead of per-seat logins.

- Repos are private, scoped to one AWS account + region.
- Auth: HTTPS Git credentials, SSH keys, or the `git-remote-codecommit` helper using AWS credentials.
- Integrates natively with CodePipeline / CodeBuild / CodeDeploy.

> ⚠️ **Availability:** AWS closed CodeCommit to *new* customers around July 2024. It only works in accounts that already had it. **Confirmed available** in the target P41 account.

## The problem we're solving

Some P41 AI Labs engineers **don't have a GitHub seat**, so they can't reach the AI Labs repos. They *do* have AWS access. So: mirror the repos into CodeCommit, where AWS/IAM access is enough to read them. GitHub stays the **source of truth**; CodeCommit is a downstream, read-oriented copy.

```
   ┌─────────────────────┐      mirror / sync     ┌──────────────────────┐
   │  Particle41 GitHub  │  ───────────────────▶  │   AWS CodeCommit     │
   │  (AI Labs repos)    │                        │   (mirror copy)      │
   │  SOURCE OF TRUTH    │                        │   read via IAM        │
   └─────────────────────┘                        └──────────────────────┘
          ▲                                                   │
          │ seat-holders push here                            │ seat-less engineers
          │                                                    ▼ clone/read with AWS creds
                                                        P41 AI Labs engineers
```

## ⚠️ The make-or-break gotcha: a mirror is one-way and destructive

`git push --mirror` makes the destination **exactly match** the source on every run. Anything committed *only* to CodeCommit gets **wiped** on the next sync.

```
GitHub [A B C D E]  ──push --mirror──▶  CodeCommit becomes [A B C D E]
                                         (any extra commit X made on
                                          CodeCommit is DELETED)
```

So the decisive question is **not** "will it wipe" (it always will) — it's **"does anyone need to contribute on the CodeCommit side?"**
- **No → only read** → a mirror is perfect.
- **Yes → contribute back** → a mirror is the *wrong tool*; needs a different design.

## "Stale" mirror = the copy falls behind

A copy is frozen at copy-time; GitHub keeps moving. The copy drifts behind ("stale"). How fresh it must stay is what drives the trigger choice.

## The architect's mental model: every sync = the same 4 blocks

Don't memorize setups — reason about which blocks change.

```
   [1] READ from GitHub ──▶ [3] COPY ──▶ [2] WRITE to CodeCommit
       (GitHub credential)   (git clone   (IAM permission)
                              --mirror /
                              fetch + push)
              ⟲ [4] TRIGGER — what kicks it off?
   (+ cross-cutting: WHERE it runs, and WHERE the credentials live)
```

| Block | One-time copy | Continuous mirror | Note |
|---|---|---|---|
| [1] Read GitHub | same mechanism, can use a human's creds | needs **long-lived, automated** creds (no human at 3am) | difference is *durability*, not method |
| [2] Write CodeCommit | **IAM** | **IAM** | **identical** — and the same system seat-less readers use |
| [3] Copy | clone once | **stateful** (keep local copy, `fetch`) or **stateless** (re-`clone` each run) | real axis = does the runner remember state between runs? |
| [4] Trigger | manual | scheduled or webhook | the biggest differentiator |

**Key insight:** IAM appears in *both* the runner's write permission and the readers' read permission. That's *why* CodeCommit solves the seat problem — one access system everyone already has.

## Trigger options

| Trigger | How | Freshness | Complexity | Integration cost |
|---|---|---|---|---|
| **Manual** | a human runs it "when requested" | stale until next run | lowest | none |
| **Scheduled (poll)** | a clock runs it every N min/hrs | lags up to N | low | none — stays inside AWS |
| **Webhook (event)** | GitHub pushes a signal on each commit | near-real-time | highest | GitHub must call *into* AWS (public endpoint, secret) |

> Rule of thumb: if "fresh within ~15 min" is fine, **scheduled poll** gives continuous-enough sync without the inbound-integration complexity of webhooks.

## Runner (the "pump") options

The runner is the heart — it holds *both* credentials and runs the copy.

| Runner | Stateful? | Triggers | Cost / footprint |
|---|---|---|---|
| Laptop / script | yes | manual only | free, but not real infra |
| **AWS Lambda** | no (ephemeral) | schedule / webhook | tiny pay-per-run; time- & repo-size-limited |
| **AWS CodeBuild** | no by default | manual / schedule / webhook | pay-per-build; handles big repos & long runs |
| EC2 | yes (persistent disk) | all three | always-on cost, you maintain it |

## Build order (what unblocks what)

1. **Create** the empty CodeCommit repo (destination).
2. **Write key** — IAM identity for the runner, allowed to push.
3. **Read key** — GitHub credential (PAT / deploy key) stored in **Secrets Manager**.
4. **Runner** — pick the compute that runs the copy.
5. **Copy** — `git clone --mirror` (or `fetch`) → `git push --mirror`.
6. **Trigger** — wire manual / schedule / webhook.
7. **Consumers** — give seat-less engineers **IAM read** + their Git auth.

> Reframe: the copy is trivial (4 git commands). The real work is the *edges* — identity, compute, secrets.

## Questions to validate with the team (before building)

1. Which GitHub repo(s)? One or many? More added over time?
2. WHO are the seat-less readers, and grant access via IAM user / role / group?
3. **Read-only or contribute-back?** (decides if a mirror is even the right tool)
4. Trigger: manual / scheduled / webhook-on-push?
5. Runner: Lambda / CodeBuild / EC2?
6. GitHub read credential — which machine token/deploy key, who owns it, store where?
7. IaC standard (Terraform / CloudFormation / CDK) and which repo holds it?
8. AWS account + region? Repo naming convention?
9. Ownership + monitoring/alerting on sync failure?

## Why this matters (career)

A small but *complete* integration problem: two source-control systems, two auth models (GitHub tokens vs AWS IAM), a data-movement job, a trigger, and a security boundary. The transferable skill isn't the git commands — it's the reasoning: find the shared skeleton, identify which blocks change, weigh trade-offs (freshness vs complexity), spot the destructive-mirror gotcha, and validate requirements before designing.
