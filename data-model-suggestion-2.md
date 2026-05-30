# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Academic Library Management · Created: 2026-05-26

## Philosophy

This model collapses the library services domain into five core tables by using JSONB columns to aggregate related data that would otherwise require separate tables. Each table represents a primary aggregate boundary: `institutions` (configuration, staff, policies, and product catalogue in JSONB), `patrons` (the full patron lifecycle including loans, holds, fines, and reading history), `catalogue` (bibliographic records with embedded items, holdings, course reserves, and circulation status), `electronic_resources` (ERM with embedded licence terms, usage statistics, and SUSHI configuration), and `ill_requests` (interlibrary loan workflow). An `ai_suggestions` table and `audit_log` complete the set.

The key insight is that academic libraries query primarily by patron ("what does this person have checked out?") or by catalogue record ("is this title available?"). By embedding items, holds, and active loans inside the catalogue record, and embedding loan history, holds, and fines inside the patron record, the two most common queries — patron account view and catalogue availability check — become single-row reads. The patron-side and catalogue-side loan data is denormalised (the same loan appears in both aggregates), with the catalogue aggregate being the source of truth for item availability and the patron aggregate being the source of truth for account status.

This approach is ideal for smaller academic and departmental libraries that need rapid deployment, flexible metadata schemas (accommodating MARC21, BIBFRAME, Dublin Core without schema changes), and minimal operational overhead.

**Best for:** Small to mid-size academic libraries prioritising development speed, flexible metadata, and single-query patron/catalogue lookups.

**Trade-offs:**
- Pro: Patron account view (loans, holds, fines) is a single-row read
- Pro: Catalogue availability (items, holds queue, reserves) is a single-row read
- Pro: New metadata schemas (MODS, EAD, custom) require no migration — just add to JSONB
- Pro: GDPR erasure is a single-row patron update
- Con: Loan data is denormalised across patron and catalogue aggregates — requires dual-write consistency
- Con: Large catalogues (millions of bib records with many items each) may have oversized rows
- Con: Cross-entity analytics (most popular titles across all patrons) require JSONB extraction
- Con: MARC field-level queries on the catalogue require GIN index on nested JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MARC21 | `catalogue.marc_json` stores full MARC record; API layer serialises to ISO 2709 / MarcXchange |
| BIBFRAME 2.0 | `catalogue.bibframe_json` stores linked-data representation alongside MARC |
| Dublin Core | `catalogue.dc_json` for OAI-PMH metadata export |
| Z39.50 / SRU | Searches map to GIN queries on `catalogue` JSONB fields |
| OAI-PMH 2.0 | `catalogue` with `oai_identifier` and `updated_at` supports incremental harvesting |
| NCIP / SIP2 | Patron account and item status served from `patrons` and `catalogue` aggregates |
| ISO 18626 | `ill_requests` table models full ISO 18626 message lifecycle |
| COUNTER 5 / SUSHI | Usage data embedded in `electronic_resources.usage_json` |
| EDIFACT | Acquisition data embedded in `catalogue.acquisitions_json` |
| KBART 2 | Coverage data embedded in `electronic_resources.coverage_json` |
| OpenURL | ISBN, ISSN, DOI fields on `catalogue` support context-object resolution |
| SAML / Shibboleth | `patrons` stores IdP reference for federated authentication |
| GDPR / FERPA | Consent fields in `patrons`; single-row erasure |
| Plan S / NIH / UKRI | OA compliance tracked in `catalogue.oa_json` |

---

## Institution Configuration

