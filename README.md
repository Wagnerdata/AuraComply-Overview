# AuraComply

**Tamper-evident audit evidence for AI decisions and operational events (EU AI Act · DORA · GDPR).**

> Public overview. **Source code is private; access available for technical review on request.**

AuraComply is a narrow audit kernel: an API and a Python SDK that record AI decisions, human-oversight actions and incident-lifecycle events in a per-tenant, SHA-256-chained ledger, and that let the customer check that recorded events were not altered afterwards.

It **produces evidence** that supports an organisation's compliance work. It does **not** make an organisation compliant, it is not a certified product, and its self-check report is not a third-party certification.

---

## What it is, and what it is not

| AuraComply is | AuraComply is not |
| --- | --- |
| A tamper-**evident** ledger: any edit to a recorded event breaks the hash chain on recomputation. | Tamper-**proof** or "immutable" storage: a database operator can still change rows; the point is that the change becomes detectable. |
| A record of the decision metadata *your* system supplies (model identity, input hashes, outcome, a screened `rationale_summary`, human-oversight status). | An explainability engine: it does not generate explanations, it seals the ones you provide. |
| Hashes and references by design; structural PII (email, IBAN, card numbers, SSN, Spanish NIF/NIE, E.164 phones) is rejected at the API before anything is written. | A PII detector for free text: semantic PII (names, addresses) is the calling system's responsibility. |
| Resilience metrics computed only from recorded events; `null` when data is insufficient, never an invented `0.0`. | A SIEM, a log aggregator or a dashboard. |

---

## How it works

```
event_hash = SHA-256( canonical(payload) ‖ prev_hash )
```

- **`POST /audit/log`**, **`POST /audit/log/batch`** (up to 100 events, one transaction): validate, seal, write a JSONL mirror to disk *before* the database insert, return the `event_hash`. An optional `idempotency_key` makes retries safe.
- **`GET /verify`**: recomputes the tenant's chain from genesis and reports the first mismatch.
- **`POST /verify_receipts`**: returns the stored `event_hash` for given `log_id`s, so the client can compare them with the receipts it kept (see below).
- **`GET /metrics/resilience`**: DORA incident-pipeline metrics (detection → classification → report latency, RTO, worst RPO, resilience-test pass rate). **`GET /metrics/delivery`**: DevOps Four Keys.
- **Python SDK** (`auracomply-sdk`, dependencies `httpx` + `pydantic` only): buffers events in an HMAC-protected local file when the API is unreachable, and keeps a local receipt (`sealed.jsonl`) of every returned `event_hash`.
- **Tenant isolation**: tenant identity comes only from the bearer token; PostgreSQL row-level security on every audit table. Tokens are stored as SHA-256 only.
- **MCP server (proof of concept, stdio transport)**: exposes sealing, delivery metrics and chain verification as Model Context Protocol tools, so an AI agent can seal its own decisions with the same bearer-token scoping.

The chain hash is specified in a normative document with test vectors, so an external auditor can reimplement the verifier in any language without the source.

Example request:

```http
POST /audit/log
Authorization: Bearer <token>
Content-Type: application/json

{
  "product": "credit_decision_model",
  "step": "deployment",
  "payload": {"model_version": "v3.2.1", "approved_by": "risk_committee"}
}
```

Response:

```json
{"status": "success", "log_id": "9a2c8e34-…", "event_hash": "8f3a2bd9e7c4…"}
```

---

## Verification model and known limits

| Threat | Detected? | How |
| --- | --- | --- |
| A recorded event is edited or deleted in the database | Yes | `/verify` (or the offline verifier) recomputes the chain and reports the first mismatch. |
| Concurrent writers fork the chain | Prevented | A per-tenant lock is taken before reading the chain head; batches chain in memory under one lock. |
| The client's local receipt file is tampered with | Yes | Every receipt line carries an HMAC under the customer's own key. |
| A privileged operator rewrites the chain **directly in the database** and recomputes every later hash | Yes, when the client runs `verify_receipts()` | `/verify` still passes (the rewritten chain is internally consistent), but the stored hashes no longer match the receipts the client kept. |
| An operator who **also controls the API** answers `/verify_receipts` with the original hashes | **Not yet** | The receipt check trusts the server's answer. Planned: Ed25519-signed chain-head checkpoints delivered to the tenant, optionally with an RFC 3161 timestamp. |

The chain is not "signed" today: integrity rests on the hash chain plus the receipts the customer holds. Signatures (cosign) apply to the released container image, not to audit events.

---

## Sample self-check report

The kernel ships an internal **integrity self-check** that runs automated checks against a deployment and writes a PDF sealed with the SHA-256 of its JSONL mirror. Each check is PASS, FAIL or NOT_RUN; the verdict is PASS only if every check passes, INCOMPLETE if any could not run. It is a consistency check of the deployment, **not a certification**.

📥 [AuraComply_Compliance_Report_Sample.pdf](docs/AuraComply_Compliance_Report_Sample.pdf)

