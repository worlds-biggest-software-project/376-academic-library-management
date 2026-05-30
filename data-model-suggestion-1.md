# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Academic Library Management · Created: 2026-05-26

## Philosophy

This model assigns a dedicated table to every core library domain concept — institutions, patrons, bibliographic records, items, loans, holds, fines, acquisitions, electronic resources, interlibrary loans, and course reserves — connected by foreign keys that enforce referential integrity across the full library services lifecycle. The design aligns directly with the entity structures used in MARC21, NCIP, ISO 18626, and COUNTER 5 so that data exchange with external systems (Z39.50 servers, ILL partners, SUSHI endpoints) maps naturally to database entities.

The normalized structure makes the relationships between bibliographic records, physical items, loans, and patrons explicit and queryable — answering questions like "which items for this bib record are currently on loan, on hold, or in transit?" requires only standard JOINs. Acquisition orders link to bib records and items, enabling fund-to-title traceability. Electronic resource licences link to bib records, enabling cost-per-use analysis when joined with COUNTER usage statistics.

This is the safest choice for libraries with established cataloguing workflows who need MARC21/BIBFRAME compliance, clean data exports, and straightforward integration with external systems via standard library protocols.

**Best for:** Academic libraries and consortia requiring strict standards compliance, clean MARC21/BIBFRAME exports, and auditable data lineage.

**Trade-offs:**
- Pro: Direct mapping to MARC21 record structure and library protocol entities (NCIP, SIP2, ISO 18626)
- Pro: Referential integrity enforced across bib record → item → loan → patron chain
- Pro: Cost-per-use analysis (electronic resources → usage stats → bib records) is a natural JOIN
- Pro: GDPR/FERPA patron data operations target well-defined tables
- Con: 14 tables with foreign keys increases migration complexity
- Con: Bib record JSONB (MARC fields) still requires application-layer validation
- Con: High-volume circulation (large consortia) may stress loan/hold tables without partitioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MARC21 (Library of Congress) | `bib_records` stores MARC leader, control fields, and data fields in structured JSONB |
| BIBFRAME 2.0 | `bib_records.bibframe_json` holds linked-data Work/Instance/Item representation |
| Dublin Core (DCMI) | `bib_records.dc_json` stores Dublin Core metadata elements for OAI-PMH export |
| Z39.50 / SRU | `bib_records` structure supports Z39.50 attribute-based search and SRU response mapping |
| OAI-PMH 2.0 | `bib_records` with `oai_identifier` and `updated_at` supports incremental harvesting |
| ISO 2709 / MarcXchange | MARC field storage in `bib_records.marc_fields_json` serialises to ISO 2709 / MarcXchange XML |
| NCIP (Z39.83) | `loans`, `holds`, `patrons`, `items` map to NCIP LookupItem, CheckOutItem, RequestItem services |
| SIP2 | `loans` and `items` support SIP2 self-checkout message fields |
| ISO 18626 | `ill_requests` table models ISO 18626 message lifecycle (request → ship → receive → return) |
| COUNTER 5 / SUSHI | `usage_statistics` stores COUNTER TR/DR/PR reports harvested via SUSHI API |
| EDIFACT | `acquisition_orders` models EDIFACT ORDERS/INVOIC message structures |
| KBART 2 | `electronic_resources` supports KBART title-list import for knowledge base synchronisation |
| OpenURL (Z39.88) | `bib_records` fields (ISSN, ISBN, DOI) support OpenURL context-object resolution |
| SAML 2.0 / Shibboleth | `patrons` stores IdP references for federated authentication via eduGAIN/Shibboleth |
| GDPR / FERPA | `patrons` includes consent and anonymisation fields; `audit_log` tracks data access |
| Plan S / NIH / UKRI | `electronic_resources` tracks open-access compliance status per funder mandate |

---

