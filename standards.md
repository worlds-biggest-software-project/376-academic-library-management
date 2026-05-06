# Standards & API Reference

> Project: Academic Library Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 2709:2008 — Format for information exchange.** Foundation for MARC record exchange. https://www.iso.org/standard/41319.html
- **ISO 18626:2017 — Interlibrary Loan Transactions.** Modern XML/JSON message protocol for resource sharing between libraries; core for ILL interoperability. https://www.iso.org/standard/63056.html
- **ISO 10160 / 10161 — ILL Application Service Definition / Protocol Specification.** Legacy ILL protocol still referenced for compatibility. https://www.iso.org/standard/27635.html
- **ISO 25577:2013 — MarcXchange.** XML representation of ISO 2709 records. https://www.iso.org/standard/43005.html
- **ISO 690:2021 — Information and documentation: Bibliographic references.** Citation guidance relevant to discovery and reference services. https://www.iso.org/standard/72642.html
- **ISO 27001 / 27701** — Information security and privacy management for hosting patron data. https://www.iso.org/standard/27001
- **ISO 5127:2017 — Information and documentation: Foundation and vocabulary.** Domain terminology baseline. https://www.iso.org/standard/59743.html

### W3C & IETF Standards

- **W3C Linked Data Platform (LDP) 1.0.** Foundation for BIBFRAME-based linked-data publishing. https://www.w3.org/TR/ldp/
- **W3C SPARQL 1.1 Protocol.** Querying linked-data catalogues. https://www.w3.org/TR/sparql11-protocol/
- **W3C Activity Streams 2.0.** For event publishing across federated library platforms. https://www.w3.org/TR/activitystreams-core/
- **schema.org Library extensions.** SEO-friendly bibliographic markup for discovery. https://schema.org/Library
- **RFC 5005 — Feed Paging and Archiving (Atom).** Used by some OAI-PMH alternatives. https://www.rfc-editor.org/rfc/rfc5005
- **RFC 5023 — Atom Publishing Protocol (AtomPub).** Used in some repository deposit workflows (SWORD). https://www.rfc-editor.org/rfc/rfc5023
- **RFC 7231 / 9110 — HTTP semantics.** Foundation for REST APIs. https://www.rfc-editor.org/rfc/rfc9110
- **RFC 6749 / 8252 — OAuth 2.0 + OAuth for Native Apps.** Used by modern library APIs. https://www.rfc-editor.org/rfc/rfc6749

### Data Model & API Specifications

- **MARC21 (Library of Congress).** Dominant bibliographic record format; MARC Bibliographic, Authority, Holdings, Classification, Community formats. https://www.loc.gov/marc/
- **BIBFRAME 2.0 (Library of Congress).** Linked-data successor to MARC21 for bibliographic description. https://www.loc.gov/bibframe/
- **Dublin Core Metadata Element Set (DCMI).** Lightweight metadata schema for digital collections. https://www.dublincore.org/specifications/
- **MODS / METS (Library of Congress).** Metadata Object Description Schema and Metadata Encoding & Transmission Standard for digital objects. https://www.loc.gov/standards/mods/ ; https://www.loc.gov/standards/mets/
- **EAD3 (Society of American Archivists).** Encoded Archival Description for archival finding aids. https://www.loc.gov/ead/
- **OAI-PMH 2.0 (Open Archives Initiative).** Protocol for metadata harvesting. https://www.openarchives.org/OAI/openarchivesprotocol.html
- **ResourceSync (NISO/OAI).** Successor protocol for synchronising web resources. https://www.niso.org/publications/z39102014-resourcesync
- **Z39.50 (NISO Z39.50-2003).** Information retrieval protocol still ubiquitous in libraries. https://www.loc.gov/z3950/agency/
- **SRU/SRW (Library of Congress).** Modernised search/retrieve service over HTTP; companion to Z39.50. https://www.loc.gov/standards/sru/
- **NCIP (NISO Z39.83) — NISO Circulation Interchange Protocol.** Patron and item information exchange across systems. https://www.niso.org/standards-committees/ncip
- **SIP2 (3M Standard Interchange Protocol).** De-facto self-check and circulation device protocol. https://www.ncip.info/uploads/7/1/4/6/7146749/sip2_developers_guide.pdf
- **OpenURL (NISO Z39.88).** Context-sensitive linking from citations to full text. https://www.niso.org/standards-committees/openurl
- **COUNTER 5 Code of Practice.** Standardised usage statistics for electronic resources. https://cop5.projectcounter.org/
- **SUSHI (NISO Z39.93) — Standardised Usage Statistics Harvesting Initiative.** REST API for harvesting COUNTER reports. https://www.niso.org/standards-committees/sushi
- **EDItEUR ONIX for Books / Serials.** Publisher metadata and licensing exchange. https://www.editeur.org/8/ONIX/
- **EDIFACT (UN/EDIFACT) Library subset (BIC, EDItEUR).** Vendor order/invoice exchange. https://www.editeur.org/89/Standards/
- **KBART 2 (NISO RP-9-2014).** Knowledge bases and related tools — title list exchange. https://www.niso.org/standards-committees/kbart
- **MathML / TEI / DAISY** for specialised content (math, scholarly text, accessible reading). https://www.w3.org/TR/MathML3/ ; https://tei-c.org/

