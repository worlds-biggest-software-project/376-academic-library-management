# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Academic Library Management · Created: 2026-05-26

## Philosophy

This model treats every state change across the library services lifecycle as an immutable event in a single append-only store. The event store is the sole source of truth — catalogue changes, item movements, patron checkouts, hold placements, fine accruals, acquisition orders, ILL requests, electronic resource licence changes, and AI suggestions are all recorded as typed events on domain streams. Materialised read models (CQRS projections) are rebuilt from these events to serve the operational queries that library staff and patrons need.

This architecture is naturally suited to academic libraries because the domain has strong temporal and audit requirements: regulators and auditors want to know who changed a catalogue record and when, GDPR requires a processing record, FERPA requires access logs for student records, and library analytics depend on historical circulation patterns. Rather than bolting audit logging onto a CRUD model, the event-sourced approach makes the event stream the primary data structure and derives CRUD-like views as projections.

The approach also enables powerful temporal queries that are impossible in CRUD models: "what was the catalogue record for this ISBN before the batch import on March 15?" or "what was this patron's loan history in the 2024-25 academic year?" are answered by replaying events to the desired point in time. For AI-native features like predictive acquisitions and cataloguing suggestions, the full event history provides rich training data without any additional ETL.

**Best for:** Libraries requiring complete audit trails, temporal queries, GDPR/FERPA compliance, and rich event data for AI model training.

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfying GDPR processing records and FERPA access logging
- Pro: Temporal queries ("what was true on date X?") are trivial — replay events to that point
- Pro: Full event history provides rich training data for AI cataloguing and demand prediction
- Pro: No separate audit logging system needed — the event store IS the audit trail
- Pro: Schema evolution is additive — new event types don't affect existing events
- Con: Read models must be projected and maintained — adds operational complexity
- Con: Event schema evolution requires careful versioning for long-lived streams (catalogue records)
- Con: Eventually consistent read models may lag during batch imports (MARC batch load)
- Con: Developers unfamiliar with CQRS/ES face a steeper learning curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MARC21 | `catalogue.*` events carry MARC field changes; `rm_catalogue` read model serves full MARC records |
| BIBFRAME 2.0 | `catalogue.bibframe_updated` events carry linked-data changes |
| Dublin Core | `catalogue.dc_metadata_set` events for OAI-PMH export preparation |
| Z39.50 / SRU | `rm_catalogue` read model serves Z39.50/SRU search responses |
| OAI-PMH 2.0 | Event timestamps on catalogue events directly support incremental OAI-PMH harvesting |
| NCIP / SIP2 | `rm_patron_account` read model serves NCIP/SIP2 patron status and item responses |
| ISO 18626 | `ill.*` events model ISO 18626 message lifecycle |
| COUNTER 5 / SUSHI | `erm.usage_harvested` events record COUNTER report data |
| EDIFACT | `acquisition.*` events record EDIFACT order/invoice processing |
| SAML / Shibboleth | `patron.authenticated` events record federated login |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion, ce_time) |
| GDPR / FERPA | Complete processing record via event stream; `patron.anonymised` event triggers projection scrub |
| Plan S / NIH / UKRI | `catalogue.oa_compliance_updated` events track funder mandate compliance changes |

---

## Event Store Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'institution', 'patron', 'catalogue', 'item',
                        'circulation', 'acquisition', 'erm',
                        'ill', 'course_reserve', 'ai', 'config'
                    )),
    stream_id       TEXT NOT NULL,              -- e.g. "catalogue:bib-uuid" or "patron:patron-uuid"
    sequence_num    BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,              -- system component origin
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_type         TEXT NOT NULL,              -- e.g. "org.library.catalogue.record_created"
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Institution isolation
    institution_id  UUID NOT NULL,
    -- Actor
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
                        'user', 'patron', 'system', 'ai',
                        'sip2_terminal', 'z39_50_client',
                        'oai_harvester', 'sushi_harvester',
                        'edifact_processor', 'iso18626_partner',
                        'marc_batch_loader', 'scheduler'
                    )),
    actor_id        TEXT NOT NULL,
    -- Tracing
    correlation_id  TEXT,
    causation_id    UUID,
    -- Privacy flags
    gdpr_relevant   BOOLEAN NOT NULL DEFAULT false,
    ferpa_relevant  BOOLEAN NOT NULL DEFAULT false,
    protocol        TEXT,                       -- z39.50, sru, sip2, ncip, oai_pmh, rest, marc_batch
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_institution ON event_store (institution_id, ce_time DESC);
CREATE INDEX idx_events_type ON event_store (event_type, ce_time DESC);
CREATE INDEX idx_events_correlation ON event_store (correlation_id);
CREATE INDEX idx_events_causation ON event_store (causation_id);
CREATE INDEX idx_events_actor ON event_store (actor_type, actor_id);
CREATE INDEX idx_events_privacy ON event_store (institution_id, stream_type) WHERE gdpr_relevant = true OR ferpa_relevant = true;