## Institution & Staff Management

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
    country_code    TEXT NOT NULL,              -- ISO 3166-1 alpha-2
    isni            TEXT,                       -- ISNI institutional identifier
    ror_id          TEXT,                       -- Research Organization Registry ID
    ringgold_id     TEXT,                       -- Ringgold identifier for publisher matching
    oclc_symbol     TEXT,                       -- OCLC institutional symbol
    oai_base_url    TEXT,                       -- OAI-PMH base URL for this institution
    shibboleth_entity_id TEXT,                  -- SAML entity ID
    ezproxy_url     TEXT,
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "currency": "GBP",
    --   "loan_policies": {"student": {"days": 28, "renewals": 3}, "faculty": {"days": 90, "renewals": 5}},
    --   "fine_rates": {"overdue_per_day_cents": 20, "lost_item_processing_cents": 1000},
    --   "hold_expiry_days": 7,
    --   "sip2_enabled": true,
    --   "ill_enabled": true,
    --   "timezone": "Europe/London"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_institutions_status ON institutions (status);
CREATE INDEX idx_institutions_oclc ON institutions (oclc_symbol);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
                        'admin', 'cataloguer', 'circulation_desk',
                        'acquisitions', 'erm_manager', 'ill_operator',
                        'reference_librarian', 'systems_librarian',
                        'collection_development', 'read_only'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'suspended', 'deactivated'
                    )),
    idp_provider    TEXT,                       -- SAML IdP entity ID
    idp_subject     TEXT,
    permissions     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, email)
);

CREATE INDEX idx_users_institution ON users (institution_id);
CREATE INDEX idx_users_role ON users (institution_id, role);
```

## Patron Management

```sql
CREATE TABLE patrons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    external_id         TEXT,                   -- campus ID / student number
    barcode             TEXT,                   -- library barcode
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
    first_name          TEXT,
    last_name           TEXT,
    email               TEXT,
    phone               TEXT,
    department          TEXT,
    faculty             TEXT,
    orcid               TEXT,                   -- ORCID researcher identifier
    -- Authentication
    idp_provider        TEXT,                   -- Shibboleth / OpenAthens entity ID
    idp_subject         TEXT,
    -- Circulation limits
    max_loans           INTEGER NOT NULL DEFAULT 20,
    max_holds           INTEGER NOT NULL DEFAULT 10,
    max_renewals        INTEGER NOT NULL DEFAULT 3,
    loan_period_days    INTEGER NOT NULL DEFAULT 28,
    -- Financial
    outstanding_fines_cents BIGINT NOT NULL DEFAULT 0,
    block_threshold_cents   BIGINT NOT NULL DEFAULT 2500,
    -- Privacy
    gdpr_consent        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"reading_history": false, "search_history": false, "analytics": true, "consent_date": "2026-01-15"}
    anonymise_after_days INTEGER,               -- auto-anonymise loan history after N days
    anonymised_at       TIMESTAMPTZ,
    -- Dates
    expiry_date         DATE,
    last_activity_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patrons_institution ON patrons (institution_id);
