# 01 — What is Kestra and Who Uses It

> Source branch: `develop` (`9e6a2a4020`). Code refs point to this repo.

## 1. One-line definition

Kestra is an **open-source, event-driven orchestration platform** for data, AI, and infrastructure workflows.

From `README.md:47`:

> It unifies scheduled and event-driven automation behind a declarative, language-agnostic interface.

## 2. Explain it like I'm new

You write a **Flow** — a small YAML file — that says *what to do*:

```yaml
id: hello_world
namespace: dev

tasks:
  - id: say_hello
    type: io.kestra.plugin.core.log.Log
    message: "Hello, World!"
```

Kestra takes care of *when and how* it runs:

- on a **schedule** (`cron: "0 9 * * *"`),
- on an **event** (file arrived in S3, message in Kafka, API webhook),
- on-demand (click Run in UI, `POST /api/v1/.../executions`, Terraform, CLI).

While it runs, Kestra gives you retries, timeouts, parallel vs sequential steps, inputs/outputs, logs, file artifacts, and a live DAG graph in the UI. The YAML is always the truth — even if you edit from the UI, the YAML updates (`README.md:74`).

## 3. The 7 words you must know

| Word | Meaning | Where in code |
|---|---|---|
| `Flow` | The YAML recipe | `core/src/main/java/io/kestra/core/models/flows/Flow.java` |
| `Execution` | One run of a Flow | `core/src/main/java/io/kestra/core/models/executions/Execution.java` |
| `Task` | One step: run Python, SQL, Docker, HTTP, etc. | `core/src/main/java/io/kestra/core/models/tasks/Task.java` |
| `Trigger` | What starts a Flow: Schedule, Poll, Webhook | `core/src/main/java/io/kestra/core/models/triggers/` |
| `Namespace / Tenant` | Folder + isolation boundary (`/api/v1/{tenant}/...`) | `webserver/.../controllers/api/NamespaceController.java`, `TenantController.java` |
| `Inputs / Outputs / Variables` | Parameters in, files/data out | `core/src/main/java/io/kestra/core/runners/FlowInputOutput.java` |
| `Pebble `{{ }}` | Template language for dynamic values | `core/src/main/java/io/kestra/core/utils/PebbleUtil.java` |

Example with all of them:

```yaml
id: etl_sales
namespace: finance
inputs:
  - id: date
    type: DATE

tasks:
  - id: extract
    type: io.kestra.plugin.jdbc.postgres.Query
    sql: "SELECT * FROM sales WHERE day = '{{ inputs.date }}'"

triggers:
  - id: daily
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 6 * * *"
```

## 4. Why teams pick it over Airflow / cron / Jenkins

1. **Language-agnostic:** Python, Node, Go, Shell, SQL, Docker, K8s — no rewrite into one language.
2. **UI + code both work:** non-coders use UI/Blueprints, engineers use Git + Terraform provider.
3. **Event-driven, not just cron:** Kafka, SQS, MQTT, webhooks, file triggers are first-class (`README.md:170-172`).
4. **Plugin ecosystem:** hundreds of plugins (`https://kestra.io/plugins`), plus custom plugins in Java.
5. **Scalable:** Executor + Worker + Queue split lets it handle millions of executions.

## 5. Who uses it — company-wise

All below are from `https://kestra.io/customers/*` (public case studies):

### Big Tech / AI

- **Apple (ML team)** — ETL across App Store, Apple Music, device logs into a central ML platform. 200 engineers, replaced Prefect/Dagster/Airflow candidates. Uses declarative YAML + retries + observability at massive scale.

### Banking / Finance / Security (strictest compliance)

- **JPMorgan Chase (cybersecurity division)** — 100+ users, billions of transactions, thousands of API pulls/week. Hourly API ingestion → dbt enrichment → Trino analysis → S3 → dashboards. No-Python requirement for analysts.
- **Crédit Agricole / CAGIP** — IT production arm of France's largest retail bank. 50+ MongoDB clusters, 100+ clusters total. Replaced Jenkins + Ansible with reusable subflows, HashiCorp Vault for secrets, self-hosted private cloud.

### Data platforms / Multi-tenant SaaS

- **Acxiom (Axiom Cloud)** — Embedded Kestra as the orchestration layer for 50+ enterprise clients. Hub-and-spoke on AWS: central control plane + per-client EKS worker pool, S3 bucket, secrets, VPC. Terraform-provisioned onboarding.
- **Sopht (Green ITOps)** — 1,240 → 6,200 jobs/day in 6 months, 99.5% success. Tenant-per-namespace + Terraform `kestra_flow` module.
- **Gorgias (e-commerce AI)** — Airbyte → BigQuery → dbt → Hightouch, plus Kestra as CI/CD engine on Kubernetes via Terraform provider.
- **Riverside (creator platform)** — Snowflake + dbt Cloud + Metaplane + Hightouch on Kestra Cloud, 2 engineers maintain everything.

### Retail / Infra / Public sector

- **Leroy Merlin France** — Self-service data mesh, 250 engineers, 140 stores, +900% data production after replacing Airflow.
- **Amdocs** — Integration environments-as-a-service (VM provision → deploy → test), shipped to prod in 2 months.
- **Gravitee, Clever Cloud, Displayce, Dataport** — API docs generation, 20TB/week monitoring offload, 1M-screen ad platform, German government private cloud.

### Takeaway by role

- **Data engineers:** replace fragile cron + Airflow DAGs.
- **Platform/infra teams:** replace Jenkins/Ansible chains with observable, Vault-backed flows.
- **Security/fraud teams:** compliant ingestion + enrichment without forcing everyone to write Python.
- **SaaS founders:** multi-tenancy (namespace + tenant + isolated workers/storage) to sell one platform to many customers.
