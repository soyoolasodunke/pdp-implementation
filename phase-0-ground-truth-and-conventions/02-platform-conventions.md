# 02: Platform Conventions

These conventions are the **naming and configuration standard for every example in the guide**. They are opinionated defaults for a generic reference platform and can be adapted per client. Where a convention depends on an unverified product behaviour, it references the validation register (file 03).

---

## 1. Environments and Resource Naming

| Item | Convention | Example |
|------|-----------|---------|
| Environment codes | `dev`, `stg`, `prd` | `prd` |
| Platform prefix | `bdp` | — |
| Generic resource name | `bdp-<env>-<component>[-<purpose>]` | `bdp-prd-trino`, `bdp-dev-kafka-connect` |
| Base DNS zone (placeholder) | `<env>.bdp.example.internal` | `prd.bdp.example.internal` |
| Service hostname | `<component>.<env>.bdp.example.internal` | `trino.prd.bdp.example.internal` |
| Business domain token | lowercase, single word | `sales`, `customer`, `risk` |
| Case rules | Kubernetes/DNS: `kebab-case`; SQL objects, Python, Kafka keys: `snake_case`; env vars: `UPPER_SNAKE` | — |

---

## 2. Kubernetes Namespace Layout

One namespace per functional area keeps RBAC, network policy and quota boundaries aligned with platform responsibilities.

| Namespace | Components | Notes |
|-----------|-----------|-------|
| `bdp-security` | Keycloak, OpenBao | Highest-privilege namespace; restricted admin access |
| `bdp-observability` | Prometheus, Alertmanager, Grafana | Cluster-wide read for scraping |
| `bdp-ingest` | NiFi, Kafka (brokers/controllers), Kafka Connect + Debezium | — |
| `bdp-storage` | MinIO (self-built image), Cassandra, PostgreSQL clusters (`pg-platform`, `pg-app`) + PgBouncer | Stateful; StorageClass per §15 |
| `bdp-lakehouse` | Lakekeeper, OpenFGA `[Prerequisite]`, (conditional) OPA bridge, Trino, ClickHouse | — |
| `bdp-processing` | Spark driver/executor pods (spark-submit on K8s) | Ephemeral pods; quota-controlled |
| `bdp-governance` | OpenMetadata server, OpenMetadata ingestion Airflow (ADR-019), OpenSearch `[Prerequisite]`, GX Core runtime jobs | — |
| `bdp-orchestration` | Airflow (KubernetesExecutor task pods run here) | — |
| `bdp-bi` | Superset (web, Celery workers, beat), Valkey `[Prerequisite]` | — |
| `bdp-ml-dev` | JupyterHub hub/proxy; user servers in `bdp-ml-users` | — |
| `bdp-ml-platform` | MLflow, ZenML server, Feast (registry/feature servers), TrustyAI operator | — |
| `bdp-ml-serving-<domain>` | KServe `InferenceService`s, FastAPI apps, TrustyAI service instance | TrustyAI service is per namespace (VR-013) |
| `kserve` | KServe controller; `kserve-ingress-gateway` Gateway | Gateway uses the Traefik GatewayClass (ADR-008/022) |
| `cert-manager` | cert-manager `[Prerequisite]` | — |
| `longhorn-system` | Longhorn `[Prerequisite]` | Privileged; PSA exemption required (VR-036) |
| `kube-system` | RKE2-packaged Traefik (customised via `HelmChartConfig`), CoreDNS, metrics-server | Managed by RKE2 |
| `bdp-edge` | Shared platform `Gateway` (`bdp-gateway`), `HTTPRoute`s, forward-auth middleware | TLS certificates from cert-manager |

---

## 3. Labels and Annotations

All workloads carry the Kubernetes recommended labels plus platform labels:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: trino            # component
    app.kubernetes.io/instance: bdp-prd-trino
    app.kubernetes.io/version: "484"
    app.kubernetes.io/component: coordinator  # role within component
    app.kubernetes.io/part-of: bdp
    app.kubernetes.io/managed-by: helm
    bdp.io/env: prd
    bdp.io/layer: lakehouse                   # ingest|storage|lakehouse|processing|governance|orchestration|bi|ml|security|observability
    bdp.io/domain: shared                     # or a business domain
    bdp.io/data-classification: internal      # public|internal|confidential|restricted
  annotations:
    prometheus.io/scrape: "true"              # see §11 (native k8s SD scraping)
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