CREATE INDEX idx_patrons_barcode ON patrons (institution_id, barcode);
CREATE INDEX idx_patrons_external ON patrons (institution_id, external_id);
CREATE INDEX idx_patrons_type ON patrons (institution_id, patron_type);
CREATE INDEX idx_patrons_status ON patrons (institution_id, status);
CREATE INDEX idx_patrons_email ON patrons (institution_id, email);
CREATE INDEX idx_patrons_orcid ON patrons (orcid) WHERE orcid IS NOT NULL;
```

## Bibliographic Records

```sql
CREATE TABLE bib_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    -- Identifiers
    control_number      TEXT,                   -- MARC 001
    oclc_number         TEXT,                   -- OCLC accession number
    isbn                TEXT[] NOT NULL DEFAULT '{}',
    issn                TEXT[] NOT NULL DEFAULT '{}',
    doi                 TEXT,
    lccn                TEXT,                   -- Library of Congress Control Number
    oai_identifier      TEXT,                   -- OAI-PMH identifier
    -- Descriptive metadata
    title               TEXT NOT NULL,
    subtitle            TEXT,
    authors             TEXT[] NOT NULL DEFAULT '{}',
    editors             TEXT[] NOT NULL DEFAULT '{}',
    publisher           TEXT,
    publication_date    TEXT,                   -- free-form per MARC 260$c
    publication_place   TEXT,
    edition             TEXT,
    physical_desc       TEXT,                   -- MARC 300
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
    language            TEXT,                   -- MARC 008/35-37 language code
    -- Classification
    lcc_call_number     TEXT,                   -- Library of Congress Classification
    dewey_number        TEXT,                   -- Dewey Decimal
    subject_headings    TEXT[] NOT NULL DEFAULT '{}',    -- LCSH
    mesh_headings       TEXT[] NOT NULL DEFAULT '{}',    -- MeSH (medical)
    -- MARC21 full record
    marc_leader         TEXT,                   -- 24-char MARC leader
    marc_fields_json    JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"tag": "100", "ind1": "1", "ind2": " ", "subfields": [{"code": "a", "value": "Smith, John,"}]},
    --   {"tag": "245", "ind1": "1", "ind2": "0", "subfields": [{"code": "a", "value": "Introduction to algorithms /"}]},
    --   {"tag": "650", "ind1": " ", "ind2": "0", "subfields": [{"code": "a", "value": "Computer algorithms."}]}
    -- ]
    -- Linked data
    bibframe_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"work_uri": "http://id.loc.gov/resources/works/...", "instance_uri": "...", "bf_type": "Text"}
    dc_json             JSONB NOT NULL DEFAULT '{}',
    -- Example: {"dc:title": "...", "dc:creator": ["..."], "dc:subject": ["..."], "dc:date": "2025"}
    -- Open access
    oa_status           TEXT CHECK (oa_status IN (
                            'gold', 'green', 'hybrid', 'bronze',
                            'closed', 'unknown'
                        )),
    oa_deposit_required BOOLEAN NOT NULL DEFAULT false,
    oa_funder_mandates  TEXT[] NOT NULL DEFAULT '{}', -- e.g. {'plan_s', 'nih', 'ukri'}
    oa_compliance_status TEXT CHECK (oa_compliance_status IN (
                            'compliant', 'non_compliant', 'pending', 'exempt', 'not_applicable'
                        )),
    -- Provenance
    source              TEXT NOT NULL CHECK (source IN (
                            'original_cataloguing', 'z39_50_import', 'sru_import',
                            'oclc_worldcat', 'oai_pmh_harvest', 'vendor_record',
                            'marc_batch_import', 'bibframe_import', 'ai_assisted'
                        )),
    cataloguing_level   TEXT CHECK (cataloguing_level IN (
                            'full', 'core', 'minimal', 'preliminary'
                        )),
    catalogued_by       UUID REFERENCES users(id),
    -- Counts (denormalised for performance)
    total_items         INTEGER NOT NULL DEFAULT 0,
    available_items     INTEGER NOT NULL DEFAULT 0,
    total_holds         INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bib_institution ON bib_records (institution_id);