### Security & Authentication Standards

- **SAML 2.0 (OASIS).** Federated identity used by Shibboleth and OpenAthens. https://docs.oasis-open.org/security/saml/v2.0/
- **Shibboleth.** Federated SSO software widely deployed in academia. https://www.shibboleth.net/
- **OpenAthens.** Hosted access management for libraries. https://www.openathens.net/
- **OAuth 2.0 + OpenID Connect.** Modern API authentication. https://openid.net/connect/
- **eduGAIN.** Inter-federation identity service for research and education. https://edugain.org/
- **REFEDS Research and Scholarship Entity Category.** Attribute release profile for academic SPs. https://refeds.org/category/research-and-scholarship
- **OWASP ASVS 4.0 / Top 10 (2025 update).** Application security baseline for hosted library platforms. https://owasp.org/www-project-application-security-verification-standard/
- **NIST SP 800-63-3 — Digital Identity Guidelines.** Authentication assurance levels. https://pages.nist.gov/800-63-3/
- **GDPR (EU 2016/679) and UK GDPR.** Patron data privacy obligations. https://eur-lex.europa.eu/eli/reg/2016/679/oj
- **FERPA (US 20 U.S.C. § 1232g).** Student-record privacy in US institutions. https://studentprivacy.ed.gov/
- **PCI DSS 4.0.** Required if libraries process card payments for fines/fees. https://www.pcisecuritystandards.org/

### MCP Server Specifications

- **Model Context Protocol (Anthropic).** Open spec for connecting LLM clients to data and tools — directly applicable for an AI-native cataloguing/reference assistant exposing MARC, OAI-PMH, and discovery operations to an agent. https://modelcontextprotocol.io
- **MCP TypeScript / Python SDKs** for implementing servers. https://github.com/modelcontextprotocol

### Open Access & Research Data Standards

- **Plan S / cOAlition S Principles.** Open-access mandate driving compliance workflows. https://www.coalition-s.org/
- **DataCite Metadata Schema 4.5.** DOI minting and research data discovery. https://schema.datacite.org/
- **CrossRef REST API.** DOI registration and citation linking. https://api.crossref.org/
- **ORCID Public API.** Researcher identifiers for institutional repositories. https://info.orcid.org/documentation/
- **FAIR Data Principles (GO FAIR).** Findable, Accessible, Interoperable, Reusable — research data baseline. https://www.go-fair.org/fair-principles/
- **SWORD v3 Protocol.** Repository deposit interoperability. https://swordapp.org/sword-v3/

## Similar Products — Developer Documentation & APIs

### Ex Libris Alma
- **Description:** Cloud-based library services platform for academic libraries.
- **API Documentation:** https://developers.exlibrisgroup.com/alma/apis/
- **SDKs/Libraries:** Community Python (`almapipy`), Ruby (`alma`); no first-party SDKs.
- **Developer Guide:** https://developers.exlibrisgroup.com/alma/integrations/
- **Standards:** REST/JSON + XML, OAI-PMH, OpenURL, SIP2, NCIP, EDIFACT
- **Authentication:** API Key per institution; SAML/Shibboleth for end users

### Innovative Sierra
- **Description:** Long-running ILS with circulation, cataloguing, acquisitions.
- **API Documentation:** https://techdocs.iii.com/sierraapi/
- **SDKs/Libraries:** Community wrappers (Python `sierra-ilsapi`).
- **Developer Guide:** https://techdocs.iii.com/sierradna/
- **Standards:** REST/JSON, Z39.50, SIP2, NCIP, EDIFACT
- **Authentication:** OAuth 2.0 client credentials

### Koha
- **Description:** Open-source ILS used worldwide.
- **API Documentation:** https://api.koha-community.org/
- **SDKs/Libraries:** REST + legacy SOAP/ILS-DI; Perl plugin SDK.
- **Developer Guide:** https://wiki.koha-community.org/wiki/Development
- **Standards:** REST/JSON, OAI-PMH, Z39.50, SRU, SIP2, NCIP
- **Authentication:** OAuth 2.0, Basic Auth, cookie session

### FOLIO
- **Description:** Microservices-based open LSP via the Okapi gateway.
- **API Documentation:** https://dev.folio.org/reference/api/
- **SDKs/Libraries:** Stripes UI framework; module SDK templates.
- **Developer Guide:** https://dev.folio.org/guides/
- **Standards:** REST/JSON, EDIFACT, SIP2, NCIP, Z39.50, SAML
- **Authentication:** Okapi token (JWT), tenant header