---

## 4. Service Endpoints and Default Ports

Ports are **upstream defaults** used in examples. Items marked † must be confirmed in Phase 2 against the reference version.

| Component | Endpoint / protocol | Default port |
|-----------|---------------------|--------------|
| NiFi 2.x | Web UI / REST API (HTTPS) | 8443 |
| Kafka (KRaft) | Broker listener / controller listener | 9092 / 9093 |
| Kafka Connect (Debezium runtime) | REST API | 8083 |
| Debezium Server (alternative runtime) | HTTP (health/metrics) | 8080 † |
| MinIO | S3 API / Console (reduced in CE; administer via `mc` and Admin API) | 9000 / 9001 |
| Lakekeeper | Iceberg REST + Management API (`/catalog`, `/management`) | 8181 |
| Cassandra | CQL / inter-node / inter-node TLS / JMX | 9042 / 7000 / 7001 / 7199 |
| PostgreSQL | SQL | 5432 |
| Spark | Driver UI / History Server / Spark Connect | 4040 / 18080 / 15002 |
| Trino | HTTP / HTTPS | 8080 / 8443 |
| ClickHouse | HTTP / native TCP / HTTPS / native TLS / Prometheus | 8123 / 9000 / 8443 / 9440 / 9363 |
| OpenMetadata | Server / admin | 8585 / 8586 |
| Airflow 3 | API server (UI + REST) | 8080 |
| Superset | Web / REST | 8088 |
| Keycloak | HTTP / HTTPS / management (health, metrics) | 8080 / 8443 / 9000 |
| OpenBao | API / cluster | 8200 / 8201 |
| Prometheus | HTTP API / UI | 9090 |
| Grafana | HTTP | 3000 |
| JupyterHub | Proxy public / Hub API | 8000 / 8081 |
| MLflow | Tracking server | 5000 |
| ZenML server | HTTP | 8080 † |
| KServe predictor | HTTP / gRPC (inside pod) | 8080 / 8081 † |
| Feast | Feature server / registry server / offline server | 6566 / 6570 / 8815 † |
| TrustyAI service | HTTP | 8080 † |
| FastAPI app (uvicorn) | HTTP | 8000 |

| OpenSearch | REST | 9200 |
| OpenFGA | HTTP / gRPC | 8080 / 8081 |
| Valkey | RESP | 6379 |
| PgBouncer | PostgreSQL protocol | 6432 † |
| Lakekeeper | Prometheus metrics | 9000 |
| Alertmanager | HTTP | 9093 |
| Traefik | Web / websecure entrypoints | 80 / 443 (host ports per RKE2 defaults) |

**Exposure rule:** Clients reach services by in-cluster DNS (`<svc>.<namespace>.svc.cluster.local`). Only user-facing UIs and approved APIs are exposed, through **Gateway API `HTTPRoute`s** attached to `bdp-edge/bdp-gateway` (Traefik), with TLS at the Gateway and OIDC at the application. Components without native OIDC use the Keycloak forward-auth middleware. Kubernetes `Ingress` objects are used only where a chart offers nothing else (ADR-022).

---

## 5. Identity and Access (Keycloak)

### 5.1 Realms and clients

| Item | Convention | Example |
|------|-----------|---------|
| Realm | One per environment: `bdp-<env>` | `bdp-prd` |
| OIDC client (interactive app) | `bdp-<component>` (confidential, auth-code + PKCE) | `bdp-superset`, `bdp-grafana` |
| OIDC client (service-to-service) | `svc-<component>-<purpose>` (client-credentials) | `svc-airflow-trino`, `svc-zenml-mlflow` |
| Client scope for group claims | `bdp-groups` → token claim `groups` (full path off) | `groups: ["data-engineer"]` |
| Audience mapping | Per resource server: `aud` includes target client ID | `aud: ["bdp-lakekeeper"]` |