CREATE INDEX idx_bib_control ON bib_records (institution_id, control_number);
CREATE INDEX idx_bib_oclc ON bib_records (oclc_number);
CREATE INDEX idx_bib_isbn ON bib_records USING GIN (isbn);
CREATE INDEX idx_bib_issn ON bib_records USING GIN (issn);
CREATE INDEX idx_bib_doi ON bib_records (doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_bib_type ON bib_records (institution_id, record_type);
CREATE INDEX idx_bib_subjects ON bib_records USING GIN (subject_headings);
CREATE INDEX idx_bib_mesh ON bib_records USING GIN (mesh_headings);
CREATE INDEX idx_bib_authors ON bib_records USING GIN (authors);
CREATE INDEX idx_bib_marc ON bib_records USING GIN (marc_fields_json);
CREATE INDEX idx_bib_title ON bib_records USING gin (to_tsvector('english', title));
CREATE INDEX idx_bib_oa ON bib_records (institution_id, oa_compliance_status) WHERE oa_deposit_required = true;
CREATE INDEX idx_bib_updated ON bib_records (institution_id, updated_at DESC);
```

## Items (Physical Holdings)

```sql
CREATE TABLE items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    bib_record_id       UUID NOT NULL REFERENCES bib_records(id),
    barcode             TEXT,
    call_number         TEXT,
    copy_number         INTEGER NOT NULL DEFAULT 1,
    volume              TEXT,                   -- for multi-volume works
    item_type           TEXT NOT NULL CHECK (item_type IN (
                            'circulating', 'reference', 'reserve',
                            'special_collections', 'periodical',
                            'bound_journal', 'thesis', 'av_material',
                            'microform', 'map', 'equipment'
                        )),
    status              TEXT NOT NULL DEFAULT 'available' CHECK (status IN (
                            'available', 'checked_out', 'on_hold',
                            'in_transit', 'in_processing', 'on_order',
                            'missing', 'lost', 'damaged', 'withdrawn',
                            'at_bindery', 'on_reserve'
                        )),
    location            TEXT NOT NULL,          -- shelving location
    sublocation         TEXT,                   -- floor, wing, shelf
    branch              TEXT,                   -- for multi-branch institutions
    temporary_location  TEXT,                   -- if temporarily moved
    -- Circulation policy
    loan_type           TEXT NOT NULL DEFAULT 'standard' CHECK (loan_type IN (
                            'standard', 'short_loan', 'overnight',
                            'in_library_only', 'non_circulating',
                            'course_reserve', 'ill_lendable'
                        )),
    -- Physical
    condition           TEXT CHECK (condition IN (
                            'good', 'fair', 'poor', 'damaged', 'fragile'
                        )),
    replacement_cost_cents BIGINT,
    -- SIP2 fields
    sip2_item_type      TEXT,                   -- SIP2 item type code
    magnetic_media      BOOLEAN NOT NULL DEFAULT false,
    -- Dates
    acquired_at         DATE,
    last_seen_at        TIMESTAMPTZ,
    last_charged_at     TIMESTAMPTZ,
    inventory_checked_at TIMESTAMPTZ,
    withdrawn_at        TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_items_institution ON items (institution_id);
CREATE INDEX idx_items_bib ON items (bib_record_id);
CREATE INDEX idx_items_barcode ON items (institution_id, barcode);
CREATE INDEX idx_items_status ON items (institution_id, status);
CREATE INDEX idx_items_location ON items (institution_id, location, sublocation);
CREATE INDEX idx_items_type ON items (institution_id, item_type);
CREATE INDEX idx_items_call ON items (institution_id, call_number);
```

## Circulation

```sql
CREATE TABLE loans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    patron_id           UUID NOT NULL REFERENCES patrons(id),
    item_id             UUID NOT NULL REFERENCES items(id),
    bib_record_id       UUID NOT NULL REFERENCES bib_records(id),
    loan_type           TEXT NOT NULL CHECK (loan_type IN (
                            'standard', 'short_loan', 'overnight',
                            'course_reserve', 'ill', 'in_library'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'renewed', 'recalled', 'overdue',
                            'returned', 'lost', 'claimed_returned'
                        )),
    checkout_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    due_date            TIMESTAMPTZ NOT NULL,
    returned_at         TIMESTAMPTZ,
    renewal_count       INTEGER NOT NULL DEFAULT 0,
    max_renewals        INTEGER NOT NULL DEFAULT 3,
    recalled            BOOLEAN NOT NULL DEFAULT false,
    recalled_at         TIMESTAMPTZ,
    recall_due_date     TIMESTAMPTZ,
    -- SIP2 checkout fields
    sip2_terminal       TEXT,                   -- self-checkout terminal ID
    desensitised        BOOLEAN NOT NULL DEFAULT false,
    -- Staff
    checked_out_by      UUID REFERENCES users(id),
    checked_in_by       UUID REFERENCES users(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_loans_institution ON loans (institution_id);
CREATE INDEX idx_loans_patron ON loans (patron_id);
CREATE INDEX idx_loans_item ON loans (item_id);
CREATE INDEX idx_loans_bib ON loans (bib_record_id);
CREATE INDEX idx_loans_status ON loans (institution_id, status);
CREATE INDEX idx_loans_due ON loans (due_date) WHERE status IN ('active', 'renewed', 'recalled');
CREATE INDEX idx_loans_overdue ON loans (institution_id, due_date) WHERE status = 'overdue';
CREATE INDEX idx_loans_checkout ON loans (institution_id, checkout_at DESC);

CREATE TABLE holds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    patron_id           UUID NOT NULL REFERENCES patrons(id),
    bib_record_id       UUID NOT NULL REFERENCES bib_records(id),
    item_id             UUID REFERENCES items(id),    -- NULL for title-level holds
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'in_transit', 'available_for_pickup',
                            'fulfilled', 'expired', 'cancelled'
                        )),
    hold_type           TEXT NOT NULL DEFAULT 'title' CHECK (hold_type IN (
                            'title', 'copy', 'recall', 'course_reserve'
                        )),
    queue_position      INTEGER,
    pickup_location     TEXT NOT NULL,
    placed_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ,
    available_at        TIMESTAMPTZ,            -- when item arrived at pickup location
    pickup_by           TIMESTAMPTZ,            -- deadline to pick up
    fulfilled_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancel_reason       TEXT,
    notification_sent   BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holds_institution ON holds (institution_id);
