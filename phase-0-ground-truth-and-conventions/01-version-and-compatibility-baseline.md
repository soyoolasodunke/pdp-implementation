# 01: Version and Compatibility Baseline

## 1. How to Read This File

- **Reference version**: the release line every example, configuration key and API reference in this guide is written against. It is a *documentation pin*, not a deployment mandate.
- **Latest observed**: the newest release found during Phase 0 research (2026-11-17). Several of these come from secondary trackers. Re-verify at pin time against the official release page linked in §6.
- **Confidence**: High = confirmed on an official project page; Medium = consistent across secondary sources; Low = single secondary source or conflicting data.

### Pinning policy

1. Pin to a **minor release line** (e.g. `3.4.x`) in the guide; pin to an **exact patch** in deployment manifests.
2. Prefer **LTS lines** where the project offers them (Prometheus, ClickHouse).
3. For projects that only maintain the latest release (Keycloak, OpenBao, NiFi), plan an upgrade cadence of at most one or two releases behind.
4. Any version change triggers a re-check of the constraints in §4.

---

## 2. Runtime Baselines

| Runtime | Reference | Rationale | Confidence |
|---------|-----------|-----------|------------|
| Kubernetes distribution | **RKE2 v1.36.x** (Kubernetes 1.36) | Customer decision (ADR-001). RKE2 v1.36 deploys Traefik as the default ingress controller (ingress-nginx deprecated in 1.36). A recent patch (v1.36.5+rke2r1) ships Traefik 3.7.13 and Canal (default, FIPS-compliant) CNI. Z2JH chart requires Kubernetes ≥ 1.28. | High |
| Python (platform code, SDKs) | **3.12** | Fits the intersection of GX Core (3.10–3.13), Airflow 3 (3.10–3.14), MLflow (≥ 3.10), FastAPI (≥ 3.10), model-gate (≥ 3.9). ZenML/Feast upper bounds still to confirm (VR-021). | Medium |
| Java (per component) | Component-specific (see §3) | Trino now requires Java 25. Debezium 3.x requires Java 17+. Other JVM components have their own minimums. Run each component on the JDK its official image ships with; do not share a JDK across components. | Medium |

---

## 3. Component Reference Versions

### 3A. Big Data and Data Engineering

