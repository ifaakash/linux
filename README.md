# Concept Knowledge Base

A personal reference of concepts, system architectures, and command-line know-how. Each topic lives in its own folder with a `README.md` so it renders cleanly on GitHub and stays self-contained.

## How this is organised

- **One folder per topic.** Folder name is the subject in plain English (e.g. `AWS Centralised Logging`).
- **Each folder has a `README.md`** with the full write-up.
- **Supporting files** (diagrams, code snippets, sub-notes) live next to the README inside the same folder.
- **This root README is the table of contents.** Add a row whenever a new topic folder is created.

## Table of Contents

| # | Topic | What it covers |
| --- | --- | --- |
| 1 | [Linux and Git Basics](./Linux%20and%20Git%20Basics/) | SSH keys for GitHub, `grep -iE`, `find` recursion, `git show --stat`, finding commit authors. |
| 2 | [AWS Centralised Logging](./AWS%20Centralised%20Logging/) | CloudWatch → Firehose → S3 → SQS → Elastic/Kibana pipeline. ECS via Fluent Bit (FireLens) sidecar. Costs, IAM, when to use which architecture. |
| 3 | [AWS CodeCommit](./AWS%20CodeCommit/) | Mirroring GitHub repos into CodeCommit so seat-less engineers get IAM-based read access. Sync models, credentials, IAM, trade-offs. |

## Adding a new topic

1. Create a new folder at the repo root with a human-readable name (Title Case, spaces allowed).
2. Inside that folder, write a `README.md` explaining the concept in layman terms — focus on **how things work** and **system architecture**, not implementation details.
3. Add a row to the table above with a one-line summary.
4. Commit.