CREATE INDEX idx_holds_patron ON holds (patron_id);
CREATE INDEX idx_holds_bib ON holds (bib_record_id);
CREATE INDEX idx_holds_item ON holds (item_id);
CREATE INDEX idx_holds_status ON holds (institution_id, status);
CREATE INDEX idx_holds_queue ON holds (bib_record_id, queue_position) WHERE status = 'pending';
CREATE INDEX idx_holds_pickup ON holds (institution_id, pickup_by) WHERE status = 'available_for_pickup';

CREATE TABLE fines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    patron_id           UUID NOT NULL REFERENCES patrons(id),
    loan_id             UUID REFERENCES loans(id),
    fine_type           TEXT NOT NULL CHECK (fine_type IN (
                            'overdue', 'lost_item', 'damaged_item',
                            'lost_item_processing', 'replacement',
                            'ill_fee', 'photocopy', 'manual'
                        )),
    status              TEXT NOT NULL DEFAULT 'outstanding' CHECK (status IN (
                            'outstanding', 'partially_paid', 'paid',
                            'waived', 'written_off', 'transferred'
                        )),
    amount_cents        BIGINT NOT NULL,
    paid_cents          BIGINT NOT NULL DEFAULT 0,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    description         TEXT NOT NULL,
    accrual_date        DATE NOT NULL,
    payment_method      TEXT CHECK (payment_method IN (
                            'card', 'cash', 'campus_account',
                            'payment_gateway', 'waiver'
                        )),
    paid_at             TIMESTAMPTZ,
    waived_by           UUID REFERENCES users(id),
    waive_reason        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fines_institution ON fines (institution_id);
CREATE INDEX idx_fines_patron ON fines (patron_id);
CREATE INDEX idx_fines_loan ON fines (loan_id);
CREATE INDEX idx_fines_status ON fines (institution_id, status);
CREATE INDEX idx_fines_outstanding ON fines (patron_id) WHERE status IN ('outstanding', 'partially_paid');
```

## Acquisitions

```sql
CREATE TABLE acquisition_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    bib_record_id       UUID REFERENCES bib_records(id),
    order_number        TEXT NOT NULL,
    order_type          TEXT NOT NULL CHECK (order_type IN (
                            'firm_order', 'standing_order', 'subscription',
                            'approval_plan', 'gift', 'exchange',
                            'demand_driven', 'ebook_package', 'database'
                        )),
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'sent', 'acknowledged', 'partially_received',
                            'received', 'invoiced', 'paid', 'cancelled',
                            'claimed'
                        )),
    vendor_name         TEXT NOT NULL,
    vendor_account      TEXT,                   -- EDIFACT SAN / vendor account code
    vendor_order_ref    TEXT,
    -- Financial
    fund_code           TEXT NOT NULL,          -- budget/fund allocation
    fiscal_year         INTEGER NOT NULL,
    estimated_cost_cents BIGINT,
    actual_cost_cents   BIGINT,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    discount_pct        REAL,
    tax_cents           BIGINT NOT NULL DEFAULT 0,
    -- EDIFACT
    edifact_message_ref TEXT,                   -- EDIFACT ORDERS message reference
    invoice_number      TEXT,
    invoice_date        DATE,
    invoice_amount_cents BIGINT,
    -- Receiving
    quantity_ordered    INTEGER NOT NULL DEFAULT 1,
    quantity_received   INTEGER NOT NULL DEFAULT 0,
    received_at         TIMESTAMPTZ,
    -- Requestor
    requested_by        UUID REFERENCES patrons(id),
    ordered_by          UUID REFERENCES users(id),
    selector_notes      TEXT,
    -- Claim tracking
    claim_count         INTEGER NOT NULL DEFAULT 0,
    last_claimed_at     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, order_number)
);