| Component | Layer | Reference line | Latest observed | Support / lifecycle notes | Runtime notes | Confidence |
|-----------|-------|----------------|-----------------|---------------------------|---------------|------------|
| Apache NiFi | Ingestion | **2.11.x** | 2.11.0 (2026-08-03); a 2.12.0 is reported by one third-party tracker | Only the latest minor is maintained. **NiFi Registry is deprecated (Feb 2026) and planned for removal in NiFi 3.0**. Use Git-based Flow Registry Clients. | Java 21 (NiFi 2.x) [VR-021] | High |
| Debezium | CDC | **3.7.x** | 3.7 GA (late Sept 2026); 3.6.3.Final maintenance | Community supports latest stable + active dev branch. Cassandra 3 connector removed in 3.7. | Java 17+; runs on Kafka Connect or Debezium Server | High |
| Apache Kafka | Streaming | **4.3.x** (4.4.x acceptable) | 4.4.0 (2026-10-22); 4.3.1 (2026-06-25) | 4.x is **KRaft-only**; ZooKeeper unsupported. | Broker Java 17+ [VR-021] | High |
| MinIO | Object storage | **Frozen last CE build; in-house self-builds thereafter** (ADR-012) | CE last listed release Oct 2025 (reported); repo archived 2026 | CE no longer maintained upstream. Bluechip owns patching and rebuilding. Image tag `minio:<upstream-release>-bdp.<n>`. | Go toolchain pinned in build pipeline | Medium |
| Apache Iceberg | Table format | **1.12.x** | 1.12.0 (2026-09-30) | Spark 3.4 module removed; supported Spark lines 3.5.9, 4.0.4, 4.1.3. | Engine runtime JARs per engine version | High |
| Lakekeeper | Iceberg REST catalog | **0.15.x** | 0.15.0 (2026-11-06) | 0.15 contains **breaking changes and security behaviour changes**. Read the upgrade notes. Remote-signing fix for Iceberg ≥ 1.12 landed in 0.14.1. | Rust; PostgreSQL metadata backend | High |
| Apache Cassandra | Operational NoSQL | **5.0.x** | 5.0.9 (2026-08-07) | Latest three majors supported (5.0, 4.1, 4.0). | Java 11/17 [VR-021] | High |
| PostgreSQL | Relational / vector | **18.x** | 18.x patch line (18.6 seen Aug 2026). **PG 19 GA planned 2026-11-19** | Both `pg-platform` and `pg-app` run 18.x (ADR-013). Do not adopt 19 until extension and client compatibility is confirmed. PG 18 support per metadata consumer: VR-032. | n/a | High |
| pgvector | Vector extension | **0.8.x** | 0.8.7 (Oct 2026, per EDB release notes) | Distro packages may lag (pgdg shows 0.8.3). | Must match PG major | Medium |
| Apache Spark | Processing | **4.1.x** | 4.1.x patch line; a 4.2.x line is also reported | Pin to 4.1 because Iceberg 1.12 officially supports 4.1.3; 4.2 is only "work in source". Spark 4.1 EOL listed 2027-06-11. | Java 17/21; Scala 2.13 | High |
| Trino | Federated SQL | **484** | 484 (2026-10-15) | Trino has no LTS. 484 has **breaking changes in the Iceberg/Hive connectors and in JSON/interval behaviour**. | **Java 25 required** | High |
| ClickHouse | Analytical OLAP | **26.8 LTS** | 26.8 LTS (2026-08-27); 26.3 LTS previous | Prefer LTS. Production serving uses MergeTree materialisation (ADR-018). The experimental `DataLakeCatalog` (Iceberg REST) is allowed only for dev/stg exploration. | C++ | High |
| GX Core | Data quality | **1.24.x** | 1.24.1 (2026-10-22) | **GX Core only (ADR-025); no GX Cloud features referenced.** GX Cloud discontinued 2026-06-01; GX Core continues with Fivetran as steward. SQLAlchemy 2.1 broke GX ≤ 1.23.1 (fixed 1.23.2). | Python 3.10–3.13 | High |
| OpenMetadata | Metadata / lineage | **2.0.x** | 2.0.7 (2026-10-27) | Monthly release cadence; 2.0 had ingestion regressions fixed in 2.0.5. | Server: Java; needs relational DB + search engine (VR-003) | High |
| Apache Airflow | Orchestration | **3.4.x** | 3.4.0 (current stable docs); 3.3.2 (2026-09-17) | Airflow 2 is EOL (22 Apr 2026). | Python 3.10–3.14; **metadata DB: PostgreSQL 14–18** (official docs); PgBouncer recommended for production; driver `postgresql+psycopg://` | High |
| Apache Superset | BI | **6.1.x** | 6.1.0 (2026-05-13) | First minor of 6.x line. | Python; metadata DB | High |

### 3B. Platform Security and Observability

| Component | Reference line | Latest observed | Support / lifecycle notes | Confidence |
|-----------|----------------|-----------------|---------------------------|------------|
| Keycloak | **26.8.x** | 26.8.1 (Oct 2026) | **Only the latest release is maintained**; frequent security fixes. Plan monthly patching. | High |
| OpenBao | **2.7.x** | 2.7.2 (2026-11-05) | Only the latest release cycle is maintained. 2.7 adds PostgreSQL horizontal scalability and control groups. | Medium |
| Prometheus | **3.13 LTS** | 3.16.0 (2026-11-05); 3.13.5 LTS | 3.13 LTS supported to 2027-07-31. Non-LTS minors get about 6 weeks of support. | High |
| Grafana | **13.x** | 13.1 (2026-06-24); later 13.x minors possible | Git Sync went GA in 13.0. | High |

### 3C. MLOps and AI Engineering

| Component | Reference line | Latest observed | Support / lifecycle notes | Confidence |
|-----------|----------------|-----------------|---------------------------|------------|
| JupyterHub | **5.5.x** (Z2JH chart **4.4.x**) | JupyterHub 5.5.1; chart 4.4.0 (2026-06-10) | JupyterHub 6.0 in development. Chart 4.4.0 is a security release. | High |
| MLflow | **3.17.x** | 3.17.1 (2026-10-13) | Official Helm chart and RBAC/Admin UI since 3.13. Fast release cadence. | High |
| ZenML | **0.96.x** | 0.96.4 (2026-09-04) | **Model Deployer component deprecated** → Deployer / Pipeline Deployments. | High |
| KServe | **0.21.x** (0.20.x fallback) | 0.21.0 (Sept 2026, third-party report); 0.20.0 (2026-08-06) | Two CRDs: `InferenceService` (predictive) and `LLMInferenceService` (GenAI). Standard (raw) vs Knative mode. One report says Helm charts for the newest release were not published (VR-011). | Medium |
| FastAPI | **0.16x** | 0.163.0 (2026-11-14) | Still pre-1.0; minor versions may contain breaking changes. **Pin exactly.** Python ≥ 3.10 since 0.129. | High |
| TrustyAI | **Operator + Service, line to confirm** | n/a | Documented mainly via Open Data Hub / OpenShift AI (VR-013). | Low |
| Feast | **0.68.x** | 0.68.0 | Pre-1.0, roughly monthly minors that can change registry behaviour. **Run ≥ 0.59 (CVE-2026-23537)** and enable auth. | Medium |
| model-gate (`bdp-model-gate`) | **0.7.x** | 0.7.0 (2026-10-15) | **MIT licence** (ADR-024). Pre-1.0; roadmap lists 0.7.1/0.7.2 fixes, 0.8.0 release automation, 1.0.0 public `ModelAdapter`. See file 05. | High |

