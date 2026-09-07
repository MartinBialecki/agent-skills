# Country Mandate Reference Guide

## Table of Contents
1. [Mandate Models Explained](#mandate-models-explained)
2. [Europe](#europe)
3. [Asia-Pacific](#asia-pacific)
4. [Americas](#americas)
5. [Middle East & Africa](#middle-east--africa)
6. [Format Standards](#format-standards)
7. [Compliance Models](#compliance-models)

## Mandate Models Explained

| Model | Description | Examples |
|-------|-------------|----------|
| **Clearance** | Invoice must be approved by tax authority before delivery | Italy (SDI), Poland (KSeF), Mexico (SAT) |
| **Real-Time Reporting (RTR)** | Invoice data reported to authority in real-time after issuance | Hungary, South Korea, India (IRP) |
| **Peppol / Decentralized CTC** | Exchange via Peppol network with optional authority reporting | Belgium, Netherlands, Australia, Singapore |
| **Centralized Exchange (CE)** | Government platform manages all invoice exchange | France (Chorus Pro), Spain (AEAT) |
| **DCTCE (5-Corner)** | Decentralized CTC + Exchange - Peppol with real-time reporting | UAE, France (PDP model) |
| **Post-Audit** | No live authority connection; records kept for inspection | US (voluntary), some legacy systems |

## Europe

### Italy - FatturaPA (SDI Clearance)
- **Format:** FatturaPA XML
- **Model:** Clearance via SDI (Sistema di Interscambio)
- **Status:** Live since 2019
- **Scope:** B2B, B2G, B2C
- **Key Features:**
  - Invoices cleared by Agenzia delle Entrate before delivery
  - Codice Destinatario required for routing
  - Digital signature (CAdES) required
  - FatturaPA 1.2 XML format
  - Supports: Invoice, Credit Note, Debit Note
- **Avalara Mandate ID:** `IT-B2B-CLEARANCE`
- **Docs:** [E-invoicing in Italy](https://www.avalara.com/vatlive/en/country-guides/europe/italy/italian-e-invoicing.html)

### Belgium - B2B Peppol
- **Format:** UBL 2.1 / Peppol BIS 3.0
- **Model:** Peppol (Decentralized)
- **Status:** Mandatory from January 2026
- **Scope:** B2B
- **Key Features:**
  - Must be able to receive from Jan 2026
  - Must be able to send from Jan 2026
  - Peppol Access Point required
  - EN 16931 compliant
- **Avalara Connection:** Prebuilt Peppol network access

### France - e-facture
- **Format:** UBL, CII, Factur-X
- **Model:** PDP (Partner Dematerialisation Platform) or Chorus Pro
- **Status:** B2B mandatory September 2026 (Wave 1)
- **Scope:** B2B
- **Timeline:**
  - Sep 2026: Large & medium enterprises (issuance + reception)
  - Sep 2027: SMEs and micro-enterprises
- **Key Features:**
  - Certified PDP or public Chorus Pro portal
  - Factur-X (PDF/A-3 + embedded XML) supported
  - Status lifecycle tracking (rejection, acceptance, payment)
- **Avalara Mandate ID:** `FR-B2B-PDP`

#### France - Annuaire bulk lookup (checking which customers are registered)

Verified 2026-08-10. `facturation.chorus-pro.gouv.fr/annuaire` is the unified reform directory: public structures plus private companies' e-invoicing reception status (routing details - PA name, addresses - are visible to PAs only).

- **Mass consultation (recherche en masse) - the bulk mechanism.** Since 2026-06-17 (DGFiP): upload a CSV/TXT list of SIREN/SIRET, **max 5,000 lines per file**, on the Portail de services Chorus Pro (Espace Annuaire). Returns each identifier's presence/status and reception platform. Official how-to: AIFE doc KB0012174 (`portail.chorus-pro.gouv.fr/aife_documentation?id=kb_article_view&sysparm_article=KB0012174`). For large customer bases, split into <=5000-line chunks, submit sequentially, merge results locally.
- **No full public dump of the new unified annuaire** (private companies included) - mass consultation is the only bulk route for it.
- **Public-structures-only annuaire (B2G) IS downloadable:** "Telecharger l'annuaire" button on the portal, or CSV at `communaute.chorus-pro.gouv.fr/annuaire-cpro` (updated weekly on Mondays; ~200k structures, ~80k services, ~27 MB XLSX). EDI alternative: daily FAR37 flux subscription.
- **PISTE API** (`api.piste.gouv.fr/cpro`, Structures API): `rechercherStructure`/`consulterStructure` for single lookups, `telechargerAnnuaireDestinataire` for the full FAR37 archive. Requires PISTE account + OAuth2 + Chorus Pro technical account; rate-limited - not for massive bulk checks.
- **Match key for PEPPOL-registered customers: SIREN/SIRET** - the annuaire's electronic addresses can be Peppol identifiers.

### Germany - XRechnung
- **Format:** UBL, CII (XRechnung)
- **Model:** Peppol / Direct
- **Status:** B2G mandatory; B2B mandatory Jan 2027 (Wave 1, >EUR 800k)
- **Scope:** B2G (live), B2B (2027-2028)
- **Key Features:**
  - XRechnung standard (UBL or CII syntax)
  - Peppol or direct exchange
  - Full universal mandate by Jan 2028

### Poland - KSeF
- **Format:** FA(3) XML
- **Model:** Centralized (KSeF - Krajowy System e-Faktur)
- **Status:** Mandatory Feb 2026
- **Timeline:**
  - Feb 2026: Large taxpayers (>PLN 200M)
  - Apr 2026: All other VAT-registered taxpayers
- **Key Features:**
  - Central government platform
  - Structured XML (FA-VAT format)
  - KSeF reference number required

### Spain - VERI*FACTU
- **Format:** UBL, Facturae, EDIFACT, CII
- **Model:** AEAT platform / Private platforms
- **Status:** VERI*FACTU Jan 2027; B2B mandate in progress
- **Key Features:**
  - VERI*FACTU: Certified invoicing software with tamper-proof chain
  - B2B exchange via AEAT or interoperable private platforms
  - Status reporting mandatory (acceptance, rejection, payment within 4 days)
  - SII (real-time reporting) users excluded from VERI*FACTU

### Hungary - RTIR
- **Format:** XML
- **Model:** Real-Time Invoice Reporting (RTIR)
- **Status:** Live
- **Scope:** B2B mandatory from July 2025 (electricity/gas); broader rollout ongoing
- **Key Features:**
  - Real-time reporting to NAV within seconds of issuance
  - XML format with specific NAV schema
  - Transaction-level data submission
- **Docs:** [Hungary RTIR](https://www.avalara.com/vatlive/en/country-guides/europe/hungary/hungary-real-time-invoice-reporting.html)

### Netherlands
- **Format:** UBL / Peppol BIS 3.0 (NLCIUS profile)
- **Model:** Peppol
- **Status:** B2G mandatory; B2B voluntary
- **Key Features:**
  - NLCIUS (National Core Invoice Use Specification)
  - Peppol BIS 3.0 for government
  - No current B2B mandate but widely adopted

### Greece - myDATA
- **Format:** XML / Peppol BIS 3.0
- **Model:** myDATA providers
- **Status:** B2G mandatory (phased); B2B approved by EU, pending legislation
- **Timeline:**
  - Sep 2025: Expanded B2G
  - Oct 2026: All businesses (B2B)

### Slovakia
- **Format:** Peppol BIS 3.0 / UN/CEFACT CII
- **Model:** Peppol 5-Corner (Digital Postman)
- **Status:** Mandatory Jan 2027
- **Key Features:**
  - Approved Digital Postman providers
  - Mandatory provider use from July 2027
  - 10-year archiving requirement

## Asia-Pacific

### India - GST e-invoicing (IRN)
- **Format:** GST JSON
- **Model:** Real-time Reporting (IRP - Invoice Registration Portal)
- **Status:** Live (turnover >= INR 5 Crore)
- **Key Features:**
  - Invoice Registration Number (IRN) from IRP
  - QR code generation
  - GSTR-1 auto-population
  - JSON format via API
  - NIC or approved portals

### Malaysia - MyInvois
- **Format:** UBL 2.1 XML / JSON
- **Model:** Clearance (MyInvois Portal)
- **Status:** Live (Phase 4, <= RM 5M turnover, Jan 2026)
- **Timeline:**
  - Phase 4: Jan 2026 (<= RM 5M turnover)
  - Relaxation period until Dec 2026
  - Smallest businesses: Jul 2026
- **Key Features:**
  - 72-hour rejection/cancellation window
  - QR code after validation
  - Unique Identification Number (UIN)
  - Consolidated e-invoices for B2C within 7 days

### Singapore - InvoiceNow
- **Format:** Peppol BIS / PINT-SG
- **Model:** Peppol (InvoiceNow)
- **Status:** Phased rollout from May 2025
- **Timeline:**
  - 2025-2026: New GST-registered businesses
  - Through 2031: All GST-registered businesses
- **Key Features:**
  - GST InvoiceNow via Peppol
  - IRAS phased requirement

### Australia
- **Format:** Peppol PINT A-NZ
- **Model:** Peppol
- **Status:** Voluntary (government adoption growing)
- **Key Features:**
  - PINT A-NZ profile
  - ATO supporting adoption
  - B2B voluntary

### New Zealand
- **Format:** Peppol PINT A-NZ
- **Model:** Peppol
- **Status:** Voluntary
- **Key Features:**
  - Same PINT A-NZ profile as Australia
  - NZBN (New Zealand Business Number) as identifier

### South Korea
- **Format:** XML (Korean standard)
- **Model:** Real-time Reporting
- **Status:** Live
- **Key Features:**
  - Real-time submission to NTS
  - Korean-standard XML format
  - Domestic-only scope

## Americas

### Mexico - CFDI 4.0
- **Format:** CFDI 4.0 XML
- **Model:** Clearance (PAC -> SAT)
- **Status:** Live (all transactions)
- **Key Features:**
  - Authorized PAC (Proveedor Autorizado de Certificación)
  - SAT (Servicio de Administración Tributaria) clearance
  - Digital signature (Sello Digital)
  - UUID from SAT
  - Addendas for B2B data

### Brazil - NF-e / NFS-e
- **Format:** XML (NF-e for goods, NFS-e for services)
- **Model:** SEFAZ Clearance
- **Status:** Live (since 2005 for NF-e)
- **Key Features:**
  - State-level SEFAZ clearance
  - DANFE (human-readable PDF)
  - Ambiente: Producao vs Homologacao
  - Eventos (Cancelamento, Carta de Correcao)
  - Complex dual system (NF-e goods, NFS-e services)

### Colombia
- **Format:** UBL 2.1
- **Model:** DIAN Clearance
- **Status:** Live
- **Key Features:**
  - UBL 2.1 XML format
  - DIAN pre-clearance
  - CUFE (Codigo Unico de Factura Electronica)

### Chile
- **Format:** XML
- **Model:** Real-time Reporting (SII)
- **Status:** Live
- **Key Features:**
  - DTE (Documento Tributario Electronico)
  - Folio authorization from SII
  - XML format with Chilean schema

### United States
- **Format:** UBL / Peppol BIS
- **Model:** Voluntary (Post-audit)
- **Status:** No federal mandate
- **Key Features:**
  - DBNAlliance (Peppol-based voluntary exchange)
  - B2B adoption market-driven
  - State-level requirements may vary

## Middle East & Africa

### UAE - DCTCE (Peppol 5-Corner)
- **Format:** XML/UBL 2.1 (PINT-AE profile)
- **Model:** DCTCE 5-Corner (Decentralized CTC + Exchange)
- **Status:** Voluntary pilot Jul 2026; Mandatory Jan 2027
- **Timeline:**
  - Jul 2026: Voluntary pilot
  - Jan 2027 (Wave 1): Revenue >= AED 50M - ASP by Jul 2026
  - Jul 2027 (Wave 2): All others - ASP by Mar 2027
  - Oct 2027: B2G transactions
- **Key Features:**
  - Peppol 5-Corner model
  - Certified Access Service Provider (ASP) required
  - TIN-based participant identifier (scheme 0235)
  - Data retention: 9 years within UAE
  - PDFs not valid after mandate dates
- **Docs:** [UAE E-Invoicing](https://orchidatax.com/e-invoicing-2026-latest-global-developments-and-what-they-mean-for-your-business/)

### Saudi Arabia - ZATCA Fatoora
- **Format:** XML (ZATCA standard)
- **Model:** ZATCA Clearance (Fatoora)
- **Status:** Live (Phase 2)
- **Key Features:**
  - Phase 1 (Dec 2021): Generation
  - Phase 2: Integration and clearance
  - Cryptographic stamp (CSID)
  - QR code with TLV Base64
  - XML with UBL structure + ZATCA extensions

### Egypt
- **Format:** XML
- **Model:** ETA Platform
- **Status:** Live
- **Key Features:**
  - ETA (Egyptian Tax Authority) portal
  - XML submission
  - Pre-clearance model

## Format Standards

### EN 16931 (European Standard)
- Core data model for EU e-invoices
- Two syntaxes: UBL 2.1 and UN/CEFACT CII
- Required for most EU mandates

### UBL 2.1 (Universal Business Language)
- XML standard by OASIS
- Used by Peppol BIS 3.0, XRechnung, NLCIUS
- Avalara ELR's standard input format

### CII (Cross Industry Invoice)
- UN/CEFACT XML format
- Alternative EN 16931 syntax
- Used in France, Germany

### Factur-X / ZUGFeRD
- Hybrid format: PDF/A-3 with embedded XML
- Used in France and Germany
- Human-readable + machine-processable

### Peppol BIS 3.0
- UBL 2.1 profile for Peppol network
- EN 16931 compliant
- Used across 30+ countries

### Peppol PINT (International)
- International extension of Peppol
- PINT A-NZ (Australia/NZ), PINT AE (UAE), etc.
- Country-specific profiles

## Compliance Models Summary

| Model | Authority Role | Invoice Validity | Examples |
|-------|---------------|-----------------|----------|
| **Clearance** | Pre-approval required | Invalid without authority stamp | Italy, Mexico, Poland, Malaysia |
| **Real-Time Reporting** | Post-issuance reporting | Valid upon issuance; reported after | Hungary, India, South Korea |
| **Peppol (Decentralized)** | Optional/no role | Valid upon exchange | Belgium, Netherlands, Australia |
| **Centralized Exchange** | Platform manages exchange | Valid upon platform registration | France (Chorus Pro) |
| **DCTCE 5-Corner** | Real-time visibility via ASP | Valid upon ASP transmission | UAE |
| **Post-Audit** | No live role | Valid upon issuance; audited later | US (voluntary) |

## Resources

- [Avalara E-Invoicing Product Page](https://www.avalara.com/us/en/products/e-invoicing.html)
- [EY Global E-invoicing Tracker PDF](https://www.ey.com/content/dam/ey-unified-site/ey-com/en-gl/technical/tax-guides/documents/en-gl-einvoicing-developments-tracker.pdf)
- [Peppol e-invoicing report](https://peppol.org/wp-content/uploads/2024/06/Billentis-Peppol-May-2024.pdf)
- [E-invoicing in Italy - Avalara](https://www.avalara.com/vatlive/en/country-guides/europe/italy/italian-e-invoicing.html)
- [E-invoicing in Netherlands - Avalara](https://www.avalara.com/vatlive/en/country-guides/europe/netherlands/netherlands-e-invoicing.html)
- [E-invoicing in UK - Avalara](https://www.avalara.com/vatlive/en/country-guides/europe/uk/uk-e-invoicing.html)
- [E-invoicing in Hungary - Avalara](https://www.avalara.com/vatlive/en/country-guides/europe/hungary/hungary-real-time-invoice-reporting.html)
- [EC eInvoicing Country Sheets](https://ec.europa.eu/digital-building-blocks/sites/spaces/einvoicingCFS/pages/718735703/2024+Italy+2024+eInvoicing+Country+Sheet)