CREATE INDEX idx_acq_institution ON acquisition_orders (institution_id);
CREATE INDEX idx_acq_bib ON acquisition_orders (bib_record_id);
CREATE INDEX idx_acq_status ON acquisition_orders (institution_id, status);
CREATE INDEX idx_acq_vendor ON acquisition_orders (institution_id, vendor_name);
CREATE INDEX idx_acq_fund ON acquisition_orders (institution_id, fund_code, fiscal_year);
CREATE INDEX idx_acq_type ON acquisition_orders (institution_id, order_type);
```

## Electronic Resource Management

```sql
CREATE TABLE electronic_resources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    bib_record_id       UUID REFERENCES bib_records(id),
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
    -- Publisher / provider
    publisher           TEXT,
    provider            TEXT,                   -- aggregator platform (EBSCO, ProQuest, JSTOR)
    platform_url        TEXT,
    -- Licence
    licence_start       DATE,
    licence_end         DATE,
    licence_type        TEXT CHECK (licence_type IN (
                            'subscription', 'perpetual_access', 'open_access',
                            'pay_per_view', 'evidence_based', 'transformative_agreement'
                        )),
    licence_terms_json  JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "simultaneous_users": "unlimited",
    --   "ill_permitted": true,
    --   "course_reserves_permitted": true,
    --   "text_mining_permitted": false,
    --   "archival_rights": "post_cancellation_access",
    --   "authorised_users": "current_students_staff_walk_in",
    --   "gdpr_dpa_signed": true
    -- }
    -- Financial
    annual_cost_cents   BIGINT,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    fund_code           TEXT,
    fiscal_year         INTEGER,
    cost_per_use_cents  BIGINT,                 -- calculated from COUNTER data
    -- KBART
    kbart_title_list    TEXT,                   -- KBART file reference
    coverage_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"start_date": "1997-01-01", "end_date": null, "embargo_months": 0, "coverage_depth": "fulltext"}
    -- COUNTER / SUSHI
    sushi_endpoint      TEXT,                   -- SUSHI API base URL
    sushi_credentials   JSONB NOT NULL DEFAULT '{}', -- encrypted at rest
    counter_compliant   BOOLEAN NOT NULL DEFAULT false,
    last_harvest_at     TIMESTAMPTZ,
    -- Access
    access_method       TEXT CHECK (access_method IN (
                            'ip_authentication', 'shibboleth', 'openathens',
                            'ezproxy', 'username_password', 'open'
                        )),
    proxy_url           TEXT,
    -- Open access
    oa_status           TEXT CHECK (oa_status IN (
                            'gold', 'diamond', 'transformative', 'hybrid', 'closed'
                        )),
    read_and_publish    BOOLEAN NOT NULL DEFAULT false,
    apc_discount_pct    REAL,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_erm_institution ON electronic_resources (institution_id);