CREATE TABLE stream_snapshots (
    stream_id       TEXT NOT NULL,
    snapshot_at     BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_at)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Event Taxonomy

### Institution & Configuration Events
```
institution.created             — new library institution onboarded
institution.configured          — policies, integrations, or auth settings changed
institution.staff_added         — staff member account created
institution.staff_role_changed  — RBAC role assignment changed
institution.staff_deactivated   — staff account deactivated
config.loan_policy_updated      — circulation policy changed
config.fine_rate_updated        — fine/fee rate changed
config.integration_configured   — SIP2, Z39.50, OAI-PMH, or SUSHI endpoint configured
```

### Catalogue Events
```
catalogue.record_created            — new bib record created (original or imported)
catalogue.record_updated            — bib record metadata modified
catalogue.marc_fields_changed       — MARC data fields added/modified/removed
catalogue.bibframe_updated          — BIBFRAME linked-data representation updated
catalogue.dc_metadata_set           — Dublin Core metadata set for OAI-PMH
catalogue.authority_heading_linked  — authority record linked to bib
catalogue.subject_heading_added     — LCSH/MeSH subject heading added
catalogue.classification_assigned   — LCC/Dewey classification assigned
catalogue.record_merged             — duplicate records merged
catalogue.record_deleted            — bib record withdrawn/deleted
catalogue.oa_compliance_updated     — open-access funder compliance status changed
catalogue.batch_imported            — batch of records imported (MARC, OAI-PMH harvest)
```

### Item Events
```
item.created                — new physical item added to bib record
item.status_changed         — available → checked_out, in_transit, etc.
item.location_changed       — item moved between locations/branches
item.condition_updated      — item condition assessed
item.barcode_assigned       — barcode assigned or changed
item.withdrawn              — item removed from collection
item.marked_missing         — item not found during inventory
item.marked_lost            — item declared lost
item.found                  — previously lost/missing item recovered
```

### Circulation Events
```
circulation.checked_out         — item checked out to patron
circulation.checked_in          — item returned
circulation.renewed             — loan renewed
circulation.recalled            — item recalled from current borrower
circulation.recall_responded    — recalled item returned
circulation.hold_placed         — hold placed on title or item
circulation.hold_available      — held item available for pickup
circulation.hold_fulfilled      — patron picked up held item
circulation.hold_expired        — hold expired (not picked up)
circulation.hold_cancelled      — hold cancelled by patron or staff
circulation.fine_accrued        — overdue fine generated
circulation.fine_paid           — fine payment received
circulation.fine_waived         — fine waived by staff
circulation.lost_item_billed    — patron billed for lost item
circulation.lost_item_returned  — lost item returned (with refund event)
circulation.sip2_checkout       — checkout via SIP2 self-service terminal
circulation.sip2_checkin        — return via SIP2 terminal
```

### Acquisition Events
```
acquisition.order_created       — purchase order created
acquisition.order_sent          — EDIFACT order sent to vendor
acquisition.order_acknowledged  — vendor acknowledged order
acquisition.item_received       — item received from vendor
acquisition.invoice_received    — vendor invoice received
acquisition.invoice_paid        — invoice paid
acquisition.order_cancelled     — order cancelled
acquisition.order_claimed       — overdue order claimed from vendor
acquisition.fund_allocated      — budget allocated to fund
acquisition.fund_spent          — expenditure recorded against fund
```

### ERM Events
```
erm.resource_added              — new electronic resource registered
erm.trial_started               — publisher trial initiated
erm.licence_negotiated          — licence terms captured (manual or AI-extracted)
erm.licence_signed              — licence agreement active
erm.licence_renewed             — licence renewed for new period
erm.licence_cancelled           — licence cancelled or lapsed
erm.usage_harvested             — COUNTER report harvested via SUSHI
erm.cost_per_use_calculated     — cost-per-use metric updated
erm.access_configured           — proxy/Shibboleth/OpenAthens access set up
erm.ai_licence_analysed         — AI extracted licence terms and flagged issues
```