---

### 3D. Platform Prerequisites (Runtime Dependencies, ADR-015)

Not part of the functional baseline. Required by baseline components or by the RKE2 platform. Exact patch versions are pinned per component in Phase 2.

| Prerequisite | Required by | Reference line / constraint | Source of truth |
|-------------|-------------|------------------------------|-----------------|
| RKE2 packaged Traefik + Gateway API CRDs | Gateway/ingress, KServe | Traefik 3.7.x (Gateway API v1.5); enable `providers.kubernetesGateway` via `HelmChartConfig` | RKE2 networking docs |
| cert-manager | KServe webhooks, internal TLS | Version per KServe 0.21 install docs (VR-002) | KServe install docs |
| Longhorn | Default StorageClass; TrustyAI PVC mode | Release supported for RKE2 1.36 (Phase 2) | Longhorn docs |
| Harbor (OCI registry) | All images; self-built MinIO | Current stable (Phase 2) | Harbor docs |
| OpenSearch | OpenMetadata 2.0 | **≥ 3.4.0** (3.0–3.3 bug with OpenMetadata 2.0) | OpenMetadata 2.0 requirements |
| OpenFGA | Lakekeeper authz | **≥ 1.11** | Lakekeeper OpenFGA docs |
| Open Policy Agent | Trino enforcement of Lakekeeper grants (conditional) | Per Lakekeeper OPA bridge docs | Lakekeeper docs |
| Valkey | Superset Celery broker, results backend, caches | Current stable (Phase 2) | Superset docs |
| PgBouncer | Production Postgres connection pooling (Airflow et al.) | Current stable | Airflow DB docs |
| Prometheus JMX Exporter | Kafka, Kafka Connect, Cassandra | Java agent; current stable | Prometheus project |
| postgres_exporter | PostgreSQL | Current stable | prometheus-community |
| Alertmanager | Prometheus alert routing | Version paired with Prometheus 3.13 LTS | Prometheus project |

---

## 4. Cross-Component Compatibility Constraints

Each constraint is a rule the guide must respect. "Check" means re-validate whenever either side changes version.

