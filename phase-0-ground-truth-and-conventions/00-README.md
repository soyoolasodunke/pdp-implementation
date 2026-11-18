# Phase 0: Ground Truth and Conventions

**Document set:** Data and AI Platform: Component Architecture, Integration and Developer Engineering Guide  
**Phase:** 0 of 6 (Foundations)  
**Version:** 1.0  
**Date:** 2026-11-17  
**Prepared for:** BDP Big Data & AI Stack project

---

## 1. Purpose

Phase 0 sets the ground truth that every later phase builds on: reference versions, naming and configuration conventions, the validation mechanism for unverified claims, documentation templates, the verified model-gate capability set, and the architecture decisions that shape the reference platform.

| # | File | What it fixes | Status |
|---|------|---------------|--------|
| 00 | `00-README.md` (this file) | Scope, assumptions, review outcomes, approval record | ✅ Approved v1.0 |
| 01 | `01-version-and-compatibility-baseline.md` | Reference versions, runtime baselines, platform prerequisites, compatibility constraints | ✅ Approved v1.0 |
| 02 | `02-platform-conventions.md` | Naming, namespaces, endpoints/ports, identity, secrets, storage, topics, tables, storage classes, registry | ✅ Approved v1.0 |
| 03 | `03-validation-register.md` | Claims requiring validation, with target phase and status | ✅ Approved v1.0 (living) |
| 04 | `04-documentation-templates.md` | Component, integration, workflow templates; claim tags; diagram legend | ✅ Approved v1.0 |
| 05 | `05-model-gate-capability-record.md` | Documented capabilities and limits of `bdp-model-gate` | ✅ Approved v1.0 |
| 06 | `06-architecture-decision-log.md` | ADR-001 … ADR-025 | ✅ Approved v1.0 (all Accepted) |

---

## 2. Key Findings: Review Outcomes

| # | Finding | Review direction | Resolution |
|---|---------|------------------|------------|
| F1 | MinIO CE unmaintained upstream | Accept a frozen build; team continues to self-build | **ADR-012.** Frozen last CE build; in-house builds with CVE monitoring, SBOM, signed images in Harbor. VR-001 closed. |
| F2 | model-gate has no documented MLflow/ZenML/KServe integration | Do as recommended | All model-gate ↔ MLflow/ZenML/KServe interactions documented as `[Custom]` platform glue (file 05 §10, ADR-017). |
| F3 | Licence: MIT vs "proprietary" | Reconcile with the owner's MIT licence | **ADR-024.** Documented as MIT. VR-020 closed. |
| F4 | ZenML Model Deployer deprecated | Use best practices | **ADR-020.** ZenML Kubernetes orchestrator for pipelines; a platform deployment step applies the KServe `InferenceService` after the gate; ZenML Kubernetes Deployer optional for multi-step inference services. VR-012 closed. |
| F5 | TrustyAI documentation centred on OpenShift | Use best practices for RKE2 | **ADR-021.** Upstream TrustyAI operator; one `TrustyAIService` per serving namespace; PVC mode on Longhorn; KServe Standard-mode payload integration validated in a lab (VR-013). |
| F6 | Runtime dependencies outside the baseline | Include the needed runtime dependencies | **ADR-015.** Named platform prerequisites: RKE2 Traefik + Gateway API, cert-manager, Longhorn, Harbor, OpenSearch ≥ 3.4.0, OpenFGA ≥ 1.11, (conditional) OPA bridge, Valkey, PgBouncer, JMX Exporter, postgres_exporter, Alertmanager, forward-auth middleware. Supported by ADR-022 (Traefik/Gateway API) and ADR-023 (storage classes). |
| F7 | Airflow metadata DB and PostgreSQL 18 | PostgreSQL 18 validated by Airflow | **Confirmed.** Airflow's set-up-database guide lists PostgreSQL 14–18. Both PG clusters run 18 (ADR-013). VR-009 closed; per-consumer PG 18 check for other components is VR-032. |
| F8 | GX Cloud discontinued | GX Core only | **ADR-025.** No GX Cloud references anywhere in the guide. |
| F9 | ClickHouse `DataLakeCatalog` experimental | MergeTree materialisation is the only production-safe option | **ADR-018.** Production serving = Iceberg → ClickHouse `ReplicatedMergeTree` materialisation; `DataLakeCatalog` only for dev/stg exploration. VR-008 closed. |

### New facts found while closing Phase 0

| Fact | Effect |
|------|--------|
| RKE2 v1.36 deploys **Traefik** by default; ingress-nginx is deprecated in 1.36 and retired upstream; in RKE2, Gateway API requires Traefik | ADR-022: Traefik + Gateway API is the north-south standard |
| KServe recommends **Gateway API** over Ingress for Standard mode (`enableGatewayApi`) | ADR-008 updated |
| OpenMetadata 2.0 requires **PostgreSQL ≥ 15** and **OpenSearch ≥ 3.4.0** (or Elasticsearch 9.x); ingestion runs on a version-coupled Airflow | ADR-015, ADR-019 |
| Lakekeeper OSS authorisation uses **OpenFGA ≥ 1.11**; Cedar is listed for Lakekeeper+ | ADR-015, CC-16 |
| Airflow recommends **PgBouncer** for production PostgreSQL | ADR-013, ADR-015 |

---

## 3. Approval Record

| Role | Decision | Date |
|------|----------|------|
| Engagement owner / reviewer (Bluechip Presales – Data & AI) | Approved Phase 0 deliverables v1.0 with the directions recorded in §2 and §3 | 2026-11-17 |
| Author (Claude) | All review directions incorporated; files re-issued as v1.0 | 2026-11-17 |

**Change control from here:** Phase 0 documents are baselined. Later changes are made by a new ADR (file 06) or a validation-register status update (file 03), and are noted in a change log at the end of the affected file.

---

## 4. Conventions Used in This Document Set

- **Claim tags** (file 04): `[Native]`, `[Connector]`, `[Config]`, `[Custom]`, `[Prerequisite]`, `[VR-nnn]`, `[Optional]` / `[Conditional]` / `[Deployment-specific]`.
- **Placeholders:** `bdp.example.internal` (DNS), `<env>` ∈ {`dev`, `stg`, `prd`}, `<domain>` (business domain).
- **Version statements** cite file 01; re-verify exact patches when pinning manifests.
