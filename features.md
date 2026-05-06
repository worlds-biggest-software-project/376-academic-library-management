# Academic Library Management — Feature & Functionality Survey

> Candidate #376 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Ex Libris Alma | Cloud LSP (commercial) | Proprietary SaaS (Clarivate) | https://exlibrisgroup.com/products/alma-library-services-platform/ |
| Innovative Sierra | Hybrid ILS (commercial) | Proprietary, on-prem or hosted | https://iii.com/products/sierra-ils/ |
| Koha | Open-source ILS | GPL v3 | https://koha-community.org |
| FOLIO | Open-source LSP | Apache 2.0 | https://www.folio.org |
| Evergreen | Open-source ILS (consortia) | GPL v2 | https://evergreen-ils.org |
| OCLC WorldShare Management Services | Cloud LSP (cooperative) | Proprietary SaaS | https://www.oclc.org/en/worldshare-management-services.html |
| EBSCO FOLIO / EDS | Discovery + LSP | Mixed (FOLIO open, EDS proprietary) | https://www.ebsco.com/products/folio |
| ProQuest Primo / Summon | Discovery layer | Proprietary | https://exlibrisgroup.com/products/primo-discovery-service/ |
| Invenio (CERN) | Open-source repository platform | MIT | https://invenio-software.org |
| SirsiDynix Symphony / BLUEcloud | ILS / LSP | Proprietary | https://www.sirsidynix.com |
| Auto-Graphics SHAREit | ILL / resource sharing | Proprietary SaaS | https://www4.auto-graphics.com/products-shareit/ |
| ReShare | Open-source resource sharing | Apache 2.0 | https://projectreshare.org |

## Feature Analysis by Solution

### Ex Libris Alma

**Core features**
- Unified management of print, electronic, and digital resources
- MARC21 + linked data (BIBFRAME) cataloguing
- Acquisitions with EDI vendor integration
- Electronic resource management (ERM) with licence tracking
- Fulfilment (circulation, course reserves, digitisation requests)
- Analytics powered by Oracle BI (Alma Analytics)
- Network Zone for consortial shared cataloguing

**Differentiating features**
- Community Zone — global shared knowledge base of e-resources
- Integrated discovery via Primo
- Real-time COUNTER 5 / SUSHI usage harvesting
- Strong consortial / multi-institution model

**UX patterns**
- Persona-based dashboards (cataloguer, acquisitions, circulation)
- Task-list driven workflow with role permissions
- Heavy configurability often requires consultant implementation

**Integration points**
- Extensive REST APIs (Alma Developer Network)
- OAI-PMH harvesting, OpenURL link resolver (SFX)
- SAML/Shibboleth, EZproxy, OpenAthens auth
- Webhooks for fulfilment events

**Known gaps**
- Steep learning curve, perceived UI complexity
- Lock-in to Clarivate ecosystem
- High licensing cost

**Licence / IP notes**
- Closed proprietary SaaS; data portability via OAI-PMH and MARC export

### Sierra (Innovative Interfaces)

**Core features**
- Circulation, cataloguing, acquisitions, serials in one client
- Sierra Web for browser access
- Decision Center analytics
- Self-check and SIP2 integration
- Course reserves

**Differentiating features**
- Mature millennium-era data model trusted by long-term customers
- Granular fine/fee management
- INN-Reach consortial circulation

**UX patterns**
- Fat-client desktop UI plus web modules
- Per-module permission system

**Integration points**
- Sierra REST APIs (v6+), SIP2, NCIP, Z39.50
- EDIFACT for vendor orders
- Shibboleth/SAML support

**Known gaps**
- Slow modernisation; UX considered dated
- Limited native ERM compared with Alma
- Migration to FOLIO / Alma is a common churn pattern

**Licence / IP notes**
- Proprietary; III is owned by Clarivate (since 2024) — same parent as Ex Libris

### Koha

**Core features**
- Full ILS: cataloguing, circulation, acquisitions, serials, OPAC
- MARC21 and UNIMARC support
- Z39.50 / SRU search and import
- Patron self-service via OPAC
- Reports via SQL builder

**Differentiating features**
- Truly open source with active global community
- Multilingual (100+ languages)
- Plugin architecture
- Deployable on-prem or via support vendors (ByWater, PTFS Europe, Biblibre)

**UX patterns**
- Dual interface — staff client and OPAC
- Bootstrap-based modern OPAC themes available

**Integration points**
- REST and SOAP APIs
- SIP2, NCIP for self-check / ILL
- OAI-PMH for harvest
- LDAP, CAS, Shibboleth

**Known gaps**
- ERM module less mature than Alma
- Discovery layer typically requires VuFind or Bibliovation
- Analytics limited without third-party tools

**Licence / IP notes**
- GPL v3 — copyleft; commercial vendor support widely available

### FOLIO

**Core features**
- Microservices LSP — modular apps (Inventory, Users, Orders, Receiving, Courses, ERM)
- Native ERM (built on Knowledge Base+ data)
- Integrated with EBSCO Discovery Service
- Linked data ready
- Tenant model supports consortia

