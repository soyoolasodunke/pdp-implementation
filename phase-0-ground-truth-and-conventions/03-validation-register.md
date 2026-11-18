# 03: Technical Validation Register

## 1. Purpose and Rules

The brief requires that "where information cannot be verified, explicitly identify it as requiring technical validation rather than making assumptions." This register is that mechanism.

**Rules:**

1. Any statement in the guide that depends on an open item carries its tag, e.g. `[VR-013]`.
2. An item is **closed** only with evidence (an official doc link for the reference version, or a reproducible lab test) **or** by an accepted ADR that removes the dependency.
3. Closed items keep their record; the resolution is written into the relevant spec.
4. **Severity:** **H** = could invalidate a workflow or force an architecture change; **M** = changes configuration or integration code; **L** = documentation accuracy only.
5. Open items do **not** block phase exit. They block only the phase listed under "Needed by".

**Status values:** `Open` · `In validation` · `Closed – confirmed` (evidence) · `Closed – decided` (ADR) · `Closed – disproved` · `Accepted risk`

---

## 2. Status Summary at Phase 0 Exit

| Status | Count | Items |
|--------|-------|-------|
| Closed – confirmed | 4 | VR-003, VR-009, VR-025, VR-026 |
| Closed – decided | 5 | VR-001, VR-008, VR-012, VR-020, VR-035 |
| Open (scheduled) | 27 | All others, each with a target phase |

---

## 3. Register

