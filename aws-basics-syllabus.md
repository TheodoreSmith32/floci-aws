# AWS Fundamentals via Floci — Syllabus

**Why floci instead of a real AWS account:** it's a local, free, open-source AWS emulator — `docker compose up` and you're talking to a fake `http://localhost:4566` that speaks the real AWS wire protocol. No billing risk, no account, same `aws` CLI commands you already know. Some services run as shallow in-process mocks; others (Lambda, RDS, MSK, Managed Flink, ElastiCache, EC2, ECS, OpenSearch...) spin up **real Docker containers** underneath, so the behavior is genuinely close to AWS, not just a stub.

**Assumes:** you already have floci running via docker-compose and the AWS CLI installed/configured.

---

## Week 1 — The AWS Service Landscape (Overview)

Before diving into any one service, get the map of the territory. AWS groups its ~250 services into rough categories — you don't need to know every service, just what each category is for and a couple of flagship names in it.

| Category | Purpose | Flagship services |
|---|---|---|
| **Compute** | Run code/workloads | EC2, Lambda, ECS, EKS, Lightsail |
| **Storage** | Store files/objects/blocks | S3, EBS, EFS |
| **Database** | Structured/semi-structured data | RDS, DynamoDB, Redshift, ElastiCache, Neptune |
| **Networking** | Connect things securely | VPC, Route53, API Gateway, CloudFront, ELB |
| **Analytics & Data** | Query/process large data | Athena, Glue, EMR, Firehose, Managed Flink |
| **Streaming/Messaging** | Move data between systems | Kinesis, MSK, SQS, SNS, EventBridge |
| **Identity & Security** | Access control, secrets | IAM, Cognito, KMS, Secrets Manager, GuardDuty |
| **Orchestration** | Coordinate multi-step workflows | Step Functions, SWF, CodePipeline |
| **Monitoring & Governance** | Observe and manage cost/compliance | CloudWatch, CloudTrail, Config, Budgets |
| **AI/ML** | Managed ML services | Bedrock, Rekognition, Comprehend, Textract |

**Hands-on:** pick 5 services you've never touched, run their `aws <service> help` in the CLI, and skim the top-level commands each supports — just enough to know what each one *does*, not how to use it yet. Cross-reference against floci's supported-services list (its GitHub README) to see which of these you can actually practice against for free.

**Goal for this week:** be able to answer "which AWS category would I reach for to do X?" — not build anything yet.

---

## Week 2 — Foundations: IAM, S3, Storage Modes

Now go deep on the two services almost everything else depends on.

- **IAM**: users, roles, groups, policies, access keys — floci accepts any non-empty credentials by default, but practicing the actual `aws iam` commands builds real muscle memory
- **S3 deeper features**: versioning, multipart upload, pre-signed URLs, object lock, event notifications, bucket policies vs IAM policies
- **Storage modes** (floci-specific): try switching `FLOCI_STORAGE_MODE` between `memory`, `persistent`, `hybrid`, and `wal` in your compose file, restart the container each time, and check whether your bucket data survives

**Hands-on:** enable versioning on your bucket, overwrite a file, recover an old version. Generate a pre-signed URL and download via `curl` without AWS credentials.

---

## Week 3 — Core App Services: DynamoDB, SQS/SNS, Lambda

- **DynamoDB**: create table, GSI/LSI, `Query` vs `Scan`, TTL, transactions, DynamoDB Streams
- **SQS + SNS**: build a basic fan-out pattern (SNS topic → multiple SQS queues)
- **Lambda**: floci runs this with **real Docker execution**, not a mock — write a small function, deploy it, trigger it from an S3 upload event or an SQS message, check logs in CloudWatch Logs (also emulated)

**Hands-on:** DynamoDB table with a stream → Lambda consumer that logs each change. This is a nice mini CDC-like pattern to compare against what you already build for real.

---

## Week 4 — Streaming: Kinesis, MSK, Firehose

Directly relevant to your background — this is the closest floci gets to your actual stack.

- **Kinesis**: streams, shards, enhanced fan-out, split/merge
- **MSK**: floci backs this with a real **Redpanda** broker (Kafka wire-protocol compatible) — same Kafka client code you already write can point at it
- **Firehose**: buffered streaming delivery into S3, with compression — a manual, AWS-native version of the sink pattern you build in PyFlink

**Hands-on:** create an MSK cluster in floci, produce/consume with your normal Kafka client pointed at it instead of your real broker, then wire a Firehose delivery stream to dump into S3.

---

## Week 5 — Data & Analytics: Glue, Athena, Managed Flink, Redshift

The most relevant week given your Trino/PyFlink work — and where you're already ahead: you've set up Trino with an Iceberg catalog backed by Glue, confirmed Glue works, and created a Glue-registered schema+table.

- **Glue**: Data Catalog + crawlers — auto-discovers schema from S3 data, feeds Athena
- **Athena**: floci runs this via an in-process **DuckDB sidecar**, doing real SQL execution over S3/Glue-backed tables — mentally the same role Trino plays for you, just AWS-managed instead of self-hosted
- **Managed Service for Apache Flink**: floci actually launches a **real Flink cluster** (JobManager + TaskManager via Docker) when you call `StartApplication`, pulling your job JAR from local S3. You could package a scaled-down version of your topic A/B/C join logic and deploy it the way you would to real AWS, entirely offline
- **Redshift**: listed as supported — good to at least run basic queries and compare the mental model to Athena (data warehouse vs serverless query-on-S3)

**Hands-on:** query your existing Glue-backed Iceberg table via Athena (not just Trino) to compare. Then try packaging a trivial Flink job JAR and deploying it as a "Managed Flink" application in floci.

---

## Optional Week 6 — Databases & Caching: RDS, ElastiCache

Ties directly into your CDC and PyFlink-lookup-store work.

- **RDS**: real Docker-backed Postgres/MySQL/MariaDB — you could point a CDC source config at this locally to sanity-check config before touching a real DB
- **ElastiCache**: real Redis/Valkey via Docker — worth testing against, since you were already weighing Redis as the lookup store for your PyFlink A/B reference-table join

**Hands-on:** spin up RDS Postgres in floci, enable logical replication, and see if your existing CDC tooling (Qlik Replicate or otherwise) can even talk to it as a source — good low-risk way to validate config changes.

---

## Optional Week 7 — Orchestration & Events: EventBridge, Step Functions

- **EventBridge**: custom buses, rules, routing to SQS/SNS/Lambda targets
- **Step Functions**: ASL-based state machines, task tokens, execution history

**Hands-on:** build a small Step Functions workflow that reads from S3, calls Lambda, and writes to DynamoDB — good intro to AWS-native orchestration if you ever need to explain/compare it against your own pipeline orchestration.

---

## Notes
- Full service list & fidelity details: floci's GitHub README (`floci-io/floci`) — some services are real Docker, others are in-process mocks; the README calls out which is which per service
- Terraform also works against floci (`floci.io/floci/docs/getting-started/terraform.md`) if you want to practice IaC on top of this later
- No billing alarms needed here — floci is local, so nothing costs money regardless of how much you spin up