### 5.2 Platform roles (realm groups)

| Group | Intent | Typical grants |
|-------|--------|---------------|
| `platform-admin` | Operate the platform | Admin on all components (break-glass audited) |
| `data-engineer` | Build pipelines | NiFi, Airflow, Spark, Kafka topics (domain-scoped), Lakekeeper write on owned namespaces |
| `data-analyst` | Query and report | Trino read, ClickHouse read, Superset |
| `data-scientist` | Explore and experiment | JupyterHub, MLflow experiments, Trino read, Feast read |
| `ml-engineer` | Productionise models | ZenML, MLflow registry, KServe namespaces, model-gate runs |
| `data-steward` | Govern data | OpenMetadata edit, GX suites |
| `viewer` | Read-only dashboards | Grafana viewer, Superset viewer |

Domain scoping uses **subgroups**: `data-engineer/sales`, `data-analyst/risk`. Components that cannot consume group claims directly need mapping, proxy or custom code (see Keycloak coverage table in file 03).

---

## 6. Secrets (OpenBao)

### 6.1 Mounts and paths

| Purpose | Engine / mount | Path convention | Example |
|---------|---------------|-----------------|---------|
| Static secrets | KV v2 at `kv/` | `kv/bdp/<env>/<component>/<purpose>` | `kv/bdp/prd/lakekeeper/pg-encryption-key` |
| Dynamic DB credentials | Database engine at `database/` | role `<env>-<consumer>-<db>-<ro\|rw>` | `prd-superset-clickhouse-ro` † (plugin coverage per DB: VR-016) |
| Internal PKI | PKI at `pki/` and `pki_int/` | role `<env>-<component>` | `prd-kafka` |
| Transit encryption | Transit at `transit/` | key `<env>-<purpose>` | `prd-feature-pii` |

### 6.2 Authentication methods

| Consumer | Auth method | Convention |
|----------|-------------|------------|
| Kubernetes workloads | `kubernetes` auth | Role `<env>-<namespace>-<serviceaccount>`; bound to SA + namespace |
| Humans (CLI/UI) | `oidc` auth → Keycloak realm `bdp-<env>` | Role `<group>`; policy by Keycloak group |
| CI pipelines | `jwt` or `approle` | Role `<env>-ci-<repo>` |

### 6.3 Policy naming and delivery

- Policy names: `<env>-<component>-<access>` (e.g. `prd-airflow-read`).
- **Delivery to pods:** OpenBao Agent injection (sidecar/init) or the application reading OpenBao directly via SDK/API. Kubernetes Secrets are synchronised only when a component cannot read files or the API (VR-017 confirms injector availability in the OpenBao Helm chart).
- **Never** put secret values in Helm values, DAG code, notebooks or Git.

---

## 7. Object Storage (MinIO)

| Bucket | Purpose | Writers | Readers |
|--------|---------|---------|---------|
| `bdp-<env>-landing` | Raw files dropped by NiFi before table ingestion | NiFi | Spark |
| `bdp-<env>-lakehouse` | Iceberg warehouse (data + metadata) managed via Lakekeeper | Spark, Trino (via Lakekeeper) | Spark, Trino, ClickHouse, Feast |
| `bdp-<env>-mlflow-artifacts` | MLflow artefacts and model files | MLflow clients | MLflow, KServe (storage initializer) |
| `bdp-<env>-zenml-artifacts` | ZenML artifact store | ZenML steps | ZenML |
| `bdp-<env>-feast` | Feast offline staging / registry files (if file-based) | Feast | Feast |
| `bdp-<env>-models` | Promoted, gate-approved model packages for serving | Deployment pipeline | KServe |
| `bdp-<env>-governance` | GX validation results / Data Docs, model-gate reports | GX, ZenML | OpenMetadata, auditors |
| `bdp-<env>-ops` | Spark event logs, NiFi provenance exports, backups | Platform jobs | Ops |