### OCLC WorldShare / WorldCat
- **Description:** Cooperative cataloguing and discovery network.
- **API Documentation:** https://developer.api.oclc.org/
- **SDKs/Libraries:** OCLC OAuth client libraries (PHP, Java, Python).
- **Developer Guide:** https://www.oclc.org/developer/home.en.html
- **Standards:** REST/JSON, MARCXML, SRU, OAI-PMH, NCIP
- **Authentication:** OAuth 2.0 (WSKey)

### EBSCO Discovery Service / FOLIO
- **Description:** Discovery index plus integrated FOLIO LSP.
- **API Documentation:** https://connect.ebsco.com/s/article/EDS-API-Documentation
- **SDKs/Libraries:** Sample apps in PHP, Ruby, .NET; FOLIO Stripes for UI.
- **Developer Guide:** https://connect.ebsco.com/s/topic/0TO1H000000PsQVWA0/eds-api
- **Standards:** REST/JSON, OpenURL, LTI 1.3
- **Authentication:** Profile/User-ID + Auth Token; SAML for patrons

### Evergreen ILS
- **Description:** Open-source ILS optimised for consortia.
- **API Documentation:** https://docs.evergreen-ils.org/3.13/_application_programming_interfaces.html
- **SDKs/Libraries:** OpenSRF service framework (Perl, JavaScript clients).
- **Developer Guide:** https://docs.evergreen-ils.org/dev/
- **Standards:** OpenSRF, Z39.50, SIP2, NCIP, OAI-PMH
- **Authentication:** OpenSRF session token

### Invenio (CERN) / InvenioRDM
- **Description:** Open-source repository platform powering Zenodo.
- **API Documentation:** https://inveniordm.docs.cern.ch/reference/rest_api_index/
- **SDKs/Libraries:** Python module ecosystem; Invenio CLI.
- **Developer Guide:** https://inveniordm.docs.cern.ch/develop/
- **Standards:** REST/JSON, OAI-PMH, DataCite, ORCID, SWORD
- **Authentication:** OAuth 2.0, API tokens, SAML

### SirsiDynix BLUEcloud
- **Description:** ILS / LSP suite by SirsiDynix.
- **API Documentation:** https://developers.sirsidynix.com/ (developer programme)
- **SDKs/Libraries:** Web Services SDK for Symphony.
- **Developer Guide:** Provided to customers via SirsiDynix Support.
- **Standards:** REST/JSON + SOAP, SIP2, NCIP, Z39.50, EDIFACT
- **Authentication:** Session token; SAML for patrons

### Auto-Graphics SHAREit
- **Description:** Cloud ILL and resource-sharing platform.
- **API Documentation:** Available to customers; ISO 18626 / NCIP messaging spec.
- **SDKs/Libraries:** None public; integration via standards.
- **Developer Guide:** https://www4.auto-graphics.com/products-shareit/
- **Standards:** ISO 18626, NCIP, SIP2, Z39.50
- **Authentication:** API key + IP allowlist

### Project ReShare
- **Description:** Open-source resource sharing aligned with FOLIO.
- **API Documentation:** https://projectreshare.org/documentation/
- **SDKs/Libraries:** FOLIO Stripes module ecosystem.
- **Developer Guide:** https://github.com/openlibraryenvironment
- **Standards:** ISO 18626, NCIP, OAI-PMH
- **Authentication:** FOLIO Okapi tokens (JWT)

### Primo / Primo VE (Ex Libris)
- **Description:** Discovery layer over Alma and CDI.
- **API Documentation:** https://developers.exlibrisgroup.com/primo/apis/
- **SDKs/Libraries:** Community React / Angular extensions.
- **Developer Guide:** https://developers.exlibrisgroup.com/primo/integrations/
- **Standards:** REST/JSON, OpenURL, OAI-PMH
- **Authentication:** API Key; SAML for patrons

## Notes

- BIBFRAME adoption is accelerating but MARC21 will remain the dominant exchange format for the foreseeable future; supporting both is essential.
- Many "open" library standards (NCIP, SIP2, ISO 18626) have inconsistent vendor implementations — interop testing matters more than spec compliance alone.
- Project COUNTER 5.1 (released 2024) is now the baseline for new SUSHI integrations; older COUNTER 4 endpoints are being retired.
- An MCP server exposing local catalogue, OAI-PMH harvest, and discovery search is a natural AI-native interface and has no established competitor in 2026.
- Plan S, NIH Public Access, and UKRI mandates continue to evolve — open-access compliance is a moving target requiring frequent rule updates.
- Card payment for fines/fees usually delegates to a campus payment gateway, avoiding direct PCI scope; verify before architecture decisions.
