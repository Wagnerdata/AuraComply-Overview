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

AuraComply produces evidence relevant to these obligations; whether an organisation meets them is for that organisation and its auditors to assess. Every citation was checked against the official text on EUR-Lex.

| Framework | Where the evidence helps |
| --- | --- |
| **EU AI Act** | Art. 12 (automatic recording of events for high-risk systems) and Art. 14 (human oversight): decision and oversight events sealed with model identity and input hashes. Art. 19(1) and 26(6) (providers and deployers keep these logs for at least six months): append-only storage; the retention period itself is the operator's policy. |
| **DORA** (Reg. (EU) 2022/2554) | Art. 9(4)(a) and Art. 12, detailed by Delegated Reg. (EU) 2024/1774 Art. 12 (integrity of data, logs protected against tampering and deletion, restoration); incident-lifecycle events for management (Art. 17), classification (Art. 18) and reporting (Art. 19); resilience-testing records (Art. 24–27). |
| **GDPR** | Art. 32(1)(b): evidence that audit records were not altered; no personal data in the chain by design. |

**Timeline.** EU AI Act high-risk obligations, as amended by the Digital Omnibus on AI (Regulation (EU) 2026/1744, OJ 24 July 2026, in force since 27 July 2026), apply from 2 December 2027 (Annex III systems, e.g. credit scoring, employment, critical infrastructure) and 2 August 2028 (Annex I, AI in regulated products). DORA has applied since 17 January 2025.

**Roadmap:** optional regulatory tags per event (e.g. `aiact:art14`, `nis2:art23`) and per-framework metric profiles, starting with NIS2 and the electricity Network Code on Cybersecurity for the energy sector. Planned, not shipped.

---

## Engineering governance log

Findings from the review of 2026-10-18, each fixed with a regression test that fails on the old code (row 11: a test that fails on the old text). Recorded here because how defects are found and closed is part of the evidence a governance tool should offer.

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
| 10 | The web container received the PostgreSQL superuser password, used only by the two verification endpoints. | Anyone able to inspect the container could read a credential that bypasses tenant isolation and can alter any record. | Verification runs under the application role with row-level security, like every other endpoint; a real-database test proves one tenant cannot see or verify another's rows, and that the role cannot update or delete audit records. The password is rotated on the next deployment. | Merged |
| 11 | The documentation cited DORA Art. 9(4)(c) as the audit-trail requirement and AI Act Art. 13 for traceability. | Art. 9(4)(c) is about access control, and traceability is Art. 12: wrong citations in a governance tool undermine the evidence. | Citations checked against EUR-Lex in one regulatory map; a test blocks the known miscitations. | Merged |
| 12 | Idempotency keys never expired, and a key reused for a different event silently discarded the new one; replays returned no log_id. | Evidence could be lost without any error, and receipts of resent events could not be verified. | Seven-day window enforced in the lookup itself, a request fingerprint that answers 409 on reuse, and the original log_id on replay; SDK 0.3.0 reports conflicts instead of retrying. Fixed in v0.3.0. | Merged |
| 13 | In the SDK's buffered mode, a single event the server rejected as invalid kept its whole batch pending and was resent forever; a burst of rejected events could also trip the circuit breaker and hold back valid ones. | Evidence queued behind one bad event stopped flowing, with nothing but a repeated warning to show it. | Rejected events are set aside in a signed local file and reported through the SDK's API, never retried and never logged by value; oversized batches are split; rejections no longer count against the breaker. Because that file can hold personal data the server refused, the SDK documents how the data controller erases one entry (GDPR Art. 17). Fixed in SDK 0.5.0. | Merged |

---

## Status

Runs locally with Docker Compose and is deployable to a single VPS; not currently offered as a hosted service. Source code is private; access available for technical review on request.

Developed with AI coding assistance (Claude Code); every change reviewed and approved by the author.

## Contact

| | |
| --- | --- |
| 📩 Email | waglex@gmail.com |
| 💼 LinkedIn | [linkedin.com/in/wagnercamposdata](https://www.linkedin.com/in/wagnercamposdata/) |