CREATE INDEX idx_erm_bib ON electronic_resources (bib_record_id);
CREATE INDEX idx_erm_status ON electronic_resources (institution_id, status);
CREATE INDEX idx_erm_type ON electronic_resources (institution_id, resource_type);
CREATE INDEX idx_erm_publisher ON electronic_resources (institution_id, publisher);
CREATE INDEX idx_erm_licence_end ON electronic_resources (licence_end) WHERE status = 'active';
CREATE INDEX idx_erm_fund ON electronic_resources (institution_id, fund_code, fiscal_year);
```

## Interlibrary Loans (ISO 18626)

```sql
CREATE TABLE ill_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    patron_id           UUID REFERENCES patrons(id),
    bib_record_id       UUID REFERENCES bib_records(id),
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
    -- ISO 18626 fields
    iso18626_request_id TEXT,                   -- ISO 18626 request identifier
    requesting_agency   TEXT NOT NULL,          -- ISIL or OCLC symbol
    supplying_agency    TEXT,                   -- ISIL or OCLC symbol
    service_type        TEXT NOT NULL CHECK (service_type IN (
                            'loan', 'copy', 'digital_copy'
                        )),
    -- Bibliographic
    title               TEXT NOT NULL,
    author              TEXT,
    isbn                TEXT,
    issn                TEXT,
    volume              TEXT,
    issue               TEXT,
    pages               TEXT,
    publication_year    TEXT,
    -- Fulfilment
    need_before_date    DATE,
    shipped_at          TIMESTAMPTZ,
    received_at         TIMESTAMPTZ,
    due_date            DATE,
    returned_at         TIMESTAMPTZ,
    delivery_method     TEXT CHECK (delivery_method IN (
                            'physical', 'electronic', 'ariel', 'email', 'ftp'
                        )),
    -- Copyright
    copyright_compliance TEXT CHECK (copyright_compliance IN (
                            'ccg', 'ccl', 'fair_use', 'cla_licence',
                            'copyright_fee_paid', 'open_access', 'other'
                        )),
    -- Financial
    fee_cents           BIGINT NOT NULL DEFAULT 0,
    fee_currency        TEXT NOT NULL DEFAULT 'GBP',
    ifm_transaction_id  TEXT,                   -- OCLC IFM reference
    -- Lender selection
    lender_string       TEXT[] NOT NULL DEFAULT '{}', -- ordered list of potential lenders
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
CREATE INDEX idx_ill_iso ON ill_requests (iso18626_request_id);
CREATE INDEX idx_ill_agency ON ill_requests (supplying_agency);
CREATE INDEX idx_ill_created ON ill_requests (institution_id, created_at DESC);
```

## Course Reserves

```sql
CREATE TABLE course_reserves (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    bib_record_id       UUID NOT NULL REFERENCES bib_records(id),
    item_id             UUID REFERENCES items(id),     -- NULL for e-resources
    electronic_resource_id UUID REFERENCES electronic_resources(id),
    course_code         TEXT NOT NULL,
    course_name         TEXT NOT NULL,
    instructor          TEXT NOT NULL,
    department          TEXT,
    term                TEXT NOT NULL,          -- e.g. "2026-spring", "2026-autumn"
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'pending', 'active', 'expired', 'removed'
                        )),
    reserve_type        TEXT NOT NULL CHECK (reserve_type IN (
                            'required', 'recommended', 'supplementary'
                        )),
    loan_period_override TEXT CHECK (loan_period_override IN (
                            'in_library_only', '2_hours', '4_hours',
                            'overnight', '3_days', '7_days'
                        )),
    -- Copyright
    copyright_status    TEXT CHECK (copyright_status IN (
                            'owned', 'licensed', 'fair_use', 'cla_scanned',
                            'open_access', 'link_only'
                        )),
    pages_scanned       TEXT,                   -- e.g. "pp. 45-67"
    -- LTI integration
    lms_course_id       TEXT,                   -- LTI course identifier
    lms_link_url        TEXT,
    requested_by        UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reserves_institution ON course_reserves (institution_id);