| ID | Area | Item to validate | Sev | Method | Needed by | Status / resolution |
|----|------|------------------|-----|--------|-----------|---------------------|
| VR-001 | MinIO | Distribution and support model | H | Decision | — | **Closed – decided** (ADR-012): frozen last CE build; in-house self-build with CVE monitoring, SBOM, signed images. |
| VR-002 | KServe | Gateway API on RKE2 Traefik: compatibility of KServe's Gateway API usage with Traefik 3.7 / Gateway API v1.5+; cert-manager version for KServe 0.21 | H | Lab install on RKE2 1.36 | Phase 2 (MLOps batch) | Open, scope narrowed by ADR-008/022 |
| VR-003 | OpenMetadata | Search engine and DB requirements for 2.0.x | H | Official docs | — | **Closed – confirmed:** requires MySQL ≥ 8.0.42 or PostgreSQL ≥ 15, plus Elasticsearch 9.x or OpenSearch 3.x (**use ≥ 3.4.0**; 3.0–3.3 bug); Java 21. OpenSearch selected (ADR-015). |
| VR-004 | Observability | Native Prometheus endpoint per component; exporter list | M | Docs + lab scrape | Phase 2 (batch 5) | Open. Exporters for Kafka/Connect/Cassandra/PostgreSQL already accepted (ADR-015). |
| VR-005 | Kafka / Debezium | JSON serialisation and schema evolution end-to-end (Debezium → Kafka → Spark → Iceberg) | M | Lab | Phase 3 | Open |
| VR-006 | Spark / Airflow | Airflow operator for native `spark-submit` on Kubernetes | M | Lab DAG | Phase 3 | Open |
| VR-007 | Iceberg / Lakekeeper | Per-engine REST-catalog feature coverage (credential vending vs remote signing, views, nested namespaces, v3 types, OAuth2 via Keycloak) for Spark, Trino, ClickHouse, Feast, NiFi | H | Lab matrix | Phase 3 | Open |
| VR-008 | ClickHouse | `DataLakeCatalog` maturity | H | Decision | — | **Closed – decided** (ADR-018): production serving uses MergeTree materialisation; `DataLakeCatalog` only for dev/stg exploration. |
| VR-009 | Airflow | Metadata DB on PostgreSQL 18 | M | Official docs | — | **Closed – confirmed:** Airflow set-up-database guide (3.4.0) lists PostgreSQL 14, 15, 16, 17, 18; PgBouncer recommended for production. |
| VR-010 | MinIO ↔ Keycloak | Identity features remaining in the frozen CE build (OIDC/STS, LDAP, policy claims) | H | Lab with frozen build | Phase 2 (batch 2) | Open |
| VR-011 | KServe | Install artefacts for 0.21.x (OCI Helm charts at `ghcr.io/kserve/charts` vs release manifests) | M | Registry check | Phase 2 (MLOps batch) | Open. OCI charts are documented for recent releases; confirm for 0.21. |
| VR-012 | ZenML ↔ KServe | Deployment path given Model Deployer deprecation | H | Decision | — | **Closed – decided** (ADR-020): platform deployment step applies the `InferenceService`; lab test in Phase 3. |
| VR-013 | TrustyAI | Upstream operator on RKE2 (PSA/CIS compliance), KServe Standard-mode payload logging to `TrustyAIService`, PVC mode on Longhorn | H | Lab | Phase 2 (MLOps batch) | Open, direction set by ADR-021 |
| VR-014 | Feast | Contrib stores at 0.68: Spark/Trino offline over Iceberg via Lakekeeper; PostgreSQL (pgvector) and Cassandra online | M | Lab feature repo | Phase 3 | Open |
| VR-015 | Airflow ↔ OpenBao | Airflow HashiCorp Vault secrets backend against OpenBao API | M | Lab | Phase 3 | Open |
| VR-016 | OpenBao | Database secrets engine coverage for PostgreSQL, Cassandra, ClickHouse | M | Plugin docs + lab | Phase 2 (batch 5) | Open |
| VR-017 | OpenBao | Agent injector / CSI provider in OpenBao Helm chart (2.7.x) | M | Chart review + lab | Phase 2 (batch 5) | Open |
| VR-018 | ZenML | Supported server DB engines for 0.96 (PostgreSQL vs MySQL only) | M | Official docs | Phase 2 (MLOps batch) | Open. If MySQL-only, raised as a new ADR-015 prerequisite. |
| VR-019 | Prometheus | OTLP receiver in 3.13 LTS; OTel-emitting components (e.g. Airflow) | M | Docs + lab | Phase 2 (batch 5) | Open |
| VR-020 | model-gate | Licence | M | Owner decision | — | **Closed – decided** (ADR-024): MIT. |
| VR-021 | Runtimes | Java min/max per JVM component; Python upper bounds for ZenML 0.96 and Feast 0.68 | L | Official requirements | Phase 2 (each batch) | Open |
| VR-022 | PostgreSQL 18 | OAuth authentication validator module compatible with Keycloak tokens | M | PG 18 docs + lab | Phase 2 (batch 2) | Open |
| VR-023 | MLflow ↔ Keycloak | SSO for MLflow 3.17 (native RBAC + OIDC vs Gateway forward-auth) | M | Docs + lab | Phase 2 (MLOps batch) | Open |
| VR-024 | KServe | Endpoint authN/authZ in Standard mode via Gateway + Keycloak | H | Lab | Phase 2 (MLOps batch) | Open |
| VR-025 | Superset | Broker/cache dependency | M | Official docs | — | **Closed – confirmed:** async queries, alerts/reports and thumbnails need a Celery broker, results backend and cache; Valkey selected (ADR-015). |
| VR-026 | Lakekeeper | Authorisation backend in OSS 0.15 | H | Official docs | — | **Closed – confirmed:** OpenFGA (≥ 1.11) is the OSS fine-grained authoriser; Cedar listed for Lakekeeper+; OPA bridge for Trino (ADR-015). |
| VR-027 | OpenMetadata | Connector availability in 2.0.x for Iceberg (REST/Lakekeeper), Trino, ClickHouse, Kafka, Superset, Airflow, MLflow, PostgreSQL, Cassandra, NiFi, Spark lineage | M | Connector docs | Phase 2 (batch 4) | Open |
| VR-028 | Debezium → Iceberg | Kafka → Spark Structured Streaming vs Iceberg Kafka Connect sink, with Lakekeeper | M | Lab | Phase 3 | Open |
| VR-029 | NiFi → Iceberg | NiFi 2.11 Iceberg processors with REST catalog vs landing files for Spark | M | Docs + lab | Phase 3 | Open |
| VR-030 | KServe ↔ MLflow | Serving MLflow-format models on KServe (runtime used for `mlflow` format), storage initializer pulling from MinIO with OpenBao-delivered credentials | H | Lab | Phase 3 | Open |
| VR-031 | Kafka Connect | Securing the Connect REST API (auth extensions vs Gateway forward-auth vs network policy only) | H | Docs + lab | Phase 2 (batch 1) | Open |
| VR-032 | PostgreSQL 18 | PG 18 support for every `pg-platform` consumer: Keycloak, Superset, MLflow, Lakekeeper, Feast registry, Grafana, JupyterHub, ZenML, OpenFGA | M | Official docs per component | Phase 2 (each batch) | Open (ADR-013) |
| VR-033 | TrustyAI | DATABASE storage mode engines supported upstream (docs show MariaDB/MySQL); PostgreSQL support | M | Operator docs | Phase 2 (MLOps batch) | Open. PVC mode is the default meanwhile (ADR-021). |
| VR-034 | cert-manager ↔ OpenBao | cert-manager Vault-compatible issuer against OpenBao PKI | M | Lab | Phase 2 (batch 5) | Open |
| VR-035 | OpenMetadata | Ingestion runtime placement | M | Decision | — | **Closed – decided** (ADR-019): dedicated, version-coupled Airflow instance. |
| VR-036 | RKE2 hardening | Pod Security Admission / CIS profile compatibility for each component (privileged needs, e.g. Longhorn, OpenSearch `vm.max_map_count`, Spark, NiFi) | M | Lab on CIS-profile RKE2 | Phase 2 (each batch) | Open |