```sql
CREATE TABLE institutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    institution_type TEXT NOT NULL CHECK (institution_type IN (
                        'university', 'college', 'research_institute',
                        'consortium', 'departmental', 'special'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'provisioning', 'active', 'suspended', 'archived'
                    )),
    country_code    TEXT NOT NULL,
    identifiers_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"isni": "0000 0001 2181 7878", "ror": "https://ror.org/...", "ringgold": "6387", "oclc_symbol": "OXF"}
    auth_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "shibboleth_entity_id": "https://idp.ox.ac.uk/shibboleth",
    --   "openathens_org": "oxford",
    --   "ezproxy_url": "https://ezproxy.bodleian.ox.ac.uk/login",
    --   "edugain_member": true
    -- }
    policies_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "loan_policies": {
    --     "undergraduate": {"days": 28, "renewals": 3, "max_loans": 20},
    --     "faculty": {"days": 90, "renewals": 5, "max_loans": 50}
    --   },
    --   "fine_rates": {"overdue_per_day_cents": 20, "lost_processing_cents": 1000},
    --   "hold_expiry_days": 7,
    --   "anonymise_loans_after_days": 180,
    --   "currency": "GBP",
    --   "timezone": "Europe/London"
    -- }
    staff_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "uuid", "email": "librarian@uni.ac.uk", "name": "Jane Smith",
    --    "role": "cataloguer", "status": "active", "last_login": "2026-05-25"}
    -- ]
    integrations_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "sip2_enabled": true, "sip2_terminals": ["circ-desk-1", "self-check-2"],
    --   "oai_pmh_base_url": "https://library.uni.ac.uk/oai",
    --   "z39_50_port": 2100,
    --   "ill_enabled": true, "ill_default_lenders": ["BL", "NLS", "OXF"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_institutions_status ON institutions (status);
CREATE INDEX idx_institutions_identifiers ON institutions USING GIN (identifiers_json);
```

## Patron Aggregate

```sql
CREATE TABLE patrons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    external_id         TEXT,
    barcode             TEXT,
    patron_type         TEXT NOT NULL CHECK (patron_type IN (
                            'undergraduate', 'postgraduate', 'doctoral',
                            'faculty', 'staff', 'visiting_scholar',
                            'alumni', 'community', 'ill_partner',
                            'departmental', 'system'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'suspended', 'expired', 'blocked',
                            'anonymised'
                        )),
    -- Identity
    identity_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "first_name": "Alice", "last_name": "Chen",
    --   "email": "alice.chen@uni.ac.uk", "phone": "+44...",
    --   "department": "Computer Science", "faculty": "Engineering",
    --   "orcid": "0000-0002-1234-5678",
    --   "idp_provider": "https://idp.uni.ac.uk/shibboleth",
    --   "idp_subject": "alice.chen@uni.ac.uk"
    -- }
    -- Active loans
    loans_json          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "loan-uuid", "item_barcode": "30000012345",
    --     "bib_id": "bib-uuid", "title": "Introduction to Algorithms",
    --     "call_number": "QA76.73 .C55",
    --     "checkout_at": "2026-05-01T10:30:00Z",
    --     "due_date": "2026-05-29T23:59:00Z",
    --     "renewal_count": 1, "max_renewals": 3,
    --     "status": "active", "loan_type": "standard",
    --     "recalled": false
    --   }
    -- ]
    -- Active holds
    holds_json          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "hold-uuid", "bib_id": "bib-uuid",
    --     "title": "Deep Learning", "hold_type": "title",
    --     "status": "pending", "queue_position": 2,
    --     "pickup_location": "Main Library", "placed_at": "2026-05-20"
    --   }
    -- ]
    -- Fines & payments
    fines_json          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "outstanding_cents": 240,
    --   "block_threshold_cents": 2500,
    --   "items": [
    --     {"id": "fine-uuid", "type": "overdue", "amount_cents": 240,
    --      "paid_cents": 0, "description": "Overdue: Introduction to Algorithms",
    --      "accrual_date": "2026-05-30"}
    --   ],
    --   "payment_history": [
    --     {"date": "2026-04-15", "amount_cents": 500, "method": "card", "reference": "PAY-001"}
    --   ]
    -- }
    -- Circulation limits
    limits_json         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"max_loans": 20, "max_holds": 10, "max_renewals": 3, "loan_period_days": 28}
    -- Privacy
    privacy_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "gdpr_consent": {"reading_history": false, "search_history": false, "analytics": true, "date": "2026-01-15"},
    --   "anonymise_after_days": 180,
    --   "data_retention_note": "FERPA applies"
    -- }
    anonymised_at       TIMESTAMPTZ,
    -- Engagement
    engagement_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "expiry_date": "2027-06-30",
    --   "last_activity": "2026-05-25",
    --   "total_loans_lifetime": 87,
    --   "ill_requests": 3,
    --   "preferred_subjects": ["computer science", "mathematics"]
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patrons_institution ON patrons (institution_id);
CREATE INDEX idx_patrons_barcode ON patrons (institution_id, barcode);
CREATE INDEX idx_patrons_external ON patrons (institution_id, external_id);
CREATE INDEX idx_patrons_type ON patrons (institution_id, patron_type);
CREATE INDEX idx_patrons_status ON patrons (institution_id, status);
CREATE INDEX idx_patrons_identity ON patrons USING GIN (identity_json);
CREATE INDEX idx_patrons_loans ON patrons USING GIN (loans_json);
CREATE INDEX idx_patrons_fines ON patrons USING GIN (fines_json);
```