CREATE INDEX idx_reserves_bib ON course_reserves (bib_record_id);
CREATE INDEX idx_reserves_course ON course_reserves (institution_id, course_code, term);
CREATE INDEX idx_reserves_status ON course_reserves (institution_id, status);
CREATE INDEX idx_reserves_term ON course_reserves (institution_id, term);
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
                            'bib_record', 'item', 'patron',
                            'electronic_resource', 'acquisition_order',
                            'ill_request', 'course_reserve'
                        )),
    target_entity_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    detail_json         JSONB NOT NULL DEFAULT '{}',
    -- Example (marc_field): {
    --   "suggested_fields": [
    --     {"tag": "650", "subfields": [{"code": "a", "value": "Machine learning."}], "source": "lcsh_authority"},
    --     {"tag": "655", "subfields": [{"code": "a", "value": "Textbooks."}], "source": "lcgft"}
    --   ],
    --   "existing_headings_reviewed": 3
    -- }
    explanation         TEXT,
    model_id            TEXT,
    model_version       TEXT,
    reviewed_by         UUID REFERENCES users(id),
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
    user_agent          TEXT,
    protocol            TEXT,                   -- z39.50, sru, sip2, ncip, oai_pmh, rest
    gdpr_relevant       BOOLEAN NOT NULL DEFAULT false,
    ferpa_relevant      BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_institution ON audit_log (institution_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log (institution_id, action);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
CREATE INDEX idx_audit_privacy ON audit_log (institution_id) WHERE gdpr_relevant = true OR ferpa_relevant = true;
```

---

## Cross-Domain Query Examples

### Cost-per-use analysis for electronic resources

```sql
SELECT er.resource_name, er.publisher, er.annual_cost_cents,
       COALESCE(SUM(us.total_item_requests), 0) AS total_uses,
       CASE WHEN SUM(us.total_item_requests) > 0
            THEN er.annual_cost_cents / SUM(us.total_item_requests)
            ELSE NULL END AS cost_per_use_cents
FROM electronic_resources er
LEFT JOIN usage_statistics us ON us.electronic_resource_id = er.id
    AND us.report_period >= '2025-01-01' AND us.report_period < '2026-01-01'
WHERE er.institution_id = $1 AND er.status = 'active'
GROUP BY er.id
ORDER BY cost_per_use_cents DESC NULLS LAST;
-- Note: usage_statistics would be a supplementary table in production;
-- COUNTER data can alternatively be stored in electronic_resources.usage_json
```

### Open-access compliance dashboard

```sql
SELECT br.oa_compliance_status,
       UNNEST(br.oa_funder_mandates) AS funder,
       COUNT(*) AS record_count
FROM bib_records br
WHERE br.institution_id = $1
  AND br.oa_deposit_required = true
GROUP BY br.oa_compliance_status, funder
ORDER BY funder, br.oa_compliance_status;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Institution & Staff | 2 | Multi-institution/consortia support with RBAC |
| Patron Management | 1 | SAML/Shibboleth SSO, GDPR/FERPA consent tracking |
| Cataloguing | 2 | MARC21/BIBFRAME bib records + physical items/holdings |
| Circulation | 3 | Loans, holds queue, fines/payments |
| Acquisitions | 1 | EDIFACT orders with fund tracking |
| Electronic Resources | 1 | ERM with licence terms, COUNTER/SUSHI, KBART |
| Resource Sharing | 1 | ISO 18626 interlibrary loan lifecycle |
| Course Reserves | 1 | LTI-linked course reading lists |
| AI & Audit | 2 | Cataloguing/ERM AI suggestions + partitioned audit log |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Bib record → item → loan chain** — the core library data model (bibliographic record describes a work, items are physical copies, loans track circulation) is enforced by foreign keys. This makes "find all copies and their status" a two-table JOIN and "find all loans for a title" a three-table JOIN.

2. **MARC fields in structured JSONB** — `marc_fields_json` stores MARC data fields as an array of `{tag, ind1, ind2, subfields}` objects, preserving the full MARC record structure while allowing GIN-indexed containment queries. The leader and fixed-length fields are separate columns for direct access.

3. **Dual metadata representation** — `bibframe_json` and `dc_json` alongside MARC fields allow the same record to serve BIBFRAME linked-data export, OAI-PMH Dublin Core harvesting, and MARC21 exchange without lossy conversion at query time.

4. **GDPR/FERPA-aware patron model** — consent granularity (reading history, search history, analytics), configurable anonymisation timers, and explicit `anonymised_at` timestamps address the tension between library analytics and student privacy regulations.

5. **Title-level and copy-level holds** — `holds.item_id` is nullable: when NULL, the hold is against the bib record (any available copy satisfies it); when set, it targets a specific copy. This supports both public library-style "any copy" and academic "this specific edition" workflows.

6. **ISO 18626 lifecycle in ILL** — `ill_requests.status` values map directly to ISO 18626 message states, and `lender_string` with `current_lender_idx` implements the rota pattern where unfilled requests automatically try the next lender.

7. **ERM licence terms as structured JSONB** — licence terms vary enormously between publishers. Structured JSONB captures ILL permissions, concurrent user limits, text mining rights, and archival access without requiring columns for every possible licence clause.

8. **Course reserves with LTI integration** — linking course reserves to LMS course identifiers via `lms_course_id` enables automatic reading list synchronisation, while `copyright_status` tracks compliance for scanned materials.

9. **Multi-institution / consortia support** — `institution_id` on every table enables shared-catalogue consortial deployments where cataloguing can be shared (via bib record deduplication) while circulation and patron data remain institution-scoped.

10. **Open-access compliance tracking** — `oa_funder_mandates` as an array enables per-funder compliance status (a single article may be subject to Plan S, NIH, and UKRI mandates simultaneously), and `oa_compliance_status` provides a single queryable field for the compliance dashboard.
