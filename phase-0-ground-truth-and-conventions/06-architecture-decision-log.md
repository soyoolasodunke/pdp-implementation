# 06: Architecture Decision Log

Each decision fixes how the guide is written and how the reference platform is built. All decisions below were **Accepted** at the Phase 0 review on 2026-11-17. A later change requires a new ADR that supersedes the old one; accepted ADRs are not edited in place.

| ID | Decision | Status |
|----|----------|--------|
| ADR-001 | Deployment substrate: **RKE2** (Kubernetes 1.36 line) | Accepted |
| ADR-002 | Reference versions per file 01 (latest stable as of 2026-11-17) | Accepted |
| ADR-003 | Platform Python 3.12 | Accepted |
| ADR-004 | Per-component JVMs from official images; no shared JDK base | Accepted |
| ADR-005 | Lakekeeper is the only Iceberg catalog; all engines use the REST Catalog API | Accepted |
| ADR-006 | Airflow uses KubernetesExecutor | Accepted |
| ADR-007 | Spark runs on Kubernetes via native `spark-submit` (no operator) | Accepted |
| ADR-008 | KServe Standard (raw) deployment mode with Gateway API | Accepted |
| ADR-009 | Kafka payloads as JSON; no schema registry | Accepted |
| ADR-010 | Keycloak is the single identity provider (OIDC/OAuth2) | Accepted |
| ADR-011 | OpenBao Kubernetes auth + agent-based secret delivery | Accepted |
| ADR-012 | MinIO: frozen last Community Edition build; in-house self-build thereafter | Accepted |
| ADR-013 | PostgreSQL topology: `pg-platform` and `pg-app`, both on PostgreSQL 18 | Accepted |
| ADR-014 | Prometheus scraping via native Kubernetes SD + OTLP receiver; Alertmanager for notifications | Accepted |
| ADR-015 | Runtime dependencies outside the baseline are included as named platform prerequisites | Accepted |
| ADR-016 | Airflow vs ZenML orchestration boundary | Accepted |
| ADR-017 | Model promotion gated by model-gate and a platform tag | Accepted |
| ADR-018 | Production analytical serving = materialise into ClickHouse MergeTree | Accepted |
| ADR-019 | OpenMetadata ingestion runs on a dedicated, version-coupled Airflow instance | Accepted |
| ADR-020 | ZenML → KServe deployment via a platform deployment step (no deprecated Model Deployer) | Accepted |
| ADR-021 | TrustyAI on RKE2 via the upstream operator, PVC storage mode | Accepted |
| ADR-022 | North-south traffic: RKE2-packaged Traefik with Gateway API enabled | Accepted |
| ADR-023 | Storage: Longhorn as the default CSI StorageClass; local volumes for self-replicating stores | Accepted |
| ADR-024 | model-gate is documented under its MIT licence | Accepted |
| ADR-025 | GX Core only; no GX Cloud features | Accepted |

---

## ADR-001: Deployment Substrate (RKE2)

- **Decision:** The reference platform runs on **RKE2 v1.36.x** (Kubernetes 1.36), with RKE2's packaged components where they meet a platform need: Canal CNI (default), CoreDNS, metrics-server, Traefik (ADR-022), and the Helm controller (`HelmChart` / `HelmChartConfig` CRDs).
- **Rationale:** Customer-selected distribution. RKE2 is CIS-hardened by default and FIPS-capable (the Canal CNI is FIPS-compliant).
- **Consequences:**
  - RKE2-packaged add-ons are customised through `HelmChartConfig` manifests in `/var/lib/rancher/rke2/server/manifests/`, not by editing the bundled charts.
  - RKE2 ships **no** storage provisioner → ADR-023.
  - The RKE2 default CIS profile enforces Pod Security Admission. Every component spec must state its security context (non-root, read-only root FS where supported) and any namespace PSA exemption it needs.
  - OpenShift-specific procedures (e.g. TrustyAI via OpenShift AI) are **not** used.

## ADR-002: Reference Versions

- **Decision:** Pin documentation to the release lines in file 01. Airflow moves to **3.4.x** (the current stable documentation is 3.4.0).
- **Consequences:** Re-verify exact patches when building deployment manifests.

## ADR-005: Single Iceberg Catalog

- **Decision:** All Iceberg reads and writes go through Lakekeeper's REST Catalog API: Spark (`type=rest`), Trino (`iceberg.catalog.type=rest`), ClickHouse (batch materialisation reads only, ADR-018), Feast (via Spark/Trino offline stores).
- **Consequences:** No direct metadata-file access in production paths and no secondary catalogs. Engine REST feature differences are tracked in VR-007.

## ADR-006: Airflow Executor