---

## 4. Preliminary Keycloak Integration Coverage (Hypotheses to Verify in Phase 2)

> **Status: preliminary.** These are working hypotheses to direct Phase 2 research. **Do not quote them as fact** until each row is confirmed. "Gateway forward-auth" refers to the Traefik middleware pattern accepted in ADR-015.

| Component | Expected mechanism | Class (hypothesis) | Related VR |
|-----------|-------------------|--------------------|-----------|
| NiFi | OIDC login | Native | — |
| Kafka | SASL/OAUTHBEARER with OIDC tokens | Native | — |
| Kafka Connect REST | Not native → Gateway forward-auth / network policy | Proxy | VR-031 |
| MinIO (frozen CE) | OIDC/STS (CE status uncertain) | Uncertain | VR-010 |
| Lakekeeper | OIDC bearer tokens (+ K8s SA tokens) | Native | VR-007 |
| Cassandra | Password/LDAP; no native OIDC | None | — |
| PostgreSQL 18 | OAuth via validator module | Conditional | VR-022 |
| Spark | K8s RBAC + downstream service auth | Indirect | — |
| Trino | OAuth2 (UI) / JWT (clients) | Native | — |
| ClickHouse (OSS) | LDAP/Kerberos; OIDC/JWT not assumed | Uncertain | — |
| GX Core | Library; caller credentials | n/a | — |
| OpenMetadata | OIDC SSO + bot JWTs | Native | — |
| Airflow 3 | Auth manager with OIDC | Native/provider | — |
| Superset | Flask-AppBuilder OAuth | Native (config) | — |
| OpenBao | OIDC/JWT auth methods | Native | — |
| Prometheus / Alertmanager | Gateway forward-auth | Proxy | — |
| Grafana | Generic OAuth | Native | — |
| JupyterHub | GenericOAuthenticator | Native (authenticator package) | — |
| MLflow | RBAC (3.13+); OIDC path uncertain | Uncertain | VR-023 |
| ZenML OSS | Uncertain | Uncertain | — |
| KServe | Gateway forward-auth / custom | Proxy | VR-024 |
| FastAPI | JWT validation against Keycloak JWKS | Custom code | — |
| TrustyAI | Uncertain | Uncertain | VR-013 |
| Feast | OIDC auth for servers | Native | VR-014 |
| model-gate | Library/CLI; no auth surface | n/a | — |

---

## 5. Preliminary Metrics Coverage (Hypotheses to Verify in Phase 2)

| Component | Expected metrics source | Exporter |
|-----------|------------------------|----------|
| NiFi 2.x | Built-in Prometheus endpoint | No (verify) |
| Kafka / Kafka Connect | JMX | **JMX Exporter** (ADR-015) |
| MinIO | Native Prometheus endpoint | No |
| Lakekeeper | Native metrics endpoint (port 9000 per docs) | No |
| Cassandra | JMX | **JMX Exporter** (ADR-015) |
| PostgreSQL | Statistics views | **postgres_exporter** (ADR-015) |
| Spark | PrometheusServlet / metrics sink | No (config) |
| Trino | `/metrics` OpenMetrics | No |
| ClickHouse | Native Prometheus port 9363 | No |
| OpenMetadata | Native metrics endpoint | No (verify) |
| Airflow 3 | OpenTelemetry → Prometheus OTLP | No (VR-019) |
| Superset | StatsD-style logger | Uncertain |
| Keycloak | Management port `/metrics` | No |
| OpenBao | `/v1/sys/metrics?format=prometheus` | No |
| Grafana / Alertmanager | `/metrics` | No |
| JupyterHub | `/hub/metrics` | No |
| MLflow | `--expose-prometheus` | No |
| ZenML | Uncertain | Uncertain |
| KServe | Native Prometheus metrics | No |
| Feast | Feature server metrics | No (verify) |
| TrustyAI | Native Prometheus metrics (fairness/drift) | No (verify) |
| OpenSearch, OpenFGA, Valkey, Traefik, Longhorn, cert-manager | Per prerequisite | Confirm in Phase 2 |
| FastAPI | Application instrumentation | Custom code |

---

## 6. Evidence Links for Closed Items

- VR-003: https://docs.open-metadata.org/v2.0.x/deployment/minimum-requirements
- VR-009: https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html
- VR-025: https://superset.apache.org/admin-docs/configuration/async-queries-celery · https://superset.apache.org/admin-docs/6.1.0/configuration/cache · https://apache.github.io/superset-kubernetes-operator/user-guide/configuration/
- VR-026: https://docs.lakekeeper.io/docs/latest/authorization/ · https://docs.lakekeeper.io/docs/nightly/docs/authorization-openfga/
- VR-001, VR-008, VR-012, VR-020, VR-035: see ADR-012, ADR-018, ADR-020, ADR-024, ADR-019 in file 06.