## Catalogue Aggregate

```sql
CREATE TABLE catalogue (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    -- Core identifiers
    control_number      TEXT,
    oclc_number         TEXT,
    isbn                TEXT[] NOT NULL DEFAULT '{}',
    issn                TEXT[] NOT NULL DEFAULT '{}',
    doi                 TEXT,
    lccn                TEXT,
    oai_identifier      TEXT,
    -- Descriptive (search-optimised top-level columns)
    title               TEXT NOT NULL,
    subtitle            TEXT,
    authors             TEXT[] NOT NULL DEFAULT '{}',
    publisher           TEXT,
    publication_date    TEXT,
    record_type         TEXT NOT NULL CHECK (record_type IN (
                            'book', 'serial', 'journal', 'e_book', 'e_journal',
                            'thesis', 'dissertation', 'conference_paper',
                            'report', 'map', 'score', 'recording',
                            'video', 'dataset', 'software', 'mixed_material',
                            'archival', 'website'
                        )),
    material_type       TEXT NOT NULL CHECK (material_type IN (
                            'print', 'electronic', 'microform', 'audiovisual',
                            'digital', 'mixed'
                        )),
    language            TEXT,
    -- Classification
    classification_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "lcc": "QA76.73 .C55",
    --   "dewey": "005.133",
    --   "subject_headings": ["Computer algorithms", "Data structures"],
    --   "mesh": [],
    --   "local_subjects": ["CS Year 2 Core"]
    -- }
    -- Full MARC record
    marc_json           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "leader": "01234nam a2200321 a 4500",
    --   "control_fields": [{"tag": "001", "value": "ocm12345678"}, {"tag": "008", "value": "..."}],
    --   "data_fields": [
    --     {"tag": "100", "ind1": "1", "ind2": " ", "subfields": [{"code": "a", "value": "Cormen, Thomas H."}]},
    --     {"tag": "245", "ind1": "1", "ind2": "0", "subfields": [{"code": "a", "value": "Introduction to algorithms /"}]}
    --   ]
    -- }
    -- Linked data
    bibframe_json       JSONB NOT NULL DEFAULT '{}',
    dc_json             JSONB NOT NULL DEFAULT '{}',
    -- Items (physical holdings)
    items_json          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "item-uuid", "barcode": "30000012345",
    --     "call_number": "QA76.73 .C55", "copy": 1,
    --     "type": "circulating", "status": "checked_out",
    --     "location": "Main Library", "sublocation": "Level 3",
    --     "loan_type": "standard",
    --     "condition": "good", "replacement_cost_cents": 5500,
    --     "current_loan": {
    --       "patron_id": "patron-uuid", "checkout_at": "2026-05-01",
    --       "due_date": "2026-05-29", "renewals": 1
    --     }
    --   },
    --   {
    --     "id": "item-uuid-2", "barcode": "30000012346",
    --     "call_number": "QA76.73 .C55", "copy": 2,
    --     "type": "reference", "status": "available",
    --     "location": "Science Library", "loan_type": "in_library_only"
    --   }
    -- ]
    -- Holds queue
    holds_json          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "hold-uuid", "patron_id": "patron-uuid", "type": "title",
    --    "status": "pending", "queue_position": 1, "pickup": "Main Library", "placed_at": "2026-05-20"}
    -- ]
    -- Course reserves
    reserves_json       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"course_code": "CS201", "course_name": "Algorithms", "instructor": "Prof. Smith",
    --    "term": "2026-autumn", "reserve_type": "required", "loan_period": "overnight",
    --    "copyright_status": "owned"}
    -- ]
    -- Acquisitions
    acquisitions_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"order_number": "PO-2026-001", "type": "firm_order", "status": "received",
    --    "vendor": "Baker & Taylor", "fund": "SCIBOOKS", "cost_cents": 5500,
    --    "ordered_at": "2026-03-01", "received_at": "2026-03-15"}
    -- ]
    -- Open access
    oa_json             JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "status": "green", "deposit_required": true,
    --   "funder_mandates": ["ukri", "plan_s"],
    --   "compliance_status": "compliant",
    --   "repository_url": "https://ora.ox.ac.uk/objects/...",
    --   "embargo_end": "2027-01-01"
    -- }
    -- Source & cataloguing
    source              TEXT NOT NULL CHECK (source IN (
                            'original_cataloguing', 'z39_50_import', 'sru_import',
                            'oclc_worldcat', 'oai_pmh_harvest', 'vendor_record',
                            'marc_batch_import', 'bibframe_import', 'ai_assisted'
                        )),
    cataloguing_level   TEXT CHECK (cataloguing_level IN (
                            'full', 'core', 'minimal', 'preliminary'
                        )),
    -- Computed counts
    total_items         INTEGER NOT NULL DEFAULT 0,
    available_items     INTEGER NOT NULL DEFAULT 0,
    total_holds         INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_catalogue_institution ON catalogue (institution_id);
CREATE INDEX idx_catalogue_control ON catalogue (institution_id, control_number);
CREATE INDEX idx_catalogue_oclc ON catalogue (oclc_number);
CREATE INDEX idx_catalogue_isbn ON catalogue USING GIN (isbn);
CREATE INDEX idx_catalogue_issn ON catalogue USING GIN (issn);
CREATE INDEX idx_catalogue_doi ON catalogue (doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_catalogue_type ON catalogue (institution_id, record_type);
CREATE INDEX idx_catalogue_authors ON catalogue USING GIN (authors);
CREATE INDEX idx_catalogue_title ON catalogue USING gin (to_tsvector('english', title));
CREATE INDEX idx_catalogue_classification ON catalogue USING GIN (classification_json);
CREATE INDEX idx_catalogue_marc ON catalogue USING GIN (marc_json);
CREATE INDEX idx_catalogue_items ON catalogue USING GIN (items_json);
CREATE INDEX idx_catalogue_oa ON catalogue USING GIN (oa_json);
CREATE INDEX idx_catalogue_updated ON catalogue (institution_id, updated_at DESC);
```