- **Decision:** KubernetesExecutor.
- **Rationale:** Per-task pod isolation, native fit with OpenBao Kubernetes auth, no Celery broker for Airflow.
- **Trade-off:** Task start-up latency. Very short, high-frequency tasks should be batched.

## ADR-007: Spark Submission

- **Decision:** Native `spark-submit --master k8s://https://<api-server>` (cluster mode), triggered from Airflow. The exact Airflow operator is confirmed in Phase 3 (VR-006).
- **Rationale:** No dependency on a Spark operator.

## ADR-008: KServe Mode

- **Decision:** KServe **Standard (raw deployment) mode** with **Gateway API** (`enableGatewayApi: true`). The `kserveIngressGateway` points at a Gateway served by the RKE2 Traefik GatewayClass (ADR-022).
- **Rationale:** KServe documents Gateway API as the recommended network option and Kubernetes Ingress support as deprecated. Standard mode avoids Knative and a service mesh.
- **Consequences:** No scale-to-zero. Autoscaling uses HPA. Canary rollouts are handled at Gateway/route level or by KServe's raw-mode mechanisms, documented per workflow. Gateway API version compatibility between KServe and Traefik is checked in VR-002.

## ADR-009: Kafka Serialisation

- **Decision:** JSON (Debezium JSON converter); schema handling documented explicitly.
- **Consequences:** Schema evolution is enforced downstream (Spark + Iceberg schema evolution, GX contracts). VR-005 validates end-to-end behaviour.

## ADR-010: Identity Provider

- **Decision:** One Keycloak realm per environment. OIDC for humans, client-credentials for services. Group claims drive authorisation.
- **Consequences:** Components without native OIDC are fronted by the Gateway with authentication middleware, or by custom code. Each case is tagged `[Custom]` or `[Prerequisite]` in Part II.

## ADR-011: Secrets Delivery

- **Decision:** Workloads authenticate to OpenBao with Kubernetes service-account tokens. Secrets are delivered by OpenBao Agent (injector/CSI) or read via the API. Humans use OIDC auth to OpenBao.
- **Consequences:** Depends on VR-017 (injector/CSI in the OpenBao chart). cert-manager issues internal TLS certificates, ideally backed by OpenBao PKI (VR-034).

## ADR-012: MinIO Distribution (Frozen Build + Self-Build)

- **Decision:** Use the **last published MinIO Community Edition build** as the reference baseline. The Bluechip platform team takes ownership of **building future images from the archived source**.
- **Consequences:**
  1. **Security ownership moves in-house.** The team must monitor CVEs affecting MinIO and its Go dependencies, patch and rebuild, and run image scanning in CI.
  2. The admin web console in CE is stripped. Administration is via the `mc` client and the S3/Admin APIs, which the guide documents.
  3. Identity integration with Keycloak depends on what remains in the CE server (VR-010). The guide documents the S3 API surface and marks anything edition-specific.
  4. Images are published to the internal OCI registry (ADR-015) with the tag pattern `minio:<upstream-release>-bdp.<n>`.
  5. Reproducible builds: pinned Go toolchain, SBOM generation and signed images are mandatory in the build pipeline.

## ADR-013: PostgreSQL Topology

- **Decision:** Two PostgreSQL clusters, both on **PostgreSQL 18**:
  - `pg-platform`: component metadata stores.
  - `pg-app`: OLTP and pgvector workloads.

  Production connections go through **PgBouncer**, as Airflow's documentation recommends for production Postgres.
- **Evidence:** Airflow's set-up-database guide lists PostgreSQL 14, 15, 16, 17 and 18 (Airflow 3.4.0). OpenMetadata 2.0 requires PostgreSQL ≥ 15.
- **Consequences:** PG 18 support for every other metadata consumer (Keycloak, Superset, MLflow, Lakekeeper, Feast, Grafana, JupyterHub, ZenML, OpenFGA) is confirmed per component in Phase 2 (VR-032). Any exception gets its own database line in `pg-platform`.

## ADR-014: Metrics and Alerting

- **Decision:** Prometheus scrapes via native Kubernetes service discovery and annotations. OTLP-capable components push to Prometheus's OTLP receiver. **Alertmanager** routes alerts. Grafana uses Prometheus as its primary data source. Exporters (ADR-015) cover components without native metrics.

## ADR-015: Runtime Dependencies (Platform Prerequisites)

- **Context:** "Do not add technologies" constrains the **functional baseline**. It does not exclude runtime dependencies that baseline components cannot operate without (review direction, 2026-11-17).
- **Decision:** The guide includes a **"Platform Prerequisites"** section in Part I and tags each item `[Prerequisite]`. Each one is chosen by best practice for RKE2 and by the requirements of the baseline component that needs it:

| Prerequisite | Required by | Selection rationale | Class |
|-------------|-------------|---------------------|-------|
| **RKE2** (incl. Canal CNI, CoreDNS, metrics-server, Helm controller) | All | Customer platform decision (ADR-001) | Mandatory |
| **Traefik** (RKE2-packaged) + **Gateway API CRDs** | KServe (Gateway API), all UIs/APIs | RKE2 default ingress from v1.36; RKE2's supported Gateway API implementation; ingress-nginx is retired upstream (ADR-022) | Mandatory |
| **cert-manager** | KServe webhooks; internal TLS | KServe installation prerequisite; standard Kubernetes certificate automation | Mandatory |
| **Longhorn** (CSI) | All stateful workloads; TrustyAI PVC mode | SUSE/Rancher-native distributed block storage for RKE2 (ADR-023) | Mandatory |
| **OCI container registry (Harbor)** | All images, incl. self-built MinIO | CNCF-graduated registry with scanning, signing, replication; holds self-built and mirrored images | Mandatory |
| **OpenSearch ≥ 3.4.0** | OpenMetadata 2.0 | OpenMetadata needs Elasticsearch or OpenSearch; OpenSearch (Apache-2.0) chosen; 3.4.0+ because 3.0–3.3 have a bug triggered by OpenMetadata 2.0 | Mandatory for OpenMetadata |
| **OpenFGA ≥ 1.11** | Lakekeeper authorisation | Lakekeeper OSS's fine-grained authoriser; Cedar is listed for Lakekeeper+ (commercial) | Mandatory for multi-tenant authz |
| **Open Policy Agent (Lakekeeper OPA bridge)** | Trino enforcement of Lakekeeper permissions | Lakekeeper's documented path for query engines like Trino | Conditional (if Trino must enforce catalog grants) |
| **Valkey** | Superset async queries, alerts/reports, thumbnails, caching | Superset needs a Celery broker and cache for these features; Valkey (BSD-licensed, Redis-protocol) is supported by Apache Superset's Kubernetes operator configuration | Mandatory for those Superset features |
| **PgBouncer** | Airflow (and other high-connection clients) on PostgreSQL | Airflow recommends PgBouncer for production Postgres | Mandatory (production) |
| **Prometheus JMX Exporter** | Kafka, Kafka Connect, Cassandra metrics | These expose JMX, not Prometheus format | Mandatory for observability |
| **postgres_exporter** | PostgreSQL metrics | PostgreSQL has no native Prometheus endpoint | Mandatory for observability |
| **Alertmanager** | Prometheus alert routing | Prometheus project component for notifications | Mandatory |
| **Gateway authentication middleware** (Traefik forward-auth to Keycloak) | UIs/APIs without native OIDC (e.g. Prometheus UI, Kafka Connect REST, KServe endpoints) | Uses Traefik's native middleware; exact pattern confirmed in Phase 2 (VR-024, VR-031) | Conditional |

- **Explicitly not used** (not required by any baseline component under the accepted ADRs): Knative, service mesh, schema registry, Spark operator, Prometheus Operator, ZooKeeper, MySQL/MariaDB.
- **Deployment tooling** (Kubernetes operators for Kafka, PostgreSQL, Cassandra, ClickHouse) is a Phase 2 decision per component. It is evaluated against "official Helm chart or manifests first".

## ADR-016: Orchestration Boundary (Airflow vs ZenML)

- **Decision:** **Airflow** orchestrates data pipelines: ingestion, Spark processing, GX validation, ClickHouse materialisation, Feast materialisation triggers. **ZenML** orchestrates ML pipelines: training, evaluation, model-gate, deployment, using the ZenML Kubernetes orchestrator on RKE2.
- **Handoff:** An Airflow task triggers a ZenML pipeline run, or ZenML consumes Iceberg tables published on Airflow's schedule. The mechanism is specified in Phase 3.

## ADR-017: Gated Model Promotion

- **Decision:** No model version reaches a KServe `InferenceService` without a model-gate `PASS`, or a recorded human approval after `NEEDS_REVIEW`. Enforced by pipeline design and the `gate-passed` tag convention (file 05 §10).
- **Consequences:** This is a platform convention, not a native MLflow/KServe feature. Write access to `bdp-ml-serving-*` namespaces is restricted to the deployment pipeline's service account.

## ADR-018: Production Analytical Serving (ClickHouse MergeTree)

- **Decision:** For high-concurrency, low-latency, user-facing serving (Superset dashboards, APIs), curated data is **materialised from Iceberg into ClickHouse `ReplicatedMergeTree`-family tables**. That is the only production-safe serving pattern.
- **Materialisation paths (to be detailed in Workflow 6):**
  1. Airflow-scheduled Trino `INSERT INTO clickhouse.<mart>.<table> SELECT … FROM lakehouse.<ns>.<table>` (catalog-governed read, native Trino ClickHouse connector).
  2. Airflow-scheduled ClickHouse `INSERT … SELECT` from Iceberg via ClickHouse's Iceberg table functions/engines, for high-volume loads (governance implications noted).