**Differentiating features**
- App-store extensibility (Okapi gateway)
- Community-developed apps
- Backed by Open Library Foundation, EBSCO, Index Data

**UX patterns**
- React-based Stripes UI
- Persona dashboards by app permission set

**Integration points**
- Okapi API gateway with tenant routing
- EDIFACT, SIP2, NCIP, Z39.50
- SSO via SAML

**Known gaps**
- ERM-Inventory data harmonisation still maturing
- Hosting costs / Kubernetes complexity
- Smaller community than Koha

**Licence / IP notes**
- Apache 2.0 — permissive; safe for commercial integration

### Evergreen

**Core features**
- Designed for multi-branch consortia
- Cataloguing, circulation, acquisitions, serials
- Bookbag / patron lists
- OPAC

**Differentiating features**
- Strong consortial floating collection model
- High-availability multi-site architecture
- Battle-tested at large public consortia (Georgia PINES, BC SITKA)

**UX patterns**
- Web client (XUL retired)
- Workstation registration model

**Integration points**
- Z39.50 server, SIP2, NCIP
- OpenSRF service bus
- Apache Solr discovery

**Known gaps**
- Less academic-feature focus (course reserves, ERM minimal)
- Steeper deployment learning curve

**Licence / IP notes**
- GPL v2

### OCLC WorldShare Management Services (WMS)

**Core features**
- Cloud LSP linked to WorldCat
- Acquisitions, circulation, licence manager
- Integrated with WorldCat Discovery
- ILL via WorldShare ILL / Tipasa

**Differentiating features**
- Direct WorldCat cataloguing — no MARC import workflow needed
- Cooperative knowledge base
- Built-in resource sharing across global member libraries

**UX patterns**
- Browser-based; consistent with WorldShare suite
- WorldCat record matching at point of cataloguing

**Integration points**
- WorldCat Search / Metadata APIs
- NCIP, SIP2
- SUSHI / COUNTER

**Known gaps**
- Less customisation than Alma or FOLIO
- Smaller analytics suite

**Licence / IP notes**
- Proprietary cooperative — member governance via OCLC

### EBSCO Discovery Service (EDS) / FOLIO bundle

**Core features**
- Discovery index of 130k+ sources
- Subject thesaurus, advanced search
- Mobile-optimised UI
- Integration with EBSCOhost, FOLIO, Koha

**Differentiating features**
- Largest discovery index in the market
- Concept Map visualisation
- Researcher accounts with saved searches

**UX patterns**
- Configurable widget-based UI
- Modern responsive design

**Integration points**
- EDS API (REST/JSON)
- LTI for LMS integration
- OpenURL link resolver

**Known gaps**
- Index licensing controversy with rival publishers (some content excluded)
- Customisation requires JavaScript skills

**Licence / IP notes**
- Proprietary; FOLIO components Apache 2.0

### Primo (Ex Libris)

**Core features**
- Discovery layer over Alma
- Central Discovery Index (CDI)
- Personalisation
- Browse and tag features

**Differentiating features**
- Tight Alma integration
- Esploro researcher profile bundling
- Multi-CDI publisher coverage

**UX patterns**
- Angular SPA with custom view configuration
- Personalised "Get It" link resolver

**Integration points**
- Primo VE APIs
- OpenURL, OAI-PMH, SAML

**Known gaps**
- Customisation tied to view editor
- Performance variability

**Licence / IP notes**
- Proprietary

### Invenio

**Core features**
- Repository / digital library platform
- MARC21, Dublin Core, custom JSON Schemas
- Search via Elasticsearch / OpenSearch
- File handling for large datasets

**Differentiating features**
- Backbone of CERN Document Server, Zenodo, InvenioRDM
- Supports research data management workflows
- DOI minting, ORCID integration

**UX patterns**
- Modular React/Flask UI
- Researcher-centric upload flow

**Integration points**
- REST APIs, OAI-PMH
- DataCite, ORCID, GitHub
- SAML / OAuth

**Known gaps**
- Not a full ILS — circulation, acquisitions absent
- Geared to repositories more than circulation

**Licence / IP notes**
- MIT — permissive

### SirsiDynix Symphony / BLUEcloud

**Core features**
- Symphony ILS plus BLUEcloud apps (analytics, cataloguing, mobile circ)
- Strong ILL via BLUEcloud ILL / Relais
- Self-service and EDI

**Differentiating features**
- Long history in academic and public sectors
- BLUEcloud Cataloging built on cloud microservices
- Enterprise discovery layer

**UX patterns**
- Mix of WorkFlows desktop client and BLUEcloud web modules
- Modernisation in progress

**Integration points**
- Web Services API, SIP2, NCIP, Z39.50
- Shibboleth/SAML

**Known gaps**
- Migration in progress from desktop to web
- Two parallel data models (Symphony + BLUEcloud) confuse users

**Licence / IP notes**
- Proprietary

### Auto-Graphics SHAREit

**Core features**
- Statewide / consortial ILL platform
- Union catalogue search
- Direct patron request
- Lender rota management
- Fee accounting, IFM (OCLC Interlibrary Fee Management)