## Electronic Resource Management

```sql
CREATE TABLE electronic_resources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    catalogue_id        UUID REFERENCES catalogue(id),
    resource_name       TEXT NOT NULL,
    resource_type       TEXT NOT NULL CHECK (resource_type IN (
                            'e_journal', 'e_book', 'e_book_package',
                            'database', 'streaming_media', 'dataset',
                            'open_access_journal', 'repository_item'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'trial', 'negotiating', 'active', 'lapsed',
                            'cancelled', 'free'
                        )),
    publisher           TEXT,
    provider            TEXT,
    platform_url        TEXT,
    -- Licence (full terms in JSONB)
    licence_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "start": "2026-01-01", "end": "2026-12-31",
    --   "type": "subscription",
    --   "terms": {
    --     "simultaneous_users": "unlimited",
    --     "ill_permitted": true, "course_reserves": true,
    --     "text_mining": false, "archival_rights": "post_cancellation",
    --     "authorised_users": "students_staff_walk_in",
    --     "gdpr_dpa_signed": true
    --   },
    --   "ai_extracted_flags": [
    --     {"clause": "Section 4.2", "flag": "no_text_mining", "severity": "warning"}
    --   ]
    -- }
    -- Financial
    financial_json      JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "annual_cost_cents": 1250000, "currency": "GBP",
    --   "fund_code": "EJOURNALS", "fiscal_year": 2026,
    --   "cost_per_use_cents": 342,
    --   "price_history": [
    --     {"year": 2024, "cost_cents": 1100000},
    --     {"year": 2025, "cost_cents": 1175000}
    --   ],
    --   "oa_read_and_publish": true, "apc_discount_pct": 20.0
    -- }
    -- COUNTER / SUSHI usage
    usage_json          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "sushi_endpoint": "https://publisher.com/sushi",
    --   "counter_compliant": true,
    --   "last_harvest": "2026-05-01",
    --   "reports": {
    --     "TR_J1": [
    --       {"period": "2026-01", "total_item_requests": 450},
    --       {"period": "2026-02", "total_item_requests": 380}
    --     ]
    --   },
    --   "ytd_total_item_requests": 2100,
    --   "ytd_cost_per_use_cents": 595
    -- }
    -- Coverage (KBART)
    coverage_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"start_date": "1997-01-01", "end_date": null, "embargo_months": 0, "depth": "fulltext"}
    -- Access
    access_json         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"method": "shibboleth", "proxy_url": "https://ezproxy.uni.ac.uk/login?url=...", "openathens_enabled": true}
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_erm_institution ON electronic_resources (institution_id);
CREATE INDEX idx_erm_catalogue ON electronic_resources (catalogue_id);
CREATE INDEX idx_erm_status ON electronic_resources (institution_id, status);
CREATE INDEX idx_erm_type ON electronic_resources (institution_id, resource_type);
CREATE INDEX idx_erm_publisher ON electronic_resources (institution_id, publisher);
CREATE INDEX idx_erm_licence ON electronic_resources USING GIN (licence_json);
CREATE INDEX idx_erm_usage ON electronic_resources USING GIN (usage_json);
CREATE INDEX idx_erm_financial ON electronic_resources USING GIN (financial_json);
```