```text
Integrity Self-Check Report // AuraComply
Product ID: system_audit | Client: AuraComply_HQ
Generated: 2026-10-16T23:06:15+00:00
Kernel: dev (unreleased) | Payload schema v1.4.0

Internal self-check. Not a third-party certification.

1. Operational integrity seal
   SHA-256 (Traceability Mirror):
   babaa5b75ea58285bbc1ed71af57be2068456dbad710f0aa515a620750706449

2. Check log
   check_integrity_self        PASS: hash[:16]=9edcb150642839bc (deterministic, data-sensitive)
   check_db_connectivity       PASS: SELECT 1 OK
   check_clients_table         PASS: all required columns present (6 total)
   check_audit_logs_schema     PASS: all required columns present
   check_rls_enabled           PASS: RLS enabled on audit_logs
   check_no_plaintext_tokens   PASS: all api_token_hash entries are 64-char hex
   certify_verdict             PASS

3. Privacy controls (GDPR Art. 32(1)(b))
   Structural PII rejected at the API boundary; semantic PII is the calling
   system's responsibility. The chain shows recorded events were not altered;
   it does not detect semantic PII.
```

The sample was generated on 2026-10-16 from a local development build, before the receipt cross-check shipped; its "known limitation" line about operator rewrites predates that change (see the table above for the current state).

---

## Regulatory context

AuraComply produces evidence relevant to these obligations; whether an organisation meets them is for that organisation and its auditors to assess.

| Framework | Where the evidence helps |
| --- | --- |
| **EU AI Act** | Art. 12 (automatic recording of events for high-risk systems) and Art. 14 (human oversight): decision and oversight events sealed with model identity and input hashes. |
| **DORA** (Reg. (EU) 2022/2554) | Art. 9 and Art. 12, detailed by Delegated Reg. (EU) 2024/1774 (integrity of data, logging, restoration); incident-lifecycle events for classification (Art. 18) and reporting (Art. 19); resilience-testing records. |
| **GDPR** | Art. 32(1)(b): evidence that audit records were not altered; no personal data in the chain by design. |

**Timeline.** EU AI Act high-risk obligations, as amended by the Digital Omnibus on AI (Regulation (EU) 2026/1744, OJ 24 July 2026, in force since 27 July 2026), apply from 2 December 2027 (Annex III systems, e.g. credit scoring, employment, critical infrastructure) and 2 August 2028 (Annex I, AI in regulated products). DORA has applied since 17 January 2025.

**Roadmap:** optional regulatory tags per event (e.g. `aiact:art14`, `nis2:art23`) and per-framework metric profiles, starting with NIS2 and the electricity Network Code on Cybersecurity for the energy sector. Planned, not shipped.

---

## Engineering governance log

Findings from the review of 2026-10-18, each fixed with a regression test that fails on the old code. Recorded here because how defects are found and closed is part of the evidence a governance tool should offer.

| # | Finding | Impact | Fix | Status |
| --- | --- | --- | --- | --- |
| 1 | The JSONL mirror was written to a path relative to the working directory, which the server changes at start-up. | The mirror landed in the container's ephemeral layer, not on the persistent volume: the "mirror before database" resilience guarantee was lost on every container recreation. The unit tests could not see it. | Absolute, configurable mirror root; a test pins the image setting to the volume mount. | Merged |
| 2 | Receipt lookup compared a `uuid` column with a `text[]` parameter. | **Every `/verify_receipts` call failed against a real PostgreSQL**; unit tests mock the database, and the branch had never run in CI. | Explicit `uuid[]` cast; found on the first real-database integration run. | Merged |
| 3 | No reverse-proxy route for `/verify_receipts`. | The endpoint returned 404 behind the proxy. | Own route plus a routing test. | Merged |
| 4 | The API contract referenced an error schema that did not exist. | The contract could not be used by validators or code generators. | Fixed reference plus a test that resolves every reference in the contract. | Merged |
| 5 | Two branches recorded different decisions under the same number. | Git merged them without a conflict: a silent duplicate in the decision log. | Renumbered before merge. | Merged |
| 6 | The SDK's `close()` could return while its background thread was still retrying, then close the HTTP client under it. | `close()` did not reliably finish its work; intermittent test failures. | Retry back-off interruptible on close, single final attempt; unsent events stay buffered and are resent later with the same idempotency key. SDK 0.2.0. | Merged |
| 7 | The release workflow built, signed and published any version tag, without running tests. | An untested image could be published, and its signature cannot be withdrawn from the public transparency log. | Release gate: only commits on `main`, only after the full test suites pass in the same run. | Merged |
| 8 | Tests that did not run in CI: the API-contract tests could skip silently, the SDK suite and its 95 % coverage gate never ran, and branches without a pull request got no CI. | Green CI that did not check what it appeared to check. | Contract-test dependency made mandatory; SDK job with coverage gate; CI on every pushed branch. | Merged |
| 9 | A newer database library changed its default driver. | Fresh container builds failed at start-up. | Dependency pinned; dependency lock file planned. | Merged |

---

## Status

Runs locally with Docker Compose and is deployable to a single VPS; not currently offered as a hosted service. Source code is private; access available for technical review on request.

Developed with AI coding assistance (Claude Code); every change reviewed and approved by the author.

## Contact

| | |
| --- | --- |
| 📩 Email | waglex@gmail.com |
| 💼 LinkedIn | [linkedin.com/in/wagnercamposdata](https://www.linkedin.com/in/wagnercamposdata/) |