**Differentiating features**
- ILS-agnostic (works alongside Koha, Alma, Sierra, etc.)
- VDX-style mediated workflow
- Multi-state consortia deployments (e.g. NCIP-driven)

**UX patterns**
- Browser-based staff and patron portals
- Configurable lender strings

**Integration points**
- NCIP, ISO 18626 (resource sharing)
- SIP2 integration with host ILS
- Z39.50 union catalogue

**Known gaps**
- Standalone deployment requires sync with host ILS
- Lower visibility outside North America

**Licence / IP notes**
- Proprietary SaaS

### ReShare

**Core features**
- Open-source resource sharing (peer-to-peer)
- ISO 18626 messaging
- Shared index for consortia
- FOLIO-aligned data model

**Differentiating features**
- Built for next-gen consortial sharing on FOLIO foundations
- Open-source alternative to Tipasa / SHAREit
- Project Reshare governance

**UX patterns**
- Stripes-based UI (FOLIO conventions)
- Patron and staff modules

**Integration points**
- ISO 18626, NCIP, OAI-PMH
- FOLIO Inventory and Users apps

**Known gaps**
- Younger ecosystem
- Limited deployments outside founding consortia

**Licence / IP notes**
- Apache 2.0

## Cross-Cutting Feature Themes

### Table-Stakes Features
- MARC21 cataloguing with Z39.50/SRU import
- Circulation (loans, holds, renewals, fines, recalls)
- Patron management with SAML/Shibboleth SSO
- Acquisitions with EDI vendor exchange
- ERM with COUNTER 5 / SUSHI usage harvesting
- Discovery layer with faceted search and OpenURL resolution
- ILL with ISO 18626 / NCIP messaging
- OAI-PMH harvest endpoint
- Bulk export/import (MARC, CSV)
- Reporting / dashboard for circulation and budget

### Differentiating Features
- Linked-data (BIBFRAME) cataloguing alongside MARC21
- Cooperative shared knowledge base for e-resources
- Microservices / app-store extensibility (FOLIO model)
- Native research data management and DOI minting
- Consortial floating collections and shared catalogue zones
- Persona-driven workflow dashboards
- Built-in licence negotiation / cost-per-use intelligence

### Underserved Areas / Opportunities
- Plain-language ERM negotiation support — extracting key terms from publisher contracts
- Open-access compliance automation across funder mandates (Plan S, NIH, UKRI)
- Researcher-facing discovery that ranks by relevance and openness, not just metadata match
- Real-time collection development advice with usage and citation signals
- Automated subject classification and authority control quality assurance
- Migration tooling between MARC, BIBFRAME, and schema.org without data loss
- Lightweight LSP for small / departmental academic libraries (current options too heavy)

### AI-Augmentation Candidates
- Auto-suggest MARC fields and authority headings during cataloguing
- Natural-language reference / chat assistant against the local catalogue
- Contract intelligence for ERM (extract licence terms, identify red flags)
- Demand-driven acquisitions with predictive usage modelling
- Duplicate / authority record detection across federated catalogues
- Subject classification suggestions (LCSH, MeSH, custom thesauri)
- Open-access compliance triage from manuscript metadata
- Conversational discovery layer with citation-style answers

## Legal & IP Summary

Open-source options (Koha GPL v3, FOLIO Apache 2.0, Evergreen GPL v2, Invenio MIT, ReShare Apache 2.0) span permissive and copyleft licences. Building atop FOLIO, Invenio, or ReShare offers the most permissive base and avoids GPL viral effects. MARC21 is a Library of Congress standard (no royalties); BIBFRAME is also open. COUNTER and SUSHI are open standards governed by Project COUNTER and NISO. ISO 18626 (resource sharing) requires ISO standard purchase but no per-use fee. No patent encumbrances were identified for the core ILS workflows; proprietary discovery indices (CDI, EDS) are licence-restricted and cannot be redistributed. Care is needed when integrating with WorldCat — OCLC member terms restrict redistribution of bibliographic records.

## Recommended Feature Scope

**Must-have (MVP)**
- MARC21 cataloguing with Z39.50/SRU import and OCLC WorldCat search
- Patron / circulation module with holds, renewals, fines, SIP2 self-check
- SAML/Shibboleth SSO with EZproxy/OpenAthens compatibility
- AI-assisted cataloguing (subject heading and MARC field suggestions)
- OAI-PMH provider and consumer
- Basic acquisitions with EDIFACT vendor orders
- Reporting dashboard (circulation, budget, ILL)

**Should-have (v1.1)**
- ERM with COUNTER 5 / SUSHI harvesting and cost-per-use analytics
- ISO 18626 + NCIP resource-sharing module
- Discovery layer with relevance ranking and faceted search
- BIBFRAME export / linked-data publication
- Open-access compliance dashboard for major funder mandates
- LLM-powered reference assistant against local catalogue and discovery index

**Nice-to-have (backlog)**
- AI contract analysis for publisher licence negotiation
- DOI minting and research data management (Invenio-style)
- Course reserves with LTI integration to LMS
- Consortial floating collections / shared cataloguing zones
- Predictive demand-driven acquisitions
- Mobile patron app with push notifications