Object-key prefix pattern inside buckets: `<domain>/<dataset_or_project>/<yyyy>/<mm>/<dd>/…` for time-partitioned raw files.

---

## 8. Lakehouse (Lakekeeper + Iceberg)

| Item | Convention | Example |
|------|-----------|---------|
| Lakekeeper project | `bdp-<env>` | `bdp-prd` |
| Lakekeeper warehouse | `lakehouse` (one per env, backed by `bdp-<env>-lakehouse`) | `lakehouse` |
| Engine catalog name | `lakehouse` in Spark, Trino and ClickHouse | `lakehouse.silver_sales.orders` |
| Namespace (single level) | `<layer>_<domain>`; layers `bronze`, `silver`, `gold` | `bronze_crm`, `gold_sales` |
| Table | `snake_case`, singular entity or explicit fact/dim prefix | `customer`, `fact_order_line`, `dim_product` |
| Iceberg format version | v2 by default; v3 features only after engine validation (VR-007) | — |
| Partitioning | Hidden partitioning with transforms (`days(event_ts)`, `bucket(16, id)`) | — |

Single-level namespaces avoid inconsistent nested-namespace support across engines (VR-007).

---

## 9. Streaming (Kafka, Debezium)

| Item | Convention | Example |
|------|-----------|---------|
| CDC topic prefix (`topic.prefix`) | `cdc.<source_system>` | `cdc.crm` |
| CDC topic (Debezium default shape) | `<prefix>.<schema>.<table>` | `cdc.crm.public.customer` |
| Business event topic | `evt.<domain>.<event_name>.v<n>` | `evt.payments.card_authorised.v1` |
| Dead-letter topic | `<source_topic>.dlq` | `cdc.crm.public.customer.dlq` |
| Connect internal topics | `_connect.<cluster>.configs/offsets/status` | `_connect.cdc.offsets` |
| Consumer group | `cg.<application>.<purpose>` | `cg.spark-bronze-crm.ingest` |
| Defaults | RF = 3, `min.insync.replicas = 2`, `cleanup.policy = delete` (CDC compaction per use case) | — |
| Serialisation | JSON converter with explicit schema handling; **no schema registry in baseline** (ADR-009) | — |

---

## 10. Databases

### 10.1 PostgreSQL instances

| Cluster | Purpose | PG line | Databases |
|---------|---------|---------|-----------|
| `pg-platform` | Component metadata stores (behind PgBouncer) | 18 (ADR-013; per-consumer check VR-032) | `keycloak`, `lakekeeper`, `airflow`, `superset`, `mlflow`, `openmetadata`, `feast_registry`, `jupyterhub`, `grafana`, `openfga`, `airflow_omd` (OpenMetadata ingestion Airflow), `zenml` (VR-018) |
| `pg-app` | Application OLTP and vector workloads | 18 + pgvector 0.8.x | `<domain>_app`, `<domain>_vector` |

Per-database owner role `<db>_owner`; application roles `<db>_rw` / `<db>_ro`, preferably issued dynamically by OpenBao.

### 10.2 Cassandra

- Keyspace: `<domain>_<purpose>` (e.g. `risk_online_features`, `customer_profile`).
- Replication: `NetworkTopologyStrategy` with RF = 3 per DC.
- Table design follows **query-first modelling**; partition key documented per table.

### 10.3 ClickHouse

- Databases: `mart_<domain>` for serving marts; `stg_<domain>` for staging; `lake_<catalog>` for experimental `DataLakeCatalog` mounts.
- Engines: `ReplicatedMergeTree` family for production marts.

---

## 11. Observability

| Item | Convention |
|------|-----------|
| Scrape model | Prometheus **native Kubernetes service discovery** using the `prometheus.io/*` annotations (§3). No operator CRDs unless approved (ADR-014). |
| OTLP | Components emitting OpenTelemetry metrics send them to Prometheus's OTLP receiver (VR-019). |
| Mandatory metric labels | `env`, `namespace`, `component`, `instance`, `domain` (where applicable) |
| Grafana folders | `Platform / <layer>`; dashboards named `<component> – <view>` |
| Alert naming | `BDP<Component><Condition>` (e.g. `BDPKafkaUnderReplicatedPartitions`) |
| Log format | JSON to stdout; fields `ts`, `level`, `component`, `env`, `trace_id`, `msg` |

