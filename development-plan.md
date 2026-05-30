# Academic Library Management — Phased Development Plan

> Project: 376-academic-library-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files into a concrete, phased implementation spec for an AI-native, open-source library services platform (LSP) for academic and research libraries.

**Data model adopted:** `data-model-suggestion-1.md` (Entity-Centric Normalized Relational). It is the recommended choice for a standards-compliant LSP — its 14-table schema maps directly onto MARC21, NCIP, SIP2, ISO 18626, and COUNTER 5 entities, enforces referential integrity across the `bib_record → item → loan → patron` chain, and produces clean MARC/OAI-PMH exports. The schema in this plan is taken from that file with no re-derivation.

---

## Product Summary

**What it does.** A unified library services platform that covers cataloguing & metadata (MARC21 + BIBFRAME), circulation (loans/holds/renewals/recalls/fines, SIP2 self-check), patron management with federated SSO (SAML/Shibboleth/OpenAthens), acquisitions (EDIFACT vendor orders), electronic resource management (COUNTER 5/SUSHI harvesting, cost-per-use), interlibrary loans (ISO 18626/NCIP), a discovery layer (faceted search + OpenURL), and open-access compliance dashboards — with an AI layer that auto-suggests MARC fields and subject headings, extracts licence terms from publisher contracts, and answers patron reference queries against the local catalogue.

**Who uses it.** Cataloguers, circulation-desk staff, acquisitions/ERM managers, ILL operators, reference librarians, and systems librarians at universities and research institutions; consortia procurement leads; and patrons (via OPAC/discovery). Multi-institution/consortia is a first-class concern (`institution_id` scopes every table).

**Key differentiators (AI-native).** No incumbent (Alma, Sierra, Koha, FOLIO) ships a built-in MCP server exposing catalogue/OAI-PMH/discovery to an LLM agent, AI-assisted cataloguing, contract-intelligence for ERM, or conversational reference with citation-style answers. The platform targets the underserved "lightweight LSP for small/departmental academic libraries" gap with a permissively licensed, easy-to-self-host stack.