- **Restriction:** ClickHouse's experimental `DataLakeCatalog` database engine is permitted **only for ad-hoc exploration in `dev`/`stg`**, never as a production serving path.
- **Closes:** VR-008 (as a design decision).

## ADR-019: OpenMetadata Ingestion Runtime

- **Decision:** OpenMetadata's ingestion framework runs on a **dedicated Airflow instance** (Apache Airflow, baseline technology) using the OpenMetadata ingestion image, deployed in `bdp-governance`.
- **Rationale:** OpenMetadata releases have specific Airflow compatibility requirements. Coupling its ingestion runtime to the platform's data-pipeline Airflow would tie two upgrade cycles together.
- **Consequences:** Two Airflow deployments with distinct responsibilities. Both share the Keycloak and OpenBao patterns. The platform Airflow can still trigger OpenMetadata ingestion via the OpenMetadata API where lineage freshness requires it.

## ADR-020: ZenML → KServe Deployment

- **Decision:**
  - **Do not use** ZenML's deprecated Model Deployer abstraction.
  - Training/evaluation pipelines run on the **ZenML Kubernetes orchestrator**, with MLflow as experiment tracker and model registry.
  - Deployment to KServe is a **platform deployment step** in a ZenML pipeline `[Custom]`. After model-gate approval it creates or patches an `InferenceService` through the Kubernetes API (or the KServe Python SDK). `storageUri` points at the approved model artefact in MinIO, and the step waits for `Ready` before switching the MLflow `champion` alias.
  - ZenML's newer **Kubernetes Deployer** (pipeline deployments as HTTP services) is reserved for multi-step inference pipelines that do not fit a single KServe predictor `[Optional]`.
- **Rationale:** Keeps KServe as the single model-serving control plane, uses only supported ZenML abstractions, and puts the governance gate in a deterministic position.
- **Closes:** VR-012 (design). Lab verification is a Phase 3 integration test.

## ADR-021: TrustyAI on RKE2

- **Decision:**
  - Install the **upstream TrustyAI Kubernetes operator** (`trustyai-service-operator`, Apache-2.0) from its release manifests into `bdp-ml-platform`.
  - Deploy one `TrustyAIService` custom resource per serving namespace (`bdp-ml-serving-<domain>`).
  - Default storage mode is **PVC on Longhorn** (DATABASE mode documentation targets MariaDB/MySQL, which is outside the baseline; VR-033).
  - Integrate with KServe Standard mode inference payload logging.
  - Prometheus scrapes TrustyAI fairness/drift metrics.
- **Rationale:** Vendor distributions (OpenShift AI, Alauda) are built on this same upstream operator. The upstream operator is the portable path on RKE2.
- **Consequences:** KServe raw-mode ↔ TrustyAI payload integration and RKE2 Pod Security compliance are validated in a lab (VR-013). Workflow 14 documents only what that validation confirms.

## ADR-022: North-South Traffic (Traefik + Gateway API)

- **Decision:** Use RKE2's packaged **Traefik** as the ingress and Gateway API implementation. Enable `providers.kubernetesGateway` via `HelmChartConfig`. Use **Gateway API (`Gateway`, `HTTPRoute`)** for new routes; Ingress resources only where a component's chart offers nothing else.
- **Rationale:** RKE2 v1.36 deploys Traefik by default. RKE2 documents Traefik as the way to use Gateway API. ingress-nginx is retired upstream and deprecated in RKE2 1.36.
- **Consequences:** TLS terminates at the Gateway (certificates from cert-manager). Keycloak forward-auth middleware where needed (ADR-015). Experimental Gateway API resources (e.g. TCPRoute) only if explicitly required.

## ADR-023: Storage Classes

- **Decision:**
  - **Longhorn** is the default StorageClass (`longhorn`, replica count 3) for stateful components without their own replication.
  - Components that replicate data themselves (Kafka, Cassandra, ClickHouse, OpenSearch, MinIO erasure sets) use a **single-replica/local-disk class** (`longhorn-local` with replica count 1 and strict-local data locality, or local PVs), so data is not replicated twice.
- **Consequences:** Exact class parameters are fixed per component in Phase 2.

## ADR-024: model-gate Licence

- **Decision:** The guide documents model-gate under the **MIT licence** published by its owner. "Proprietary component" in the brief means it is a **platform-specific governance component**, not a licensing restriction.
- **Closes:** VR-020.

## ADR-025: Data Quality Scope

- **Decision:** **GX Core only.** No GX Cloud features, APIs or UI are referenced. Validation results are stored in `bdp-<env>-governance` and surfaced via OpenMetadata.