### ILL Events
```
ill.request_created             — ILL request submitted
ill.request_validated           — request bibliographically verified
ill.request_sent                — request sent to lender (ISO 18626)
ill.lender_responded            — lender will_supply / cannot_supply
ill.item_shipped                — lender shipped item
ill.item_received               — borrowing library received item
ill.checked_out_to_patron       — item given to requesting patron
ill.recalled                    — lending library recalled item
ill.returned_by_patron          — patron returned ILL item
ill.returned_to_lender          — item shipped back to lending library
ill.completed                   — ILL transaction completed
ill.unfilled                    — request unfilled after all lenders tried
ill.fee_charged                 — ILL fee applied
ill.lender_rotated              — request moved to next lender in rota
```

### Course Reserve Events
```
reserve.item_placed             — item placed on course reserve
reserve.item_removed            — item removed from reserve
reserve.term_activated          — reserve list activated for academic term
reserve.term_expired            — reserve list expired at term end
reserve.copyright_cleared       — copyright compliance verified
reserve.lms_linked              — reserve linked to LMS via LTI
```

### AI Events
```
ai.marc_field_suggested         — AI suggested MARC fields for bib record
ai.subject_heading_suggested    — AI suggested subject headings (LCSH/MeSH)
ai.authority_match_suggested    — AI identified potential authority match
ai.duplicate_detected           — AI detected potential duplicate records
ai.licence_terms_extracted      — AI extracted key terms from licence document
ai.licence_red_flag_raised      — AI flagged problematic licence clause
ai.oa_compliance_assessed       — AI assessed open-access compliance status
ai.acquisition_recommended      — AI recommended title for acquisition
ai.reference_answered           — AI reference assistant answered patron query
ai.usage_forecasted             — AI predicted future usage for collection development
ai.suggestion_accepted          — librarian accepted AI suggestion
ai.suggestion_rejected          — librarian rejected AI suggestion
```

---

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_catalogue (
    bib_id              UUID PRIMARY KEY,
    institution_id      UUID NOT NULL,
    control_number      TEXT,
    oclc_number         TEXT,
    isbn                TEXT[] NOT NULL DEFAULT '{}',
    issn                TEXT[] NOT NULL DEFAULT '{}',
    doi                 TEXT,
    title               TEXT NOT NULL,
    authors             TEXT[] NOT NULL DEFAULT '{}',
    publisher           TEXT,
    publication_date    TEXT,
    record_type         TEXT NOT NULL,
    material_type       TEXT NOT NULL,
    language            TEXT,
    classification      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"lcc": "QA76.73 .C55", "dewey": "005.133", "subjects": ["Computer algorithms"], "mesh": []}
    marc                JSONB NOT NULL DEFAULT '{}',
    bibframe            JSONB NOT NULL DEFAULT '{}',
    dc                  JSONB NOT NULL DEFAULT '{}',
    items               JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "item-uuid", "barcode": "30000012345", "call_number": "QA76.73 .C55",
    --    "status": "available", "location": "Main Library", "loan_type": "standard"}
    -- ]
    holds               JSONB NOT NULL DEFAULT '[]',
    reserves            JSONB NOT NULL DEFAULT '[]',
    acquisitions        JSONB NOT NULL DEFAULT '[]',
    oa_status           JSONB NOT NULL DEFAULT '{}',
    source              TEXT,
    total_items         INTEGER NOT NULL DEFAULT 0,
    available_items     INTEGER NOT NULL DEFAULT 0,
    total_holds         INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_cat_institution ON rm_catalogue (institution_id);
CREATE INDEX idx_rm_cat_control ON rm_catalogue (institution_id, control_number);
CREATE INDEX idx_rm_cat_oclc ON rm_catalogue (oclc_number);
CREATE INDEX idx_rm_cat_isbn ON rm_catalogue USING GIN (isbn);
CREATE INDEX idx_rm_cat_issn ON rm_catalogue USING GIN (issn);
CREATE INDEX idx_rm_cat_title ON rm_catalogue USING gin (to_tsvector('english', title));
CREATE INDEX idx_rm_cat_authors ON rm_catalogue USING GIN (authors);
CREATE INDEX idx_rm_cat_classification ON rm_catalogue USING GIN (classification);
CREATE INDEX idx_rm_cat_type ON rm_catalogue (institution_id, record_type);

