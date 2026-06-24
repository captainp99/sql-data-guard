# sql-data-guard — Safety Layer for LLM ↔ Database Interactions

> **Hackathon Jury Brief** · Presentation-ready technical & business analysis
> Prepared: 24 June 2026

---

## 1. Project Overview

**sql-data-guard** is an open-source safety layer that sits between an application (especially an **LLM-powered** one) and a database. It inspects every SQL query *before execution*, decides whether the query is allowed to run against a declarative **restriction policy**, and — when possible — **automatically rewrites the query** so it becomes compliant rather than simply rejecting it.

It is delivered in four complementary forms so it can drop into almost any stack:

| Integration | What it is | Use case |
|---|---|---|
| **Python library** (`pip install sql-data-guard`) | `verify_sql(sql, config, dialect)` | In-process guard inside your app |
| **REST API** (Flask + Swagger UI) | `POST /verify-sql` | Language-agnostic microservice |
| **MCP Wrapper** (Docker) | Transparent proxy around an MCP database server | Guards AI-agent tool calls |
| **Dify Plugin** | Tool node for the Dify LLM-orchestration platform | No-code/low-code AI workflows |

**Core engine:** Built on [`sqlglot`](https://github.com/tobymao/sqlglot), it parses SQL into an Abstract Syntax Tree (AST), reasons about it structurally (not via regex/string matching), and emits a structured verdict:

```json
{ "allowed": false, "errors": [...], "fixed": "SELECT id, account_id FROM orders WHERE account_id = 123", "risk": 0.3 }
```

---

## 2. Problem Statement

SQL injection has been a top-tier vulnerability for two decades — and the rise of **natural-language-to-SQL** via LLMs has made it *worse*, not better:

- **Prepared statements don't help.** They secure a query's *structure*, but LLM-generated SQL is dynamic — there is no fixed parameterized form to protect.
- **LLMs can be manipulated** (prompt injection) into generating queries that read columns, rows, or tables they should never touch.
- **Database permissions aren't expressive enough.** Fine-grained **row-level** (multi-tenant isolation) and **column-level** access control, driven by business logic, often cannot be expressed in the native DB permission model.
- **Regulatory exposure is real.** GDPR / CCPA penalize unauthorized data exposure; OWASP explicitly flags poor LLM-to-DB sandboxing.

**The gap:** there is no lightweight, vendor-neutral layer that *understands* a query's intent and enforces fine-grained data-access policy at runtime — for both human- and AI-generated SQL.

**sql-data-guard fills that gap.**

---

## 3. Current Architecture

```
                          ┌──────────────────────────────────────────────┐
   App / LLM Agent ──SQL──►│              sql-data-guard core              │
                          │  verify_sql(sql, config, dialect)             │
                          │                                                │
                          │   1. Config validation (validate_restrictions)│
                          │   2. Parse → AST            (sqlglot)          │
                          │   3. Statement gate (block DDL/DML)           │
                          │   4. FROM/JOIN/CTE table allow-list           │
                          │   5. SELECT column allow-list                 │
                          │   6. WHERE static-expression detection (1=1)  │
                          │   7. Restriction verification + auto-fix      │
                          │   8. Risk scoring                             │
                          └───────────────────┬────────────────────────────┘
                                              │ {allowed, errors, fixed, risk}
                          ┌───────────────────┴────────────────────────────┐
   Distribution layer:   │  Library  │  REST/Swagger │  MCP Wrapper │ Dify  │
                          └─────────────────────────────────────────────────┘
```

**Key modules** (`src/sql_data_guard/`):

| Module | Responsibility |
|---|---|
| `sql_data_guard.py` | Orchestrator; statement gating, SELECT/FROM verification, fix assembly |
| `restriction_validation.py` | Validates the *policy config* (operators, value types) |
| `restriction_verification.py` | Checks the *query* against restrictions; injects missing `WHERE` conditions |
| `verification_context.py` | Stateful container — errors, risk list, dynamic (CTE/subquery) tables |
| `verification_utils.py` | AST traversal helpers (`split_to_expressions`, `find_direct`) |

**Design strengths in the architecture:**
- **AST-based, not regex** — resistant to obfuscation that defeats string matching.
- **Allow-list by default** — tables and columns must be explicitly permitted.
- **Defense in depth** — it *complements* (never replaces) DB permissions.
- **Recursive** — handles CTEs, subqueries, UNION, JOINs, lateral joins, UNNEST.

---

## 4. Key Features

1. **Allow-list enforcement** for tables and columns; everything else is stripped or rejected.
2. **Row-level security via restrictions** — e.g. force `account_id = 123` for multi-tenant isolation.
3. **Automatic query rewriting (`fixed`)** — adds missing restrictions, expands `SELECT *` to allowed columns, removes disallowed columns, neutralizes always-true expressions.
3a. **Column-level masking / redaction** *(implemented)* — sensitive columns are rewritten in-place to `redact` (`'****'`), `hash` (`MD5`), or `partial` (`****1234`) instead of being dropped. The result keeps the same column name and shape, so the application keeps working while raw PII never leaves the boundary. Mask expressions are generated per-dialect and execute natively.
4. **Statement gating** — blocks `DELETE`, `INSERT`, `UPDATE`, `CREATE`, and raw commands (read-only posture).
5. **Static / always-true injection detection** — `WHERE 1=1`, `OR 'a'='a'` rewritten to `FALSE`.
6. **Risk score (0.0–1.0)** — quantitative signal for logging, alerting, or step-up controls.
7. **Multi-dialect parsing** via sqlglot (SQLite, Postgres, MySQL, BigQuery, Snowflake, Trino, DuckDB, Athena… 30+).
8. **Rich restriction operators** — `=`, `>`, `<`, `>=`, `<=`, `BETWEEN`, `IN`.
9. **Four deployment surfaces** — library, REST, MCP wrapper, Dify plugin.
10. **Interactive Swagger UI** for live exploration and testing.

---

## 5. API Analysis

### `POST /verify-sql` (Flask + flasgger)

**Request:**
```json
{
  "sql": "SELECT * FROM orders WHERE account_id = 123",
  "config": {
    "tables": [{
      "table_name": "orders",
      "columns": ["id", "product_name", "account_id"],
      "restrictions": [{ "column": "account_id", "value": 123 }]
    }]
  },
  "dialect": "sqlite"
}
```

**Response:**
```json
{ "allowed": false, "errors": ["SELECT * is not allowed"],
  "fixed": "SELECT id, product_name, account_id FROM orders WHERE account_id = 123",
  "risk": 0.1 }
```

| Aspect | Assessment |
|---|---|
| **Endpoint design** | ✅ Clean, single-purpose, JSON in/out |
| **Documentation** | ✅ Swagger/OpenAPI auto-generated, with examples |
| **Input validation** | ✅ Checks JSON, presence of `sql` and `config`; ⚠️ no schema enforcement on `config` |
| **Error handling** | ✅ Clear 400s; ✅ verdicts returned as 200 with structured body |
| **Statelessness** | ✅ Fully stateless — horizontally scalable |
| **Security posture** | ⚠️ **No authentication, no rate limiting, no audit logging** |
| **Observability** | ⚠️ Logs to stderr only; no metrics/tracing endpoint |

**Verdict:** The API surface is elegant and demo-friendly, but it is currently an *unauthenticated, unthrottled* service — acceptable for a sidecar behind a trusted network boundary, but a production gap (see §7).

---

## 6. Strengths

- 🛡️ **Genuine, timely problem** — LLM-to-SQL safety is a live, under-served pain point.
- 🌳 **AST-based engine** — structurally sound; far stronger than regex/keyword filtering.
- 🔧 **Auto-fix is a differentiator** — most tools only *block*; this one *repairs* and keeps the app working.
- 🔌 **Four integration surfaces** — meets developers where they are (code, HTTP, MCP, Dify).
- 🧪 **Strong test suite** — ~3,200 lines, 150+ tests, 145 data-driven JSONL cases, **adversarial LLM tests** simulating prompt injection via real Claude models (Bedrock).
- 🚀 **Mature release automation** — tag → TestPyPI → PyPI → multi-arch GHCR Docker → Dify, with provenance attestation.
- 🐍 **Broad compatibility** — Python 3.8–3.12 verified in CI.
- 📦 **Minimal footprint** — single core dependency (`sqlglot`).

---

## 7. Areas for Improvement

Findings from a full source review, grouped by severity.

### 🔴 Security-relevant
1. **Config-value injection in rewriting** — `_format_value()` adds quotes for strings but does **not escape embedded single quotes**. If policy config is ever user-influenced, the *fixed* query can itself carry injection. **Fix: parameterize or escape rigorously.**
2. **`assert len(values) == 1` in production path** — assertions are stripped under `python -O`, turning a guard into a silent no-op. **Fix: raise `ValueError`.**
3. **No authn/authz, rate limiting, or audit trail** on the REST API or MCP wrapper.

### 🟠 Correctness / robustness
4. **String comparison for numeric restrictions** — `str(exp.right.this) < values[0]` compares lexicographically (`"9" < "10"` is false). **Fix: type-aware comparison.**
5. **`IN` validation requires exactly 2 values** — defeats the purpose of `IN`; contradicts the manual (which says "multiple values"). **Fix: allow arbitrary length.**
6. **Risk score is a plain average** — one critical 0.9 error diluted by minor 0.1 errors can read as low risk. **Fix: use max or weighted aggregation.**
7. **Missing operators** — no `LIKE`, `IS NULL`, `NOT IN`, set membership beyond `IN`; `NOT` expressions are rejected wholesale.
8. **No subquery/CTE depth limit** — pathological nesting is a DoS vector.

### 🟡 Coverage / quality
9. **Dialect breadth untested** — only SQLite, DuckDB, Trino exercised; Postgres/MySQL/BigQuery/Snowflake unverified despite being advertised.
10. **No Docker / MCP-wrapper integration tests**; REST tested with only ~6 cases.
11. **Example config bug** — Postgres example sets `"dialect": "sqlite"`.
12. **No performance / fuzz / property-based testing.**

---

## 8. Proposed Enhancements

### Feature enhancements
- **Policy-as-code with presets & inheritance** — named, reusable policies (per-role, per-tenant) instead of inlining JSON at every call site.
- **Richer restriction grammar** — `LIKE`, `IS [NOT] NULL`, `NOT IN`, regex match, and **dynamic values** (e.g. `account_id = :current_user_account`) resolved from request context.
- ✅ **Column-level masking / redaction** — **DELIVERED** in this iteration (see §4 feature 3a and the Demo Flow). Columns are masked rather than dropped via `redact` / `hash` / `partial` policies.
- **Explain mode** — return a human-readable rationale for each decision (great for audits and developer trust).

### Performance optimization
- **Verification result cache** (LRU keyed on `hash(sql + config)`) — repeated agent queries verify instantly.
- **AST parse-cache** and **complexity/depth limits** to bound worst-case cost and block DoS-shaped queries.
- **Async REST server** (e.g. FastAPI/ASGI) for higher concurrency than the current Flask dev server.

### Security improvements
- **Auth + rate limiting + structured audit log** on REST/MCP (API keys/JWT, per-tenant quotas, tamper-evident decision log).
- **Parameterized fix output** — emit `fixed_sql` + bound params, not string-interpolated literals.
- **Harden the assertion/escaping issues** in §7 (#1, #2, #4).

### Scalability & maintainability
- **Centralized policy service** with versioning and a management API/UI.
- **Property-based & fuzz testing** (Hypothesis) plus a multi-dialect conformance matrix in CI.
- **OpenTelemetry** traces/metrics; Prometheus `/metrics` endpoint.

### AI / automation opportunities
- **AI-assisted policy generation** — point it at a schema (or sample queries) and have an LLM *propose* a least-privilege restriction config, which a human approves.
- **Anomaly detection on risk scores** — learn each tenant's normal query shape and flag drift.
- **Self-expanding adversarial test corpus** — an agent loop that continually generates new injection/jailbreak attempts and adds the survivors to the regression suite.
- **Natural-language policy authoring** — "analysts can only see their region's orders, no PII" → generated config.

---

## 9. Technical Roadmap

| Horizon | Theme | Deliverables |
|---|---|---|
| **Now → 1 mo (Harden)** | Close correctness/security gaps | Fix `assert`→`ValueError`, value escaping, numeric comparison, `IN` length; add API key + rate limit + audit log |
| **1–3 mo (Scale)** | Production readiness | Result/parse caching, depth limits, ASGI server, OpenTelemetry, multi-dialect CI conformance matrix |
| **3–6 mo (Govern)** | Policy platform | Centralized versioned policy service + management UI; column masking; dynamic context-bound restrictions; explain mode |
| **6 mo+ (Intelligence)** | AI layer | AI policy generation from schema, risk-based anomaly detection, self-growing adversarial test corpus |

---

## 10. Innovation Opportunities

- **"Guardrail for the AI-data stack."** As MCP and agentic tooling proliferate, a transparent, drop-in SQL guard for *every* agent tool call is a category-defining position. The existing MCP wrapper is the seed of this.
- **Auto-fix as the killer feature.** Blocking is commodity; *repairing* a query so the application keeps working — while silently enforcing tenant isolation — is rare and demo-spectacular.
- **Compliance-as-output.** Emit a signed, structured decision record per query → instant GDPR/CCPA audit evidence.
- **Closed-loop AI safety.** LLM generates SQL → guard verifies/fixes → guard's findings feed back as adversarial training/test data. A system that *gets safer the more it is attacked.*

---

## 11. Demo Flow (≈5 minutes)

1. **The hook (30s).** "We let an LLM write SQL against our orders DB — watch it try to break out."
2. **Open Swagger UI** at `http://localhost:5000/apidocs` → expand `POST /verify-sql`.
3. **Scenario A — Tenant breakout.** Submit `SELECT id FROM orders WHERE account_id = 456` (wrong tenant) → response shows `allowed: false` and a **`fixed`** query that re-pins `account_id = 123`. *"It didn't just block it — it corrected it."*
4. **Scenario B — Classic injection.** Submit `SELECT id FROM orders WHERE account_id = 123 OR 1=1` → always-true expression detected and neutralized to `FALSE`.
5. **Scenario C — Over-broad read.** Submit `SELECT * FROM orders WHERE account_id = 123` → `SELECT *` rejected and expanded to the explicit allow-listed columns.
6. **Scenario D — Destructive intent.** Submit `DELETE FROM orders` → blocked outright (read-only gate), `risk: 0.9`.
7. **Scenario E — PII masking (the "wow").** With `column_masks` configured, submit `SELECT id, credit_card FROM users` → the `fixed` query returns `SELECT id, '****' || SUBSTRING(CAST(credit_card AS VARCHAR), -4) AS credit_card FROM users`. Run both: the raw query shows `4111111111111234`; the guarded query shows `****1234`. *"Same column, same app — the PII simply never leaves the database boundary. That's GDPR data-minimization, automatically."*
8. **The agentic finale.** Show the **MCP wrapper** transparently guarding a live AI agent's tool call — the agent never sees the forbidden rows.
8. **Close on the risk score & roadmap** — "Every decision is scored, auditable, and the policy can be AI-generated."

---

## 12. Conclusion

**sql-data-guard targets one of the most pressing, least-solved problems in the AI era: safely connecting LLMs to real databases.** Its AST-based engine, allow-list-by-default posture, and standout **auto-fix** capability already make it more than a prototype — backed by a 150+-test suite (including adversarial LLM testing) and a mature, multi-channel release pipeline.

The path forward is clear and achievable: **harden** the handful of correctness/security gaps, **scale** with caching and observability, **govern** with a centralized policy platform, and **differentiate** with an AI layer that authors policies and learns from attacks. Positioned at the center of the emerging MCP/agentic-AI stack, sql-data-guard has a credible path from a strong open-source library to **the default guardrail for AI-to-database access.**

> **One-line pitch:** *Prepared statements protect a query's shape — sql-data-guard protects what the query is allowed to see, even when an AI wrote it.*

---

### Appendix — Project Facts at a Glance
- **Language / core dep:** Python 3.8–3.12, `sqlglot` (only runtime dependency)
- **Parsing:** AST-based, 30+ SQL dialects
- **Tests:** ~3,200 LOC, 150+ tests, 145 JSONL data cases, LLM adversarial suite (Claude via Bedrock)
- **Distribution:** PyPI · GHCR Docker (REST + MCP, multi-arch) · Dify plugin
- **License:** MIT
- **API:** `POST /verify-sql` with interactive Swagger UI at `/apidocs`
