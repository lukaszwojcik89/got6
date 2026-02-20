# got6

## No-Second-Chances LLM Platform Blueprint

This repository now documents a production architecture for an LLM platform that is **strict by default**: it fails closed on uncertainty, minimizes data retention, and enforces policy, evidence, and verification at every stage.

## System Goal

Build an LLM platform that:

- Fails closed on uncertainty, missing evidence, policy conflicts, and schema violations.
- Never trains on customer data by default.
- Minimizes retention, redacts logs, encrypts all sensitive data, and isolates tenants.
- Uses tools and citations for factuality and recency.
- Has measurable reliability, security, and privacy guarantees through evals and audits.

## 1) High-Level Architecture

Treat the system as a **pipeline**, not a single chatbot process.

1. **Gateway (API edge)**
   - AuthN/Z (OIDC/SAML), RBAC/ABAC, tenant resolution.
   - Request shaping (max tokens, allowed tools/sources).
   - Rate limiting, abuse controls, anomaly detection.
   - Pre-check content policy enforcement.

2. **Policy Engine (central decision point)**
   - Deterministic rules for allowed answers, refusals, required citations/tool verification.
   - Per-tenant constraints: retention, residency, model/tool/connector allowlists.
   - Produces a machine-readable decision contract for downstream components.

3. **Orchestrator (execution controller)**
   - Runs retrieve -> generate -> verify -> redact -> deliver.
   - Enforces bounded retries, timeouts, circuit breakers, and kill switches.
   - Prevents direct model access to internet/databases (tool-mediated only).

4. **Retrieval Layer (RAG with enforcement)**
   - Ingestion with ACLs, chunking, embeddings, provenance metadata.
   - Query-time checks for tenant isolation + user permissions + need-to-know scope.
   - Returns evidence bundles with source IDs.
   - Treats retrieved text as untrusted input (prompt injection defense).

5. **Tooling Layer (controlled actions)**
   - Supports search, internal KB, read-only DB access, ticketing, calendar/email, code execution.
   - Applies sandboxing, egress deny-by-default, request signing, scoped permissions.
   - Produces structured outputs with provenance.

6. **Model Layer (multi-model)**
   - Generator model for candidate outputs under strict schema.
   - Independent verifier model for evidence/policy checks.
   - Optional specialist models (code/math/translation/etc.).
   - Router selects model family by task risk and domain.

7. **Output Guard**
   - Hard schema validation.
   - PII/secret detection and redaction/blocking.
   - Citation coverage enforcement.
   - Post-generation safety checks.

8. **Observability + Audit**
   - Metadata-only operational telemetry where possible.
   - Security logs for policy decisions, data access, and tool execution.
   - Redacted/encrypted short-retention content logs only when explicitly enabled.

## 2) Security Design: "No Data Leaks" (Practical)

- **Data minimization:** no sensitive persistence by default; strict TTL if temporary storage is necessary.
- **Crypto posture:** encryption at rest everywhere; per-tenant keys via KMS/HSM; envelope encryption for content blobs.
- **Tenant isolation:** prefer per-tenant stores/keys/runtime where feasible; enforce row-level + service-layer checks at minimum.
- **Egress controls:** deny-by-default networking; tool allowlists; blocked arbitrary URLs/uploads unless explicitly allowed.
- **Secret handling:** never place secrets in prompts; add scrubbers for key/JWT/private-key patterns.
- **Prompt injection defense:** keep instructions separate from retrieved data; mark retrieved/tool text as untrusted.
- **Access controls:** RBAC/ABAC with break-glass access that is time-bound and fully audited.
- **Tamper-resistant auditing:** append-only logs + alerting on suspicious access patterns.

## 3) Reliability Design: "No Mistakes" (Practical)

- **Fail-closed contract:** no evidence, failed schema, verifier uncertainty, or tool timeout => refusal/safe degradation.
- **Structured outputs:** all model interactions use strict JSON schemas.
- **Two-pass flow:**
  - Generator emits answer, claims, evidence pointers, uncertainty flags.
  - Verifier checks claim-evidence alignment, policy compliance, and citation requirements.
- **Tool-first facts:** current/precise facts must come from tools, not model memory.
- **Deterministic templates:** high-stakes domains use constrained response formats and explicit refusal language.

## 4) Model Strategy

1. **Fastest path:** frontier base model + strong orchestration controls.
2. **Cost path:** fine-tuned small models for narrow tasks + router.
3. **Research path:** train frontier model (highest cost/time/risk).

Core principle: safety comes from orchestration + verification, not blind trust in base model quality.

## 5) Evaluation and Release Gates

- **Test suites:**
  - Unit tests for policy/schema/tool wrappers.
  - Integration tests for end-to-end workflow.
  - Regression golden sets for expected answers/refusals.
  - Adversarial tests for jailbreak/injection/exfiltration.
- **Operational metrics:** refusal rate, verifier-fail rate, citation coverage, tool error/latency, leakage detector hits.
- **Release gates:** eval pass required for prompt/model changes, canary rollouts per tenant, instant rollback support.

## 6) Deployment Topology (Reference)

- Kubernetes (or equivalent), separated environments/namespaces.
- Tooling services isolated in restricted segments.
- Tenant-isolated vector stores (or strict partitioning + enforcement).
- KMS/HSM-backed key management.
- WAF at edge; mTLS internally; workload identity.
- Signed CI/CD artifacts, SBOM generation, and vulnerability scans.

## 7) Operational Safety

- Incident response workflow: detect -> contain -> rotate/revoke -> notify -> postmortem.
- Backup/restore drills run routinely.
- Access and audit review cadence.
- Quarterly red-team exercises.

## 8) Suggested Build Order

### Phase 1: Foundations

1. Gateway + Auth + tenant isolation skeleton.
2. Policy engine v1 (refusal, tool requirements, citation requirements).
3. Orchestrator v1 (retrieve -> generate -> verify -> output guard).

### Phase 2: Retrieval + Tools

1. Ingestion pipeline with ACL metadata + vector storage.
2. Tool layer with sandboxing, allowlists, and audit logs.
3. Prompt injection defenses + verifier enforcement.

### Phase 3: Hardening + Evals

1. Full eval harness + regression/adversarial suites.
2. Canary + rollback + release gates.
3. Monitoring/alerts + incident response rehearsal.

## 9) Definition of Done (Acceptance Criteria)

### Privacy

- Content logging disabled by default.
- If enabled: redacted, encrypted, and TTL-enforced.
- Per-tenant isolation + keys validated by audit.
- Egress deny-by-default with allowlisted tools and auditable calls.

### Reliability

- Schema validation enforced at each stage.
- Verifier blocks unsupported claims.
- Citation-required policies consistently enforced.
- Refusal behavior is deterministic and tested.

### Security

- Prompt injection evals meet agreed thresholds.
- Secret exfiltration attempts are detected and blocked.
- Retrieval ACLs are enforced end-to-end and cannot be bypassed.