CREATE TABLE rm_patron_account (
    patron_id           UUID PRIMARY KEY,
    institution_id      UUID NOT NULL,
    external_id         TEXT,
    barcode             TEXT,
    patron_type         TEXT NOT NULL,
    status              TEXT NOT NULL,
    identity            JSONB NOT NULL DEFAULT '{}',
    active_loans        JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"loan_id": "uuid", "item_barcode": "30000012345", "title": "Introduction to Algorithms",
    --    "checkout_at": "2026-05-01", "due_date": "2026-05-29", "renewals": 1, "status": "active"}
    -- ]
    active_holds        JSONB NOT NULL DEFAULT '[]',
    fines               JSONB NOT NULL DEFAULT '{}',
    -- Example: {"outstanding_cents": 240, "items": [...], "payments": [...]}
    limits              JSONB NOT NULL DEFAULT '{}',
    privacy             JSONB NOT NULL DEFAULT '{}',
    engagement          JSONB NOT NULL DEFAULT '{}',
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_patron_institution ON rm_patron_account (institution_id);
CREATE INDEX idx_rm_patron_barcode ON rm_patron_account (institution_id, barcode);
CREATE INDEX idx_rm_patron_external ON rm_patron_account (institution_id, external_id);
CREATE INDEX idx_rm_patron_type ON rm_patron_account (institution_id, patron_type);
CREATE INDEX idx_rm_patron_status ON rm_patron_account (institution_id, status);

CREATE TABLE rm_circulation_dashboard (
    institution_id      UUID NOT NULL,
    period              DATE NOT NULL,          -- daily dashboard
    statistics          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "checkouts": 245, "checkins": 230, "renewals": 67,
    --   "holds_placed": 34, "holds_fulfilled": 28, "holds_expired": 3,
    --   "fines_accrued_cents": 4800, "fines_collected_cents": 3200, "fines_waived_cents": 600,
    --   "overdue_items": 42, "lost_items": 2,
    --   "sip2_checkouts": 180, "sip2_checkins": 165,
    --   "by_patron_type": {"undergraduate": 150, "postgraduate": 45, "faculty": 30, "staff": 20},
    --   "by_location": {"Main Library": 180, "Science Library": 65},
    --   "busiest_hour": 14
    -- }
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (institution_id, period)
);

CREATE TABLE rm_erm_dashboard (
    resource_id         UUID PRIMARY KEY,
    institution_id      UUID NOT NULL,
    resource_name       TEXT NOT NULL,
    resource_type       TEXT NOT NULL,
    status              TEXT NOT NULL,
    publisher           TEXT,
    licence             JSONB NOT NULL DEFAULT '{}',
    financial           JSONB NOT NULL DEFAULT '{}',
    usage               JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "ytd_requests": 2100, "cost_per_use_cents": 595,
    --   "monthly": [{"period": "2026-01", "requests": 450}, ...],
    --   "trend": "increasing"
    -- }
    coverage            JSONB NOT NULL DEFAULT '{}',
    access              JSONB NOT NULL DEFAULT '{}',
    ai_flags            JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"clause": "Section 4.2", "flag": "no_text_mining", "severity": "warning"}]
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_erm_institution ON rm_erm_dashboard (institution_id);
CREATE INDEX idx_rm_erm_status ON rm_erm_dashboard (institution_id, status);
CREATE INDEX idx_rm_erm_type ON rm_erm_dashboard (institution_id, resource_type);

CREATE TABLE rm_ill_tracker (
    request_id          UUID PRIMARY KEY,
    institution_id      UUID NOT NULL,
    request_type        TEXT NOT NULL,
    status              TEXT NOT NULL,
    patron_id           UUID,
    bib_info            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"title": "...", "author": "...", "isbn": "...", "volume": "3"}
    agencies            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"requesting": "OXF", "supplying": "BL", "rota": ["BL", "NLS", "CAM"], "current_idx": 0}
    fulfilment          JSONB NOT NULL DEFAULT '{}',
    fee                 JSONB NOT NULL DEFAULT '{}',
    timeline            JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"event": "request_created", "at": "2026-05-20", "by": "patron"},
    --   {"event": "request_sent", "at": "2026-05-20", "to": "BL"},
    --   {"event": "item_shipped", "at": "2026-05-22"}
    -- ]
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_ill_institution ON rm_ill_tracker (institution_id);
CREATE INDEX idx_rm_ill_status ON rm_ill_tracker (institution_id, status);
CREATE INDEX idx_rm_ill_patron ON rm_ill_tracker (patron_id);
```

---

## Event-Driven Query Examples

### Catalogue record history (temporal query)

```sql
-- Reconstruct the catalogue record as it was on a specific date
SELECT event_type, event_data, ce_time, actor_type, actor_id
FROM event_store
WHERE stream_id = 'catalogue:bib-uuid-123'
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY sequence_num ASC;
-- Application replays these events to reconstruct the record state at March 15
```

### Patron circulation history for academic year

```sql
-- All circulation events for a patron during 2025-26 academic year
SELECT ce_time, event_type,
       event_data->>'title' AS title,
       event_data->>'item_barcode' AS barcode
