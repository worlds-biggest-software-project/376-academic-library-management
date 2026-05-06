# Academic Library Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source library services platform unifying cataloguing, circulation, interlibrary loans, and discovery for academic and research libraries.

Academic Library Management is a candidate project to build a modern library services platform (LSP) for universities and research institutions. It targets the gap between legacy ILS deployments designed for print-only workflows and the realities of e-resource licensing, federated discovery, open-access mandates, and data-driven collection development.

---

## Why Academic Library Management?

- **Legacy ILS limitations:** Traditional integrated library systems were built for print workflows and struggle with modern e-resource licensing and discovery needs.
- **Vendor consolidation and lock-in:** The dominant LSPs (Alma, Sierra) are now under the same parent (Clarivate), and licensing costs and ecosystem lock-in are common complaints.
- **Open-source gaps:** Koha's ERM module is less mature than Alma's; FOLIO has Kubernetes/hosting complexity and a smaller community than Koha; Evergreen lacks academic-specific features like course reserves and ERM.
- **Compliance burden:** Open-access mandates from UKRI, NIH, and Plan S impose deposit and licence requirements that current systems do not automate well.
- **Underserved small libraries:** Existing options are too heavy for small or departmental academic libraries.

---

## Key Features

### Cataloguing and Metadata

- MARC21 cataloguing with Z39.50 / SRU import and OCLC WorldCat search
- BIBFRAME linked-data export and publication
- Authority control and batch import from vendor records
- OAI-PMH provider and consumer endpoints

### Circulation and Patron Services

- Loans, holds, renewals, recalls, and fine management
- SIP2 self-checkout integration
- Course reserves and reading-room workflows
- SAML/Shibboleth SSO with EZproxy and OpenAthens compatibility

### Electronic Resource Management

- Licence tracking and trial management
- COUNTER 5 / SUSHI usage harvesting
- Cost-per-use analysis and publisher negotiation data

### Acquisitions and Resource Sharing

- EDIFACT vendor orders, invoice processing, and budget tracking
- ISO 18626 and NCIP-based interlibrary loan and consortial sharing
- Lender selection, copyright compliance, and use-fee accounting

### Discovery and Compliance

- Unified discovery layer with relevance ranking and faceted search
- OpenURL link resolver integration
- Open-access compliance dashboard for major funder mandates (Plan S, NIH, UKRI)
- Reporting dashboards for circulation, ILL, and budget

---

## AI-Native Advantage

AI capabilities target the most labour-intensive and judgement-heavy library workflows. Auto-suggested MARC fields and authority headings reduce cataloguing effort, while subject classification suggestions (LCSH, MeSH) improve consistency. Contract intelligence extracts licence terms and red flags from publisher agreements to support ERM negotiation. A conversational reference assistant answers patron queries against the local catalogue and discovery index, and predictive usage modelling supports demand-driven acquisitions.

---

## Tech Stack & Deployment

The platform is expected to support self-hosted and cloud deployment, aligned with open standards used across the academic library sector: MARC21, BIBFRAME, Dublin Core, MODS, METS, Z39.50/SRU, OAI-PMH, COUNTER 5 / SUSHI, ISO 18626, NCIP, SIP2, EDIFACT, and SAML/Shibboleth. REST APIs and federated identity are core integration points. Building on top of permissively licensed foundations such as FOLIO (Apache 2.0), Invenio (MIT), or ReShare (Apache 2.0) avoids GPL viral effects and supports commercial integration.

---

## Market Context

Library management is an actively tracked category on Gartner and G2, with Research.com publishing a ranked list of 20 platforms for 2026. Incumbent LSPs include Alma (Ex Libris / Clarivate), Sierra (Innovative Interfaces / Clarivate), OCLC WorldShare, and SirsiDynix Symphony / BLUEcloud, alongside open-source options Koha, FOLIO, Evergreen, and Invenio. Primary buyers are university and research-institution library directors and consortia procurement leads operating under sustained budget pressure. Domain availability is rated Medium and demand Low in the candidates table (complexity 6).

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