## Interlibrary Loans

```sql
CREATE TABLE ill_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    patron_id           UUID REFERENCES patrons(id),
    catalogue_id        UUID REFERENCES catalogue(id),
    request_type        TEXT NOT NULL CHECK (request_type IN (
                            'borrowing', 'lending', 'document_supply'
                        )),
    status              TEXT NOT NULL DEFAULT 'new' CHECK (status IN (
                            'new', 'validated', 'sent', 'received_by_lender',
                            'will_supply', 'shipped', 'received',
                            'checked_out_to_patron', 'recalled',
                            'returned_by_patron', 'returned_to_lender',
                            'completed', 'unfilled', 'cancelled', 'expired'
                        )),
    -- ISO 18626 + bibliographic
    request_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "iso18626_id": "ILL-2026-0042",
    --   "requesting_agency": "OXF", "supplying_agency": "BL",
    --   "service_type": "loan",
    --   "bib": {"title": "...", "author": "...", "isbn": "...", "volume": "3", "pages": "45-67"},
    --   "need_before": "2026-06-30",
    --   "delivery_method": "physical",
    --   "copyright": "cla_licence"
    -- }
    -- Fulfilment
    fulfilment_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "shipped_at": "2026-05-22", "received_at": "2026-05-24",
    --   "due_date": "2026-06-21", "returned_at": null,
    --   "tracking": "RM-1234567890"
    -- }
    -- Financial
    fee_json            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"fee_cents": 1200, "currency": "GBP", "ifm_id": "IFM-2026-001", "invoiced": true}
    -- Lender rota
    lender_rota         TEXT[] NOT NULL DEFAULT '{}',
    current_lender_idx  INTEGER NOT NULL DEFAULT 0,
    unfilled_reason     TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ill_institution ON ill_requests (institution_id);
CREATE INDEX idx_ill_patron ON ill_requests (patron_id);
CREATE INDEX idx_ill_status ON ill_requests (institution_id, status);
CREATE INDEX idx_ill_type ON ill_requests (institution_id, request_type);
CREATE INDEX idx_ill_request ON ill_requests USING GIN (request_json);
CREATE INDEX idx_ill_created ON ill_requests (institution_id, created_at DESC);
```

## AI Suggestions & Audit

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'marc_field', 'subject_heading', 'authority_match',
                            'classification', 'duplicate_detection',
                            'licence_term_extraction', 'licence_red_flag',
                            'oa_compliance', 'acquisition_recommendation',
                            'reference_answer', 'usage_forecast',
                            'catalogue_enrichment'
                        )),
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'accepted', 'rejected',
                            'auto_applied', 'expired'
                        )),
    confidence          REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    target_entity_type  TEXT NOT NULL CHECK (target_entity_type IN (
                            'catalogue', 'patron', 'electronic_resource',
                            'ill_request', 'institution'
                        )),
    target_entity_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    detail_json         JSONB NOT NULL DEFAULT '{}',
    explanation         TEXT,
    model_id            TEXT,
    model_version       TEXT,
    reviewed_by         TEXT,
    reviewed_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_institution ON ai_suggestions (institution_id);
CREATE INDEX idx_ai_type ON ai_suggestions (institution_id, suggestion_type);
CREATE INDEX idx_ai_status ON ai_suggestions (institution_id, status);
CREATE INDEX idx_ai_target ON ai_suggestions (target_entity_type, target_entity_id);

CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user', 'patron', 'system', 'ai',
                            'sip2_terminal', 'z39_50_client',
                            'oai_harvester', 'sushi_harvester',
                            'edifact_processor', 'iso18626_partner'
                        )),
    actor_id            TEXT NOT NULL,
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB NOT NULL DEFAULT '{}',
    ip_address          TEXT,
    protocol            TEXT,
    gdpr_relevant       BOOLEAN NOT NULL DEFAULT false,
    ferpa_relevant      BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_institution ON audit_log (institution_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
CREATE INDEX idx_audit_privacy ON audit_log (institution_id) WHERE gdpr_relevant = true OR ferpa_relevant = true;
```

---

## Aggregate Query Examples

### Patron account view (single-row read)

```sql
SELECT identity_json, patron_type, status,
       loans_json, holds_json, fines_json, limits_json,
       engagement_json, privacy_json
FROM patrons
WHERE institution_id = $1 AND id = $2;
```

### Catalogue availability check (single-row read)

```sql
SELECT title, authors, record_type, material_type,
       classification_json, items_json, holds_json,
       total_items, available_items, total_holds,
       reserves_json, oa_json
FROM catalogue
WHERE institution_id = $1 AND id = $2;
```

### Find all available copies of a title by ISBN

```sql
SELECT id, title, items_json
FROM catalogue
WHERE institution_id = $1
  AND isbn @> ARRAY['978-0262046305']
  AND available_items > 0;
```

### ERM cost-per-use ranking

```sql
SELECT resource_name, publisher,
       (financial_json->>'annual_cost_cents')::BIGINT AS cost,
       (usage_json->>'ytd_total_item_requests')::INTEGER AS uses,
       (financial_json->>'cost_per_use_cents')::BIGINT AS cpu
FROM electronic_resources
WHERE institution_id = $1 AND status = 'active'
ORDER BY (financial_json->>'cost_per_use_cents')::BIGINT DESC NULLS LAST;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Institution Configuration | 1 | Staff, policies, integrations all in JSONB |
| Patron Aggregate | 1 | Loans, holds, fines, privacy all embedded |
| Catalogue Aggregate | 1 | MARC, items, holds, reserves, acquisitions, OA all embedded |
| Electronic Resources | 1 | Licence terms, COUNTER usage, financial history embedded |
| Interlibrary Loans | 1 | ISO 18626 lifecycle with lender rota |
| AI & Audit | 2 | AI suggestions + partitioned audit log |
| **Total** | **7** | |

---

## Key Design Decisions

1. **Patron as BSS-equivalent aggregate** — loans, holds, fines, and engagement data are embedded in the patron row because the most common circulation operation (patron account lookup at the desk or via OPAC) needs all of this data simultaneously. A SIP2 Patron Status Request returns a single JSON extraction.

2. **Catalogue as inventory aggregate** — items, holds queue, course reserves, and acquisitions are embedded in the catalogue row because the most common discovery operation (item availability check) needs item status, hold count, and reserve status together. This also makes the OAI-PMH response a single-row read.

3. **Denormalised loans across patron and catalogue** — the same loan appears in `patrons.loans_json` (patron's perspective) and `catalogue.items_json[].current_loan` (item's perspective). The catalogue aggregate is the source of truth for item availability; the patron aggregate is the source of truth for account status. Dual-write consistency is managed at the application layer.

4. **MARC record as structured JSONB** — `marc_json` stores leader, control fields, and data fields as structured JSON. This supports full MARC round-tripping (no data loss) while enabling GIN-indexed queries on tag/subfield values for discovery.

5. **ERM with embedded COUNTER data** — usage statistics (harvested via SUSHI) are stored directly in `electronic_resources.usage_json` alongside licence terms and financial data. This makes the cost-per-use calculation a single-row read — the ERM manager's most common query.

6. **ILL as a separate table** — interlibrary loans involve external agencies and have their own lifecycle (ISO 18626 states). Unlike internal loans (embedded in patron/catalogue), ILL requests need independent tracking, lender rota management, and fee accounting that doesn't map to the patron or catalogue aggregate.

7. **Staff in institution JSONB** — for small to mid-size libraries with 5-50 staff, embedding users in `institutions.staff_json` avoids a separate table. For larger deployments, this would need promotion to its own table.

8. **GDPR erasure is a single update** — anonymising a patron means updating one row: clear `identity_json`, set `anonymised_at`, optionally clear `loans_json` entries older than the retention period. No cascading across multiple tables.

9. **Open-access compliance in catalogue** — `oa_json` on each catalogue record tracks funder mandates, compliance status, and repository deposit URLs, enabling a compliance dashboard via a single query with GIN index filtering.

10. **Flexible metadata via multiple JSONB representations** — a single catalogue record can hold MARC (`marc_json`), BIBFRAME (`bibframe_json`), and Dublin Core (`dc_json`) simultaneously. New formats (MODS, EAD) can be added as additional JSONB fields without migration.