FROM event_store
WHERE stream_id = 'circulation:patron-uuid-456'
  AND ce_time BETWEEN '2025-09-01' AND '2026-06-30'
  AND event_type IN ('circulation.checked_out', 'circulation.renewed', 'circulation.checked_in')
ORDER BY ce_time ASC;
```

### GDPR data access audit

```sql
-- All events involving patron data for a specific patron
SELECT ce_time, event_type, actor_type, actor_id, protocol
FROM event_store
WHERE institution_id = $1
  AND stream_id LIKE 'patron:patron-uuid%'
  AND gdpr_relevant = true
ORDER BY ce_time DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | Partitioned event store + snapshots + projection checkpoints |
| Read Models | 5 | Catalogue, patron account, circulation dashboard, ERM dashboard, ILL tracker |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Single unified event store across all library functions** — cataloguing, circulation, acquisitions, ERM, and ILL events share one partitioned table. This makes cross-domain queries (e.g. "all events related to this bib record across cataloguing, circulation, and acquisitions") a single-table query.

2. **CloudEvents envelope** — `ce_source`, `ce_type`, `ce_specversion`, and `ce_time` follow the CloudEvents 1.0 spec, enabling events to be published to external systems (consortial partners, discovery services, analytics platforms) in a standard format.

3. **11 stream types covering full library lifecycle** — institution, patron, catalogue, item, circulation, acquisition, erm, ill, course_reserve, ai, config. Each has its own event taxonomy but shares the same storage and indexing.

4. **OAI-PMH as a natural event query** — incremental OAI-PMH harvesting maps directly to "all catalogue events since timestamp X." The event store's `ce_time` index serves the `from` parameter of OAI-PMH ListRecords without any additional infrastructure.

5. **GDPR/FERPA as event properties** — every event is tagged with `gdpr_relevant` and `ferpa_relevant` flags. A `patron.anonymised` event triggers the projection to scrub PII from `rm_patron_account`. The event store retains the anonymisation event but original PII events can be cryptographically erased using per-patron encryption keys.

6. **Circulation events bridge patron and catalogue streams** — a `circulation.checked_out` event appears on both the `circulation:patron-uuid` stream and references the `item:item-uuid` stream. Both the `rm_patron_account` and `rm_catalogue` projections consume the same event, maintaining consistency without dual-writes.

7. **Five read models for five user personas** — `rm_catalogue` (cataloguers and patrons), `rm_patron_account` (circulation desk and patron self-service), `rm_circulation_dashboard` (circulation managers), `rm_erm_dashboard` (ERM managers), `rm_ill_tracker` (ILL operators). Each is optimised for its primary use case.

8. **MARC field changes as fine-grained events** — `catalogue.marc_fields_changed` events carry only the delta (which fields were added, modified, or removed), not the entire record. This enables audit of individual field changes and efficient event replay for large catalogue records.

9. **Batch imports as compound events** — `catalogue.batch_imported` events record the batch metadata (source, count, import parameters) and link via `correlation_id` to the individual `catalogue.record_created` events. This keeps individual record provenance while enabling batch-level reporting.

10. **AI training from event replay** — the full event history (cataloguing decisions, classification assignments, authority linking, circulation patterns, acquisition decisions) provides rich training data for AI models without any ETL pipeline. The AI module replays events from the store to train and validate its suggestions.

11. **Circulation dashboard as daily projection** — `rm_circulation_dashboard` produces one row per institution per day, aggregating checkout counts, fine totals, hold statistics, and patron type breakdowns. Library managers get instant statistics without counting millions of individual events.

12. **Protocol tracking on events** — `protocol` records whether an event originated from REST API, SIP2 terminal, Z39.50 client, OAI-PMH harvester, MARC batch loader, or ISO 18626 partner. This enables protocol-specific analytics and debugging.
