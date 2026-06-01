# AWS CodeCommit (GitHub → CodeCommit Mirror)

> **Status: in progress.** This write-up grows as we work through the design. Architecture decisions are being reasoned out question-by-question, not just copied in.

## What it is

AWS CodeCommit is a **fully managed private Git host** that lives inside your AWS account — AWS's own GitHub/GitLab. It speaks plain Git (clone, push, pull, branches, PRs), but access is governed by **IAM** instead of per-seat logins.

- Repos are private, scoped to one AWS account + region.
- Auth options: HTTPS Git credentials, SSH keys, or the `git-remote-codecommit` helper using your AWS credentials.
- Integrates natively with CodePipeline / CodeBuild / CodeDeploy.

> ⚠️ **Availability:** AWS closed CodeCommit to *new* customers around July 2024. It only works in accounts that already had it. **Confirmed available** in the target P41 account.

## The problem we're solving

Some P41 AI Labs engineers **don't have a GitHub seat**, so they can't access the AI Labs repos. They *do* have AWS access. So: mirror the repos into CodeCommit, where AWS/IAM access is enough to read them.

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

Key constraint from the ticket: mirroring is **on-demand per repo** ("when requested"), and GitHub remains the **source of truth** — CodeCommit is downstream and read-oriented.

## Design decisions (being worked through)

| Decision | Options | Chosen | Why |
| --- | --- | --- | --- |
| Sync model | one-time copy vs continuous mirror | _TBD_ | _TBD_ |
| Where sync runs | laptop / Lambda / CI runner / EC2 | _TBD_ | _TBD_ |
| Credentials | GitHub read + CodeCommit write | _TBD_ | _TBD_ |
| Reader access | IAM users / roles / groups | _TBD_ | _TBD_ |

## Why this matters (career)

This is a small but complete **integration** problem: two source-control systems, two auth models (GitHub tokens vs AWS IAM), a data-movement job, scheduling/triggering, and a security boundary. Reasoning through it the way an architect would — trade-offs, failure modes, least privilege — is the transferable skill.

## Open questions / next steps

- Decide sync model, runner location, credential handling, and reader IAM.
- (Later) Define a "quick debugging mode" workflow preference.