| ID | Constraint | Components | Implication for the guide |
|----|-----------|------------|---------------------------|
| CC-01 | Spark Iceberg runtime JAR must match **Spark minor + Scala version + Iceberg version** (e.g. `iceberg-spark-runtime-4.1_2.13:1.12.x`). | Spark, Iceberg | All Spark examples use this exact coordinate pattern. |
| CC-02 | All engines talk to Lakekeeper over the **Iceberg REST Catalog API**. Feature coverage (views, remote signing, credential vending, nested namespaces) differs by engine. | Lakekeeper, Spark, Trino, ClickHouse, Feast | Integration specs must state which REST features each engine actually uses (VR-007). |
| CC-03 | Trino 484 requires **Java 25**. | Trino | The Trino image is built and run separately; no shared JDK base. |
| CC-04 | Kafka 4.x is **KRaft-only**. Debezium (Kafka Connect) must use a client compatible with the broker version. | Kafka, Debezium | No ZooKeeper anywhere in the guide. Connect worker version is aligned with the broker line. |
| CC-05 | Debezium connectors capture from **PostgreSQL** (logical decoding, `pgoutput`) and others. The source PG must have `wal_level=logical` and replication slots sized correctly. | Debezium, PostgreSQL | Workflow 2 configures the source database explicitly. |
| CC-06 | GX Core supports Python 3.10–3.13, and SQLAlchemy 2.1 broke GX ≤ 1.23.1. | GX Core, Airflow, ZenML | Pin GX ≥ 1.23.2 and SQLAlchemy consistently in shared images. |
| CC-07 | Airflow 3.4 metadata DB supports **PostgreSQL 14–18** (official). | Airflow, PostgreSQL | Airflow metadata runs on PG 18 in `pg-platform`, via PgBouncer (ADR-013). |
| CC-08 | ClickHouse reads Iceberg via the `Iceberg*` table functions/engines or the experimental `DataLakeCatalog` database engine (REST). | ClickHouse, Iceberg, Lakekeeper, Trino | Production: materialise into `ReplicatedMergeTree` (via Trino ClickHouse connector or ClickHouse `INSERT … SELECT`), per ADR-018. `DataLakeCatalog` only in dev/stg. |
| CC-09 | pgvector build must match the PostgreSQL major. Distro packages may lag upstream. | PostgreSQL, pgvector | Image build pins both. |
| CC-10 | Feast Spark/Trino offline stores and PostgreSQL/Cassandra online stores are **contrib** plugins (not core). | Feast, Spark, Trino, PostgreSQL, Cassandra | Feast integration specs are labelled `[Connector – contrib]` and need test coverage (VR-014). |
| CC-11 | MLflow 3.x uses the `LoggedModel` entity and a changed model-logging API compared with 2.x. | MLflow, ZenML, Feast | Examples use MLflow 3 APIs only; ZenML's MLflow integration must support 3.x (confirmed since ZenML 0.91.2). |
| CC-12 | KServe needs **cert-manager** for admission webhooks. In Standard mode it uses **Gateway API** (`enableGatewayApi: true`), served on RKE2 by Traefik with `providers.kubernetesGateway` enabled. | KServe, Traefik, cert-manager | ADR-008, ADR-022; version fit checked in VR-002. |
| CC-13 | OpenMetadata 2.0 needs MySQL ≥ 8.0.42 or **PostgreSQL ≥ 15**, plus Elasticsearch 9.x or **OpenSearch 3.x (≥ 3.4.0)**, and Java 21. | OpenMetadata, PostgreSQL, OpenSearch | PostgreSQL 18 + OpenSearch ≥ 3.4.0 (ADR-015). |
| CC-16 | Lakekeeper fine-grained authorisation requires **OpenFGA ≥ 1.11**. `LAKEKEEPER__AUTHZ_BACKEND=openfga` must be set **before** `lakekeeper migrate`. | Lakekeeper, OpenFGA | Bootstrap sequence documented in Part II. |
| CC-17 | OpenMetadata ingestion requires the Airflow version bundled/pinned by the OpenMetadata release. | OpenMetadata, Airflow | Dedicated ingestion Airflow (ADR-019). |
| CC-18 | RKE2 Traefik's Gateway API version (Traefik 3.7.x ↔ Gateway API v1.5) must cover the Gateway API resources KServe creates. | RKE2, Traefik, KServe | VR-002. |
| CC-19 | Before the April 2026 RKE2 patch releases, disabling Traefik after enabling it removed the Gateway API CRDs. | RKE2 | Use RKE2 ≥ the April 2026 patches; never disable the packaged Traefik once Gateway API is in use. |
| CC-14 | Keycloak and OpenBao both support only their latest release line. | Keycloak, OpenBao | Ops guidance includes a patch cadence. |
| CC-15 | model-gate's statistical checks need roughly ≥ 500 validation rows to avoid `UNCERTAIN` outcomes. | model-gate, ZenML | Workflow 12 states minimum validation-set sizing. |

---

## 5. Supply-Chain and Lifecycle Risks

| Risk | Component | Note |
|------|-----------|------|
| Upstream unmaintained | MinIO CE | Accepted (ADR-012): frozen build + in-house rebuilds with CVE monitoring, SBOM and image signing. |
| No provenance attestations | model-gate | Release files uploaded without Trusted Publishing (planned for 0.8.0). Mirror and hash-pin in an internal index. |
| Pre-1.0 API churn | FastAPI, Feast, ZenML, model-gate, Lakekeeper, KServe | Pin exact versions; isolate upgrades. |
| Stewardship change | GX Core | Monitor Fivetran-stewarded releases. |
| Recent critical CVEs | Feast (< 0.59), Keycloak (frequent), NiFi (2026 CVEs) | Stay on the latest patch releases. |

---

## 6. Sources

Official sources first; secondary trackers marked (2°).

