# BHL data flow — exploratory notes

> Working notes from the initial exploration of the `bhl-us` codebase. Some of this content is source material for the Mermaid diagrams in `/diagrams`; some is reference detail (code pointers, stored-procedure counts) that doesn't belong on the diagrams themselves. Treat it as a scratchpad rather than authoritative documentation — if it disagrees with the diagrams, the diagrams win.
>
> Source: `diagrams/BHL Internal Data Flows.drawio.svg` (revised March 23, 2026)
> Codebase: `bhl-us` (C# / ASP.NET / SQL Server)

---

## Overview

The BHL system is built around two central hubs:

- **BHL Services Private API** — the write-back gateway through which almost every background job commits data to the production database
- **Message Queue (RabbitMQ)** — the async backbone decoupling search indexing and PDF generation from the processes that trigger them

The diagram covers approximately 30 components across four physical tiers: Web Server, App Server, Database Server, and File/Search Server.

---

## Infrastructure & Storage

| Component | Technology | Role |
|-----------|-----------|------|
| BHL DB | SQL Server | Central production database |
| BHLImport DB | SQL Server | Import staging database |
| IAAnalysis DB | SQL Server | Internet Archive analysis database |
| bhlindex DB | PostgreSQL | Name/taxon indexing database |
| Search Engine | Elasticsearch | Full-text and faceted search |
| Static Files | File share | OCR text, scandata, images, PDFs |
| Message Queue | RabbitMQ | Async messaging between services |
| SMTP Service | — | Outbound email (centralised: components POST to `/v1/Email` on the BHL Services Private API or SiteServices API; only the Search Indexer talks to SMTP directly) |

---

## Subsystems & Data Flows

The ASCII sketches below are source material for the Mermaid sub-diagrams and will eventually be replaced.

### Internet Archive Import

The most deeply coupled subsystem — touches five storage systems.

```
IAAnalysis DB ──────────────────────────────► IA Harvest
IA Analysis Harvest ──► IAAnalysis DB            │
                                                 ├──► BHLImport DB        (IA item metadata)
                                                 ├──► BHL DB
                                                 ├──► Static Files        (Scandata/DJVU/MARC/OCR)
                                                 ├──► Message Queue       (index + PDF messages)
                                                 └──► BHL Services API
```

Key code: `IAHarvest/`, `IAHarvestAsync/`, `IAAnalysisHarvest/`, `BHLImportServer/`
Database: `Database-BHLImport/` (~120 stored procedures)

---

### Search Indexing

```
BHL DB ──► Search Queue Load ──► Message Queue ──► Search Indexer ──► Elasticsearch
OCR Refresh ───────────────────► Message Queue
Text Import Processor ─────────► Message Queue
```

Key code: `BHLSearchIndexer/`, `SearchElastic/`

---

### OCR & Text

```
Static Files ──► SiteServices Private API     (served to web)
Text Import Processor ──► OCR Files
OCR Refresh ────────────► OCR Files
OCR Files ─┬──► OCR Export to Figshare
           └──► Name Refresh
```

Key code: `BHLSearchIndexer/DataAccess.cs` (OCR retrieval), `IAHarvest/` (DJVU extraction)

---

### PDF Generation

```
Message Queue ──────────────────► Pre-Gen PDF Generator ──► Pre-gen PDF Files
Public APIs ────────────────────► Pre-Gen PDF Generator
DJVU Files ─────────────────────► Custom PDF Generator ──► Custom PDF Files
Custom PDF Generator ───────────► BHL Services API        (data; also POSTs to /v1/Email for user-facing "PDF ready" notification)
```

---

### Name / Taxon Processing

```
BHL DB ──► Name File Generator ──► BHL Services API
OCR Files ───────────────────────► Name Refresh ──► BHL Services API
bhlindex (GN tool) ──────────────► bhlindex DB
```

Key code: `BHLSearchIndexer/` (name indexing), `IIIFUtility/NameManifest.cs` (name display — see IIIF status note below)

---

### External Harvests

All follow the same pattern: fetch from external source → write via BHL Services API.

| Harvester | Source | Destinations |
|-----------|--------|-------------|
| Flickr Tag Harvest | Flickr API | BHLImport DB, BHL Services API |
| Flickr Thumb Grab | Flickr API | BHL Services API |
| Wikidata Harvest | Wikidata | BHL DB, BHL Services API |
| Biostor Harvest | Biostor | BHLImport DB, BHL DB, BHL Services API |
| Macaw OAI Harvest | Macaw Server | Macaw Web Site |

---

### APIs & Web Tier

```
BHL DB ──► SiteServices Private API ─┬──► Public Web Site
                                     ├──► Admin Web Site
                                     └──► Public APIs ─┬──► Public Web Site
                                                       └──► Macaw Web Site
BHL DB ──► Public APIs ──► Public Web Site
BHL Services API ──► BHL DB
```

---

### Other Processors

| Processor | Inputs | Outputs |
|-----------|--------|---------|
| Export Processor | BHL Services API | Public Web Site (export files) |
| METS Upload | — | BHL Services API |
| DOI Processor | — | BHL Services API |
| AWS Sync | Static Files, Message Queue | AWS |
| IIIF | BHL DB, Static Files (scandata) | IIIF manifests (Presentation API 2.x) — **not in production use; see IIIF status note** |

Key code: `BHLUSWeb2/Controllers/IIIFController.cs`, `IIIFUtility/`

---

## IIIF status

Code exists in `BHLUSWeb2/Controllers/IIIFController.cs` and `IIIFUtility/` for generating IIIF Presentation API 2.x manifests, but **IIIF is not currently used in production**. Page images are served by the Internet Archive BookReader. Treat the IIIF components as dormant until/unless BHL decides to adopt them.

---

## Email (clarification)

Earlier versions of the diagram labelled many edges "Data + Email messages", which could suggest components email each other. They don't. The pattern is:

- Most batch processors POST to `/v1/Email` on the **BHL Services Private API** for admin notifications on job completion/error.
- The Public and Admin web sites POST to `/Email` on the **SiteServices API** for user-facing transactional mail (API keys, feedback acknowledgements, password resets).
- Only the **Search Indexer** bypasses the API and talks to SMTP directly (via MailKit) for critical error alerts.
- The PDF Generator also sends end-user "your PDF is ready" emails through the Private API.

20+ components post email; see `diagrams/` for the per-diagram markers.