---

## 12. Orchestration and Pipelines

| Item | Convention | Example |
|------|-----------|---------|
| Airflow DAG ID | `<domain>__<pipeline>__<cadence>` | `sales__orders_bronze_to_silver__hourly` |
| Airflow executor | KubernetesExecutor (avoids a Celery broker outside the baseline) | — |
| Airflow connection ID | `<component>_<purpose>` | `trino_default`, `lakekeeper_rest` |
| Airflow secrets | OpenBao via Airflow's secrets backend (compatibility of the HashiCorp provider with OpenBao: VR-015) | — |
| ZenML stack | `bdp-<env>-<purpose>` | `bdp-prd-training` |
| ZenML pipeline | `<domain>_<use_case>_<stage>` | `risk_fraud_training` |

---

## 13. ML Lifecycle Naming

| Item | Convention | Example |
|------|-----------|---------|
| MLflow experiment | `/<domain>/<use_case>` | `/risk/fraud_detection` |
| MLflow registered model | `<domain>.<model_name>` | `risk.fraud_xgb` |
| Model aliases | `challenger` (candidate), `champion` (production); `gate-passed` tag set only by the governance step | — |
| model-gate report | `bdp-<env>-governance/model-gate/<model>/<version>/gate_report.json` (+ `.html`) | — |
| Feast project | `bdp_<domain>` | `bdp_risk` |
| Feast feature view | `<entity>_<feature_group>` | `card_txn_aggregates_1h` |
| KServe InferenceService | `<model-name>-<variant>` in `bdp-ml-serving-<domain>` | `fraud-xgb-v3` |
| FastAPI service | `<domain>-<capability>-api` | `risk-scoring-api` |

---

## 14. Configuration and Code Repositories

| Repository | Content |
|------------|---------|
| `bdp-platform-infra` | Helm values, Kubernetes manifests, per-env overlays |
| `bdp-data-pipelines` | Airflow DAGs, Spark jobs, NiFi flow definitions (Git registry client), GX suites |
| `bdp-ml-pipelines` | ZenML pipelines, Feast feature repos, model-gate configs |
| `bdp-serving` | KServe manifests, FastAPI services |

Environment variable naming in platform code: `BDP_<COMPONENT>_<SETTING>` (e.g. `BDP_TRINO_HOST`). Native product variables (e.g. `MLFLOW_TRACKING_URI`, `LAKEKEEPER__*`) keep their upstream names.

---

## 15. Storage Classes (RKE2 + Longhorn, ADR-023)

| StorageClass | Backing | Replication | Used by |
|--------------|---------|-------------|---------|
| `longhorn` (default) | Longhorn distributed block | 3 replicas | PostgreSQL (if not using app-level replicas), NiFi repositories, Airflow logs, JupyterHub user volumes, TrustyAI PVC mode, Grafana, Prometheus (single instance) |
| `longhorn-local` | Longhorn, strict-local data locality | 1 replica | Kafka, Cassandra, ClickHouse, OpenSearch, MinIO drives: stores that replicate at the application level |

Rule: never stack application-level replication on top of 3-way Longhorn replication for the same data.

---

## 16. Container Images and Registry

| Item | Convention | Example |
|------|-----------|---------|
| Registry | Internal Harbor `registry.bdp.example.internal` `[Prerequisite]` | — |
| Projects | `bdp-mirror` (pinned upstream images), `bdp-build` (self-built: MinIO, Spark/Iceberg images, ZenML pipeline images, FastAPI services) | `registry.bdp.example.internal/bdp-build/minio:<last-ce-release>-bdp.1` |
| Pinning | Deployments reference images by digest; tags are informational | `@sha256:…` |
| Supply chain | Image scanning on push; SBOM and signature mandatory for `bdp-build` | — |