- NiFi downloads and Registry deprecation: https://nifi.apache.org/download/ · release notes: https://cwiki.apache.org/confluence/display/NIFI/Release%20Notes
- Debezium blog/tag index: https://debezium.io/tag/cassandra
- Kafka 4.4.0 announcement: https://kafka.apache.org/blog/2026/10/22/apache-kafka-4.4.0-release-announcement/ · 4.3.1: https://kafka.apache.org/blog/2026/06/25/apache-kafka-4.3.1-release-announcement/
- MinIO CE status (2°): https://itsfoss.com/news/minio-moves-away-from-open-source/ · https://www.youngju.dev/blog/2026-07-17-minio-archived-garage-seaweedfs-ceph-rgw.en
- Iceberg releases: https://iceberg.apache.org/releases/ · 1.12 analysis (2°): https://www.dremio.com/blog/apache-iceberg-1-12-0-whats-new-breaking-changes-and-upgrade-guide
- Lakekeeper release notes: https://docs.lakekeeper.io/about/release-notes/
- Cassandra: https://endoflife.date/cassandra (2°) · https://cassandra.apache.org/download/
- PostgreSQL 19 schedule (2°): https://releasebot.io/updates/postgresql · pgvector notes: https://www.enterprisedb.com/docs/pg_extensions/pgvector/rel_notes/
- Spark: https://spark.apache.org/news/ · Spark 4.1 EOL (2°): https://endoflife.ai/apache-spark/4.1
- Trino 484 (2°, contributor call): https://github-wiki-see.page/m/trinodb/trino/wiki/Contributor-meetings · Trino release notes: https://trino.io/docs/current/release.html
- ClickHouse changelog: https://clickhouse.com/docs/category/changelog · Iceberg catalog (EDB docs): https://www.enterprisedb.com/docs/clickhouse/latest/reading_iceberg/
- GX Core changelog: https://docs.greatexpectations.io/docs/changelog · stewardship: https://www.greatexpectations.io/blog/an-update-from-great-expectations
- OpenMetadata releases: https://docs.open-metadata.org/latest/releases
- Airflow release notes: https://airflow.apache.org/docs/apache-airflow/3.3.2/release_notes.html · requirements (2°): https://versionlog.com/apache-airflow/3/
- Superset (2°): https://en.wikipedia.org/wiki/Apache_Superset · https://preset.io/blog/apache-superset-6-1-release/
- Keycloak 26.8.1: https://www.keycloak.org/2026/10/keycloak-2681-released
- OpenBao release notes: https://openbao.org/community/release-notes/2-6-0/ · 2.7 (2°): https://infoq.com/news/2026/11/vault-fork-openbao-27
- Prometheus lifecycle (2°): https://endoflife.date/prometheus · 3.13 LTS: https://github.com/prometheus/prometheus/releases/tag/v3.13.0
- Grafana 13.1: https://grafana.com/blog/grafana-13-1-release-all-the-latest-features
- Z2JH changelog: https://z2jh.jupyter.org/en/stable/changelog.html
- MLflow changelog: https://raw.githubusercontent.com/mlflow/mlflow/master/CHANGELOG.md
- ZenML changelog: https://docs.zenml.io/changelog/server-sdk · Model Deployer deprecation: https://docs.zenml.io/stacks/model-deployers
- KServe 0.17/0.18 release posts: https://kserve.github.io/website/blog/kserve-0.18-release · installation: https://kserve.github.io/website/docs/install/kserve-install
- TrustyAI operator: https://github.com/trustyai-explainability/trustyai-service-operator
- Feast releases: https://github.com/feast-dev/feast/releases
- FastAPI release notes: https://fastapi.tiangolo.com/release-notes/
- model-gate: https://pypi.org/project/bdp-model-gate/0.7.0/
- Kubernetes releases: https://kubernetes.io/releases/
- RKE2 networking services (Traefik, Gateway API): https://docs.rke2.io/networking/networking_services · RKE2 v1.36 release notes: https://documentation.suse.com/en-us/cloudnative/rke2/latest/en/release-notes/v1.36.X.html
- Airflow database set-up (PostgreSQL 14–18): https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html
- OpenMetadata 2.0 requirements: https://docs.open-metadata.org/v2.0.x/deployment/minimum-requirements
- Lakekeeper authorisation / OpenFGA: https://docs.lakekeeper.io/docs/latest/authorization/
- KServe Kubernetes deployment (Gateway API recommended): https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment
- Superset async queries via Celery: https://superset.apache.org/admin-docs/configuration/async-queries-celery
- ZenML Kubernetes Deployer: https://docs.zenml.io/stacks/stack-components/deployers/kubernetes
- TrustyAI operator: https://github.com/trustyai-explainability/trustyai-service-operator