**Deployment model.** Self-hosted (Docker Compose) and cloud, single-binary-ish operations for small libraries; horizontally scalable workers for large consortia. REST API first; OPAC/staff web UI; CLI for batch ops; MCP server for AI clients.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The differentiating work is LLM-heavy (MARC suggestion, contract intelligence, reference assistant) and protocol-heavy (MARC/MARCXML, OAI-PMH, SUSHI, ISO 18626). Python has the only mature library-domain ecosystem: `pymarc` (MARC21/MARCXML/ISO 2709), `sickle`/custom OAI-PMH, plus first-class LLM SDKs. |
| API framework | **FastAPI** | Generates OpenAPI 3.1 automatically (a portability/standards requirement from `research.md`), async for slow upstream calls (Z39.50, SUSHI, OCLC, LLM), Pydantic v2 validation maps cleanly to the JSONB-heavy schema. |
| ASGI server | **Uvicorn** behind **Gunicorn** (uvicorn workers) | Standard production FastAPI deployment. |
| Database | **PostgreSQL 16** | The adopted data model uses UUID PKs, `JSONB` + `GIN` indexes (MARC fields, ISBN/ISSN arrays), full-text `tsvector`, `TEXT[]` arrays, and `PARTITION BY RANGE` on the audit log — all PostgreSQL-native. No SQLite fallback: the GIN/array/partition features are load-bearing. |
| Search / discovery | **OpenSearch 2.x** | Discovery layer needs relevance ranking + faceted search across heterogeneous records; Postgres FTS handles staff-side lookups but not faceted discovery at scale. OpenSearch is Apache-2.0 (licence-clean, matches the FOLIO/Invenio precedent). |
| ORM / DB access | **SQLAlchemy 2.0 (async)** + **Alembic** | Mature async ORM; Alembic gives reproducible, reversible migrations — essential for the "10–15 year lifecycle / migration safety" requirement in `research.md`. |
| Task queue | **Celery** + **Redis** broker/result backend | Async workloads: SUSHI harvesting, OAI-PMH harvest, EDIFACT processing, ILL rota progression, overdue/fine accrual, LLM batch jobs, OpenSearch indexing. Redis also serves as cache and SIP2 session store. |
| Scheduler | **Celery Beat** | Periodic jobs: nightly SUSHI harvest, overdue sweep, hold-expiry sweep, licence-renewal alerts, OA-compliance recheck. |
| LLM access | **Provider-abstracted client** (Anthropic Claude default; OpenAI/Azure/local via env) | AI features must be swappable for self-hosted/regulated deployments. A thin `LLMClient` interface isolates the SDK. |
| MCP server | **`mcp` Python SDK** (Model Context Protocol) | `standards.md` flags an MCP server over catalogue/OAI-PMH/discovery as a no-competitor AI-native interface. Shipped as a separate process reusing the service layer. |
| Frontend | **React 18 + TypeScript + Vite**, **TanStack Query**, **shadcn/ui + Tailwind** | Two SPAs: persona-driven **staff console** and patron-facing **discovery/OPAC**. React matches the sector (FOLIO Stripes, Primo, EDS). |
| Auth (staff/patrons) | **SAML 2.0** via `python3-saml`; **OAuth2/OIDC** via `authlib`; local JWT sessions | Federated SSO (Shibboleth/OpenAthens/eduGAIN) is a core academic requirement; OAuth2 client-credentials secures the REST API (matches Koha/Sierra/FOLIO API auth). |
| MARC handling | **pymarc** | De-facto Python MARC21 / MARCXML / ISO 2709 library; round-trips the `marc_fields_json` structure in the data model. |
| Protocol libs | `zeep`/custom (Z39.50 via `PyZ3950` or yaz-client subprocess), `lxml` (MARCXML, OAI-PMH, ISO 18626, EDIFACT-to-XML), `stdnum` (ISBN/ISSN/DOI validation) | Library protocols are XML/structured-text heavy. |
| Validation / types | **Pydantic v2** | Request/response schemas, config models, JSONB sub-document validation. |
| Testing | **pytest** + **pytest-asyncio** + **httpx** (ASGI test client) + **testcontainers** (Postgres/Redis/OpenSearch) + **respx**/`responses` (HTTP mocking) | Integration tests need real Postgres (GIN/arrays/partitions) and OpenSearch; testcontainers spins them up reproducibly. |
| Code quality | **Ruff** (lint+format), **mypy** (strict), **pre-commit** | Standard modern Python toolchain. |
| Package / deps | **uv** + `pyproject.toml` | Fast, reproducible installs. |
| Frontend tooling | **pnpm**, **Vitest**, **Playwright** (E2E), **ESLint/Prettier** | Standard React stack. |
| Containerisation | **Docker** + **docker-compose** | Self-hosted is a primary deployment mode; compose bundles api/worker/beat/postgres/redis/opensearch/mcp. |
| Secrets at rest | **`cryptography` (Fernet)** for SUSHI/IdP credentials in JSONB | Schema stores `sushi_credentials` "encrypted at rest"; envelope-encrypt with a key from env/KMS. |
| Observability | **structlog** (JSON logs), **OpenTelemetry** (traces), **Prometheus** `/metrics` | Multi-protocol system needs per-protocol tracing (the schema's `audit_log.protocol` column mirrors this). |
| Standards compliance targets | MARC21, ISO 2709, MarcXchange (ISO 25577), MARCXML, BIBFRAME 2.0, Dublin Core, OAI-PMH 2.0, Z39.50/SRU, NCIP (Z39.83), SIP2, ISO 18626, COUNTER 5.1 / SUSHI (Z39.93), KBART 2, OpenURL (Z39.88), EDIFACT, OpenAPI 3.1, SAML 2.0, OAuth2/OIDC, OWASP ASVS 4.0, GDPR/FERPA | Driven by `standards.md`; referenced per-phase below. |

### Project Structure

```
academic-library-management/
├── pyproject.toml
├── uv.lock
├── README.md
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── migrations/                         # Alembic
│   └── versions/
├── src/
│   └── liberty/                        # package name
│       ├── __init__.py
│       ├── main.py                     # FastAPI app factory
│       ├── config.py                   # Pydantic Settings
│       ├── db.py                       # async engine/session
│       ├── deps.py                     # FastAPI dependencies (auth, tenant, db)
│       ├── security/
│       │   ├── jwt.py
│       │   ├── saml.py
│       │   ├── oauth.py                # OAuth2 client-credentials for API
│       │   ├── crypto.py               # Fernet envelope encryption
│       │   └── rbac.py                 # role → permission resolution
│       ├── models/                     # SQLAlchemy ORM (one module per domain)
│       │   ├── base.py                 # Base, mixins (TenantMixin, TimestampMixin)
│       │   ├── institution.py          # institutions, users
│       │   ├── patron.py
│       │   ├── catalogue.py            # bib_records, items
│       │   ├── circulation.py          # loans, holds, fines
│       │   ├── acquisitions.py
│       │   ├── erm.py                  # electronic_resources, usage_statistics
│       │   ├── ill.py                  # ill_requests
│       │   ├── reserves.py             # course_reserves
│       │   └── ai_audit.py             # ai_suggestions, audit_log
│       ├── schemas/                    # Pydantic request/response models
│       ├── services/                   # business logic (transport-agnostic)
│       │   ├── circulation.py          # checkout/checkin/renew/recall state machine
│       │   ├── holds.py                # queue management
│       │   ├── fines.py
│       │   ├── cataloguing.py
│       │   ├── acquisitions.py
│       │   ├── erm.py                  # cost-per-use, licence lifecycle
│       │   ├── ill.py                  # ISO 18626 lifecycle + rota
│       │   ├── reserves.py
│       │   └── audit.py
│       ├── marc/                       # MARC21 / MARCXML / ISO2709 / BIBFRAME / DC
│       │   ├── record.py               # marc_fields_json <-> pymarc Record
│       │   ├── iso2709.py
│       │   ├── marcxml.py
│       │   ├── bibframe.py
│       │   └── dublin_core.py
│       ├── protocols/                  # external protocol adapters
│       │   ├── z3950.py                # Z39.50 client
│       │   ├── sru.py                  # SRU client + server
│       │   ├── oai_pmh/                # provider + harvester (consumer)
│       │   ├── sip2/                   # SIP2 self-check server (TCP)
│       │   ├── ncip.py                 # NCIP messages
│       │   ├── iso18626.py             # ILL messaging
│       │   ├── sushi.py                # COUNTER 5 SUSHI harvester
│       │   ├── edifact.py              # ORDERS / INVOIC
│       │   └── openurl.py              # link resolver
│       ├── discovery/                  # OpenSearch indexing + query
│       │   ├── index.py                # mappings, bulk indexer
│       │   ├── search.py               # faceted query builder, ranking
│       │   └── projector.py            # bib_record -> discovery doc
│       ├── ai/
│       │   ├── client.py               # LLMClient abstraction
│       │   ├── cataloguing.py          # MARC/subject suggestions
│       │   ├── contract.py             # licence term extraction
│       │   ├── reference.py            # RAG reference assistant
│       │   ├── prompts/                # prompt templates (versioned)
│       │   └── suggestions.py          # persist/apply ai_suggestions
│       ├── compliance/
│       │   └── open_access.py          # Plan S / NIH / UKRI rule engine
│       ├── api/                        # FastAPI routers (thin; call services)
│       │   └── v1/
│       │       ├── catalogue.py
│       │       ├── circulation.py
│       │       ├── patrons.py
│       │       ├── acquisitions.py
│       │       ├── erm.py
│       │       ├── ill.py
│       │       ├── reserves.py
│       │       ├── discovery.py
│       │       ├── compliance.py
│       │       ├── ai.py
│       │       └── admin.py
│       ├── tasks/                      # Celery tasks
│       │   ├── app.py                  # Celery app + Beat schedule
│       │   ├── harvest.py              # SUSHI, OAI-PMH
│       │   ├── circulation.py          # overdue/fine/hold sweeps
│       │   ├── indexing.py             # OpenSearch sync
│       │   └── ai.py                   # batch suggestion jobs
│       ├── mcp_server/                 # MCP server (separate entrypoint)
│       │   └── server.py
│       └── cli/                        # Typer CLI (batch import/export, admin)
│           └── main.py
├── frontend/
│   ├── staff/                          # staff console SPA
│   └── discovery/                      # patron OPAC/discovery SPA
└── tests/
    ├── conftest.py                     # fixtures: db, client, tenant, factories
    ├── fixtures/                       # sample MARC, MARCXML, SUSHI, ISO18626, EDIFACT
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (models / services / protocols / api / tasks / ai) so each phase adds modules without restructuring.

---

## Phase 1: Foundation, Schema & Multi-Tenancy

### Purpose
Establish the runnable skeleton: project scaffold, configuration, the full PostgreSQL schema via Alembic, async DB access, multi-tenant scoping, RBAC primitives, the audit log, and a health endpoint. After this phase the app boots, migrations apply, and every later phase has tables and a tenant/auth context to build on. This phase is foundational and deliberately small in feature surface but wide in schema.

### Tasks

#### 1.1 — Project scaffold & configuration

**What**: Create the `liberty` package, `pyproject.toml` (uv), FastAPI app factory, Pydantic `Settings`, Docker/compose, and tooling config (ruff, mypy, pre-commit).

**Design**:
- `Settings` (Pydantic `BaseSettings`) reads env with defaults:
```python
class Settings(BaseSettings):
    app_env: str = "development"          # development|staging|production
    database_url: str                      # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    opensearch_url: str = "http://localhost:9200"
    jwt_secret: str
    fernet_key: str                        # base64 32-byte key for at-rest crypto
    llm_provider: str = "anthropic"        # anthropic|openai|azure|local
    llm_api_key: str | None = None
    llm_model: str = "claude-opus-4-8"
    saml_metadata_dir: str = "./saml"
    default_currency: str = "GBP"
    cors_origins: list[str] = ["http://localhost:5173"]
    model_config = SettingsConfigDict(env_prefix="LIBERTY_", env_file=".env")
```
- `create_app() -> FastAPI` wires routers, exception handlers, CORS, OpenTelemetry, `/healthz`, `/metrics`, and `/openapi.json` (OpenAPI 3.1).
- `docker-compose.yml` services: `api`, `worker`, `beat`, `postgres:16`, `redis:7`, `opensearch:2`, `mcp`.

**Testing**:
- `Unit: Settings with full env → all fields populated, defaults applied`
- `Unit: missing LIBERTY_DATABASE_URL → ValidationError naming database_url`
- `Integration: GET /healthz → 200 {"status":"ok","db":"ok","redis":"ok"}` (testcontainers)
- `E2E: docker compose up → api container healthcheck passes`

#### 1.2 — Database layer & base models

**What**: Async SQLAlchemy engine/session, declarative `Base`, and shared mixins.

**Design**:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(),
                                                 onupdate=func.now())

class TenantMixin:
    institution_id: Mapped[UUID] = mapped_column(ForeignKey("institutions.id"),
                                                 nullable=False, index=True)
```
- `get_session()` async dependency yields a session bound to request scope.
- UUID PKs via `server_default=text("gen_random_uuid()")` (Postgres `pgcrypto`).

**Testing**:
- `Unit: model with TimestampMixin inserted → created_at/updated_at non-null`
- `Integration: update row → updated_at advances` (real Postgres)

#### 1.3 — Full schema migration (data-model-suggestion-1)

**What**: Alembic migration creating all 14 tables, indexes, CHECK constraints, the GIN/array/tsvector indexes, and the partitioned `audit_log`, exactly as specified in `data-model-suggestion-1.md`.

**Design**:
- Tables (verbatim from the adopted data model): `institutions`, `users`, `patrons`, `bib_records`, `items`, `loans`, `holds`, `fines`, `acquisition_orders`, `electronic_resources`, `ill_requests`, `course_reserves`, `ai_suggestions`, `audit_log`.
- Add the supplementary `usage_statistics` table referenced by the cost-per-use query (deferred population to Phase 7 but created here so FKs are stable):
```sql
CREATE TABLE usage_statistics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES institutions(id),
    electronic_resource_id UUID NOT NULL REFERENCES electronic_resources(id),
    report_type TEXT NOT NULL CHECK (report_type IN ('TR','DR','PR','IR')),
    report_period DATE NOT NULL,            -- first day of month
    metric_type TEXT NOT NULL,              -- COUNTER 5 metric e.g. Total_Item_Requests
    total_item_requests BIGINT NOT NULL DEFAULT 0,
    raw_json JSONB NOT NULL DEFAULT '{}',
    harvested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (electronic_resource_id, report_type, report_period, metric_type)
);
CREATE INDEX idx_usage_resource ON usage_statistics (electronic_resource_id, report_period);
```
- Create `audit_log` as `PARTITION BY RANGE (created_at)` plus the current+next month partitions; a Celery Beat job (Phase 7) rolls partitions monthly.
- Enable extensions `pgcrypto` (gen_random_uuid) in the first migration.
- `alembic downgrade base` must cleanly drop everything (reversibility = migration-safety requirement).

**Testing**:
- `Integration: alembic upgrade head → all 15 tables + expected indexes present` (query `pg_indexes`)
- `Integration: insert bib_record with marc_fields_json → GIN containment query @> finds it`
- `Integration: insert loan referencing missing item → ForeignKeyViolation`
- `Integration: bib_record.record_type='bogus' → CheckViolation`
- `Integration: alembic upgrade head → downgrade base → upgrade head succeeds idempotently`

#### 1.4 — Multi-tenancy & RBAC

**What**: Resolve the current institution + user/role on each request and enforce role→permission checks.

**Design**:
- `TenantContext(institution_id, user_id, role, permissions)` built in `deps.py` from the JWT (Phase 2 issues it; here a dev header `X-Institution-Id` + `X-Debug-Role` is accepted only when `app_env=development`).
- `rbac.py` maps the 10 `users.role` values to permission sets, e.g.:
```python
ROLE_PERMISSIONS = {
  "cataloguer": {"bib:read","bib:write","item:read","item:write","ai:catalogue"},
  "circulation_desk": {"loan:write","hold:write","fine:write","patron:read"},
  "erm_manager": {"erm:read","erm:write","sushi:harvest","ai:contract"},
  # ...admin: "*"
}
def require(perm: str) -> Callable  # FastAPI dependency raising 403
```
- Every service method takes `ctx: TenantContext` and filters queries by `institution_id`.

**Testing**:
- `Unit: require('bib:write') with cataloguer ctx → passes; with read_only ctx → 403`
- `Integration: request without tenant context in production env → 401`
- `Integration: cataloguer at institution A cannot read bib_record of institution B → 404`

#### 1.5 — Audit logging service

**What**: Central `audit.record(...)` writing to the partitioned `audit_log`, with GDPR/FERPA flags and protocol attribution.

**Design**:
```python
async def record(ctx, *, actor_type, actor_id, action, entity_type, entity_id,
                 changes: dict, protocol: str | None = None,
                 gdpr: bool = False, ferpa: bool = False, request=None): ...
```
- Called by services on every mutation; `protocol` set by protocol adapters (`sip2`, `ncip`, `oai_pmh`, `rest`, ...).

**Testing**:
- `Unit: record() with gdpr=True → row has gdpr_relevant=true`
- `Integration: checkout via service → audit_log has action='loan.checkout', protocol='rest'`

### Definition of Done
All tasks implemented; `alembic upgrade head`/`downgrade base` round-trip green; unit+integration tests pass; ruff+mypy clean; `docker compose up` healthy; `/openapi.json` served.

---

## Phase 2: Identity, Authentication & Authorization

### Purpose
Make the platform multi-user and secure: staff/patron login via SAML/Shibboleth and OIDC, local JWT sessions, OAuth2 client-credentials for the REST API, and at-rest encryption for stored credentials. This unlocks real `TenantContext` and protects every endpoint, satisfying the federated-identity core requirement (`research.md` §4) and OWASP ASVS baseline (`standards.md`).

### Tasks

#### 2.1 — Local JWT sessions & API OAuth2

**What**: Issue/verify JWTs for staff sessions; OAuth2 client-credentials grant for machine API clients.

**Design**:
- `POST /api/v1/auth/token` (client-credentials): body `{client_id, client_secret}` → `{access_token, token_type:"bearer", expires_in}`. Clients are rows in `users` with `role` + an issued secret hash.
- JWT claims: `sub` (user id), `inst` (institution_id), `role`, `perms`, `exp`. HS256 with `jwt_secret`; 1h access tokens.
- `get_current_context()` dependency decodes JWT → `TenantContext`.

**Testing**:
- `Unit: encode→decode JWT round-trips claims`
- `Integration: POST /auth/token valid creds → 200 token; invalid → 401`
- `Integration: protected endpoint with expired token → 401 "token expired"`

#### 2.2 — SAML 2.0 / Shibboleth SSO (staff + patrons)

**What**: SP-initiated SAML login with per-institution IdP metadata; map IdP subject → `users`/`patrons`.

**Design**:
- `GET /api/v1/auth/saml/{institution_slug}/login` → redirect to IdP.
- `POST /api/v1/auth/saml/{institution_slug}/acs` (Assertion Consumer Service) → validate assertion (`python3-saml`), upsert patron/user keyed on `(idp_provider, idp_subject)`, issue JWT.
- IdP metadata loaded from `saml_metadata_dir/{slug}.xml`; SP entity id = `institutions.shibboleth_entity_id`.
- Honour REFEDS R&S attribute bundle (eduPersonPrincipalName, mail, displayName, eduPersonScopedAffiliation → `patron_type` mapping).

**Testing**:
- `Integration (mocked IdP): valid signed assertion → patron upserted, JWT returned`
- `Integration: tampered/invalid signature → 401, no upsert`
- `Unit: eduPersonScopedAffiliation 'student@uni' → patron_type='undergraduate'`

#### 2.3 — OIDC login & at-rest credential encryption

**What**: OIDC (authlib) as alternative to SAML; Fernet envelope-encryption helpers for `sushi_credentials`, `idp` secrets.

**Design**:
- `crypto.encrypt_json(d: dict) -> str` / `decrypt_json(s) -> dict` using `fernet_key`; stored fields prefixed `enc:`.
- OIDC discovery via well-known URL configured per institution `config_json.oidc`.

**Testing**:
- `Unit: encrypt_json→decrypt_json round-trips; ciphertext != plaintext`
- `Integration (mocked OIDC): code exchange → user upserted, JWT issued`

### Definition of Done
SAML + OIDC + client-credentials flows pass integration tests with mocked IdPs; all `/api/v1/*` endpoints reject anonymous requests; credentials never stored in plaintext; mypy/ruff clean; OWASP ASVS auth checklist items (session expiry, no secret in logs) verified.

---

## Phase 3: Cataloguing & MARC Core

### Purpose
Deliver the heart of the catalogue: create/read/update/delete bibliographic records and physical items, with lossless MARC21 ↔ JSONB conversion and import/export in ISO 2709, MARCXML, Dublin Core, and BIBFRAME. This is the value core — everything downstream (circulation, discovery, ILL, ERM) references `bib_records`/`items`. Standards: MARC21, ISO 2709, MarcXchange (ISO 25577), MARCXML, BIBFRAME 2.0, Dublin Core.

### Tasks

#### 3.1 — MARC record model & JSONB round-trip

**What**: Convert between the schema's `marc_fields_json` structure and `pymarc.Record`, and derive the denormalised columns (`title`, `authors`, `isbn`, `subject_headings`, ...).

**Design**:
```python
# marc_fields_json element shape (from data model):
# {"tag":"245","ind1":"1","ind2":"0","subfields":[{"code":"a","value":"..."}]}
def json_to_pymarc(leader: str, fields: list[dict]) -> pymarc.Record: ...
def pymarc_to_json(rec: pymarc.Record) -> tuple[str, list[dict]]: ...
def derive_columns(rec: pymarc.Record) -> DerivedBib:
    # 245$a/$b -> title/subtitle; 100/700 -> authors; 020 -> isbn[]; 022 -> issn[];
    # 650 -> subject_headings[]; 050 -> lcc_call_number; 082 -> dewey_number;
    # 008/35-37 -> language; leader/06-07 -> record_type/material_type mapping
```

**Testing**:
- `Fixture: load fixtures/sample.mrc → json_to_pymarc→pymarc_to_json byte-identical leader+fields`
- `Unit: record with 245$a$b → title and subtitle split correctly`
- `Unit: two 020 fields → isbn array length 2, normalised via stdnum`
- `Unit: malformed leader (<24 chars) → ValueError naming field`

#### 3.2 — Bib record & item CRUD API

**What**: REST CRUD for `bib_records` and `items` with validation and audit.

**Design**:
- `POST /api/v1/catalogue/bib` body `{leader, fields[], source}` → derive columns, persist, return record. `source` defaults `original_cataloguing`.
- `PUT /api/v1/catalogue/bib/{id}` re-derives columns and bumps `updated_at` (drives OAI-PMH incremental harvest).
- `POST /api/v1/catalogue/bib/{id}/items` create item; maintains `bib_records.total_items/available_items` counters in a transaction.
- `GET /api/v1/catalogue/bib?isbn=&title=&subject=` Postgres-FTS/GIN staff search (discovery-grade search is Phase 8).
- Item status transitions validated against the CHECK enum.

**Testing**:
- `Integration: POST bib with 245 → 201, title column populated, audit row written`
- `Integration: POST item → bib.total_items incremented to 1`
- `Integration: cataloguer role required → read_only POST returns 403`
- `Integration: GET ?isbn=978... → returns the record (GIN array index used)`

#### 3.3 — Import/export: ISO 2709, MARCXML, Dublin Core, BIBFRAME

**What**: Batch import/export endpoints + CLI for the four serialisations.

**Design**:
- `POST /api/v1/catalogue/import` multipart file + `format` in `{iso2709, marcxml}` → streams records, dedups on `control_number`/`oclc_number`/`isbn`, returns `{created, updated, skipped, errors[]}`.
- `GET /api/v1/catalogue/bib/{id}.{ext}` where ext ∈ `mrc|xml|dc.xml|jsonld` serialises to ISO 2709 / MARCXML / Dublin Core XML / BIBFRAME JSON-LD using `marc/` converters; populates `bibframe_json`/`dc_json` lazily on first export.
- BIBFRAME mapping: MARC→BIBFRAME 2.0 Work/Instance/Item (use LC conversion rules subset); store `work_uri`/`instance_uri`.
- CLI: `liberty catalogue import <file> --format marcxml --institution <slug>`.

**Testing**:
- `Fixture: import fixtures/batch.xml (MARCXML, 50 recs) → created=50`
- `Integration: re-import same file → updated=50, created=0 (dedup on control_number)`
- `Integration: GET /bib/{id}.dc.xml → valid Dublin Core, dc:title matches`
- `Integration: GET /bib/{id}.jsonld → BIBFRAME has bf:Work + bf:Instance`
- `Unit: ISO 2709 export → re-parse with pymarc yields equal record`

### Definition of Done
MARC round-trips byte-stable on fixtures; CRUD + import/export endpoints in OpenAPI spec; counters consistent under concurrent item adds (transaction test); ruff/mypy clean.

---

## Phase 4: Circulation & Patron Services

### Purpose
Add the daily operational engine: patron records, checkout/checkin/renew/recall, the holds queue, and fines — the workflows that justify a library system. Builds directly on Phase 3 items and Phase 1/2 patrons/auth. Implements the circulation state machine and fine accrual logic that NCIP and SIP2 (Phase 6) will later drive.

### Tasks

#### 4.1 — Patron management

**What**: CRUD for `patrons` with GDPR/FERPA consent fields and circulation limits.

**Design**:
- `POST/GET/PUT /api/v1/patrons`; `gdpr_consent` validated against a Pydantic sub-model `{reading_history:bool, search_history:bool, analytics:bool, consent_date:date}`.
- `POST /api/v1/patrons/{id}/anonymise` → set `status='anonymised'`, null PII columns, set `anonymised_at`; loans retained but patron_id reassigned to a tombstone patron (GDPR erasure while preserving aggregate stats).

**Testing**:
- `Integration: POST patron → 201, default max_loans=20`
- `Integration: anonymise → PII null, status='anonymised', audit gdpr_relevant=true`
- `Unit: gdpr_consent missing consent_date → ValidationError`

#### 4.2 — Circulation state machine (checkout/checkin/renew/recall)

**What**: Core circulation service enforcing policy and item/loan state transitions.

**Design**:
- States: loan `active → renewed → overdue → returned | lost | claimed_returned`; recall sets `recalled=true`, shortens `recall_due_date`.
- `checkout(ctx, patron_id, item_barcode)`:
  1. resolve item; assert `status='available'` (or `on_hold` for this patron).
  2. enforce `patron.max_loans`, block if `outstanding_fines_cents > block_threshold_cents`.
  3. compute `due_date` from institution `loan_policies[patron_type]` / item `loan_type`.
  4. insert `loans`, set item `status='checked_out'`, decrement `bib.available_items`.
  5. audit `loan.checkout`.
- `checkin(item_barcode)`: close loan, accrue overdue fine if late (Phase 4.4), set item `available`/`in_transit`/`on_hold` (if hold queue non-empty → trigger hold fulfilment), increment counters.
- `renew(loan_id)`: assert `renewal_count < max_renewals` and not recalled; extend `due_date`.
- All steps in one DB transaction; `SELECT ... FOR UPDATE` on item to prevent double-checkout.

**Testing**:
- `Integration: checkout available item → loan active, item checked_out, available_items--`
- `Integration: checkout already-checked-out item → 409 conflict, no second loan`
- `Integration: patron over max_loans → 422 "loan limit exceeded"`
- `Integration: patron over fine threshold → 422 "blocked: outstanding fines"`
- `Integration: renew recalled loan → 422; renew at max_renewals → 422`
- `Integration: concurrent checkout of same item (two sessions) → exactly one succeeds`

#### 4.3 — Holds queue

**What**: Title- and copy-level holds with queue positions and pickup lifecycle.

**Design**:
- `place_hold(ctx, patron_id, bib_id, item_id?, pickup_location)`: insert `holds` with `queue_position = max(position)+1` for that bib; `hold_type='title'` when `item_id` NULL.
- On checkin of an item whose bib has pending holds: set earliest hold `status='available_for_pickup'`, set `available_at`/`pickup_by`, item `status='on_hold'`, flag `notification_sent` for the notifier (Phase 7).
- Expiry sweep (Phase 7 Beat) moves stale `available_for_pickup` → `expired` and promotes next in queue.

**Testing**:
- `Integration: two patrons hold same bib → queue_position 1 and 2`
- `Integration: checkin satisfies hold #1 → status available_for_pickup, item on_hold`
- `Integration: cancel hold #1 → hold #2 promoted to position 1`

#### 4.4 — Fines & payments

**What**: Fine accrual, payment, waiver; keep `patron.outstanding_fines_cents` in sync.

**Design**:
- Overdue fine = `ceil(days_late) * fine_rates.overdue_per_day_cents` capped at `replacement_cost_cents`.
- `pay(fine_id, amount, method)` → update `paid_cents`, status `partially_paid|paid`; `method='payment_gateway'` returns a redirect handle (campus gateway, no PCI scope — per `standards.md` note).
- `waive(fine_id, reason)` requires `fine:waive` perm; records `waived_by`.
- A DB trigger or service-level recompute keeps `outstanding_fines_cents = SUM(amount-paid where status outstanding/partial)`.

**Testing**:
- `Unit: 5 days late @20c → 100c fine; capped at replacement_cost`
- `Integration: pay partial → status partially_paid, outstanding recomputed`
- `Integration: waive without perm → 403`

### Definition of Done
Circulation state machine covered by happy-path + concurrency + edge tests; counters and `outstanding_fines_cents` provably consistent; OpenAPI updated; ruff/mypy clean.

---

## Phase 5: AI-Assisted Cataloguing (Differentiator)

### Purpose
Ship the first AI-native differentiator: while cataloguing, suggest MARC fields and subject headings (LCSH/MeSH), detect probable duplicates, and propose classification — all persisted to `ai_suggestions` with confidence and a human accept/reject loop. This is an MVP "must-have" (`features.md`) and the most labour-saving AI workflow. Depends on Phase 3 catalogue.

### Tasks

#### 5.1 — LLM client abstraction & prompt management

**What**: Provider-agnostic `LLMClient` with structured-output support and versioned prompt templates.

**Design**:
```python
class LLMClient(Protocol):
    async def complete(self, *, system: str, user: str,
                       schema: type[BaseModel] | None = None,
                       max_tokens: int = 2000) -> LLMResult: ...
# implementations: AnthropicClient (default), OpenAIClient, LocalClient
```
- Prompts in `ai/prompts/*.txt` with `{placeholders}`; each carries a `version` string stored in `ai_suggestions.model_version`.
- Cost/latency captured and logged; retries with backoff on rate limits.

**Testing**:
- `Unit: AnthropicClient.complete with schema → parses to Pydantic model` (SDK mocked)
- `Unit: provider switch via settings.llm_provider → correct class instantiated`
- `Unit: rate-limit error → retried then surfaced after N attempts`

#### 5.2 — MARC field & subject-heading suggestions

**What**: Given a draft bib record, suggest additional MARC fields and LCSH/MeSH headings.

**Design**:
- System prompt: "You are a MARC21 cataloguing assistant. Given the existing fields, suggest additional valid MARC fields and LCSH/MeSH subject headings. Return strict JSON." Output schema mirrors `ai_suggestions.detail_json`:
```python
class MarcSuggestion(BaseModel):
    suggested_fields: list[MarcField]   # {tag, subfields, source}
    subject_headings: list[SubjectHeading]  # {scheme:'lcsh'|'mesh', value, confidence}
    rationale: str
```
- `POST /api/v1/ai/catalogue/{bib_id}/suggest` → call LLM, persist one `ai_suggestions` row (`suggestion_type='subject_heading'`/`'marc_field'`, `status='pending'`, `confidence` from model), return suggestions.
- `POST /api/v1/ai/suggestions/{id}/accept` → merge `suggested_fields` into `bib.marc_fields_json`, re-derive columns, set suggestion `accepted` + `reviewed_by`. `.../reject` sets `rejected`.
- Guardrail: validate every suggested field tag against a MARC21 tag allowlist before applying.

**Testing**:
- `Integration (mocked LLM): suggest → ai_suggestions row pending, confidence set`
- `Integration: accept → bib.marc_fields_json gains 650 field, columns re-derived`
- `Unit: suggested invalid tag '9XX-bogus' → rejected by guardrail, not applied`
- `Integration: accept without ai:catalogue perm → 403`

#### 5.3 — Duplicate detection & classification suggestion

**What**: Flag likely-duplicate bib records and suggest LCC/Dewey classification.

**Design**:
- Candidate generation: match on normalised ISBN/ISSN/OCLC, then title+author trigram similarity (`pg_trgm`); send top-K candidate pairs to LLM for `is_duplicate` judgement → `ai_suggestions` (`suggestion_type='duplicate_detection'`, `target_entity_type='bib_record'`).
- Classification: prompt suggests `lcc_call_number`/`dewey_number` with confidence.

**Testing**:
- `Integration: two near-identical bibs → duplicate suggestion created with both ids in detail_json`
- `Integration (mocked LLM): classify → suggestion has lcc_call_number`

### Definition of Done
Suggestion lifecycle (create→accept/reject) works end-to-end with mocked LLM; guardrails prevent invalid MARC writes; provider abstraction switchable; all suggestions auditable; ruff/mypy clean.

---

## Phase 6: Library Protocols — Z39.50/SRU, SIP2, NCIP, OAI-PMH

### Purpose
Connect the platform to the wider library ecosystem via the de-facto protocols, enabling record import from OCLC/peer catalogues, self-checkout machines, cross-system circulation, and metadata harvesting. These are MVP/v1.1 table-stakes (`features.md`) and the interoperability backbone (`research.md` §4). Each adapter is transport for existing services (Phase 3/4), so this phase adds adapters, not new business logic.

### Tasks

#### 6.1 — Z39.50 client & SRU client/server

**What**: Search remote catalogues (Z39.50 + SRU) and expose an SRU server over the local catalogue.

**Design**:
- Z39.50 client (via `yaz-client` subprocess or `PyZ3950`): `search(target, query) -> list[pymarc.Record]`; target presets for OCLC WorldCat, LC.
- `POST /api/v1/catalogue/external-search` body `{target, isbn|title|author}` → returns candidate MARC records for one-click import (writes `source='z39_50_import'`/`'sru_import'`).
- SRU server: `GET /sru?version=2.0&operation=searchRetrieve&query=...` → SRU XML response (CQL → SQL translation for dc.title, dc.identifier, bath.isbn). Standards: Z39.50, SRU/CQL.

**Testing**:
- `Integration (mocked target): external-search isbn → candidate records returned`
- `Integration: GET /sru?query=dc.title=algorithms → valid SRU XML, numberOfRecords correct`
- `Unit: CQL 'bath.isbn=978...' → correct SQL WHERE`

#### 6.2 — OAI-PMH provider & harvester

**What**: Expose an OAI-PMH 2.0 provider over `bib_records` and harvest from remote repositories.

**Design**:
- Provider: `GET /oai?verb=...` supporting `Identify`, `ListMetadataFormats` (oai_dc, marc21, mods), `ListRecords`, `GetRecord`, `ListIdentifiers` with resumption tokens; incremental via `bib_records.updated_at` + `oai_identifier`. Returns Dublin Core / MARCXML per metadataPrefix.
- Harvester (Celery task): `harvest_oai(source_url, set?, from?)` pulls records, maps to bib records (`source='oai_pmh_harvest'`), dedups.

**Testing**:
- `Integration: GET /oai?verb=ListRecords&metadataPrefix=oai_dc → valid OAI-PMH XML`
- `Integration: GET /oai?verb=GetRecord&identifier=...&metadataPrefix=marc21 → MARCXML`
- `Integration: resumptionToken paging returns all records across pages`
- `Integration (mocked endpoint): harvest_oai → N bib_records created with oai_identifier`

#### 6.3 — SIP2 self-checkout server

**What**: TCP SIP2 server so self-check kiosks can checkout/checkin/renew and query patron status.

**Design**:
- asyncio TCP server parsing SIP2 messages; map: `11`(Checkout)→`checkout`, `09`(Checkin)→`checkin`, `29`(Renew)→`renew`, `63`(Patron Info)→patron summary, `93`(Login), `99`(SC Status). Build `12`/`10`/`30`/`64`/`94`/`98` responses with correct field codes and checksums.
- Each call sets `audit_log.protocol='sip2'`, `actor_type='sip2_terminal'`; `loans.sip2_terminal`/`desensitised` populated.

**Testing**:
- `Unit: parse Checkout (11) message → correct patron/item barcodes extracted`
- `Integration: 11 Checkout valid → 12 response with CirculationStatus ok, loan created`
- `Integration: 11 Checkout for blocked patron → 12 with denial + screen message`
- `Unit: response checksum matches SIP2 spec`

#### 6.4 — NCIP service

**What**: NCIP (Z39.83) endpoint for patron/item exchange (drives ILL and consortial circ).

**Design**:
- `POST /ncip` XML; handle `LookupItem`, `LookupUser`, `CheckOutItem`, `CheckInItem`, `RequestItem`, `AcceptItem` → delegate to circulation/holds services; respond with NCIP XML. `protocol='ncip'`.

**Testing**:
- `Integration: CheckOutItem NCIP message → loan created, NCIP response with due date`
- `Integration: LookupUser → patron summary in NCIP XML`
- `Integration: RequestItem for unavailable item → hold placed`

### Definition of Done
Each protocol adapter passes fixture-driven request/response tests; SIP2 checksum/field encoding verified against spec samples; SRU/OAI-PMH XML validates against schemas; protocol attribution appears in audit log; ruff/mypy clean.

---

## Phase 7: Acquisitions, ERM & Open-Access Compliance

### Purpose
Add collection-spend and e-resource management: EDIFACT vendor orders/invoices with fund tracking, electronic-resource licences, COUNTER 5/SUSHI usage harvesting with cost-per-use analytics, and an open-access compliance engine for Plan S/NIH/UKRI mandates. These are MVP (basic acquisitions) and v1.1 (ERM, OA dashboard) scope from `features.md`. Introduces the background-job machinery used by later sweeps.

### Tasks

#### 7.1 — Acquisitions & EDIFACT

**What**: Purchase orders with fund/fiscal-year tracking and EDIFACT ORDERS/INVOIC exchange.

**Design**:
- CRUD on `acquisition_orders`; state machine `pending→sent→acknowledged→partially_received→received→invoiced→paid` (+`cancelled`/`claimed`).
- `POST /api/v1/acquisitions/{id}/send` → generate EDIFACT ORDERS message (`edifact.build_orders(order)`), set `edifact_message_ref`, status `sent`.
- `POST /api/v1/acquisitions/edifact/invoic` ingest vendor INVOIC → match on `vendor_order_ref`, populate invoice fields, status `invoiced`.
- Receiving increments `quantity_received`; full receipt creates `items` linked to the bib record (fund-to-title traceability).
- Fund summary endpoint aggregates committed/spent by `fund_code`+`fiscal_year`.

**Testing**:
- `Integration: create order → send → EDIFACT ORDERS string parses, status sent`
- `Integration: ingest INVOIC fixture → matched, invoice_amount populated`
- `Integration: receive qty=order qty → status received, items created`
- `Unit: fund summary sums actual_cost_cents per fund/year`

#### 7.2 — Electronic resources & licence lifecycle

**What**: CRUD for `electronic_resources` with structured licence terms, KBART import, encrypted SUSHI credentials.

**Design**:
- `licence_terms_json` validated by Pydantic sub-model (simultaneous_users, ill_permitted, text_mining_permitted, archival_rights, ...).
- `POST /api/v1/erm/kbart-import` parses KBART 2 TSV → upsert resources + coverage.
- `sushi_credentials` stored via Fernet (`enc:` prefix); never returned in API responses.
- Licence-expiry alerts surfaced by a Beat job (`licence_end` within 90 days).

**Testing**:
- `Integration: create e-resource → sushi_credentials stored encrypted, absent from GET`
- `Integration: KBART import fixture → resources created with coverage_json`
- `Unit: licence_terms with bad enum → ValidationError`

#### 7.3 — COUNTER 5 / SUSHI harvesting & cost-per-use

**What**: Harvest COUNTER 5.1 reports via SUSHI into `usage_statistics`; compute cost-per-use.

**Design**:
- `sushi.fetch_report(resource, report_type, period)` calls `{sushi_endpoint}/reports/{tr|dr|pr|ir}` with decrypted creds; parse COUNTER 5.1 JSON → upsert `usage_statistics` rows (dedup on the unique key).
- Celery Beat: monthly `harvest_all_sushi` across active resources; updates `electronic_resources.last_harvest_at` and recomputes `cost_per_use_cents` using the cost-per-use query from the data model.
- `GET /api/v1/erm/cost-per-use?fiscal_year=` returns the ranked analysis.

**Testing**:
- `Integration (mocked SUSHI): fetch TR fixture → usage_statistics rows upserted`
- `Integration: re-harvest same period → no duplicate rows (unique constraint)`
- `Integration: cost-per-use endpoint → resource with 0 uses sorts last (NULLS LAST)`
- `Unit: cost_per_use = annual_cost / total_uses computed correctly`

#### 7.4 — Open-access compliance engine + background sweeps

**What**: Per-funder OA compliance evaluation (Plan S/NIH/UKRI) and the periodic Beat jobs (overdue, hold-expiry, audit-partition roll, licence alerts).

**Design**:
- `compliance/open_access.py` rule engine: for each bib with `oa_deposit_required`, evaluate each mandate in `oa_funder_mandates` against rules (e.g. Plan S: CC-BY + immediate deposit; NIH: PMC deposit within 12 months) → set `oa_compliance_status`.
- `GET /api/v1/compliance/open-access` returns the dashboard aggregation query from the data model (status × funder counts).
- Celery Beat schedule: `sweep_overdue` (mark loans overdue, accrue fines), `sweep_hold_expiry`, `roll_audit_partition` (create next month's `audit_log` partition), `licence_renewal_alerts`, `harvest_all_sushi`.

**Testing**:
- `Unit: Plan S rule on CC-BY immediate-deposit record → compliant; embargoed → non_compliant`
- `Integration: compliance dashboard → counts grouped by status+funder`
- `Integration: sweep_overdue → loan past due becomes overdue + fine accrued`
- `Integration: roll_audit_partition → next-month partition exists`

### Definition of Done
EDIFACT/KBART/SUSHI fixtures process correctly; cost-per-use and OA dashboard return correct aggregates; Beat jobs registered and unit-tested; encrypted creds never leak; ruff/mypy clean.

---

## Phase 8: Discovery Layer (OpenSearch)

### Purpose
Provide patron-facing unified discovery: relevance-ranked, faceted search across bib records (and, where licensing permits, e-resources), plus OpenURL link resolution. This is the v1.1 discovery feature and the surface the reference assistant (Phase 9) retrieves from. Depends on the catalogue (Phase 3) and ERM (Phase 7).

### Tasks

#### 8.1 — Indexing pipeline

**What**: Project bib records/items/e-resources into OpenSearch with a discovery mapping; keep the index in sync.

**Design**:
- `discovery/projector.py`: `bib_record → discovery_doc` (title, authors, subjects, format, year, availability rollup, oa_status, full-text of selected MARC fields).
- Mapping: text fields with analyzers + `keyword` sub-fields for facets (`record_type`, `language`, `subject_headings`, `oa_status`, availability).
- Sync: Celery task `index_bib(id)` enqueued on bib/item create/update/delete; bulk `reindex_all` CLI command.

**Testing**:
- `Integration (testcontainers OpenSearch): create bib → doc indexed, GET by id present`
- `Integration: update bib title → reindexed doc reflects new title`
- `Integration: delete bib → doc removed`

#### 8.2 — Faceted search API

**What**: Discovery query endpoint with ranking, facets, pagination.

**Design**:
- `GET /api/v1/discovery/search?q=&format=&language=&subject=&oa=&available=&page=&size=&sort=` → `{total, results[], facets{format[],language[],subject[],oa[]}}`.
- Ranking: BM25 with field boosts (title^3, authors^2, subjects^1.5); optional boost for available + open-access items (the "rank by openness" opportunity in `features.md`).
- OpenURL resolver: `GET /api/v1/discovery/openurl?...` parses an OpenURL 1.0 context object (ISSN/DOI/dates) → resolves to local holdings or e-resource access URL.

**Testing**:
- `Integration: search q=algorithms → ranked hits, title matches boosted first`
- `Integration: facet filter format=e_book → only e_books, facet counts correct`
- `Integration: openurl with DOI → resolves to matching e-resource access URL`
- `Integration: openurl no match → 404 with appropriate OpenURL response`

### Definition of Done
Indexing stays consistent with the catalogue (create/update/delete tests); faceted search + OpenURL pass integration tests against a real OpenSearch container; ranking verified on fixtures; ruff/mypy clean.

---

## Phase 9: Interlibrary Loans (ISO 18626) & AI Reference + Contract Intelligence

### Purpose
Complete resource sharing and the remaining AI differentiators: ISO 18626/NCIP interlibrary loans with lender-rota automation, a conversational reference assistant (RAG over the discovery index), and contract intelligence that extracts licence terms/red flags from publisher agreements. ILL is v1.1 table-stakes; the AI features are the headline differentiators (`features.md` AI-augmentation list). Depends on Phases 3, 6, 7, 8.

### Tasks

#### 9.1 — ILL workflow & ISO 18626 messaging

**What**: Borrowing/lending requests through the full ISO 18626 lifecycle with automatic lender-rota progression.

**Design**:
- `ill_requests.status` transitions map 1:1 to ISO 18626 states (`new→validated→sent→received_by_lender→will_supply→shipped→received→checked_out_to_patron→returned_to_lender→completed`, plus `unfilled/cancelled/expired`).
- `iso18626.build_request(req)` / `parse_message(xml)`; `POST /iso18626` endpoint receives partner messages (Request, Supplying/Requesting Agency messages).
- Rota: `lender_string[]` + `current_lender_idx`; on `unfilled`/timeout, advance to next lender and resend; when exhausted → `unfilled` with reason.
- On receipt, optionally check out to patron (NCIP/circulation reuse); copyright compliance recorded; `fee_cents`/`ifm_transaction_id` for fee accounting.

**Testing**:
- `Integration: create borrowing request → ISO 18626 Request XML built, status sent`
- `Integration (mocked partner): 'Unfilled' response → advances to next lender, resends`
- `Integration: exhaust lender_string → status unfilled with reason`
- `Integration: 'Loaned'+ship → status shipped; receive → received`
- `Unit: ISO 18626 XML parse of supplier message → correct status mapping`

#### 9.2 — MCP server

**What**: An MCP server exposing catalogue search, OAI-PMH harvest status, and discovery search as tools for AI clients (the no-competitor interface from `standards.md`).

**Design**:
- `mcp_server/server.py` registers tools: `search_catalogue(query)`, `get_bib(id)`, `discovery_search(q, filters)`, `harvest_status()`; all reuse the service layer with a scoped service-account `TenantContext`.
- Separate process/entrypoint (`python -m liberty.mcp_server`); read-only by default.

**Testing**:
- `Integration: MCP search_catalogue tool → returns bib summaries`
- `Unit: tool schema advertised matches spec`
- `Integration: write tools absent in read-only mode`

#### 9.3 — Conversational reference assistant (RAG)

**What**: Natural-language Q&A grounded in the local catalogue/discovery index with citation-style answers.

**Design**:
- `POST /api/v1/ai/reference` body `{question, patron_id?}`:
  1. retrieve top-K from OpenSearch (Phase 8) for the question.
  2. prompt LLM with retrieved snippets; instruct citation-style answers referencing call numbers/availability.
  3. persist `ai_suggestions` (`suggestion_type='reference_answer'`).
- Guardrail: answer only from retrieved context; if no hits, say so (no hallucinated holdings).

**Testing**:
- `Integration (mocked LLM + real OpenSearch): question about an indexed title → answer cites that record`
- `Unit: zero retrieval hits → "no matching holdings" response, no fabricated citation`

#### 9.4 — Contract intelligence for ERM

**What**: Extract structured licence terms and flag red-flag clauses from an uploaded publisher agreement.

**Design**:
- `POST /api/v1/ai/erm/{resource_id}/analyse-contract` multipart (PDF/text) → extract text, prompt LLM to return the `licence_terms_json` structure + `red_flags[]` (e.g. no post-cancellation access, text-mining prohibited, auto-renewal, confidentiality clause).
- Persist as `ai_suggestions` (`'licence_term_extraction'` / `'licence_red_flag'`); on accept, merge into `electronic_resources.licence_terms_json`.

**Testing**:
- `Integration (mocked LLM): analyse contract fixture → terms extracted, red_flags populated, suggestion pending`
- `Integration: accept → electronic_resources.licence_terms_json updated`

### Definition of Done
ILL lifecycle + rota pass fixture-driven ISO 18626 tests; MCP tools callable and read-only; reference assistant grounded (no-hallucination test passes); contract analysis round-trips to licence terms; ruff/mypy clean.

---

## Phase 10: Frontend — Staff Console & Discovery/OPAC

### Purpose
Deliver the human interfaces: a persona-driven staff console (cataloguing, circulation, acquisitions, ERM, ILL, reports) and a patron-facing discovery/OPAC. Consumes the REST API from all prior phases. The two SPAs can be built in parallel once the API is stable.

### Tasks

#### 10.1 — Shared frontend foundation & auth

**What**: Vite/React/TS scaffolds for both SPAs, shared API client (generated from OpenAPI 3.1), TanStack Query, shadcn/ui, SAML/OIDC/JWT login flow.

**Design**:
- Generate a typed client from `/openapi.json`; central `apiClient` injecting bearer token; 401 → re-auth.
- Persona routing keyed on `role` claim; permission-gated nav.

**Testing**:
- `Unit (Vitest): apiClient attaches Authorization header; 401 triggers logout`
- `E2E (Playwright): login via mocked IdP → staff dashboard renders for role`

#### 10.2 — Staff console modules

**What**: Cataloguing (MARC editor with AI-suggestion accept/reject), circulation desk (checkout/checkin/holds/fines), acquisitions, ERM (cost-per-use + contract analysis), ILL queue, OA-compliance dashboard, reports.

**Design**:
- MARC editor renders `marc_fields_json` as editable tag/indicator/subfield grid; "Suggest" button calls Phase 5 and shows accept/reject chips with confidence.
- Circulation desk: barcode-scan checkout/checkin, patron summary, holds queue, fine pay/waive.
- Dashboards: charts for circulation, budget by fund, cost-per-use, OA compliance (status×funder), ILL performance.

**Testing**:
- `E2E: catalogue a book, run AI suggest, accept a 650 → record shows new subject`
- `E2E: checkout by barcode → loan appears on patron card; checkin clears it`
- `E2E: OA dashboard renders counts matching API`

#### 10.3 — Discovery / OPAC

**What**: Patron search UI with facets, record detail with availability + "Get It" (OpenURL), place-hold, and the reference-assistant chat.

**Design**:
- Search page hits Phase 8 discovery API; facet sidebar; result list with availability badges and openness indicator.
- Record detail: holdings/items, place-hold action (auth required), OpenURL resolve button.
- Reference chat panel calls Phase 9 reference endpoint, renders cited answers.

**Testing**:
- `E2E: search 'algorithms', filter format=book → results update, facet counts shown`
- `E2E: place hold on a title (logged-in patron) → confirmation, hold in patron account`
- `E2E: ask reference chat a question → cited answer rendered`

### Definition of Done
Both SPAs build (`pnpm build`); Vitest unit + Playwright E2E green against a seeded backend; accessibility smoke checks pass; ESLint/Prettier clean.

---

## Phase 11: Hardening, Observability & Deployment

### Purpose
Make the platform production-ready: security hardening to OWASP ASVS, privacy controls for GDPR/FERPA, observability, performance/partitioning for large consortia, and reproducible deployment. Cross-cuts all phases; runs last but validates everything.

### Tasks

#### 11.1 — Security & privacy hardening

**What**: OWASP ASVS 4.0 pass, rate limiting, GDPR/FERPA data-subject operations, secret hygiene.

**Design**:
- Rate limiting (Redis) on auth + public endpoints; input validation already via Pydantic; SQL-injection-safe via ORM params.
- GDPR: `GET /api/v1/patrons/{id}/export` (data portability) and the Phase 4 anonymise; FERPA: reading-history access gated by `gdpr_consent.reading_history`.
- Secrets scanning in CI; ensure encrypted fields never logged (structlog processor redaction).

**Testing**:
- `Integration: 100 rapid /auth/token attempts → rate-limited (429)`
- `Integration: patron export → JSON bundle of patron's loans/holds/fines`
- `Unit: log processor redacts sushi_credentials/jwt_secret`
- `Security: OWASP ASVS checklist (auth, access control, crypto storage) reviewed`

#### 11.2 — Observability

**What**: Structured logs, OpenTelemetry traces (per protocol), Prometheus metrics, dashboards.

**Design**:
- `audit_log.protocol` mirrored in trace attributes; metrics: request latency, loan/checkin counts, SUSHI harvest success, Celery queue depth, LLM token spend.
- `/metrics` Prometheus endpoint; sample Grafana dashboards in `deploy/`.

**Testing**:
- `Integration: checkout → trace span 'circulation.checkout' emitted`
- `Integration: /metrics exposes liberty_loans_total counter`

#### 11.3 — Performance & deployment

**What**: Partitioning/indexing for scale, load-test scenarios, production compose/Helm, backup/restore.

**Design**:
- Verify `loans`/`holds` query plans use the targeted partial indexes; document optional partitioning for very large consortia.
- Load-test (Locust) scenarios: checkout/checkin burst, discovery search QPS.
- Production: hardened Dockerfiles (non-root), Gunicorn worker tuning, healthchecks, `pg_dump`/restore runbook, Alembic migration-on-deploy gate.

**Testing**:
- `Perf: 500 concurrent checkouts → p95 latency within target, zero double-checkouts`
- `Perf: discovery search 200 QPS → p95 within target`
- `Integration: backup → restore into fresh DB → row counts match`

### Definition of Done
ASVS review complete; GDPR/FERPA operations tested; metrics/traces emitted; load-test targets met; production compose deploys with migrations applied; backup/restore verified.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema & Multi-Tenancy   ── required by everything
    │
Phase 2: Identity, Auth & Authorization       ── requires Phase 1
    │
Phase 3: Cataloguing & MARC Core              ── requires Phase 2  (value core)
    │
    ├── Phase 4: Circulation & Patron Services ── requires Phase 3
    │       │
    │       └── Phase 6: Protocols (Z39.50/SRU/SIP2/NCIP/OAI-PMH) ── requires Phases 3,4
    │
    └── Phase 5: AI-Assisted Cataloguing       ── requires Phase 3   (parallel with Phase 4)
            │
Phase 7: Acquisitions, ERM & OA Compliance     ── requires Phase 3 (+4 for items on receipt)
    │
Phase 8: Discovery Layer (OpenSearch)          ── requires Phases 3,7
    │
Phase 9: ILL (ISO 18626) + AI Reference/Contract ── requires Phases 3,6,7,8
    │
Phase 10: Frontend (Staff + Discovery/OPAC)    ── requires a stable REST API (3-9)
    │
Phase 11: Hardening, Observability & Deployment ── cross-cuts; runs last
```

**Parallelism opportunities**
- **Phase 4 (circulation)** and **Phase 5 (AI cataloguing)** can be built concurrently — both depend only on Phase 3.
- Within **Phase 6**, the four protocol adapters (Z39.50/SRU, OAI-PMH, SIP2, NCIP) are independent and can be split across developers once Phases 3/4 exist.
- **Phase 7's** three streams (acquisitions/EDIFACT, ERM/SUSHI, OA-compliance) are largely independent.
- The two **Phase 10** SPAs (staff console, discovery/OPAC) can be built in parallel against the stable API.
- **Phase 11** observability/security work can begin incrementally during Phases 6–10.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass (`pytest`), including the named edge cases.
3. Linting and formatting pass (`ruff check` + `ruff format --check`).
4. Static type checking passes (`mypy --strict`).
5. Docker build succeeds and `docker compose up` reaches healthy state.
6. The phase's feature works end-to-end (manual or E2E test through the API/CLI/UI).
7. New configuration options are documented in `.env.example` and the README.
8. New/changed API endpoints appear correctly in the auto-generated OpenAPI 3.1 spec.
9. Any schema change ships as a reversible Alembic migration (`upgrade`/`downgrade` both tested).
10. New mutations write appropriate `audit_log` entries with correct `protocol` attribution.
11. Standards-touching code is validated against a committed fixture (MARC, MARCXML, SIP2, ISO 18626, SUSHI, EDIFACT, OAI-PMH XML) where applicable.
12. AI features are tested with a mocked `LLMClient`; no test requires a live LLM/network call by default.
```
